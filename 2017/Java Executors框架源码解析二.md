#Java Executors框架源码解析二
---

###2.2 第二类方法相关源码解析

####2.2.1 ScheduledExecutorService是一种可以发送延时任务或者周期性任务的ExecutorService，第二类的构造方法如下

1、newSingleThreadScheduledExecutor()

```
	public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }
```

2、newSingleThreadScheduledExecutor(ThreadFactory threadFactory)

```
	public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1, threadFactory));
    }
```

3、newScheduledThreadPool(int corePoolSize)

```
	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```

4、newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory)

```
	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }
```

5、unconfigurableScheduledExecutorService(ScheduledExecutorService executor)

```
	public static ScheduledExecutorService unconfigurableScheduledExecutorService(ScheduledExecutorService executor) {
        if (executor == null)
            throw new NullPointerException();
        return new DelegatedScheduledExecutorService(executor);
    }
```

由上述构造方法可知第二类方法中的核心类是ScheduledThreadPoolExecutor，其类声明及构造方法如下

```
public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor implements ScheduledExecutorService

	public ScheduledThreadPoolExecutor(int corePoolSize,ThreadFactory threadFactory,RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,new DelayedWorkQueue(), threadFactory, handler);
    }
```

注意ScheduledThreadPoolExecutor方法中并没有提供关于maximumPoolSize参数设置，并且将其设置为最大int值,同时将任务队列设置为DelayedWorkQueue.

####2.2.2 Schedule系列方法相关解析

1、Schedule方法

```
	//以Callable参数为例，
	public <V> ScheduledFuture<V> schedule(Callable<V> callable,long delay,TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        //封装任务对象
        RunnableScheduledFuture<V> t = decorateTask(callable,new ScheduledFutureTask<V>(callable,triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }
```

2、scheduleAtFixedRate方法

```
	public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,null,triggerTime(initialDelay, unit),unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```

3、scheduleWithFixedDelay方法

```
	public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft = new ScheduledFutureTask<Void>(command,null,triggerTime(initialDelay, unit),
        															unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```

由上可见这三种方法都是将任务进行封装，最终则是由delayedExecute方法来执行

4、delayedExecute方法

```
	private void delayedExecute(RunnableScheduledFuture<?> task) {
        if (isShutdown()){
        //非运行态则直接拒绝任务
            reject(task);
        }
        else {
            super.getQueue().add(task);//添加任务到队列
            if (isShutdown() &&!canRunInCurrentRunState(task.isPeriodic()) && remove(task)){
            //条件检查，如果非运行态、任务不可运行于当前状态，则移除任务成功则取消任务
                task.cancel(false);
            }
            else{
                ensurePrestart();//启动线程准备执行任务，仅仅检查线程池中线程的数量是否到达corePoolSize，不够则尝试创建一个新的
            }
        }
    }
```

注意这里任务的处理的逻辑，如果是周期任务，canRunInCurrentRunState默认返回false，非周期任务默认返回true。而且这里没有直接创建一个线程立马执行任务的逻辑，而是只创建一个线程，然后准备去取任务，而且线程类复用了ThreadPoolExecutor的线程类。

5、Task的封装

从上面任务提交的方法来看我们通过各种方法提交的任务都被包装为ScheduledFutureTask类型，ScheduledFutureTask类型的继承图如下

![](https://github.com/stdnull/StudyNotes/blob/master/2017/picture/schedule_task_class.png)

依次查看ScheduledFutureTask继承的类、实现的接口的文档说明，例如Delayed接口继承了Comparable接口用于比较延迟时间,主要方法用于计算当前Task剩余的delay时间，最终可以得到的结论ScheduledFutureTask是一种可取消的、可定时处理的的一种Future和Runable混合型任务对象。

此外可以发现到目前为止还没有出现任务的延迟时间的处理、循环任务的调度等等，只是任务对象本身可定时处理，但是这样是不够的，还需要一定的机制配合才能完成，那么既然线程取任务的方式没有变，那么任务调度的核心肯定在于队列中。

5、DelayedWorkQueue

从ScheduledThreadPoolExecutor的构造方法中可以清楚的发现其所使用的任务队列类型为DelayedWorkQueue。但是由于Worker工作者线程类没有变化，所以接下来主要从DelayedWorkQueue的操作来分析ScheduledThreadPoolExecutor如何实现定时任务。从类的声明看来DelayedWorkQueue始终BlockQueue.


(1)提交任务到队列的方法

```
	//同add方法相同，其他的元素方法最终也调用了offer
    public boolean add(Runnable e) {
        return offer(e);
    }

    public boolean offer(Runnable x) {
            if (x == null)
                throw new NullPointerException();
            RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = size;
                if (i >= queue.length){
                //数组空间不够时，增长空间每次50%,直至最大
                    grow();
                }
                size = i + 1;
                if (i == 0) {
                //第一个数据的设置
                    queue[0] = e;
                    setIndex(e, 0);仅仅设置heapIndex = 0;
                } else {
                //非第一个元素，需要对队列的数据做移动操作
                    siftUp(i, e);
                }
                if (queue[0] == e) {
                //信号通知
                    leader = null;
                    available.signal();
                }
            } finally {
                lock.unlock();
            }
            return true;
        }
```

上面的方法我觉得如果对堆这种数据结果比较熟悉的话，就能一下从调整数组数据的siftup方法名马上猜出来DelayedWorkQueue或许是利用堆来实现的一种根据Task延迟时间来排序的优先权队列。当然这个还需要根据siftup方法的操作来进一步确定

```
	private void siftUp(int k, RunnableScheduledFuture<?> key) {
	//其中k为key存储的目标位置
        while (k > 0) {
            int parent = (k - 1) >>> 1;//与(k-1)/2计算结果相同
            RunnableScheduledFuture<?> e = queue[parent];
            if (key.compareTo(e) >= 0){
           	//如果key的延迟时间长大于其父节点e，则结束
                break;
            }
            //否则交换其与其父节点的位置
            queue[k] = e;
            setIndex(e, k);
            k = parent;
        }
        queue[k] = key;//赋值
        setIndex(key, k);//heapIndex赋值
    }
```

siftup中是典型的堆增加数据时元素调整算法，到这里我觉得基本可以下定论了，DelayedWorkQueue是一种按照任务启动时间排序的一种小端堆，在堆顶的元素必定是最先要执行的任务对象。

(2)从任务队列取任务的方法

take方法

```
	public RunnableScheduledFuture<?> take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
            //死循环任务获取
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0){
                    //到达执行时间，取任务、调整堆、返回数据
                        return finishPoll(first);
                    }
                    first = null; // don't retain ref while waiting
                    if (leader != null){
                    //当前有leader，表示其他线程正在等待取任务，线程等待
                        available.await();
                    }
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;//Leader-Follower Pattern
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread){
                            //释放leader，以便让下条线程成为leader
                                leader = null;
                            }
                        }
                    }
                }
            }
        } finally {
            if (leader == null && queue[0] != null)
                available.signal();
            lock.unlock();
        }
    }
```

poll重载系列方法，只分析带有等待机制的poll(long timeout, TimeUnit unit)，poll()方法较为简单为立即返回，到达执行时间则返回task，否则返回null

```
    /**
    * timeout 取任务最大等待时长
    * unit timeout时长单位
    **/
    public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null) {
                //当前无任务则根据timeout决定立即返回还是等待
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0){
                    //如果任务执行时长立即返回
                        return finishPoll(first);
                    }
                    if (nanos <= 0){
                        return null;
                    }
                    first = null; //等待时不持有引用
                    if (nanos < delay || leader != null){
                    //如果执行时间未到或者leader线程不为空，等待
                        nanos = available.awaitNanos(nanos);
                    }
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                        	//等待时间达到
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;//leader置空，从而下个Follower线程能够成为leader
                        }
                    }
                }
            }
        } finally {
            if (leader == null && queue[0] != null)
                available.signal();
            lock.unlock();
        }
    }
```

基本的获取task机制了解之后，再看poll与take方法最后获取数据时调用的finishPoll方法

```
	private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
        int s = --size;
        RunnableScheduledFuture<?> x = queue[s];
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);//调整堆数据
        setIndex(f, -1);
        return f;
    }
```

可以看到主要是获取完数据后对于堆的数据调整，维持小端堆数据的有序性

```
	/**
	* k 调整开始位置
	* key 调整开始元素
	* 调整方法为首先将最后一个元素移到最前面，然后依次比较交换数据元素
	**/
	private void siftDown(int k, RunnableScheduledFuture<?> key) {
        int half = size >>> 1;
        while (k < half) {
            int child = (k << 1) + 1;//左子节点
            RunnableScheduledFuture<?> c = queue[child];//左子节点元素
            int right = child + 1;//右子节点
            if (right < size && c.compareTo(queue[right]) > 0){
            //左 > 右
                c = queue[child = right];
            }
            if (key.compareTo(c) <= 0){
            //当前父元素小于最小的子元素，退出
                break;
            }
            queue[k] = c;//交换值
            setIndex(c, k);
            k = child;
        }
        queue[k] = key;//节点赋值
        setIndex(key, k);
    }
```

其实上面的方法看下来其中有一个看起来我觉得有点怪的变量leader，可以看到它的作用貌似仅仅在取任务的时候赋值为当前的线程的引用，然后就被置为null了，后来从注释中看到这是一种Leader-Follower 模式,这种模式有消除线程间的内存交换等优点(具体可可google or baidu).

(3)任务移除

```
	public boolean remove(Object x) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = indexOf(x);//移除任务索引查找
            if (i < 0)
                return false;
            setIndex(queue[i], -1);
            int s = --size;
            RunnableScheduledFuture<?> replacement = queue[s];
            queue[s] = null;
            if (s != i) {
            //堆调整
                siftDown(i, replacement);
                if (queue[i] == replacement)
                    siftUp(i, replacement);
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    private int indexOf(Object x) {
        if (x != null) {
            if (x instanceof ScheduledFutureTask) {
                int i = ((ScheduledFutureTask) x).heapIndex;
                //heapIndex用于快速定位Task
                if (i >= 0 && i < size && queue[i] == x)
                    return i;
            } else {
                for (int i = 0; i < size; i++)
                    if (x.equals(queue[i]))
                        return i;
            }
        }
        return -1;
    }
```

####2.2.3 ScheduledExecutorService实现方式总结
由上分析可见第二类线程池的实现的核心是小端堆的实现，其他的逻辑基本与ThreadPoolExecutor类似，只是线程开启的要更为严格，不会超过corePoolSize.


###2.3 第三类方法相关源码解析

####2.3.1 不可定制的线程池，构造方法如下

1、unconfigurableExecutorService(ExecutorService executor)

```

    public static ExecutorService unconfigurableExecutorService(ExecutorService executor) {
        if (executor == null)
            throw new NullPointerException();
        return new DelegatedExecutorService(executor);
    }
```

2、unconfigurableScheduledExecutorService(ScheduledExecutorService executor) 

```
    public static ScheduledExecutorService unconfigurableScheduledExecutorService(ScheduledExecutorService executor) {
        if (executor == null)
            throw new NullPointerException();
        return new DelegatedScheduledExecutorService(executor);
    }
```

从构造方法中可以清晰的看出不可定制的线程池是通过代理的方式去除了Executor的相关set方法来实现，此外第四类方法、第五类方法与第三类方法都较为简单，在此不再赘述，至此Executors类提供的服务基本分析结束(除了ForkJoin框架)对于线程池的一些知识点有了进一步的认识。