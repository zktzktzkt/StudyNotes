# Handler部分相关代码分析
------
**一、Handler是什么?**

文档是这么说的：
>A Handler allows you to send and process Message and Runnable objects associated with a thread's MessageQueue. Each Handler instance is associated with a single thread and that thread's message queue. When you create a new Handler, it is bound to the thread / message queue of the thread that is creating it -- from that point on, it will deliver messages and runnables to that message queue and execute them as they come out of the message queue. 
There are two main uses for a Handler: (1) to schedule messages and runnables to be executed as some point in the future; and (2) to enqueue an action to be performed on a different thread than your own

大意是说一个Handler可以处理与之相关的线程的MessageQueen的消息、Runable对象，同时可以发送消息、Runnable对象。Handler的两个主要用途是处理某个将要发生的事件、向另一个线程发送消息

**二、基本使用**

```
	@Override
    public void run() {
       	Looper.prepare();
       	handler=new Handler(Looper.myLooper(),new HandlerCallBack());
       	Looper.loop();
    }
``` 

**三、问题详解**

由一、二我们知道Handler需要绑定一个线程、消息队列，那么是在哪里绑定的呢？二中的Looper又是什么？
首先从二中的代码开始看Looper.prepare()这个方法做了什么

```
	public static void prepare() {
       	prepare(true);//在Loop的prepare方法调用了重载方法
    }

    //可以看出来在这里创建了Looper对象，并放在了ThreadLocal中
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

	//在Looper的构造方法中创建了消息队列
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
再看第二行Handler的初始化方法

```
    public Handler(Looper looper, Callback callback) {
        this(looper, callback, false);//调用了另一个构造方法
    }

    //在这里绑定了Looper及其消息队列
    public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

最后看最后一行Looper.loop()方法中我们主要关注这段代码

```
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
                    // No message indicates that the message queue is quitting.
            return;
        }  
        略.....
        msg.target.dispatchMessage(msg);
        略.....
        msg.recycleUnchecked();
    }
```

可以看到首先利用MessageQueue的next方法来获取一个Messaage对象，然后获取msg对应的Handler即target对象(那么就是说虽然一个线程只能有一个Looper，用ThreadLocal保证的,但是一个Looper可以为多个Handler服务)，分发消息到Handler中，最后回收消息，所以已经用过的Message对象不能再用。**所以这里的逻辑就是获取消息、处理消息、回收对象**

接下来再看Message的target对象即Handler对象怎么绑定的，从Message类的部分Obtain方法中看以看到在其中有对Handler赋值，但是如果没有使用这类方法来创建Message对象呢？那可能从哪赋值呢，从另一点看消息是用Handler对象发送的，这里有Handler的引用又有Message的引用，那么可以猜想这里也有可能绑定Handler，发送消息最终调用了如下方法来将消息插入队列

```

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;//为target对象赋值
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```

最后看Handler对消息的处理，源代码如下
```
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);//调用Message对象自己的Callback对象来处理
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);//空转
        }
    }
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```
* 可以看出来，如果Message对象本身有其callback的实现，则直接运行Message的callback实现，否则
如果Handler的callBack对象不为空则直接调用，否则空转.

最后值得一提的是一般Message对象有两种方法来获取，一种是直接new一个，一种是obtain系列方法其代码如下
```
	public static Message obtain() {//从消息池重获取一个对象
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();//消息池为空则直接创建
    }

    void recycleUnchecked() {//回收时将对象加入，头插法
        略...
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```
**可以看出来Message类中是存在一个消息池的，而且是使用链表的形式来实现的.**

**四、总结**

从上面的分析可以知道Handler的工作有Handler、Looper、MessageQueue、Message构成，其中Handler用于消息的处理、Looper是一个调度者，它在一个死循环中不断的从MessageQueue中获取消息，而MessageQueen用于存储消息来解耦，Message是消息的载体。其次要提的是我认为使用Message的Obtain方法更高效，比较省。最后从上面的分析可以看到在Hanlder的流程中,存在一个MessageQueue->Message->Handler的引用，这也就是在主线程中使用Handler时，如果消息还未被处理完，则可能会导致内存泄漏，知道消息处理完之后才可能被释放

    