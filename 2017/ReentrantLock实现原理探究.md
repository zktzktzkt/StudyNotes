# ReentrantLock实现原理探究
## 1、ReentrantLock介绍
>A reentrant mutual exclusion Lock with the same basic behavior and semantics as the implicit monitor lock accessed using synchronized methods and statements, but with extended capabilities.

简单的说ReentrantLock是一种扩展的synchronized可重入互斥锁。主要有一下几个特性

+ 一个ReentrantLock总是被当前获得锁的线程所拥有
+ ReentrantLock支持公平锁与非公平锁(tryLock方法例外)
+ 如其他内置锁表现相同，反序列化的对象总是非锁定的
+ 支持同一线程最大数为2147483647的递归获取锁

## 2、Sync继承结构
由于ReentrantLock内部方法执行时基本上都将调用Sync的相应方法去完成，即使用Sync类来作为基础的锁的同步控制，所以Sync是ReentrantLock的核心对象。所以先简单看下Sync类的结构，如下图

![](https://github.com/stdnull/StudyNotes/blob/master/2017/picture/sync_class.png)

## 3、基本方法分析

### 3.1 构造方法
```
	public ReentrantLock() {
        sync = new NonfairSync();//Sync的子类
    }
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

默认的构造方法是非同步锁(文档指出这样的效率相对更加高),同时也可根据参数来创建公平锁。

### 3.2 lock方法
```
	//非公平锁的lock方法
	final void lock() {
        if (compareAndSetState(0, 1)){
        //CAS状态赋值，如果成功表示当前锁没有被获取、设置当前线程获得锁
            setExclusiveOwnerThread(Thread.currentThread());
        }
        else{
            acquire(1);//请求锁
        }
    }

    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)){
        //如果获取失败、线程等待时获取锁成功且在等待时被中断
            selfInterrupt();//调用线程interrupt方法，清除中断标记
        }
    }
```

由上可见非公平锁获取锁，首先优先利用CAS机制根据当前状态来获取锁，其次通过acquire方式获取锁，利用acquire方式获取锁又分为两步-tryAcquire、acquireQueued。先看tryAcquire方法如何工作

```
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
        //当前锁未被获取，尝试CAS机制先置状态位、然后获取锁
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
            //置状态位失败，说明有其他线程获得锁
        }
        else if (current == getExclusiveOwnerThread()) {
        //递归形式获取锁
            int nextc = c + acquires;
            if (nextc < 0) // 重复获取次数溢出
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

```

至此可见tryAcquire和平常我们的逻辑处理基本类似-首先检查状态、然后赋值、返回结果。此外可以看出Sync的状态同步依靠的是CAS机制来保证线程并发时状态更新的一致性。在看下一步addWaiter方法以及acquireQueued方法之前，先简单介绍一下Node类

#### Node类
由文档介绍可知Node类为锁请求等待队列的数据结构(***链表***)的实现，其中等待队列是一种CLH锁(一般用于自旋锁)队列的变形，这里借用了的它的基本机制用于blocking synchronizers.

Node元素中waitStatus几种状态

+ SIGNAL = -1:表明后继节点需要unpark-如果当前Node的后继正在或者将要block，当前的Node在释放锁、或者取消获取锁时需要unpark后继节点.
+ CANCELLED = 1:表明线程已取消-Node由于超时或者中断，Node永远不会离开这种状态。
+ CONDITION = -2:表明线程正在条件等待-当前Node处于condition的等待队列中
+ PROPAGATE = -3:表示下一个acquireShare需要无条件传播.
+ 一般状态 = 0:不属于上述任何一种情况，一般初始化为0

再看addWaiter方法

```
	/**
	* 此处mode的值为null,指示mode为互斥性锁EXCLUSIVE，此外还有共享性锁SHARE
	**/
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);//Node状态初始化为0
        Node pred = tail;
        if (pred != null) {
        //如果等待队列的尾结点不空
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
            //pred节点赋值成功，尝试插入节点，成功则直接返回，如果赋值失败表明有其他的线程插入了一次节点
                pred.next = node;
                return node;
            }
        }
        //插入节点、死循环知道插入成功
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
        for (;;) {
        //自旋，死循环插入等待节点到链表中
            Node t = tail;
            if (t == null) {
            //尾结点为空，初始化链表设置头节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
            	//插入节点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

上面的操作如同方法名，由于当前线程暂时无法获取锁，向等待队列中添加一个节点，等待其他线程释放，那么既然上面的方法已经想队列中添加了等待节点，那么acquireQueue方法是干嘛的呢？

```

    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();//获取前驱节点
                if (p == head && tryAcquire(arg)) {//arg默认是1
                //如果前驱节点为头节点且尝试获取锁成功
                //如果节点的前驱节点为头节点-表明当前链表中只有一个线程
                    setHead(node);
                    p.next = null; //gc
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()){
                	//Node结点状态更新成功且线程block时被中断
                    interrupted = true;
                }
            }
        } finally {
            if (failed){
            	//如果获取锁失败，即成功获得锁
                cancelAcquire(node);
            }
        }
    }

    //更新获取锁失败的Node节点的状态信息，返回true表示当前线程需要block，用于设置前驱状态为signal
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL){
        //前驱节点release时会通知当前节点
            return true;
        }
        if (ws > 0) {
        //表明前去Node pred取消获取锁，跳过取消节点
            do {
                node.prev = pred = pred.prev;//节点向前移动
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //设置前驱为通知状态,返回false,由于节点需要再次尝试是否能够获取锁
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

	//使用LockSupport block线程、返回当前线程是否interrupted
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }

    //阻塞线程，直至线程被中断、或者其他线程unpark
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        //调用park阻塞线程
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }

    //获取锁失败的处理办法，但是从理论上来说应该没有失败的可能，除非线程意外死亡
    private void cancelAcquire(Node node) {
        if (node == null)
            return;
        node.thread = null;
        Node pred = node.prev;
        while (pred.waitStatus > 0){
        //跳过已取消的节点，将取消节点从链表上摘链
            node.prev = pred = pred.prev;
        }
        Node predNext = pred.next;
        node.waitStatus = Node.CANCELLED;//至当前节点的状态为取消
        if (node == tail && compareAndSetTail(node, pred)) {
        	//如果当前的节点为最后一个节点，将其前驱节点置为最后一个节点
            compareAndSetNext(pred, predNext, null);
        } 
        else {
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
            //如果当前节点不是第一个节点、(非取消状态且置SIGANL状态成功)、前驱节点线程不空
                Node next = node.next;
                //重新链接链表
                if (next != null && next.waitStatus <= 0){
                    compareAndSetNext(pred, predNext, next);
                }
            } else {
            	//链表尾开始向node节点遍历、unpark唤醒一个正在等待的线程距离Node的线程
                unparkSuccessor(node);
            }

            node.next = node; 
        }
    }
```

至此ReentrantLock以非公平锁模式获取锁的方法基本结束，主要流程如下

1、首先检查状态，锁为空闲则尝试更改状态，立即获取锁，将锁拥有这置为当前线程

2、锁不空闲，尝试获取锁，成功则返回

3、获取失败，首先根据当前线程添加新的Node节点方法等待队列中

4、死循环中操作当前请求的等待节点，处于block-请求锁-block-请求锁的循环中直到成功。

5、出现异常、将等待节点摘链，***唤醒最近的后继等待节点***,lock方法结束。但是这里的异常如果不是线程崩溃之类的破坏性异常，lock返回返回，线程岂不是在没有获取到锁的情况下执行了临界区的代码？

### 3.3 公平的lock方法

```
	final void lock() {
        acquire(1);
    }

    public final void acquire(int arg) {
    	//tryAcquirfe方法在FairSync中override，acqiureQueued逻辑不变
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            //如果当前等待队列中没有其他线程且锁状态更新成功则获得锁，立即返回
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

```

从tryAcqiure方法逻辑的变化可以看到的是，当有新请求时，不会因为当前锁为空闲状态就立即分配锁的所有权而是首先检查排队队列是否为空，如果存在等待的线程，则无法获取到所有权，只能首先等待进入队列再获取。

### 3.4 unlock方法

```
	public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0){
            //唤醒最长等待时间的节点
                unparkSuccessor(h);
            }
            return true;
        }
        return false;
    }
```

### 3.5 lockInterruptibly方法

```
 	public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public final void acquireInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()){
                	//线程中断，抛出异常推出
                    throw new InterruptedException();
                }
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

和lock对比起来，lockInterruptibly方法只是将原先lock方法中线程被中断的处理逻辑改为了一旦中断则抛出异常立即退出。

### 3.6 tryLock()方法

```
	public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

tryLock由上可见，只尝试获取锁一次，不进入等待队列，一旦失败立即返回，相应的tryLock(long timeout, TimeUnit unit)方法则通过LockSupport方法睡眠机制来实现。

### 3.7 tryLock(long timeout, TimeUnit unit)方法

```
	public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)//等待时间超出立即退出
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) && nanosTimeout > spinForTimeoutThreshold){
                	//超时逻辑处理，线程睡眠时间
                    LockSupport.parkNanos(this, nanosTimeout);
                }
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

## 4、总结

ReentrantLock核心同步机制主要由CAS机制来保证，同时维护一个等待队列来保存暂时无法获取到锁的线程。而公平锁的机制主要区别为在线程请求锁时，公平锁不论当前锁是否空闲都会检查当前等待队列中是否有等待线程，如果有则直接进入等待不会获取，而每当unlock调用用时，会首先唤醒最先进入等待队列的睡眠线程。此外基础的线程阻塞逻辑由LockSupport来提供。










