# Android Apk的安装与PackageManagerService备忘

这篇主要用作个人阅读PackageManagerService代码的一些备忘,PackageManagerService的细节过多且限于水平一路代码看一下还是有很多地方细节搞不清,但是对于安装apk的过程至少心里有了一个较为清晰的认识,同时还意外知道了系统data/system下文件的提供了很多信息.

## 1. Android如何安装应用

+ step 1-进入安装入口

 一般来说有两种安装形式一种通过pm命令(由Pm类处理)来安装、一种是通过intent(由PackageInstallerActivity处理)来安装应用.而这两种最后都是经过PackageManagerService来安装apk应用,入口如下

```
//Called when a downloaded package installation has been confirmed by the user,此处注释是在2.3的代码上
//6.0中已经删了,这里借用用来说明
public void installPackage(String originPath, IPackageInstallObserver2 observer,
        int installFlags, String installerPackageName, VerificationParams verificationParams, String packageAbiOverride) {
    installPackageAsUser(originPath, observer, installFlags, installerPackageName,
            verificationParams, packageAbiOverride, UserHandle.getCallingUserId());
}
```

+ step 2-发送安装命令

```
@Override
public void installPackageAsUser(String originPath, IPackageInstallObserver2 observer, int installFlags, 
		String installerPackageName, VerificationParams verificationParams, String packageAbiOverride, int userId) {
    final File originFile = new File(originPath);
    final OriginInfo origin = OriginInfo.fromUntrustedFile(originFile);
    //利用Handler发送一条消息通过处理安装apk的操作
    final Message msg = mHandler.obtainMessage(INIT_COPY);
    msg.obj = new InstallParams(origin, null, observer, installFlags, installerPackageName,
            null, verificationParams, user, packageAbiOverride, null);
    mHandler.sendMessage(msg);
}
```

此处创建了一个描述原始apk信息的一个对象OriginInfo,初始构造只包含了待安装apk的文件位置

+ step 3-连接安装服务

PackageHandler处理**INIT_COPY**消息过程如下

```
switch (msg.what) {
    case INIT_COPY: {
        HandlerParams params = (HandlerParams) msg.obj;
        int idx = mPendingInstalls.size();
        if (DEBUG_INSTALL) Slog.i(TAG, "init_copy idx=" + idx + ": " + params);
        //如果当前没有服务没有连接首先连接服务,然后将消息加入到pending队列
        if (!mBound) {
            if (!connectToService()) {
                Slog.e(TAG, "Failed to bind to media container service");
                params.serviceError();
                return;
            } else {
                mPendingInstalls.add(idx, params);
            }
        } else {
            mPendingInstalls.add(idx, params);
            //服务已连接,发送消息通知服务处理第一个安装过程
            if (idx == 0) {
                mHandler.sendEmptyMessage(MCS_BOUND);
            }
        }
        break;
    }
```

+ step 4-处理消息

PackageHandler处理**MCS_BOUND**消息过程如下

```
case MCS_BOUND: {
    if (DEBUG_INSTALL) Slog.i(TAG, "mcs_bound");
    if (msg.obj != null) {
        mContainerService = (IMediaContainerService) msg.obj;
    }
    //略...
    else if (mPendingInstalls.size() > 0) {
        HandlerParams params = mPendingInstalls.get(0);
        if (params != null) {
        	//startCopy完成安装操作
            if (params.startCopy()) {
                //略...
            }
        }
    } 
    //略...
    break;
}
```

这里startCopy方法不仅仅完成了如其名所暗示的将apk文件拷贝到系统安装目录下的操作,其实是完成了安装过程中的所有操作包括apk解析、dex优化、应用安装信息更新等等

+ step 5-apk的拷贝

由名可见此处主要处理关于apk copy的一些操作,核心方法如下
```
public void handleStartCopy() throws RemoteException {
    //略...
    final InstallArgs args = createInstallArgs(this);//一般应该是FileInstallArgs
    mArgs = args;
    //略...
    ret = args.copyApk(mContainerService, true);
    //略...
    mRet = ret;//PackageManager.INSTALL_SUCCEEDED
}
```

其中copyApk方法将apk中解析到的apk以及native so库拷贝到系统安装目录,但这里只是使用了一个临时文件名,直到安装成功之后才会将名字改为正确的系统安装路径(doRename),具体方法可以参见***类FileInstallArgs***和***类DefaultContainerService***

+ step 6-开始安装,发送安装结果

```
@Override
void handleReturnCode() {
    if (mArgs != null) {
        processPendingInstall(mArgs, mRet);
    }
}

private void processPendingInstall(final InstallArgs args, final int currentStatus) {
    mHandler.post(new Runnable() {
        public void run() {
            //略....
            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                args.doPreInstall(res.returnCode);
                synchronized (mInstallLock) {
                    installPackageLI(args, res);//安装apk
                }
                args.doPostInstall(res.returnCode, res.uid);
            }
            //略...
            if (!doRestore) {
                //回送安装结果
                Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0);
                mHandler.sendMessage(msg);
            }
        }
    });
}
```

+ step 7-解析apk包,修改临时文件夹为正式文件名,为安装做准备

开始进行安装操作,包括apk的解析、apk的dex优化、应用安装信息更新到系统

```
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    //apk的解析,提取一个apk相关的所有数据包括代码路径、native库的使用等具体可参见PackageParser.Package的注释
    PackageParser pp = new PackageParser();
    pp.setSeparateProcesses(mSeparateProcesses);
    pp.setDisplayMetrics(mMetrics);
    final PackageParser.Package pkg;
    try {
    	//使用PackageParser解析apk包
        pkg = pp.parsePackage(tmpPackageFile, parseFlags);
    } catch (PackageParserException e) {
        res.setError("Failed parse during installPackageLI", e);
        return;
    }
    //略....
    else if (!forwardLocked && !pkg.applicationInfo.isExternalAsec()) {
        // Enable SCAN_NO_DEX flag to skip dexopt at a later stage
        scanFlags |= SCAN_NO_DEX;
        try {
            derivePackageAbi(pkg, new File(pkg.codePath), args.abiOverride, true /* extract libs */);
        } catch (PackageManagerException pme) {
            Slog.e(TAG, "Error deriving application ABI", pme);
            res.setError(INSTALL_FAILED_INTERNAL_ERROR, "Error deriving application ABI");
            return;
        }
        //执行dex优化
        int result = mPackageDexOptimizer
                .performDexOpt(pkg, null /* instruction sets */, false /* forceDex */,
                        false /* defer */, false /* inclDependencies */);
        if (result == PackageDexOptimizer.DEX_OPT_FAILED) {
            res.setError(INSTALL_FAILED_DEXOPT, "Dexopt failed for " + pkg.codePath);
            return;
        }
    }
    //将之前的临时文件目录改回正式的文件目录
    if (!args.doRename(res.returnCode, pkg, oldCodePath)) {
        res.setError(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
        return;
    }
    startIntentFilterVerifications(args.user.getIdentifier(), replace, pkg);
    //这里应用升级和应用新安装调用了不同的方法,但是前者最终还是回到了后者调用的方法
    //这里只看新安装应用的方式
    if (replace) {
        replacePackageLI(pkg, parseFlags, scanFlags | SCAN_REPLACING, args.user,
                installerPackageName, volumeUuid, res);
    } else {
    	//开始安装操作
        installNewPackageLI(pkg, parseFlags, scanFlags | SCAN_DELETE_DATA_ON_FAILURES,
                args.user, installerPackageName, volumeUuid, res);
    }
    //略...
}
```

+ step 7-真正执行安装操作,创建相关文件夹、解决content-provider问题等等,具体参见scanPackageLI

```
private void installNewPackageLI(PackageParser.Package pkg, int parseFlags, int scanFlags,
        UserHandle user, String installerPackageName, String volumeUuid,
        PackageInstalledInfo res) {
    try {
    	//此处开始实质性执行安装操作,将解析apk得到的数据存到内存中、dex优化、冲突处理等等
        PackageParser.Package newPackage = scanPackageLI(pkg, parseFlags, scanFlags, System.currentTimeMillis(), user);
        //将安装结果写入到packages.xml
        updateSettingsLI(newPackage, installerPackageName, volumeUuid, null, null, res, user);
        //略...
    } catch (PackageManagerException e) {
        res.setError("Package couldn't be installed in " + pkg.codePath, e);
    }
}
```

最后scanPackageLI会将apk的信息存入到变量mPackages(存储了系统安装的所有信息信息)中,代码段如下

![](https://github.com/stdnull/StudyNotes/blob/master/resources/blog/2017/package_in_system_memory.png)

## 2. 系统开机时应用信息的初始化

系统开机时会启动PackageManagerService来读取已安装应用,并解析apk将信息存入到内存中,同时还会根据需要进行dex优化的操作

step 1 - SystemServer的启动

 首先简要说明下SystemServer是什么-SystemService是Android Framework的核心进程，它管理一切与 Android 相关的服务，包括窗口、安装的应用、当前运行的应用、通知、网络、时间、电量、同步帐号、音频、视频、蓝牙、WIFI、地理位置 等一系列的功能,它是系统开机时由zygote进程启动,入口如下

 ```
 	/**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```

开机时PackageManagerService的启动以及开机时apk的管理操作均在SystemServer中调用、处理,通过三个不同的启动服务的函数调用PackageManagerService来初始化系统应用安装消息

```
try {
    startBootstrapServices();
    startCoreServices();
    startOtherServices();
} catch (Throwable ex) {
    Slog.e("System", "******************************************");
    Slog.e("System", "************ Failure starting system services", ex);
    throw ex;
}
```

+ step 2 -SystemServer#startBootstrapServices(),创建mPackageManagerService

> Starts the small tangle of critical services that are needed to get the system off the ground.  These services have complex mutual dependencies which is why we initialize them all in one place here.  Unless your service is also entwined in these dependencies, it should be initialized in one of the other functions.

```
// Start the package manager.
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
mFirstBoot = mPackageManagerService.isFirstBoot();
mPackageManager = mSystemContext.getPackageManager();
```

+ step 3-PackageManagerService初始化

```
public PackageManagerService(Context context, Installer installer,boolean factoryTest, boolean onlyCore) {
        //略...

        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {
            mHandlerThread = new ServiceThread(TAG,Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            mHandler = new PackageHandler(mHandlerThread.getLooper());
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);
            //创建应用相关的文件夹
            File dataDir = Environment.getDataDirectory();
            mAppDataDir = new File(dataDir, "data");
            mAppInstallDir = new File(dataDir, "app");
            mAppLib32InstallDir = new File(dataDir, "app-lib");
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();
            mUserAppDataDir = new File(dataDir, "user");
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
            //略...
            //判断是否需要进行dex优化
            final String bootClassPath = System.getenv("BOOTCLASSPATH");
            final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");
            //略...
            if (mSharedLibraries.size() > 0) {
                // NOTE: For now, we're compiling these system "shared libraries"
                // (and framework jars) into all available architectures. It's possible
                // to compile them only when we come across an app that uses them (there's
                // already logic for that in scanPackageLI) but that adds some complexity.
                for (String dexCodeInstructionSet : dexCodeInstructionSets) {
                    for (SharedLibraryEntry libEntry : mSharedLibraries.values()) {
                        final String lib = libEntry.path;
                        if (lib == null) {
                            continue;
                        }

                        try {
                            int dexoptNeeded = DexFile.getDexOptNeeded(lib, null, dexCodeInstructionSet, false);
                            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                                alreadyDexOpted.add(lib);
                                mInstaller.dexopt(lib, Process.SYSTEM_UID, true, dexCodeInstructionSet, dexoptNeeded);
                            }
                        } //略...
                    }
                }
            }
            //略...
            //解析所有安装应用将信息存到内存中
            File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
            scanDirLI(vendorOverlayDir, PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

            // Find base frameworks (resource packages without code).
            scanDirLI(frameworkDir, PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR | PackageParser.PARSE_IS_PRIVILEGED, scanFlags | SCAN_NO_DEX, 0);

            // Collected privileged system packages.
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM| PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

            // Collect ordinary system packages.
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // Collect all vendor packages.
            File vendorAppDir = new File("/vendor/app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
            scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // Collect all OEM packages.
            final File oemAppDir = new File(Environment.getOemDirectory(), "app");
            scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);
            //略...

            if (!mOnlyCore) {
                EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                        SystemClock.uptimeMillis());
                scanDirLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);

                scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);
            }
    }
```

scanDirLI中又调用了scanPackageLI方法,也就是系统开机时又会重新进行一遍安装的部分操作,主要是将应用信息存入到内存中

### 3.应用安装相关文件夹及部分变量说明

+ data/data:用户应用持久化文件存储位置

+ data/app:用户app安装存储文件夹

+ data/app-lib:应用32位native库存储位置

+ data/user:第三方用户持久化文件存储位置

+ data/app-private:包含foward-lock app的私有部分,[forward-lock参考](https://stackoverflow.com/questions/3104037/any-reason-at-all-to-forward-lock-a-free-app)

+ data/app-asec:加密应用相关

+ system/app:系统应用安装位置

+ system/framework:系统依赖jar位置

+ system/lib:系统依赖的native 库位置

+ data/system:存储描述关于系统信息的一些文件,例如应用安装信息packages.xml

+ SystemServer-mPackages:存储系统解析到安装应用的信息

### 4. PathClassLoader如何获取到这些信息

从LoadedApk类下getClassLoader中ClassLoader的创建可以获得一些信息

```
public ClassLoader getClassLoader() {
        synchronized (this) {
            if (mClassLoader != null) {
                return mClassLoader;
            }

            if (mIncludeCode && !mPackageName.equals("android")) {
                
                if (!Objects.equals(mPackageName, ActivityThread.currentPackageName())) {
                    final String isa = VMRuntime.getRuntime().vmInstructionSet();
                    try {
                        ActivityThread.getPackageManager().performDexOptIfNeeded(mPackageName, isa);
                    } catch (RemoteException re) {
                        // Ignored.
                    }
                }
                //准备容器存储代码路径
                final List<String> zipPaths = new ArrayList<>();
                final List<String> apkPaths = new ArrayList<>();
                final List<String> libPaths = new ArrayList<>();
                //将获取的文件目录信息放入容器中,代码略...
                zipPaths.add(mAppDir);
                if (mSplitAppDirs != null) {
                    Collections.addAll(zipPaths, mSplitAppDirs);
                }
                libPaths.add(mLibDir);
                //略... Add path to libraries in apk for current abi
                if (mApplicationInfo.primaryCpuAbi != null) {
                    for (String apk : apkPaths) {
                      libPaths.add(apk + "!/lib/" + mApplicationInfo.primaryCpuAbi);
                    }
                }
                //创建PathClassLoader
                mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip, lib, mBaseClassLoader);

                StrictMode.setThreadPolicy(oldPolicy);
            } 
            //....
            return mClassLoader;
        }
    }
```

即系统解析apk获取LoadedApk对象之后,该对象可以利用自身中关于apk的信息来创建ClassLoader

[参考链接](https://www.kancloud.cn/digest/androidframeworks/127788)