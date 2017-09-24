# Android Activity启动过程概述
---
## 一、相关类
+ Instrumentation类
+ ActivityManagerService类
+ ActivityThread类
+ ActivityStackSupervisor类 
+ ActivityStack
+ 以及各种Record类等等

## 二、启动流程

从LauncherActivity开始即从桌面点击应用图标开始时系统是如何开启我们应用的Activity

Step 1:获取Intent信息

```
	//响应点击时间
    protected void onListItemClick(ListView l, View v, int position, long id) {
        Intent intent = intentForPosition(position);//获取对应位置的应用Intent
        startActivity(intent);
    }
    //intentForPosition最终调用的是ActivityAdapter中的intentForPosition方法来获取Intent对象
    public Intent intentForPosition(int position) {
    	//数据源为空直接返回
        if (mActivitiesList == null) {
            return null;
        }
        //其中mIntent为一个空Intent,即mIntent = new Intent();
        Intent intent = new Intent(mIntent);
        //取出App的启动Intent的信息，并设置Intent的数据packageName,className
        ListItem item = mActivitiesList.get(position);
        intent.setClassName(item.packageName, item.className);
        if (item.extras != null) {
            intent.putExtras(item.extras);
        }
        return intent;
    }
    //接着看桌面应用的Intent对象如何获取的
    public List<ListItem> makeListItems() {
        //所说加载所有和mIntent这个空Intent匹配的Activity并拍好序
        List<ResolveInfo> list = onQueryPackageManager(mIntent);
        onSortResultList(list);
        ArrayList<ListItem> result = new ArrayList<ListItem>(list.size());
        int listSize = list.size();
        for (int i = 0; i < listSize; i++) {
            ResolveInfo resolveInfo = list.get(i);
            //填充ListItem信息
            result.add(new ListItem(mPackageManager, resolveInfo, null));
        }
        return result;
    }
```

Step 0.1 查询安装应用的信息

```
	//通过PackageManager检索Activity
    protected List<ResolveInfo> onQueryPackageManager(Intent queryIntent) {
    	//注意传入的flag值为0，而文档中没有关于flag 0 的介绍，这里注释提到的是0 代码没有flag
        return mPackageManager.queryIntentActivities(queryIntent, /* no flags */ 0);
    }
```

Step 0.3 由于PackageManager是一个虚类，我们先要找到Activity中getPackageManager是如何获取PackageManager的实例的,ContextImpl类中的实现如下
```
    public PackageManager getPackageManager() {
        if (mPackageManager != null) {
            return mPackageManager;
        }

        IPackageManager pm = ActivityThread.getPackageManager();
        if (pm != null) {
        	//最终返回的是一个使用ApplicationPackageManager包装类
            return (mPackageManager = new ApplicationPackageManager(this, pm));
        }
        return null;
    }
    public static IPackageManager getPackageManager() {
        if (sPackageManager != null) {
            return sPackageManager;
        }
        //这里getService的实例是PackageManagerSerice类的实例
        IBinder b = ServiceManager.getService("package");
        sPackageManager = IPackageManager.Stub.asInterface(b);
        return sPackageManager;
    }
```

Step 0.5 继续从ApplicationPackageManager中看检索Activity的实现

```
    public List<ResolveInfo> queryIntentActivities(Intent intent, int flags) {
        return queryIntentActivitiesAsUser(intent, flags, mContext.getUserId());
    }
    public List<ResolveInfo> queryIntentActivitiesAsUser(Intent intent, int flags, int userId) {
        try {
        	//由PackageManagerService对象mPM拿到Intent的信息，其中ResolveInfo代表的是minifest中Intent标签中的内容
            ParceledListSlice<ResolveInfo> parceledList =mPM.queryIntentActivities(intent,
            				//获取intent中的data的MIME type
                            intent.resolveTypeIfNeeded(mContext.getContentResolver()), flags, userId);
            if (parceledList == null) {
                return Collections.emptyList();
            }
            return parceledList.getList();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

Step 2 Acticvity启动流程开始

```
	/**
	* Activity.java
	*从Activity的startActivityForResult开始，这里requestCode默认为-1，options默认为0
	* Intent intent：目前只设置了要启动的activity的完整限定名
	* int requestCode : 默认-1
	* Bundle options:默认null
	**/
	public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
		//只看Actiity独立的情况
        if (mParent == null) {
        	//返回启动结果requestCode,	resultCode
            Instrumentation.ActivityResult ar =mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,intent, requestCode, options);
            //如果有启动结果发回主线程，mMainThread是ActivityThread实例
            if (ar != null) {
                mMainThread.sendActivityResult(mToken, mEmbeddedID, requestCode, ar.getResultCode(),ar.getResultData());
            }
            if (requestCode >= 0) {
                //当在Activty onCreate或者onResume方法中调用startActivity时，window不会显示以避免屏幕闪烁
                mStartedActivity = true;
            }
            ....
        } 
        ......
    }
```

Step 3

```
    /**
    * Instrumentation.java
    * Context who:启动Activity的Activity的实例对象
    * IBinder contextThread:ActivityThread的成员变量，ApplicationThread实例继承自ApplicationThreadNative
    * IBinder token: BinderProxy实例，BinderProxy和Binder类在同一个文件
    * Activity target:启动Activity的Activity
    * Intent intent:启动Activity使用的Intent
    * int requestCode: 默认为-1
    * Bundle options:默认为null
    **/
    public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target, Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        //获取启动来源
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        //每一个特定的intent都有一个ActivityMonitor对象，由Intent启动之前需检查是否有匹配的ActivityMonitor
        //第一次启动Activity时ActivityMonitor该对象为空
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    //比对Intent是否匹配
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        //ActivityMonitor阻止该Activity被启动，直接返回结果
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
        	//将Intent数据移动到剪贴板
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            //getDefault获取ActivityManagerService对象的实例来start Activity
            int result = ActivityManagerNative.getDefault().startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),token, target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

Step 4

```
    /**
    * ActivityManagerNative$ActivityManagerProxy.java
    * IApplicationThread caller:调用Activity的ApplicationThread实例
    * String callingPackage:调用Activity的包名
    * Intent intent:用于启动新Activity的Intent
    * String resolvedType:intent中数据的MIME类型
    * IBinder resultTo:调用Activity的BinderProxy实例
    * String resultWho:默认null
    * int requestCode:默认-1
    * int startFlags:默认为0
    * ProfilerInfo profilerInfo:默认为null
    * Bundler bOptions:默认null
    **/
    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        //写入启动Activity相关的数据到data中,略，其中mRemote是ActivityManagerService的IBinder
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

Step 5 

```
	/**
	*ActivityManagerNative.java
	*int code:启动方法标识
	*Parcel data:传入启动activity的数据
	*Parcel reply:用于传返回值
	*int flags:默认0
	**/
	public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IActivityManager.descriptor);
            //读出启动相关的数据
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            String callingPackage = data.readString();
            Intent intent = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            IBinder resultTo = data.readStrongBinder();
            String resultWho = data.readString();
            int requestCode = data.readInt();
            int startFlags = data.readInt();
            ProfilerInfo profilerInfo = data.readInt() != 0
                    ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
            Bundle options = data.readInt() != 0
                    ? Bundle.CREATOR.createFromParcel(data) : null;
            //startActivity位于ActivityManagerService.java中
            int result = startActivity(app, callingPackage, intent, resolvedType,
                    resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
            reply.writeNoException();
            reply.writeInt(result);
            return true;
        }

        //...此方法2000多行，不是说好方法不应该超过100行的嘛

        return super.onTransact(code, data, reply, flags);
    }
```

Step 6

```
    /**
    * ActivityManagerService.java
    * IApplicationThread caller:调用Activity的ApplicationThread实例
    * String callingPackage:调用Activity的包名
    * Intent intent:用于启动新Activity的Intent
    * String resolvedType:intent中数据的MIME类型
    * IBinder resultTo:调用Activity的BinderProxy
    * String resultWho:调用Activity的标识，默认null
    * int requestCode:默认-1
    * int startFlags:默认为0
    * ProfilerInfo profilerInfo:默认为null
    * Bundler bOptions:默认null
    **/
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

    /**
    * ActivityManagerService.java
    * IApplicationThread caller:调用Activity的ApplicationThread实例
    * String callingPackage:调用Activity的包名
    * Intent intent:用于启动新Activity的Intent
    * String resolvedType:intent中数据的MIME类型
    * IBinder resultTo:调用Activity的BinderProxy
    * String resultWho:调用Activity的标识，默认null
    * int requestCode:默认-1
    * int startFlags:默认为0
    * ProfilerInfo profilerInfo:默认为null
    * Bundler bOptions:默认null
    * int userId:多用户下为linux uid/100000,否则为0
    **/
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,Intent intent, String resolvedType, 
            IBinder resultTo, String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        //检查调用Activity的Process是否为完全封闭的沙盒，是则无法调用
        enforceNotIsolatedCaller("startActivity");
        userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);
        return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, bOptions, false, userId, null, null);
    }
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        // TODO: Switch to user app stacks here.
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```

Step 7

```
    /**
    * ActivityStackSupervior.java
    * IApplicationThread caller 调用Activity的ApplicationThread实例
    * int callingUid 默认-1
    * String callingPackage 调用Activity的包名
    * Intent intent 启动新Activity的Intent
    * String resolvedType 数据的MIME类型
    * IVoiceInteractionSession voiceSession 默认null
    * IVoiceInteractor voiceInteractor 默认null
    * IBinder resultTo 调用Activity的BinderProxy
    * String resultWho 调用Activity的标识,默认null
    * int requestCode 默认-1
    * int startFlags 默认为0
    * ProfilerInfo profilerInfo 默认null
    * IActivityManager.WaitResult outResult 默认为null
    * Configuration config 默认为null
    * Bundle bOptions 默认null
    * boolean ignoreTargetSecurity 默认false
    * int userId 
    * IActivityContainer iContainer 默认为null
    * TaskRecord inTask 默认为null
    **/
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
        // Refuse possible leaked file descriptors
        if (intent != null && intent.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        boolean componentSpecified = intent.getComponent() != null;
        // 不明为什么需要这么做
        intent = new Intent(intent);

        // 搜索要启动的组件中的信息，对应清单文件中的activity标签中
        //可能出现Activity标签未配置情况，从而抛出异常，这也就为什么Activity需要
        ActivityInfo aInfo = resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);
        ActivityContainer container = (ActivityContainer)iContainer;//默认为空
        synchronized (mService) {
            if (container != null && container.mParentActivity != null &&
                    container.mParentActivity.state != RESUMED) {
                //...
            }
            final int realCallingPid = Binder.getCallingPid();
            final int realCallingUid = Binder.getCallingUid();
            int callingPid;
            if (callingUid >= 0) {
                callingPid = -1;
            } else if (caller == null) {
                callingPid = realCallingPid;
                callingUid = realCallingUid;
            } else {
                callingPid = callingUid = -1;
            }

            final ActivityStack stack;//活动栈
            if (container == null || container.mStack.isOnHomeDisplay()) {
                stack = mFocusedStack;//使用系统桌面的Activity栈，如果是当前APP内切换则是当前的栈
            } else {
                stack = container.mStack;
            }
            //检查配置是否会发生变化
            stack.mConfigWillChange = config != null && mService.mConfiguration.diff(config) != 0;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Starting activity when config will change = " + stack.mConfigWillChange);

            final long origId = Binder.clearCallingIdentity();
            //privateFlags默认为0不进入一下方法块
            if (aInfo != null &&(aInfo.applicationInfo.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            	//....
            }

            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);

            Binder.restoreCallingIdentity(origId);

            if (stack.mConfigWillChange) {
                //...更新配置
            }

            if (outResult != null) {
                //outResult为空...
            }
            return res;
        }
    }
```

Step 8

```
	/**
    * ActivityStackSupervior.java
    * IApplicationThread caller 调用Activity的ApplicationThread实例
    * Intent intent 启动新Activity的Intent
    * String resolvedType 数据的MIME类型
    * ActivityInfo aInfo 要启动的Activity Info
    * IVoiceInteractionSession voiceSession 默认null
    * IVoiceInteractor voiceInteractor 默认null
    * IBinder resultTo 调用Activity的BinderProxy
    * String resultWho 调用Activity的标识,默认null
    * int requestCode 默认-1
    * int callingPid 一般-1
    * int callingUid 一般-1
    * String callingPackage 调用Activity的包名，非目标activity的包名
    * int realCallingPid
    * int realCallingUid
    * int startFlags 默认为0
    * Bundle options 默认null
    * ProfilerInfo profilerInfo 默认null
    * IActivityManager.WaitResult outResult 默认为null
    * Configuration config 默认为null
    * boolean ignoreTargetSecurity 默认false
    * boolean componentSpecified 默认为true
    * ActivityRecord[] outActivity 默认为null
    * IActivityContainer iContainer 默认为null
    * TaskRecord inTask 默认为null
    **/
	final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
        //ProcessRecord类用于记录当前Process的运行信息
        ProcessRecord callerApp = null;
        if (caller != null) {
        	//获取启动Activity的进程Process信息
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller
                      + " (pid=" + callingPid + ") when starting: "
                      + intent.toString());
                //无法找到调用进程信息
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }

        final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid
                    + " on display " + (container == null ? (mFocusedStack == null ?
                            Display.DEFAULT_DISPLAY : mFocusedStack.mDisplayId) :
                            (container.mActivityDisplay == null ? Display.DEFAULT_DISPLAY :
                                    container.mActivityDisplay.mDisplayId)));
        }

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
        	//根据调用者resultTo获取调用着的Activity信息
            sourceRecord = isInAnyStackLocked(resultTo);
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                    "Will send result to " + resultTo + " " + sourceRecord);
            if (sourceRecord != null) {
            	//如果启动Activity需要结果且没有finish，将结果Activity置为sourceRecord
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }

        final int launchFlags = intent.getFlags();
        //如果设置了FLAG_ACTIVITY_FORWARD_RESULT，则将activitiy的启动结果转移到新的activity
        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
           //一般Flag为0不会进入该方法
        }

        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
            // 找不到相应的component
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // 找不到相应的类, 即此处为Manifest未声明Activity错误的根源
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }

        if (err == ActivityManager.START_SUCCESS && !isCurrentProfileLocked(userId) 
        		&& (aInfo.flags & FLAG_SHOW_FOR_ALL_USERS) == 0) {
            // 尝试调用一个正在后台的activity
            err = ActivityManager.START_NOT_CURRENT_USER_ACTIVITY;
        }

        if (err == ActivityManager.START_SUCCESS && sourceRecord != null
                && sourceRecord.task.voiceSession != null) {
            //语音启动Activity时进入，检查是否允许语音启动
        }

        if (err == ActivityManager.START_SUCCESS && voiceSession != null) {
            //同上语言相关
        }
        //如果不需要结果，则resultStack为空
        final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;

        if (err != ActivityManager.START_SUCCESS) {
            //如果出现错误且需要结果，则将结果返回调用的activity
            return err;
        }

        boolean abort = false;
        //权限检查
        final int startAnyPerm = mService.checkPermission(START_ANY_ACTIVITY, callingPid, callingUid);

        if (startAnyPerm != PERMISSION_GRANTED) {
            //无权限略...
        }
        //检查intent是否被允许，同上权限检查，一般abort最终为false
        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);
        //执行测试时mController所用，一般开发没有
        if (mService.mController != null) {
            //...
        }

        if (abort) {
            //一般不进...
        }
        //创建ActivityRecord类，准备进入activity栈
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, this, container, options);
        if (outActivity != null) {
            //outActivity为空
        }

        if (r.appTimeTracker == null && sourceRecord != null) {
            //追踪用户使用app时间
            r.appTimeTracker = sourceRecord.appTimeTracker;
        }

        final ActivityStack stack = mFocusedStack;//使用当前的activity活动栈
        //前台还没有活动的activity或者活动的activity的uid和调用者的uid不同
        if (voiceSession == null && (stack.mResumedActivity == null || 
        						stack.mResumedActivity.info.applicationInfo.uid != callingUid)) {
        	//检查当前是否允许切换，如果不允许则加入到pending任务中，且返回
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                    realCallingPid, realCallingUid, "Activity start")) {
                PendingActivityLaunch pal =
                        new PendingActivityLaunch(r, sourceRecord, startFlags, stack);
                mPendingActivityLaunches.add(pal);
                ActivityOptions.abort(options);
                return ActivityManager.START_SWITCHES_CANCELED;
            }
        }

        if (mService.mDidAppSwitch) {
        	//第一次禁止app切换之后，第二次时我们允许其进行切换通过设置时间为0,比如用户按下home键
        	//第一次时没有允许，第二次时允许进行切换
            mService.mAppSwitchesAllowedTime = 0;
        } else {
            mService.mDidAppSwitch = true;
        }
        //启动被加入到pending activity中的activity，如前面所说这种activity是由于禁止被切换被加入的
        doPendingActivityLaunchesLocked(false);
        //启动activity
        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);

        if (err < 0) {
            // If someone asked to have the keyguard dismissed on the next
            // activity start, but we are not actually doing an activity
            // switch...  just dismiss the keyguard now, because we
            // probably want to see whatever is behind it.
            notifyActivityDrawnForKeyguard();
        }
        return err;
    }
```

Step 8.1

```
    /**
    *boolean doResume 
    **/
	final void doPendingActivityLaunchesLocked(boolean doResume) {
		//取出第一个正在pending的activity将其启动
        while (!mPendingActivityLaunches.isEmpty()) {
            PendingActivityLaunch pal = mPendingActivityLaunches.remove(0);
            startActivityUncheckedLocked(pal.r, pal.sourceRecord, null, null, pal.startFlags,
                    doResume && mPendingActivityLaunches.isEmpty(), null, null);
        }
    }
```

Step 9

```
    /**
    * ActivityStackSupervior.java
    * ActivityRecord r 将要启动的activity
    * ActivityRecord sourceRecord 代表调用的activity
    * IVoiceInteractionSession voiceSession 默认null
    * IVoiceInteractor voiceInteractor 默认null
    * int startFlags 默认为0
    * boolean doResume true
    * Bundle options 默认null
    * TaskRecord inTask 默认为null
    **/
    final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
        final Intent intent = r.intent;
        final int callingUid = r.launchedFromUid;

        // In some flows in to this function, we retrieve the task record and hold on to it
        // without a lock before calling back in to here...  so the task at this point may
        // not actually be in recents.  Check for that, and if it isn't in recents just
        // consider it invalid.
        if (inTask != null && !inTask.inRecents) {
            Slog.w(TAG, "Starting activity in task not in recents: " + inTask);
            inTask = null;
        }
        //查看启动模式
        final boolean launchSingleTop = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP;
        final boolean launchSingleInstance = r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE;
        final boolean launchSingleTask = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK;

        int launchFlags = intent.getFlags();
        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 &&
                (launchSingleInstance || launchSingleTask)) {
            // FLAG_ACTIVITY_DOCUMENT和启动模式存在冲突,intent和manifest有冲突，则取manifest设置的值
            Slog.i(TAG, "Ignoring FLAG_ACTIVITY_NEW_DOCUMENT, launchMode is " +
                    "\"singleInstance\" or \"singleTask\"");
            launchFlags &=
                    ~(Intent.FLAG_ACTIVITY_NEW_DOCUMENT | Intent.FLAG_ACTIVITY_MULTIPLE_TASK);
        } else {
            switch (r.info.documentLaunchMode) {
                case ActivityInfo.DOCUMENT_LAUNCH_NONE:
                    break;
                case ActivityInfo.DOCUMENT_LAUNCH_INTO_EXISTING:
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
                    break;
                case ActivityInfo.DOCUMENT_LAUNCH_ALWAYS:
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
                    break;
                case ActivityInfo.DOCUMENT_LAUNCH_NEVER:
                    launchFlags &= ~Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
                    break;
            }
        }
        //默认是false
        final boolean launchTaskBehind = r.mLaunchTaskBehind && !launchSingleTask && !launchSingleInstance
                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0;

        if (r.resultTo != null && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0
                && r.resultTo.task.stack != null) {
            //activity在新的活动栈中创建，先发送结果
            Slog.w(TAG, "Activity is launching as a new task, so cancelling activity result.");
            r.resultTo.task.stack.sendActivityResultLocked(-1,
                    r.resultTo, r.resultWho, r.requestCode,
                    Activity.RESULT_CANCELED, null);
            r.resultTo = null;
        }

        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 && r.resultTo == null) {
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;//不进
        }

        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            if (launchTaskBehind || r.info.documentLaunchMode == ActivityInfo.DOCUMENT_LAUNCH_ALWAYS) {
                launchFlags |= Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
            }
        }

        // We'll invoke onUserLeaving before onPause only if the launching
        // activity did not explicitly state that this is an automated launch.
        mUserLeaving = (launchFlags & Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;
        if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                "startActivity() => mUserLeaving=" + mUserLeaving);

        //本次设置不不需要resume时，标记状态，同在获取top run activity时忽略该activity
        if (!doResume) {
            r.delayedResume = true;
        }

        ActivityRecord notTop = (launchFlags & Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP) != 0 ? r : null;

        // START_FLAG_ONLY_IF_NEEDED标签表示一种特殊的启动模式，一般不进入
        if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
            ActivityRecord checkedCaller = sourceRecord;
            if (checkedCaller == null) {
                checkedCaller = mFocusedStack.topRunningNonDelayedActivityLocked(notTop);
            }
            //不是相同的activity的则需要启动
            if (!checkedCaller.realActivity.equals(r.realActivity)) {
                startFlags &= ~ActivityManager.START_FLAG_ONLY_IF_NEEDED;
            }
        }

        boolean addingToTask = false;
        TaskRecord reuseTask = null;

        // If the caller is not coming from another activity, but has given us an
        // explicit task into which they would like us to launch the new activity,
        // then let's see about doing that.
        //如果调用者不是来自其他的activity，而是一个具体的task，一般sourceRecord不为空
        if (sourceRecord == null && inTask != null && inTask.stack != null) {
            //...
        } else {
            inTask = null;
        }

        if (inTask == null) {
            if (sourceRecord == null) {
                //如果调用者不是来自于其他activity，则需要创建新的task
                if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0 && inTask == null) {
                    Slog.w(TAG, "startActivity called from non-Activity context; forcing " +
                            "Intent.FLAG_ACTIVITY_NEW_TASK for: " + intent);
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                // LAUNCH_SINGLE_INSTANCE属性需要创建新task
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            } else if (launchSingleInstance || launchSingleTask) {
                // 同上
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            }
        }

        ActivityInfo newTaskInfo = null;
        Intent newTaskIntent = null;
        ActivityStack sourceStack;
        if (sourceRecord != null) {
            if (sourceRecord.finishing) {
                // 如果调用的activity已经finish，则与之相关的一些属性已经为空，创建新的task
                if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                    Slog.w(TAG, "startActivity called from finishing " + sourceRecord
                            + "; forcing " + "Intent.FLAG_ACTIVITY_NEW_TASK for: " + intent);
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
                    newTaskInfo = sourceRecord.info;
                    newTaskIntent = sourceRecord.task.intent;
                }
                sourceRecord = null;
                sourceStack = null;
            } else {
                sourceStack = sourceRecord.task.stack;
            }
        } else {
            sourceStack = null;
        }

        boolean movedHome = false;
        ActivityStack targetStack;

        intent.setFlags(launchFlags);
        final boolean noAnimation = (launchFlags & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0;

        if (((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0 && (launchFlags & Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                || launchSingleInstance || launchSingleTask) {
            if (inTask == null && r.resultTo == null) {
                // SINGLE_INSTANCE activity 需要保持一个实例，所以需要先进行一次搜索
                ActivityRecord intentActivity = !launchSingleInstance ?
                        findTaskLocked(r) : findActivityLocked(intent, r.info);
                if (intentActivity != null) {
                    //重用activity的一些操作
                    if (isLockTaskModeViolation(intentActivity.task,
                            (launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                            == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
                        showLockTaskToast();
                        Slog.e(TAG, "startActivityUnchecked: Attempt to violate Lock Task Mode");
                        return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
                    }
                    if (r.task == null) {
                        r.task = intentActivity.task;
                    }
                    if (intentActivity.task.intent == null) {
                        intentActivity.task.setIntent(r);
                    }
                    targetStack = intentActivity.task.stack;
                    targetStack.mLastPausedActivity = null;
                    //准备将目标任务调到前台
                    final ActivityStack focusStack = getFocusedStack();
                    ActivityRecord curTop = (focusStack == null)
                            ? null : focusStack.topRunningNonDelayedActivityLocked(notTop);
                    boolean movedToFront = false;
                    if (curTop != null && (curTop.task != intentActivity.task ||
                            curTop.task != focusStack.topTask())) {
                        r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                        if (sourceRecord == null || (sourceStack.topActivity() != null &&
                                sourceStack.topActivity().task == sourceRecord.task)) {
                            // We really do want to push this one into the user's face, right now.
                            if (launchTaskBehind && sourceRecord != null) {
                                intentActivity.setTaskToAffiliateWith(sourceRecord.task);
                            }
                            movedHome = true;
                            //移到前台
                            targetStack.moveTaskToFrontLocked(intentActivity.task, noAnimation,
                                    options, r.appTimeTracker, "bringingFoundTaskToFront");
                            movedToFront = true;
                            if ((launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                                    == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                                intentActivity.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                            }
                            options = null;
                        }
                    }
                    if (!movedToFront) {
                        if (DEBUG_TASKS) Slog.d(TAG_TASKS, "Bring to front target: " + targetStack
                                + " from " + intentActivity);
                        targetStack.moveToFront("intentActivityFound");
                    }

                    // 调用者希望重置task
                    if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                        intentActivity = targetStack.resetTaskIfNeededLocked(intentActivity, r);
                    }
                    if ((startFlags & ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                        // 如果不需要创建一个activity，则直接resume当前的top activity
                        if (doResume) {
                            resumeTopActivitiesLocked(targetStack, null, options);

                            // 通知keyguard不会运行transition
                            if (!movedToFront) {
                                notifyActivityDrawnForKeyguard();
                            }
                        } else {
                            ActivityOptions.abort(options);
                        }
                        return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                    }
                    if ((launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                            == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK)) {
                        // 替换当前task
                        reuseTask = intentActivity.task;
                        reuseTask.performClearTaskLocked();
                        reuseTask.setIntent(r);
                    } else if ((launchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                            || launchSingleInstance || launchSingleTask) {
                        //移除当前要start的activity上面的task，大部分情况是恢复到初始状态
                        ActivityRecord top = intentActivity.task.performClearTaskLocked(r, launchFlags);
                        if (top != null) {
                            if (top.frontOfTask) {
                                top.task.setIntent(r);
                            }
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT,
                                    r, top.task);
                            //将新intent传递给已存在的activity，触发onNewIntent
                            top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                        } else {
                            //特殊情况:启动一个activity而且调用希望清空当前的活动栈
                            addingToTask = true;
                            sourceRecord = intentActivity;
                            TaskRecord task = sourceRecord.task;
                            if (task != null && task.stack == null) {
                                targetStack = computeStackFocus(sourceRecord, false /* newTask */);
                                targetStack.addTask(task, !launchTaskBehind /* toTop */, false /* moving */);
                            }

                        }
                    } else if (r.realActivity.equals(intentActivity.task.realActivity)) {
                        //top activity和要启动的activity相同
                        if (((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0 || launchSingleTop)
                        	&& intentActivity.realActivity.equals(r.realActivity)) {
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r,
                                    intentActivity.task);
                            if (intentActivity.frontOfTask) {
                                intentActivity.task.setIntent(r);
                            }
                            intentActivity.deliverNewIntentLocked(callingUid, r.intent,
                                    r.launchedFromPackage);
                        } else if (!r.intent.filterEquals(intentActivity.task.intent)) {
                            // 启动一个不同intent的activity
                            addingToTask = true;
                            sourceRecord = intentActivity;
                        }
                    } else if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
                        // 一般情况，创建一个activity并放到已有的活动栈中
                        addingToTask = true;
                        sourceRecord = intentActivity;
                    } else if (!intentActivity.task.rootWasReset) {
                        intentActivity.task.setIntent(r);
                    }
                    if (!addingToTask && reuseTask == null) {
                        if (doResume) {
                            targetStack.resumeTopActivityLocked(null, options);
                            if (!movedToFront) {
                                notifyActivityDrawnForKeyguard();
                            }
                        } else {
                            ActivityOptions.abort(options);
                        }
                        return ActivityManager.START_TASK_TO_FRONT;
                    }
                }
            }
        }

        if (r.packageName != null) {
            // 启动的activity和top activity相同时，检查activity是否是加载了一次
            ActivityStack topStack = mFocusedStack;
            ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(notTop);
            if (top != null && r.resultTo == null) {
                if (top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
                    if (top.app != null && top.app.thread != null) {
                        if ((launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0 || launchSingleTop || launchSingleTask) {
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, top,
                                    top.task);
                            //确保当前activity resume
                            topStack.mLastPausedActivity = null;
                            if (doResume) {
                                resumeTopActivitiesLocked();
                            }
                            ActivityOptions.abort(options);
                            if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                                return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                            }
                            top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                            return ActivityManager.START_DELIVERED_TO_TOP;
                        }
                    }
                }
            }

        } else {
            if (r.resultTo != null && r.resultTo.task.stack != null) {
                r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho,
                        r.requestCode, Activity.RESULT_CANCELED, null);
            }
            ActivityOptions.abort(options);
            return ActivityManager.START_CLASS_NOT_FOUND;
        }

        boolean newTask = false;
        boolean keepCurTransition = false;

        TaskRecord taskToAffiliate = launchTaskBehind && sourceRecord != null ?
                sourceRecord.task : null;

        // Should this be considered a new task?
        if (r.resultTo == null && inTask == null && !addingToTask && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("startingNewTask");

            if (reuseTask == null) {
                r.setTask(targetStack.createTaskRecord(getNextTaskId(),
                        newTaskInfo != null ? newTaskInfo : r.info,
                        newTaskIntent != null ? newTaskIntent : intent,
                        voiceSession, voiceInteractor, !launchTaskBehind /* toTop */),
                        taskToAffiliate);
                if (DEBUG_TASKS) Slog.v(TAG_TASKS,
                        "Starting new activity " + r + " in new task " + r.task);
            } else {
                r.setTask(reuseTask, taskToAffiliate);
            }
            if (isLockTaskModeViolation(r.task)) {
                Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            if (!movedHome) {
                if ((launchFlags &
                        (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                    // Caller wants to appear on home activity, so before starting
                    // their own activity we will bring home to the front.
                    r.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                }
            }
        } else if (sourceRecord != null) {
            final TaskRecord sourceTask = sourceRecord.task;
            if (isLockTaskModeViolation(sourceTask)) {
                Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            targetStack = sourceTask.stack;
            targetStack.moveToFront("sourceStackToFront");
            final TaskRecord topTask = targetStack.topTask();
            if (topTask != sourceTask) {
                targetStack.moveTaskToFrontLocked(sourceTask, noAnimation, options,
                        r.appTimeTracker, "sourceTaskToFront");
            }
            if (!addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
                ActivityRecord top = sourceTask.performClearTaskLocked(r, launchFlags);
                keepCurTransition = true;
                if (top != null) {
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                    top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    targetStack.mLastPausedActivity = null;
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null);
                    }
                    ActivityOptions.abort(options);
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            } else if (!addingToTask &&
                    (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
                //如果要启动的activity存在与其他的活动栈，准备将其移到前台
                final ActivityRecord top = sourceTask.findActivityInHistoryLocked(r);
                if (top != null) {
                    final TaskRecord task = top.task;
                    task.moveActivityToFrontLocked(top);
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, task);
                    top.updateOptionsLocked(options);
                    top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    targetStack.mLastPausedActivity = null;
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null);
                    }
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            }
            r.setTask(sourceTask, null);
            if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + r
                    + " in existing task " + r.task + " from source " + sourceRecord);

        } else if (inTask != null) {
            if (isLockTaskModeViolation(inTask)) {
                Slog.e(TAG, "Attempted Lock Task Mode violation r=" + r);
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            targetStack = inTask.stack;
            targetStack.moveTaskToFrontLocked(inTask, noAnimation, options, r.appTimeTracker,
                    "inTaskToFront");

            // 检查我们是否是要启动新的activity或者重用当前的activity
            ActivityRecord top = inTask.getTopActivity();
            if (top != null && top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
                if ((launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0 || launchSingleTop || launchSingleTask) {
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, top, top.task);
                    if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                        return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                    }
                    top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            }

            if (!addingToTask) {
                // 不希望添加activity到task中，仅仅通知我们消费了intent
                ActivityOptions.abort(options);
                return ActivityManager.START_TASK_TO_FRONT;
            }

            r.setTask(inTask, null);
            if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + r
                    + " in explicit task " + r.task);

        } else {
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("addingToTopTask");
            ActivityRecord prev = targetStack.topActivity();
            r.setTask(prev != null ? prev.task : targetStack.createTaskRecord(getNextTaskId(),
                            r.info, intent, null, null, true), null);
            mWindowManager.moveTaskToTop(r.task.taskId);
            if (DEBUG_TASKS) Slog.v(TAG_TASKS, "Starting new activity " + r
                    + " in new guessed " + r.task);
        }

        mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
                intent, r.getUriPermissionsLocked(), r.userId);

        if (sourceRecord != null && sourceRecord.isRecentsActivity()) {
            r.task.setTaskToReturnTo(RECENTS_ACTIVITY_TYPE);
        }
        if (newTask) {
            EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, r.userId, r.task.taskId);
        }
        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
        targetStack.mLastPausedActivity = null;
        //准备创建activity
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        if (!launchTaskBehind) {
            // Don't set focus on an activity that's going to the back.
            mService.setFocusedActivityLocked(r, "startedActivity");
        }
        return ActivityManager.START_SUCCESS;
    }
```

Step 10

```
    /**
    *ActivityStack.java
    *ActivityRecord r 代表将要启动的Activity
    *boolean newTask 新Activity是否位于新栈中
    *boolean doResume 默认true
    *boolean keepCurTransition
    *Bundle options 默认null
    **/
	final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
        TaskRecord rTask = r.task;
        final int taskId = rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
            //该task的中的前一个activity被移除或者ams复用了该task
            insertTaskAtTop(rTask, r);
            mWindowManager.moveTaskToTop(taskId);
        }
        TaskRecord task = null;
        if (!newTask) {
            // 启动一个已存在的activity
            boolean startIt = true;
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    continue;
                }
                if (task == r.task) {
                    //活动栈找到需要启动的activity，将其移到top
                    if (!startIt) {
                        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to task "
                                + task, new RuntimeException("here").fillInStackTrace());
                        task.addActivityToTop(r);
                        r.putInHistory();
                        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                                (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0,
                                r.userId, r.info.configChanges, task.voiceSession != null,
                                r.mLaunchTaskBehind);
                        if (VALIDATE_TOKENS) {
                            validateAppTokensLocked();
                        }
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    startIt = false;
                }
            }
        }
        if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
            mStackSupervisor.mUserLeaving = false;
            if (DEBUG_USER_LEAVING) Slog.v(TAG_USER_LEAVING,
                    "startActivity() behind front, mUserLeaving=false");
        }

        task = r.task;

        // Slot the activity into the history stack and proceed
        if (DEBUG_ADD_REMOVE) Slog.i(TAG, "Adding activity " + r + " to stack to task " + task,
                new RuntimeException("here").fillInStackTrace());
        task.addActivityToTop(r);
        task.setFrontOfTask();

        r.putInHistory();
        if (!isHomeStack() || numActivities() > 0) {
            //切换活动栈或者activity时准备显示preview window
            boolean showStartingIcon = newTask;
            ProcessRecord proc = r.app;
            if (proc == null) {
                proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
            }
            if (proc == null || proc.thread == null) {
                showStartingIcon = true;
            }
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                    "Prepare open transition: starting " + r);
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else {
                mWindowManager.prepareAppTransition(newTask
                        ? r.mLaunchTaskBehind
                                ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                                : AppTransition.TRANSIT_TASK_OPEN
                        : AppTransition.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            mWindowManager.addAppToken(task.mActivities.indexOf(r),
                    r.appToken, r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                    r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
            boolean doShow = true;
            if (newTask) {
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && new ActivityOptions(options).getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {
                mWindowManager.setAppVisibility(r.appToken, true);
                ensureActivitiesVisibleLocked(null, 0);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                // 准备判断是否需要keep preview window
                ActivityRecord prev = mResumedActivity;
                if (prev != null) {
                    // activity处于不同的栈
                    if (prev.task != r.task) {
                        prev = null;
                    }
                    // activity已经显示
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                mWindowManager.setAppStartingWindow(r.appToken, r.packageName, r.theme,
                        mService.compatibilityInfoForPackageLocked( r.info.applicationInfo), r.nonLocalizedLabel,
                        r.labelRes, r.icon, r.logo, r.windowFlags,
                        prev != null ? prev.appToken : null, showStartingIcon);
                r.mStartingWindowShown = true;
            }
        } else {
            mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                    r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                    r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
            ActivityOptions.abort(options);
            options = null;
        }
        if (VALIDATE_TOKENS) {
            validateAppTokensLocked();
        }

        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```

Step 11

```
	/**
	* ActivityStackSupervior.java
	* ActivityStack targetStack 要启动的Activity所在的活动栈
	* ActivityRecord target 目标Activity
	* Bundle targetOptions 默认null
	**/
	boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target, Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        //判断当前的活动栈是否在前台
        if (isFrontStack(targetStack)) {
        	//是，则resume其top activity
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    continue;
                }
                if (isFrontStack(stack)) {
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
```

Step 12

```
    /**
    *ActivityStack.java
    *ActivityRecord prev 当前栈中正在resume的Activity
    *Bundle options 默认null
    **/
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            return false;
        }
        boolean result = false;
        try {
            //防止递归
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```

Step 13

```
	/**
	*ActivityStack.java
	**/
	private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        if (DEBUG_LOCKSCREEN) mService.logLockScreen("");

        if (!mService.mBooting && !mService.mBooted) {
        	//还未完成开机
            return false;
        }

        ActivityRecord parent = mActivityContainer.mParentActivity;
        if ((parent != null && parent.state != ActivityState.RESUMED) || !mActivityContainer.isAttachedLocked()) {
            return false;
        }
        //如果top activity之下有activity正在初始化而且存在window display，则移除这可能导致该activity不会resume
        cancelInitializingActivities();

        // 从top开始找第一个没有finish的activity
        final ActivityRecord next = topRunningActivityLocked(null);

        final boolean userLeaving = mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving = false;

        final TaskRecord prevTask = prev != null ? prev.task : null;
        if (next == null) {
            //当前栈没有未finish的activity，切换task显示，may be home
        }

        next.delayedResume = false;

        //top activity就是要resume的，什么都不用做
        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed " + next);
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        final TaskRecord nextTask = next.task;
        if (prevTask != null && prevTask.stack == this && prevTask.isOverHomeStack() && prev.finishing && prev.frontOfTask) {
            if (DEBUG_STACK)  mStackSupervisor.validateTopActivitiesLocked();
            if (prevTask == nextTask) {
                prevTask.setFrontOfTask();
            } else if (prevTask != topTask()) {
                // This task is going away but it was supposed to return to the home stack.
                // Now the task above it has to return to the home task instead.
                final int taskNdx = mTaskHistory.indexOf(prevTask) + 1;
                mTaskHistory.get(taskNdx).setTaskToReturnTo(HOME_ACTIVITY_TYPE);
            } else if (!isOnHomeDisplay()) {
                return false;
            } else if (!isHomeStack()){
                if (DEBUG_STATES) Slog.d(TAG_STATES,
                        "resumeTopActivityLocked: Launching home next");
                final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                        HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
                return isOnHomeDisplay() &&
                        mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, "prevFinished");
            }
        }

        // If we are sleeping, and there is no resumed activity, and the top
        // activity is paused, well that is the state we want.
        //如果当前处于睡眠中，没有resumed activity 且top activity 处于pasused,确保pending transitions被执行
        if (mService.isSleepingOrShuttingDown() && mLastPausedActivity == next && mStackSupervisor.allPausedActivitiesComplete()) {
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Going to sleep and all paused");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        // 检查用于该活动的用户是否已启动
        if (mService.mStartedUsers.get(next.userId) == null) {
            Slog.w(TAG, "Skipping resume of top activity " + next
                    + ": user " + next.userId + " is stopped");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        // 从不正确的等待中移除
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mStackSupervisor.mWaitingVisibleActivities.remove(next);

        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resuming " + next);

        // 如果正在pause top activity则不做任何事
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    "resumeTopActivityLocked: Skip resume: some activity pausing.");
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return false;
        }

        if (false) {
            //不明觉厉,闹着玩？...
        }

        mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

        // resmume top activity之前需要poause 当前的 activity
        boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
        //暂停其他后台的活动栈
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
        //当前的resume activity不为空
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
        if (pausing) {
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG_STATES,
                    "resumeTopActivityLocked: Skip resume: need to start pausing");
            // pause成功，刷新LRU缓存
            if (next.app != null && next.app.thread != null) {
                mService.updateLruProcessLocked(next.app, true, null);
            }
            if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
            return true;
        }

        // 确保由于设备睡眠引起的stop完成
        if (mService.isSleeping() && mLastNoHistoryActivity != null &&
                !mLastNoHistoryActivity.finishing) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "no-history finish of " + mLastNoHistoryActivity + " on new resume");
            requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                    null, "resume-no-history", false);
            mLastNoHistoryActivity = null;
        }

        if (prev != null && prev != next) {
            if (!mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                    && next != null && !next.nowVisible) {
                mStackSupervisor.mWaitingVisibleActivities.add(prev);
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Resuming top, waiting visible to hide: " + prev);
            } else {
                // 前一个activity finished，隐藏其window
                if (prev.finishing) {
                    mWindowManager.setAppVisibility(prev.appToken, false);
                    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                            "Not waiting for visible to hide: " + prev + ", waitingVisible="
                            + mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                            + ", nowVisible=" + next.nowVisible);
                } else {
                    if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                            "Previous already visible but still waiting to hide: " + prev
                            + ", waitingVisible="
                            + mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                            + ", nowVisible=" + next.nowVisible);
                }
            }
        }

        // 准备启动活动
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    next.packageName, false, next.userId); /* TODO: Verify if correct userid */
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + next.packageName + ": " + e);
        }

        // 开始启动activity，隐藏preview window
        boolean anim = true;
        if (prev != null) {
            //...
        } else {
            //...        
        }
        //...动画相关

        ActivityStack lastStack = mStackSupervisor.getLastStack();
        //activity所在进程已存在的情况
        if (next.app != null && next.app.thread != null) {
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next);

            // activity正在成为可见
            mWindowManager.setAppVisibility(next.appToken, true);

            // 启动ticks以手机app性能信息
            next.startLaunchTickingLocked();

            ActivityRecord lastResumedActivity =
                    lastStack == null ? null :lastStack.mResumedActivity;
            ActivityState lastState = next.state;

            mService.updateCpuStats();

            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next + " (in existing)");
            next.state = ActivityState.RESUMED;
            mResumedActivity = next;
            next.task.touchActiveTime();
            mRecentTasks.addLocked(next.task);
            mService.updateLruProcessLocked(next.app, true, null);
            updateLRUListLocked(next);
            mService.updateOomAdjLocked();

            // Have the window manager re-evaluate the orientation of
            // the screen based on the new activity order.
            boolean notUpdated = true;
            if (mStackSupervisor.isFrontStack(this)) {
                Configuration config = mWindowManager.updateOrientationFromAppTokens(
                        mService.mConfiguration,
                        next.mayFreezeScreenLocked(next.app) ? next.appToken : null);
                if (config != null) {
                    next.frozenBeforeDestroy = true;
                }
                notUpdated = !mService.updateConfigurationLocked(config, next, false, false);
            }

            if (notUpdated) {
                // 配置变化需要创建一个新的activity,但是需要确保当前仍在top并重新计划一次run
                ActivityRecord nextNext = topRunningActivityLocked(null);
                if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                        "Activity config changed during resume: " + next
                        + ", new next: " + nextNext);
                if (nextNext != next) {
                    mStackSupervisor.scheduleResumeTopActivities();//递归再来一次
                }
                if (mStackSupervisor.reportResumedActivityLocked(next)) {
                    mNoAnimActivities.clear();
                    if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                    return true;
                }
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return false;
            }

            try {
                // 发送所有的pending result
                ArrayList<ResultInfo> a = next.results;
                if (a != null) {
                    final int N = a.size();
                    if (!next.finishing && N > 0) {
                        if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                                "Delivering results to " + next + ": " + a);
                        next.app.thread.scheduleSendResult(next.appToken, a);
                    }
                }

                if (next.newIntents != null) {
                    next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
                }

                EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                        System.identityHashCode(next), next.task.taskId, next.shortComponentName);

                next.sleeping = false;
                mService.showAskCompatModeDialogLocked(next);
                next.app.pendingUiClean = true;
                next.app.forceProcessStateUpTo(mService.mTopProcessState);
                next.clearOptionsLocked();
                //这里最终调用onResume方法
                next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                        mService.isNextTransitionForward(), resumeAnimOptions);

                mStackSupervisor.checkReadyForSleepLocked();

                if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Resumed " + next);
            } catch (Exception e) {
                // Whoops, need to restart this activity!
                if (DEBUG_STATES) Slog.v(TAG_STATES, "Resume failed; resetting state to "
                        + lastState + ": " + next);
                next.state = lastState;
                if (lastStack != null) {
                    lastStack.mResumedActivity = lastResumedActivity;
                }
                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else  if (SHOW_APP_STARTING_PREVIEW && lastStack != null &&
                        mStackSupervisor.isFrontStack(lastStack)) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(next.info.applicationInfo),
                            next.nonLocalizedLabel, next.labelRes, next.icon, next.logo,
                            next.windowFlags, null, true);
                }
                mStackSupervisor.startSpecificActivityLocked(next, true, false);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.visible = true;
                completeResumeLocked(next);
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
                return true;
            }
            next.stopped = false;

        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.logo, next.windowFlags,
                            null, true);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }

        if (DEBUG_STACK) mStackSupervisor.validateTopActivitiesLocked();
        return true;
    }
```

Step 14

```
	/**
	*ActivityStackSupersivor.java
	*ActivityRecord 待启动的Activity
	*boolean andResume 默认true
	*boolean checkConfig 默认true 
	**/
	void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);
        //Application已经启动的情况
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                //启动activity(命名感人!!!)
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
        }
        //未启动或者process挂掉，尝试重启进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

Step 15

```
	/**
	*ActivityStackSupersivor.java
	*ActivityRecord 待启动的Activity
	*ProcessRecord app当前进程
	*boolean andResume 默认true
	*boolean checkConfig 默认true 
	**/
	final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

        if (andResume) {
            r.startFreezingScreenLocked(app, 0);
            mWindowManager.setAppVisibility(r.appToken, true);

            // 启动tick收集性能信息
            r.startLaunchTickingLocked();
        }

        //更新configuration
        if (checkConfig) {
            Configuration config = mWindowManager.updateOrientationFromAppTokens(
                    mService.mConfiguration,
                    r.mayFreezeScreenLocked(app) ? r.appToken : null);
            mService.updateConfigurationLocked(config, r, false, false);
        }

        r.app = app;
        app.waitingToKill = null;
        r.launchCount++;
        r.lastLaunchTime = SystemClock.uptimeMillis();

        if (DEBUG_ALL) Slog.v(TAG, "Launching: " + r);

        int idx = app.activities.indexOf(r);
        if (idx < 0) {
            app.activities.add(r);
        }
        mService.updateLruProcessLocked(app, true, null);
        mService.updateOomAdjLocked();

        final TaskRecord task = r.task;
        if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE ||
                task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV) {
            setLockTaskModeLocked(task, LOCK_TASK_MODE_LOCKED, "mLockTaskAuth==LAUNCHABLE", false);
        }

        final ActivityStack stack = task.stack;
        try {
            if (app.thread == null) {
                throw new RemoteException();
            }
            List<ResultInfo> results = null;
            List<ReferrerIntent> newIntents = null;
            if (andResume) {
                results = r.results;
                newIntents = r.newIntents;
            }
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                    "Launching: " + r + " icicle=" + r.icicle + " with results=" + results
                    + " newIntents=" + newIntents + " andResume=" + andResume);
            if (andResume) {
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY,
                        r.userId, System.identityHashCode(r),
                        task.taskId, r.shortComponentName);
            }
            if (r.isHomeActivity() && r.isNotResolverActivity()) {
                // Home process is the root process of the task.
                mService.mHomeProcess = task.mActivities.get(0).app;
            }
            mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());
            r.sleeping = false;
            r.forceNewConfig = false;
            mService.showAskCompatModeDialogLocked(r);
            r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
            ProfilerInfo profilerInfo = null;
            if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
                if (mService.mProfileProc == null || mService.mProfileProc == app) {
                    mService.mProfileProc = app;
                    final String profileFile = mService.mProfileFile;
                    if (profileFile != null) {
                        ParcelFileDescriptor profileFd = mService.mProfileFd;
                        if (profileFd != null) {
                            try {
                                profileFd = profileFd.dup();
                            } catch (IOException e) {
                                if (profileFd != null) {
                                    try {
                                        profileFd.close();
                                    } catch (IOException o) {
                                    }
                                    profileFd = null;
                                }
                            }
                        }

                        profilerInfo = new ProfilerInfo(profileFile, profileFd,
                                mService.mSamplingInterval, mService.mAutoStopProfiler);
                    }
                }
            }

            if (andResume) {
                app.hasShownUi = true;
                app.pendingUiClean = true;
            }
            app.forceProcessStateUpTo(mService.mTopProcessState);
            //这里启动activity
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

            if ((app.info.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
                if (app.processName.equals(app.info.packageName)) {
                    if (mService.mHeavyWeightProcess != null
                            && mService.mHeavyWeightProcess != app) {
                        Slog.w(TAG, "Starting new heavy weight process " + app
                                + " when already running "
                                + mService.mHeavyWeightProcess);
                    }
                    mService.mHeavyWeightProcess = app;
                    Message msg = mService.mHandler.obtainMessage(
                            ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                    msg.obj = r;
                    mService.mHandler.sendMessage(msg);
                }
            }

        } catch (RemoteException e) {
            if (r.launchFailed) {
                // 第二次启动失败放弃尝试
                Slog.e(TAG, "Second failure launching "
                      + r.intent.getComponent().flattenToShortString()
                      + ", giving up", e);
                mService.appDiedLocked(app);
                stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                        "2nd-crash", false);
                return false;
            }

            //第一次启动失败，重试一次
            app.activities.remove(r);
            throw e;
        }

        r.launchFailed = false;
        if (stack.updateLRUListLocked(r)) {
            Slog.w(TAG, "Activity " + r
                  + " being launched, but already in LRU list");
        }

        if (andResume) {
            // 部分进程启动，ActivityThread 需resume
            stack.minimalResumeActivityLocked(r);
        } else {
            // activity不在resume开始，而是pause或者stop
            if (DEBUG_STATES) Slog.v(TAG_STATES,
                    "Moving to STOPPED: " + r + " (starting in stopped state)");
            r.state = STOPPED;
            r.stopped = true;
        }
        if (isFrontStack(stack)) {
            mService.startSetupActivityLocked();
        }

        mService.mServices.updateServiceConnectionActivitiesLocked(r.app);

        return true;
    }
```

Step 15

```
	/**
	*ApplicationThreadNative$ApplicationThreadProxy.java
	*只是将ActivityRecord中的参数提取出来进行传递
	**/
	public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        data.writeStrongBinder(token);
        data.writeInt(ident);
        info.writeToParcel(data, 0);
        curConfig.writeToParcel(data, 0);
        if (overrideConfig != null) {
            data.writeInt(1);
            overrideConfig.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        compatInfo.writeToParcel(data, 0);
        data.writeString(referrer);
        data.writeStrongBinder(voiceInteractor != null ? voiceInteractor.asBinder() : null);
        data.writeInt(procState);
        data.writeBundle(state);
        data.writePersistableBundle(persistentState);
        data.writeTypedList(pendingResults);
        data.writeTypedList(pendingNewIntents);
        data.writeInt(notResumed ? 1 : 0);
        data.writeInt(isForward ? 1 : 0);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);//ipc操作
        data.recycle();
    }
```

Step 16

```
	/**
	*ApplicationThreadNative
	**/
	case SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION:
        {
            data.enforceInterface(IApplicationThread.descriptor);
            Intent intent = Intent.CREATOR.createFromParcel(data);
            IBinder b = data.readStrongBinder();
            int ident = data.readInt();
            ActivityInfo info = ActivityInfo.CREATOR.createFromParcel(data);
            Configuration curConfig = Configuration.CREATOR.createFromParcel(data);
            Configuration overrideConfig = null;
            if (data.readInt() != 0) {
                overrideConfig = Configuration.CREATOR.createFromParcel(data);
            }
            CompatibilityInfo compatInfo = CompatibilityInfo.CREATOR.createFromParcel(data);
            String referrer = data.readString();
            IVoiceInteractor voiceInteractor = IVoiceInteractor.Stub.asInterface(
                    data.readStrongBinder());
            int procState = data.readInt();
            Bundle state = data.readBundle();
            PersistableBundle persistentState = data.readPersistableBundle();
            List<ResultInfo> ri = data.createTypedArrayList(ResultInfo.CREATOR);
            List<ReferrerIntent> pi = data.createTypedArrayList(ReferrerIntent.CREATOR);
            boolean notResumed = data.readInt() != 0;
            boolean isForward = data.readInt() != 0;
            ProfilerInfo profilerInfo = data.readInt() != 0
                    ? ProfilerInfo.CREATOR.createFromParcel(data) : null;
            //launch activity
            scheduleLaunchActivity(intent, b, ident, info, curConfig, overrideConfig, compatInfo,
                    referrer, voiceInteractor, procState, state, persistentState, ri, pi,
                    notResumed, isForward, profilerInfo);
            return true;
        }
```

Step 17

```
	/**
	*ApplicationThread.java(位于ActivityThread类中)
	**/
	public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident, ActivityInfo info, 
				Configuration curConfig, Configuration overrideConfig, CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {
            updateProcessState(procState, false);
            ActivityClientRecord r = new ActivityClientRecord();
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);
            //发送LAUNCH ACTIVITY的消息
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```

Step 18

```
	/**
	*ActivtyThread$H.java
	**/
	case LAUNCH_ACTIVITY: 
	{
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
        final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
        r.packageInfo = getPackageInfoNoCheck(
        r.activityInfo.applicationInfo, r.compatInfo);
        handleLaunchActivity(r, null);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    } break;
```

Step 19

```
	/**
	*ActivityThread.java
	**/
	private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }

        //更新confirguration
        handleConfigurationChanged(null, null);

        if (localLOGV) Slog.v(
            TAG, "Handling launch of " + r);

        // Initialize before creating the activity
        WindowManagerGlobal.initialize();
        //调用onCreate、onRestoreInstanceState
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            //调用performReume时调用onStart、onResume
            handleResumeActivity(r.token, false, r.isForward,!r.activity.mFinished && !r.startsNotResumed);

            if (!r.activity.mFinished && r.startsNotResumed) {
                
                try {
                    r.activity.mCalled = false;
                    mInstrumentation.callActivityOnPause(r.activity);
                    if (r.isPreHoneycomb()) {
                        r.state = oldState;
                    }
                    if (!r.activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPause()");
                    }

                } catch (SuperNotCalledException e) {
                    throw e;

                } catch (Exception e) {
                    if (!mInstrumentation.onException(r.activity, e)) {
                        throw new RuntimeException(
                                "Unable to pause activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
                    }
                }
                r.paused = true;
            }
        } else {
            // 出现错误则取消
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {
                // Ignore
            }
        }
    }
```

Step 20

```
	/**
	*ActivityThread.java
	**/
	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,r.activityInfo.targetActivity);
        }

        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            //这里反射创建activity实例
            activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException("Unable to instantiate activity " + component + ": " + e.toString(), e);
            }
        }

        try {
        	//反射创建Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            if (activity != null) {
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                //设置activity主题
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                //这里调用onCreate方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    activity.performStart();
                    r.stopped = false;
                }
                //调用onRestoreInstanceState方法
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```

## 三、总结

由于开始看DroidPlugin框架相关的源码，所以这里也根据上述过程画了一个简单调用流程图，如下

![](https://github.com/stdnull/StudyNotes/blob/master/resources/blog/2016/start_activity_call_part1.png)
![](https://github.com/stdnull/StudyNotes/blob/master/resources/blog/2016/start_activity_call_part2.png)
所以总的看来，整个调用过程为Activity类将调用请求到->ActivityManagerService->ActivityStackSupervior/ActivityStack进行Activity校验和活动栈的处理->ActivityThread尽心Activity的创建、生命周期的调用。

此外参考[这篇博客](http://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/)进行Hook时,发现他只是hook ActivityThread中的mInstrument,明明与Activity调用逻辑不同，但是最后hook是成功的，一时不解，但是通过上述源码的调用过程会发现，**Activity中的mInstrument变量实质上是ActivityThread 在performLaunchActivity方法中创建Activity时将自己的mInstrument attch给Actvity的**

洋洋洒洒贴了2000行左右的Android源代码和刚开始想象的结果出入还是蛮大的，限于水平，Activity的启动过程很多细节都看不懂，只能去看对于的变量的实例究竟是哪一个(SourceInsight帮了忙，否则基本的函数调用流程都搞不清)，草草的走一个流程。看完的感觉就是Activity的启动周期设计到的类实在有点多，而且很多逻辑都有点复杂而且还有重合的地方，还有Android的这部分的源代码有的地方写的实在不懂为什么这么写，感觉写的很混乱。此外看过之后系统中的一些过于性能的统计类真的是蛮多的，有的都没想到竟然会统计。
2016已然要结束，希望2017能够进一步学习到Android中更系统一点的东西。
