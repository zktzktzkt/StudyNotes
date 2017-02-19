#Java Executors框架源码解析一
---

##1、概要
在开发并发程序时，我们可能或多或少都要用到线程池来复用线程以达到较高的性能，但是我们自己写的线程池往往不能达到最大的利用率。考虑到并发的问题，JDK1.5之后引入Executors框架，这样Java中便自带了一个线程池，这边blog也是为了弄清JDK是如何实现线程池，复用线程。

##2、Executors具体作用
据文档所描述：Executors是一种工厂类或者工具类，封装了关于 Executor, ExecutorService, ScheduledExecutorService, ThreadFactory, and Callable类的操作。具体提供了一下积累方法

+ 用于创建并返回一个使用了通常使用的配置的ExecutorService
+ 用于创建并返回一个使用通常使用的配置的ScheduledExecutorService
+ 用于创建并返回一个包装了的ExecutorService，它通过私有化一个特定的方法禁用了一些配置
+ 用于创建并返回一个线程工厂，用于创建已知状态的线程
+ 用于创建并返回一个不同于其他闭包样式的Callable，这样便可用于需要Callable的方法中执行

###2.1第一类方法相关源码解析
2.1.1、ExecutorService是一种提供了处理异步任务方法的类，Executors类中提供了构造方法如下几个

1、newCachedThreadPool(ThreadFactory threadFactory)

```
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>(),threadFactory);
    }
```

2、newCachedThreadPool()

```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>());
    }
```

3、newFixedThreadPool(int nThreads, ThreadFactory threadFactory)

```
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>(),threadFactory);
    }
```

4、newFixedThreadPool(int nThreads)

```
	public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
    }
```
5、newSingleThreadExecutor(ThreadFactory threadFactory)

```
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>(),threadFactory));
    }
```

6、5newSingleThreadExecutor()

```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>()));
    }
```
7、newWorkStealingPool(int parallelism)

```
public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool(parallelism,ForkJoinPool.defaultForkJoinWorkerThreadFactory,null, true);
    }
```

8、newWorkStealingPool()

```
public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool(Runtime.getRuntime().availableProcessors(),ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```

9、unconfigurableExecutorService(ExecutorService executor)

```
public static ExecutorService unconfigurableExecutorService(ExecutorService executor) {
        if (executor == null)
            throw new NullPointerException();
        return new DelegatedExecutorService(executor);
    }
```

由上构造方法可知我们平常使用Executors的new系列方法创建的ExecutorService大部分实质上是一个ThreadPoolExecutor，其他的ForkJoin，一下的讨论也主要基于ThreadPoolExecutor的源码进行讨论。

2.1.2 ThreadPoolExecutor的几种运行状态

与线程的状态类似，线程池也定义了线程池的几种运行状态，概述如下

1、Running = -536870912:接受新task且处理排队的任务

2、Shutdown = 0:不接受新任务，处理排队的任务

3、Stop = 536870912:不接受新任务、不处理排队的任务，打断正在执行的任务

4、Tidying = 1073741824:所有的任务都已结束，工作线程数量等于0，将要执行terminate方法

5、Terminate = 1610612736:terminate()方法已经执行完毕

ThreadPoolExecutor中状态切换时机如下

+ RUNNING -> SHUTDOWN：当调用shutdown()方法时

+ (RUNNING or SHUTDOWN) -> STOP：仅当shutdownNow()方法调用
 
+ SHUTDOWN -> TIDYING：在tryTerminate中被设置,当状态为SHUTDOWN时且队列和线程池为空

4、STOP -> TIDYING：在tryTerminate中被设置,当状态为STOP时且队列和线程池为空

5、TIDYING -> TERMINATED：在tryTerminate设置，当TIDYING设置完毕、terminate执行完毕之后




2.1.3 ThreadPoolExecutor的构造参数说明

+ corePoolSize:常驻于线程池的数量，如果设置allowCoreThreadTimeOut同样会超时关闭线程池
+ maximumPoolSize:线程池最大线程数
+ workQueue:任务缓存队列
+ keepAliveTime:线程空闲关闭时间
+ threadFactory:线程池利用Factory用于创建线程
+ handler:拒绝任务处理器

2.1.4 ThreadPoolExecutor工作流程分析

1、Task的提交

```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
        //如果当前线程数量少于corePoolSize，则尝试直接创建线程处理任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
        //如果当前正在运行状态、且任务添加到队列成功
            int recheck = ctl.get();//二次check状态
            if (! isRunning(recheck) && remove(command)){
            //如果非运行状态且移出队列成功，拒绝任务
                reject(command);
            }
            else if (workerCountOf(recheck) == 0){
            //当前工作线程为零，尝试添加空任务?
                addWorker(null, false);
            }
        }
        else if (!addWorker(command, false)){
        //尝试添加任务失败，线程池已饱和
            reject(command);
        }
    }
```
其中必要参数解释如下

AtomicInteger ctl:利用位运算封装了两个不同概念的字段于一个Integer
workerCount:指示当前工作中的线程的数量，利用后29位
runState:只是当前线程池的状态

2、addWorker方法解析

```
	/**
	* firstTask 创建新线程所使用的任务对象
	* core 标识使用的线程池比较数量
	**/
	private boolean addWorker(Runnable firstTask, boolean core) {
		//retry代码块标签
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            if (rs >= SHUTDOWN &&! (rs == SHUTDOWN &&firstTask == null &&! workQueue.isEmpty())){
            //非运行状态且(运行状态不为SHUTDOWN或任务为空或队列为空)
                return false;
            }

            for (;;) {
                int wc = workerCountOf(c);
                //CAPACITY=2^29-1，线程超过容量不创建新线程
                if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c)){
                //满足创建线程条件且增加workCount成功跳出循环
                    break retry;
                }
                c = ctl.get(); 
                if (runStateOf(c) != rs){
                //线程池状态发生变化退出循环
                    continue retry;
                }
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);//创建线程
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();//获取锁，维护同步状态
                try {
                    int rs = runStateOf(ctl.get());
                    if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    //线程池处于运行状态或者（线程池已经SHUTDOWN且头任务为空）
                        if (t.isAlive())
                            throw new IllegalThreadStateException();
                        workers.add(w);//线程引用存储
                        int s = workers.size();
                        if (s > largestPoolSize){
                        //更新最大线程数量
                            largestPoolSize = s;
                        }
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                //启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (!workerStarted){
            //添加失败的后续处理
                addWorkerFailed(w);
            }
        }
        return workerStarted;
    }
```

至此其实增加任务的逻辑已经结束了，但是我们似乎没有看到线程复用相关的逻辑，但是我们可以看到的时我们提交的任务被交给了Worker处理同时通过搜索任务的队列workQueue来使用，也可以了解到复用逻辑位于Worker线程中.

3、Worker线程构造方法

```
Worker(Runnable firstTask) {
    setState(-1); 
    this.firstTask = firstTask;//保存处理任务
    this.thread = getThreadFactory().newThread(this);//利用线程工厂创建线程
}
```

4、Worker的Run方法

```
	//将方法代理到外部ThreadPoolExecutor的runWorker方法
	public void run() {
        runWorker(this);
    }

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock();//允许中断
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
            //在死循环中不断取任务
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                	(Thread.interrupted() &&runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
                    wt.interrupt();
                try {
                	//如果线程池状态<=SHUTDOWN且线程没有interrupted则执行任务
                    beforeExecute(wt, task);//Hook函数
                    Throwable thrown = null;
                    try {
                        task.run();//执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);//Hook函数
                    }
                } finally {
                    task = null;
                    w.completedTasks++;//统计
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
        	//线程结束
            processWorkerExit(w, completedAbruptly);
        }
    }
```

至此可以看出线程池的线程复用核心方法是一个死循环，这样可以免去线程创建的时间开销较少线程切换的次数，但是貌似我们没有看见超时的逻辑在哪，所以接着看getTask()函数如何获取任务

4、getTask方法解析

```
	private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
        //再死循环中取任务
            int c = ctl.get();
            int rs = runStateOf(c);
            //线程池状态不可执行任务
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;//线程是否需要超时退出

            if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
            //如果线程数量过大或者(超时且(线程数>1或者队列空))
                if (compareAndDecrementWorkerCount(c)){
                //减少失败，返回
                    return null;
                }
                continue;
            }

            try {
                Runnable r = timed ?workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;//取任务超时
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

显然超时退出机制服用了阻塞队列的超时机制，如果任务取超时则任务线程空闲时间过长，退出线程。至此任务的处理逻辑已经结束，那么接着看线程结束后的处理逻辑。

5、processWorkerExit方法解析

```
	private void processWorkerExit(Worker w, boolean completedAbruptly) {

        if (completedAbruptly){
        //非正常退出，更新线程数量
            decrementWorkerCount();
        }
        final ReentrantLock mainLock = this.mainLock;//临界量同步
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);//移除线程引用
        } finally {
            mainLock.unlock();
        }

        tryTerminate();//尝试转换到terminate状态
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
        //可执行任务状态
            if (!completedAbruptly) {
            //正常退出
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty()){
                    min = 1;
                }
                //如果当前工作线程大于corePoolSize结束方法
                if (workerCountOf(c) >= min)
                    return;
            }
            //否则新增线程，保证线程池中缓存的线程数,线程池填充
            addWorker(null, false);
        }
    }
```

6、tryTerminate方法

```
	final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||runStateAtLeast(c, TIDYING) ||(runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty())){
            //线程池正在运行或者处于TIDYING或者(处于SHUTFOWN且队列非空)，直接返回
                return;
            }
            if (workerCountOf(c) != 0) {
            //如果运行数量不为0，尝试interrupt一个线程
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();//同步
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                //尝试设置线程池状态为TIDYING成功则准备调用terminated方法
                    try {
                        terminated();//调用Hook方法terminated
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));//设置状态
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
        }
    }
```

从这里可以看到的是每次一个线程池中的线程结束，线程池都会检查状态来决定是否需要切换状态到关闭

7、Shutdown与ShutdownNow方法

```
	public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);//状态切换到SHUTDOWN
            interruptIdleWorkers();//尝试interrrupt空闲线程
            onShutdown(); //hook方法
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);//状态切换到STOP
            interruptWorkers();//interruput所有线程
            tasks = drainQueue();//提取未完成任务
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```

由上看出二种方法区别在于状态切换、线程的处理以及对于排队任务的处理，后者相对而言关闭的更为彻底。

8、Task被reject时的处理

由上述分析可知当线程池处于满负荷状态时提交的任务会被拒绝处理，此时线程池会调用reject方法，如下

```
	final void reject(Runnable command) {
        handler.rejectedExecution(command, this);//直接将任务代理给拒绝任务处理器
    }
```






