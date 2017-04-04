# AsyncTask源码分析
---

## 一、AsyncTask简介

AsyncTask适用于后台线程处理任务并将结果发送到UI线程的一个帮助类(围绕Thread和Handler),避免操作线程和Handler，但是它的理想使用场景是相对比较短的耗时任务，其一共包括四个步骤onPreExecute, doInBackground, onProgressUpdate and onPostExecute.

接着一个重要的问题就是它的执行任务的顺序问题:在文档中提到AsyncTask首次被引入时其任务顺序执行于一个单独的后台线程，在1.6之后修改为任务可以利用线程池来并发执行，在3.0之后为了避免一些应用问题又改为单条线程执行任务的逻辑，但是提供了可选的并发执行任务的方法executeOnExecutor(java.util.concurrent.Executor, Object[])

## 二、实现分析

一般来说我们都是从executor方法开始后台任务的调用，而这个方法最后调用的是如下方法

```

	public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            //此处为检查状态，如果当前任务已经在执行或者已经结束则抛出异常，所以一个AsyncTask只能被执行一次
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;//保存参数将传给doInBackground方法
        exec.execute(mFuture);

        return this;
    }

```
在上面的方法中可以看到该任务在执行之前调用了onPreExecute()方法，所以可以在后台任务开始之前在主线程中做一些UI操作的准备工作，同时保存了传入的参数，执行后台任务然后返回自身的引用。接着我们关注上面的三个变量**mWork、exec、mFuture**

先看exec变量,首先可以知道的它是方法的参数且任务是交由它进行执行，且execute调用时传入的参数为**sDefaultExecutor**，那么我们可以看一下该变量的创建

```
	//可以看到只有一个地方对它进行了改变及类初始化的时候
	public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    //而该方法是隐藏的所以忽略
	public static void setDefaultExecutor(Executor exec) {
        sDefaultExecutor = exec;
    }

    //而类SerialExecutor的声明是这样的
    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();//一个双端任务队列
        Runnable mActive;
        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {//插入一个新任务到任务队列中
            	//以下的任务任务逻辑很简单，执行完一个任务则调度下一个任务开始执行
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {//当前任务为空则开始取任务进行执行
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {//取任务执行交由线程池执行
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

```
由上面的代码可以看到，任务最后是交由一个THREAD_POOL_EXECUTOR线程池进行执行，由它的创建可以看到该线程池可以并发执行多个任务，但是由于交付任务的Executor SerialExecutor是一个结束才交付下一个任务所以默认的任务执行顺序是串行的，所以我们可以自己实现一个并发交付任务的executor来进行多任务的交付来让AsyncTask来并发执行任务。

再看exec执行的任务对象mFuture对象,首先还是看它的声明与创建的时机

```

	private final FutureTask<Result> mFuture;//类型为一个Future任务,当然从名字也可看出来
	//创建逻辑位于构造方法中，调用的方法和逻辑并不多，注意这里传入了mWorker这个参数
	mFuture = new FutureTask<Result>(mWorker) {
            //done方法在该Future任务执行完毕后才执行
            protected void done() {
                try {
                    postResultIfNotInvoked(get());//没有执行则调用Future的get()方法获取结果并post
                }
                //.... 
                catch (CancellationException e) {
                    postResultIfNotInvoked(null);//取消任务则不返回结果
                }
            }
        };

    //postResultIfNotInvoked的调用逻辑如下两个方法
    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {//如果false则调用postResult
            postResult(result);
        }
    }
    //这里则是利用Handler来传递结果到UI线程，该Handler的创建是利用Looper.mainLooper()
    private Result postResult(Result result) {
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

```

接着看mWork变量，它的声明和创建的时机如下

```
	//变量的声明以及对应的类的定义如下
	private final WorkerRunnable<Params, Result> mWorker;
	private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
    //在AsyncTask的构造方法中初始化如下,然后即使初始化mFuture对象并传入mWorker对象
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);//执行标记设为true
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            Result result = doInBackground(mParams);//执行后台任务
            Binder.flushPendingCommands();
            return postResult(result);//发送结果
        }
    };

```

最后看AsyncTask如果将事件发送给主线程

```
	//Handler的任务完成的回调，可以看到onPostExecute的调用时机
	private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());//利用UI线程的Looper来调度事件
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    result.mTask.finish(result.mData[0]);//回送结果
                    break;
                case MESSAGE_POST_PROGRESS://进度更新
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

```

## 三、总结

从上面的分析看来AsyncTask是利用线程池来处理后台任务，但是是在主线程中利用SerialExecutor对象来进行任务的调度，而该类是串行的提交任务，所以AsyncTask也是串行的执行，因此我们可以自己实现一个对应的调度类使得AsyncTask可以并发的执行任务。然后关于任务的包装则是通过Callable类、Future类来进行包装即mWorker、mFuture变量的协作，接着关于与主线程的通信则是利用mainLooper的Handler来进行。**最后值得注意的是AsyncTask的相关的线程池、任务提交类等都是静态成员对象**