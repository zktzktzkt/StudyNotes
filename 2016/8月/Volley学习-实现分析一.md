# Volley学习-实现分析一
---

## 一、要弄清的问题

+ RequestQueue的工作原理
+ 网络请求如何分发到各个线程
+ 网络请求是如何返回结果调度到主线程中
+ Volley的缓存实现
+ ImageLoader的实现原理

## 二、代码分析

从基本的使用代码demo开始看起

```

	RequestQueue queue=Volley.newRequestQueue(this);
	StringRequest request=new StringRequest(Request.Method.GET, url,
        new Response.Listener<String>() {
            @Override
       	    public void onResponse(String s) {
                mTvText.setText(s);
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError volleyError) {
                Log.e("tag","error="+volleyError.toString());
            }
    });
    queue.add(request);

```

RequestQueue相关分析

经过一系列的静态方法的调用，最终调用的创建代码如下

```
	//context用于创建cache目录，stack用于网络相关默认为空，maxDiskCacheBytes磁盘缓存大小，-1表默认大小
	public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes) {
		File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);//缓存目录的创建
		String userAgent = "volley/0";
		try {
		    String packageName = context.getPackageName();
		    PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
		    userAgent = packageName + "/" + info.versionCode;
		    } catch (NameNotFoundException e) {
		    }
		    if (stack == null) {
		        if (Build.VERSION.SDK_INT >= 9) {
		            stack = new HurlStack();
		        } else {
		            //低于SDK9，HTTPUrlConnection靠不住
		            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
		        }
		    }
		    Network network = new BasicNetwork(stack);
		    RequestQueue queue;
		    if (maxDiskCacheBytes <= -1){//根据不同的maxDiskCacheBytes创建RequestQueue
		        queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
		    }
		    else{
		        queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
		    }
		    queue.start();
		    return queue;
		}

```

由上面的代码看来创建一个RequestQueue是由磁盘缓存、网络这两个参数来创建，接着来看
		
```

	public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        mCache = cache;//磁盘缓存
        mNetwork = network;//网络请求
        mDispatchers = new NetworkDispatcher[threadPoolSize];//网络分发
        mDelivery = delivery;//完成回调responses、errors，new ExecutorDelivery(new Handler(Looper.getMainLooper()))
    }

```

RquestQueue还有几个重要的变量为

+ mSequenceGenerator = new AtomicInteger();//为请求生成单调增加的序号
+ mWaitingRequests = new HashMap();//暂存正在请求的重复请求
+ mCurrentRequests = new HashSet();//正在被dispatcher处理的请求的集合或者正在等待RequestQueue的请求
+ mCacheQueue = new PriorityBlockingQueue();//缓存分流队列
+ mNetworkQueue = new PriorityBlockingQueue();//已经完成的请求的集合
+ mFinishedListeners = new ArrayList();//完成请求的回掉

再看RqeuestQueue是如何开始工作的，由start方法开始

```
	//该方法的注释为开始这个队列的调度，由代码也可以很清楚的看到这个操作
    public void start() {
        stop();  // 确保网络调度和缓存调度被关闭
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);//请求队列、网络、缓存、回调
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }

```

可见事件的分发、传递本身不在RequestQueue中进行，先放开这个不看，由demo看它的add、cancel方法的实现

```

	public <T> Request<T> add(Request<T> request) {
        request.setRequestQueue(this);//绑定Queue
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }
        ....
        if (!request.shouldCache()) {//该请求不需要缓存，直接进入网络请求队列等待处理
            mNetworkQueue.add(request);
            return request;
        }

        // 如果正在请求的与之重复则插入到暂存队列中，即重复请求只发一个
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey)) {
                Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {//没有重复则加入到缓存队列和暂存队列中
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }

    //这个方法比较简单就是从等待队列中一个个对比tag,进行删除
    public void cancelAll(RequestFilter filter) {
        synchronized (mCurrentRequests) {
            for (Request<?> request : mCurrentRequests) {
                if (filter.apply(request)) {
                    request.cancel();
                }
            }
        }
    }

```
