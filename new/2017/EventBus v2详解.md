# EventBus V2原理解析

之所以题目叫EventBus V2原理解析是由于本篇主要的代码都是基于EventBus V2的分支进行原理分析,虽然V2最近一次提交已经是两年前了,但是架不住公司里面的项目用的是v2分支,这里主要分析EventBus实现的基本原理,根据EventBus [HOWTO.md](https://github.com/greenrobot/EventBus/blob/v2/HOWTO.md)这篇文章的教程的思路来一个个点的分析,建议阅读本文之前先看完**HOWTO.md**,这样对于EventBus的使用有一个基本的了解,方式理解本文

## 订阅者如何注册

一般的注册方式如下或者说入口如下

```
EventBus.getDefault().register(this);//其中this为订阅者

private synchronized void register(Object subscriber, boolean sticky, int priority) {
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriber.getClass());
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod, sticky, priority);
        }
}
```

这里将注册过程分成了两步,由名字来看首先是寻找订阅者中的方法、然后完成继续完成订阅的步骤.先来看第一步方法寻找

这里一共有两个问题:

+ 这一步是在寻找什么方法
+ 如何寻找

如果看过HOW-TO.md中的教程的话,应该能够回答出上面的问题,由于EventBus V2是不同Button点击的那种监听器的回调,EventBus的事件订阅是基于约定来实现的,即所有订阅者回调方法名师固定的,必须叫onEvent、onEventMainThread等.所以这一步是在寻找订阅者中的onEvent系列方法,寻找方式也就很容易想到,我们一般获取类的方法信息的方式**反射**

```
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods;
        //首先查找方法是否存在缓存,key为订阅类
        synchronized (methodCache) {
            subscriberMethods = methodCache.get(subscriberClass);
        }
        if (subscriberMethods != null) {
            return subscriberMethods;//立即返回
        }
        subscriberMethods = new ArrayList<SubscriberMethod>();
        Class<?> clazz = subscriberClass;
        HashMap<String, Class> eventTypesFound = new HashMap<String, Class>();
        StringBuilder methodKeyBuilder = new StringBuilder();
        while (clazz != null) {//直到Object类
            String name = clazz.getName();
            //略...
            // Starting with EventBus 2.2 we enforced methods to be public (might change with annotations again)
            try {
                // This is faster than getMethods, especially when subscribers a fat classes like Activities
                Method[] methods = clazz.getDeclaredMethods();
                filterSubscriberMethods(subscriberMethods, eventTypesFound, methodKeyBuilder, methods);
            } catch (Throwable th) {
                //略... java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            }
            clazz = clazz.getSuperclass();//继续沿继承链向父类查找onEvent系列回调方法
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass + " has no public methods called " + ON_EVENT_METHOD_NAME);
        } else {
            synchronized (methodCache) {
                methodCache.put(subscriberClass, subscriberMethods);
            }
            return subscriberMethods;
        }
    }
```

代码显而易见是基于反射的查找,其中filterSubscriberMethods方法主要是根据方法名、方法签名精准查找的过程,同时还会生成一个Map映射关系,**key为onEvent回调方法名+订阅事件的类名,值为订阅者的类名**,有兴趣的可以进一步分析该方法的实现,但是需要注意一个细节,由于当两个具有继承关系的类同时都订阅事件时Map映射关系的处理.

接着看第二步如何完成订阅过程


```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) {
    Class<?> eventType = subscriberMethod.eventType;//eventType是指onEvent回调中方的Event参数的Class Type
    //根据事件类型获取所有的订阅者
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod, priority);
    //首先将eventType的新的订阅者添加到eventType的订阅者队列中
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<Subscription>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }
    //根据订阅者的优先级将订阅者插入到特定位置
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || newSubscription.priority > subscriptions.get(i).priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);//获取订阅者订阅的所有事件
    //将事件添加到订阅者的订阅事件队列中
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<Class<?>>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    //sticky event在后面再继续看
    if (sticky) {
        if (eventInheritance) {
            //默认为false,不进入这个方法,略
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```

ok,到这里应该就比较清楚了,第二步就是在订阅者和订阅事件之间建立关联

## Publisher如何post事件

```
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();//
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (!postingState.isPosting) {//最初值为false,标志当前是否在publish event
        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
        postingState.isPosting = true;//标记为设为true
        if (postingState.canceled) {//标记当前是否仍处于是否取消事件发布状态中
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                //这里就是根据eventType从eventType中的Map中取出订阅者的队列,然后按照优先级来依次执行订阅者的onEvent方法
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

postSingleEvent方法的执行细节不过多展开,只开最后一步EventBus线程的调度,也就是对于onEvent、onEventMainThread、onEventBackgroundThread等的处理,如下

```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case PostThread://onEvent直接在当前调用线程执行
            invokeSubscriber(subscription, event);//method.invoke
            break;
        case MainThread://onEventMainThread
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);//利用Handler回调
            }
            break;
        case BackgroundThread:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case Async:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

## Sticky Event如何publish

由于Sticky Event是先发布后订阅,所有先看StickyEvent的发布过程

```
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    // Should be posted after it is putted, in case the subscriber wants to remove immediately
    post(event);
    }
```

过程很简单,先把sticky事件存储起来、然后将其和普通event一样publish一遍,当然我觉得后面的post是没有必要的

## Sticky Event订阅者如何注册

和普通Event订阅者注册逻辑基本上是一样的除了上面遗留的最后一步

```
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) {
    //略...
    if (sticky) {
        if (eventInheritance) {
            //默认为false,不进入这个方法,略
        } else {
            Object stickyEvent = stickyEvents.get(eventType);//由于已经post过了,此处便能获取之前publish的事件
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}

private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            //调用订阅者方法,注意这里只调用了一个,而非普通事件的总线形式
            postToSubscription(newSubscription, stickyEvent, Looper.getMainLooper() == Looper.myLooper());
        }
    }
```

## 原理总结

虽然仿佛并没有什么原理需要总结,但是还是写一下-EventBus的工作方式如文档所示的,它是基于约定的一种通过反射实现的的 publish/subscribe 模式.