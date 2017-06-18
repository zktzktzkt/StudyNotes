# LeakCanary 框架简要解析

## 前言

如题,本篇主要从整体的逻辑上分析LeakCanary框架的实现以及总结其中使用的一些个人觉得比较有趣、实用的Android API,具体LeakCanary是什么、有什么好处、为什么要用,可以参见LeakCanary的[wiki-FAQ](https://github.com/square/leakcanary/wiki/FAQ),其中的解释已经非常详尽,这里就不画蛇添足了.

## LeakCanary 框架解析

### 1 LeakCanary实现原理

实现原理LeakCanary项目的github已经有了相关解释,这里再总结一下如下

<p>  当开发者想要检测一个变量是否存在内存泄漏时,直接通过RefWatcher的watch方法来进行监控,然后RefWatcher会为要监控的对象创建一个WeakReference,以及生成一个UUID来作为key表示指定变量,之后会在主线程空闲时在非主线程中判断变量是否存在泄露. 首先移除已经被回收的变量,紧接着判断监控的变量是否在为已经被移除,如果已经移除立即返回,如果没有移除,执行GC然后在判断一次变量是否被回收,如果没有回收,则立即执行dump heap,然后分析heap的使用情况来得到内存泄漏的引用链.</p>

<p>  大致的使用原理如上所述,此外默认情况下LeakCanary只监控Activity的引用状况的实现则是通过注册了一个全局的ActivityLifecycleCallbacks来监控Acitivity的使用情况,在Actiivty的onDestroy方法执行时dump heap来分析是否存在内存泄漏</p>

### 2 LeakCanary核心服务类说明

+ AndroidWatchExecutor - 一个异步任务处理类,主要在该线程中完成变量的引用可达性检测、GC的触发、dump heap文件等操作,并将结果发送给HeapAnalyzerService

+ HeapAnalyzerService - AndroidWatchExecutor处理完任务后交由处理,主要是一个用于分析heap的文件的服务,在此处解析、构造出引用泄露链

+ DisplayLeakService - 根据HeapAnalyzerService的结果来显示通知内容,重命名dump文件等操作

+ RefWatcher - 通用的对象引用监控类,对外提供了watch方法来判断一个Object是否存在内存泄漏

## tips

+ 使用PackageManager 启用/禁用组件

LeakCanary相关代码如下

```
public static void setEnabledBlocking(Context appContext, Class<?> componentClass,
      boolean enabled) {
    ComponentName component = new ComponentName(appContext, componentClass);
    PackageManager packageManager = appContext.getPackageManager();
    int newState = enabled ? COMPONENT_ENABLED_STATE_ENABLED : COMPONENT_ENABLED_STATE_DISABLED;
    // Blocks on IPC.
    packageManager.setComponentEnabledSetting(component, newState, DONT_KILL_APP);
  }
```

此外需要注意的是方法中的注释**Blocks on IPC**,所以为了避免主线程阻塞,LeakCanary是在一个名为File-IO单独的线程池中执行该方法的.

+ Debug类的使用

在用LeakCanary之前,我原以为LeakCanary是用Runtime来执行一个命令行来dump heap,看之后发现Debug类已经提供了很多相关的API,LeakCanary主要使用Debug 来dump heap的内存情况以及使用Debug来判断是否attach了调试器

+ 确保GC执行的方法

```
@Override 
public void runGc() {
  // Code taken from AOSP FinalizationTest:
  // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
  // java/lang/ref/FinalizationTester.java
  // System.gc() does not garbage collect every time. Runtime.gc() is
  // more likely to perfom a gc.
  Runtime.getRuntime().gc();
  enqueueReferences();
  System.runFinalization();
}
```

+ 注册多个入口Activity

通过在manifest文件将多个Activity的intent-fileter注册为android.intent.action.MAIN/android.intent.category.LAUNCHER来实现一个应用多个入口图标

+ AndroidExcludedRefs 枚举用法

由于枚举类型平常用的很少,这里刚看见LeakCanary的用法的时候还有点奇怪,重新看了一下枚举相关的用法,原来枚举类型也可以定义构造方法、抽象方法、成员变量等,而且每一个枚举成员都和类的类型一样, 此外EnumSet.allOf API可以方便的从一个枚举类中获取枚举成员变量的集合.具体使用可以参见AndroidExcludedRefs.