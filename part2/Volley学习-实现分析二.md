# Volley学习-实现分析二
---

接一继续分析，从一中可以看到RequestQueue只是负责初始化各种队列、将Rquest加进队列中，真正的网络请求、线程通信在其他地方实现，这里也体现出了消息队列对于解耦的作用，先从RquestQueue的start方法中的两个start的两个东西看是看起,它们类是这样的

```
//原来都是线程！！！
public class CacheDispatcher extends Thread
public class NetworkDispatcher extends Thread

```

先看NetworkDispatcher的构造方法

```
	//绑定队列、绑定网络、绑定缓存类、绑定回调
	public NetworkDispatcher(BlockingQueue<Request<?>> queue,Network network, Cache cache,ResponseDelivery delivery) {
        mQueue = queue;
        mNetwork = network;
        mCache = cache;
        mDelivery = delivery;
    }

```

再看它的run方法(部分),

```
	@Override
    public void run() {
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            Request<?> request;
            ...
            request = mQueue.take();//取一个Rquest，阻塞队列，可以防止多线程的混乱
            ...//退出线程的
            try {
                request.addMarker("network-queue-take");
                if (request.isCanceled()) {//判断是否取消
                    request.finish("network-discard-cancelled");
                    continue;
                }
                addTrafficStatsTag(request);

                // 执行请求
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // 304则直接结束，不再请求
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // 解析响应
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");
                //缓存数据
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // 进行回调
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }

```

接下来先留着回调这个问题不看，先看CacheDispatcher的实现,先看构造方法

```
	//绑定缓存分流队列、网络请求队列、磁盘缓存的实现、回调相关类
	public CacheDispatcher(BlockingQueue<Request<?>> cacheQueue, BlockingQueue<Request<?>> networkQueue,Cache cache, ResponseDelivery delivery) {
        mCacheQueue = cacheQueue;
        mNetworkQueue = networkQueue;
        mCache = cache;
        mDelivery = delivery;
    }

```

接下来是run方法的部分

```
	@Override
    public void run() {
        mCache.initialize();//初始化缓存，阻塞方法
        while (true) {
            try {
                final Request<?> request = mCacheQueue.take();//获取一个缓存分流队列的元素，一直阻塞到返回一个元素
                request.addMarker("cache-queue-take");
                if (request.isCanceled()) {//取消则直接结束
                    request.finish("cache-discard-canceled");
                    continue;
                }
                Cache.Entry entry = mCache.get(request.getCacheKey());//取缓存数据
                if (entry == null) {//空则加入到网络请求队列，可以看到NetworkDispatcher中队列的数据来自这里
                    request.addMarker("cache-miss");
                    mNetworkQueue.put(request);
                    continue;
                }
                if (entry.isExpired()) {//缓存过期同样也是加入到请求队列中
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }
                request.addMarker("cache-hit");//缓存命中，直接解析数据
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");

                if (!entry.refreshNeeded()) {//命中且不需要刷新，直接回掉
                    mDelivery.postResponse(request, response);
                } else {命中且需要刷新，将response回调给用户，同时加入网络请求
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);
                    response.intermediate = true;
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }

```

## 阶段性总结
从这两篇来看，RequestQueue本身作用仅在于启动了一条缓存相关的线程、数条网络请求相关的线程、将请求加入队列，感觉叫RequestManager更合适，同时从NetworkDispatcher、CacheDispatcher的run方法的实现来看，前者是取请求、发请求回调，后者是取请求、检查缓存、回调，那么原理图中的缓存命中直接回调实在哪里实现的呢？关键在于RquestQueue中的PriorityBlockingQueue<Request<?>> mNetworkQueue变量。

通过上面两篇的分析基本可以分析出开头要弄清的前两个问题

**RequestQueue的工作原理：**

其实RequestQueue除了start方法之外，最重要的逻辑在add方法中，由于add方法向请求队列中发送了请求,缓存线程、网络请求线程才能够运转，否则就是一直阻塞。看一中的分析我们可以看到在add方法中一共有三个成员变量存在mCurrentRequests、mWaitingRequests、mNetworkQueue、mCacheQueue，其中最后一个是CacheDispatcher的队列，所以请求最开始是被送往这里(参见volley的原理图，先查看缓存再请求)，如果该请求不允许缓存则是直接发送到网络请求队列。而前面两个队列从add的代码中可以看出来，**如果mWaitRequests队列中存在add传进来的请求，则它不会被送入mCacheQueue或者mNetworkQueue中，而是被直接加入到mWaitingRequests中的一个列表集合中，也就是说这个请求实际上不会进行网络请求!!!**那么它们被存到List中干嘛呢？我们可以从RquestQueue中的finish方法中知道，其部分代码如下

```

	//该函数在Request请求完成后被回调
	<T> void finish(Request<T> request) {
        synchronized (mCurrentRequests) {//移除请求结束的请求
            mCurrentRequests.remove(request);
        }
        synchronized (mFinishedListeners) {
          for (RequestFinishedListener<T> listener : mFinishedListeners) {
            listener.onRequestFinished(request);
          }
        }
        if (request.shouldCache()) {
            synchronized (mWaitingRequests) {
                String cacheKey = request.getCacheKey();
                Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);
                if (waitingRequests != null) {
                    // 看以看到这里请求又被加入到mCacheQueue中了，但是由于CacheDispatcher会根据缓存来决定是否送入网络请求
                    //所以一般来说缓存应该还没有失效，这些请求将会直接回调到主线程进行响应
                    mCacheQueue.addAll(waitingRequests);
                }
            }
        }
    }
```

**网络请求如何分发到各个线程**

由上面RequestQueue的分析可以看到，RequestQueue会根据是否需要缓存决定将请求送入缓存队列或者网络请求队列(亦或者进行请求合并),那么请求到了网络请求队列自然不必多说它肯定会被处理并且回调回主线程，那么进入缓存的队列的请求是如何进入网络请求队列队列的呢？由CacheQueue中的run方法可以看到该缓存线程根据缓存是否命中来决定是进行结束请求回调、还是送入请求队列，如果缓存未命中、缓存命中但是已失效都会将请求送入网络请求队列，**一个特殊的情况是缓存命中且未过期但是即将过期，则这个请求同样会结束并且回调，但是话增加一个附加任务再一次将该请求送入网络请求队列中排队**