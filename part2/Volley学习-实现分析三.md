#Volley学习-实现分析三#
---
接上文接着看后面的三个问题

##1、网络请求是如何返回结果调度到主线程中##

可想而知肯定在CacheDispatcher、NetworkDispatcher中找，从注释中可以看到它们请求回调的代码分别为一下

```
	//CacheDispatcher
	if (!entry.refreshNeeded()) {
        mDelivery.postResponse(request, response);
    } else {
    mDelivery.postResponse(request, response, new Runnable() {
        @Override
        public void run() {
       
            mNetworkQueue.put(request);
           
            }
        });
    }

    //NetworkDispatcher
    mDelivery.postResponse(request, response);

    //请求被取消等不需要回调的情况则直接调用finish方法
    request.finish("not-modified");
```
从上面的代码可以看出来mDelivery这个参数扮演了一个重要的角色，它将请求及其结果甚至还有额外的runnable回调到某个地方。那么它是从哪里传进来的，又是在哪里创建的？回头看RequestQueue的start方法，它在创建CacheDispatcher、NetworkDispatcher时将其作为构造函数的参数传入，而在RequestQueue的构造方法中mDelivery是这么创建的
```
	//看出来了吗？它获取了主线程的Looper，并创建了Handler,要知道主线程的消息驱动也是通过该Looper
	mDelivery=new ExecutorDelivery(new Handler(Looper.getMainLooper()))

	//ExecutorDelivery的构造方法是这样的
	public ExecutorDelivery(final Handler handler) {
        // 仅仅让Executor包裹了一下handler
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }

    //而其parseResponse方法最终会到这里
    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    //由上面的代码看到她给Handler post了一个这样的runnable，而这个handler的消息处理正是在主线程！！！
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        //代码精简后如下，最后是Request执行相关的deliver方法，在执行额外的runnable
        public void run() {
            ....
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");//最终也调用了finish
            }
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }
    //deliverResponse方法在Request类中是一个抽象方法，而StringRequest中是这么实现的
    @Override
    protected void deliverResponse(String response) {
        mListener.onResponse(response);/mListener是创建请求时的成功回调
    }

    //此外请求取消时执行的finish方法是这样的,看出来是调用了RequestQueue的finish方法，其实也就是
    //为了将对象从RequestQueue的队列中清除

    void finish(final String tag) {
        if (mRequestQueue != null) {
            mRequestQueue.finish(this);
        }
        ...
    }
```

由上面的一些列分析可以看到最终回调到主线程是利用了主线程的Looper来创建的Handler post一个Runnable来做的，这样就是在主线程中回调了创建Request的监听

##2、缓存的实现##

由前面的一些分析可以看到缓存的额逻辑主要在CacheDispatcher中，而且只是用了磁盘缓存没有使用内存缓存，在这里这种看Volley的磁盘缓存实现类DiskBaseCache的实现，而且只关注put、get方法

还是先由其构造函数来看

```
	//很简单，传入了缓存目录及缓存最大值
	public DiskBasedCache(File rootDirectory, int maxCacheSizeInBytes) {
        mRootDirectory = rootDirectory;
        mMaxCacheSizeInBytes = maxCacheSizeInBytes;
    }

```

再看get和put方法

```

	//get方法的逻辑很简单，没有则返回null，否则直接从磁盘中读数据返回
    public synchronized Entry get(String key) {
        CacheHeader entry = mEntries.get(key);
        // if the entry does not exist, return.
        if (entry == null) {
            return null;
        }
        File file = getFileForKey(key);//根据key返回文件
        CountingInputStream cis = null;
        try {
            cis = new CountingInputStream(new BufferedInputStream(new FileInputStream(file)));
            CacheHeader.readHeader(cis); 
            byte[] data = streamToBytes(cis, (int) (file.length() - cis.bytesRead));
            return entry.toCacheEntry(data);
        } catch (IOException e) {
            VolleyLog.d("%s: %s", file.getAbsolutePath(), e.toString());
            remove(key);
            return null;
        } 
        ...
    }

    //逻辑也比较简单首先调用pruneIfNeeded，然后写入数据到磁盘，存入相关头字段等数据
    //而不是实体数据
    @Override
    public synchronized void put(String key, Entry entry) {
        pruneIfNeeded(entry.data.length);//
        File file = getFileForKey(key);
        try {
            BufferedOutputStream fos = new BufferedOutputStream(new FileOutputStream(file));
            CacheHeader e = new CacheHeader(key, entry);
            boolean success = e.writeHeader(fos);
            ....
            fos.write(entry.data);
            fos.close();
            putEntry(key, e);
            return;
        } catch (IOException e) {
        }
        boolean deleted = file.delete();
        if (!deleted) {
            VolleyLog.d("Could not clean up file %s", file.getAbsolutePath());
        }
    }

	//看一下pruneIfNeeded的逻辑，原来是根据缓存的大小来判断当前缓存是否满了
	//如果满了，则要删除部分数据
	private void pruneIfNeeded(int neededSpace) {
        if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes) {
            return;
        }

        long before = mTotalSize;
        int prunedFiles = 0;

        Iterator<Map.Entry<String, CacheHeader>> iterator = mEntries.entrySet().iterator();
        while (iterator.hasNext()) {//枚举已存的部分数据
            Map.Entry<String, CacheHeader> entry = iterator.next();
            CacheHeader e = entry.getValue();
            boolean deleted = getFileForKey(e.key).delete();
            if (deleted) {
                mTotalSize -= e.size;
            } else {
            	........
            }
            iterator.remove();
            prunedFiles++;
            //如果剩余的空间大于10%则不再删除，删除的逻辑相当的简单
            if ((mTotalSize + neededSpace) < mMaxCacheSizeInBytes * HYSTERESIS_FACTOR) {
                break;
            }
        }
    }

```

到这里可以看见缓存的实现没有内存缓存，只是在内存中存了用于组成文件名的数据，返回的数据实体存在磁盘中，如果不够则删除部分，其次值得注意的是volley没有对磁盘满的情况进行处理，只是简单的和最大缓存值对比，可见如果磁盘空间不够则会抛出异常

##3、ImageLoader的实现原理##

由文档看来这个一个用来请求图像的帮助类，由基本使用和创建来看

```


	//构造方法
	public ImageLoader(RequestQueue queue, ImageCache imageCache) {
        mRequestQueue = queue;
        mCache = imageCache;
    }
    //一个利用LruCache来实现的mCache
    mImageLoader = new ImageLoader(mRequestQueue,
                new ImageLoader.ImageCache() {
            private final LruCache<String, Bitmap>
                    cache = new LruCache<String, Bitmap>(20);

            @Override
            public Bitmap getBitmap(String url) {
                return cache.get(url);
            }

            @Override
            public void putBitmap(String url, Bitmap bitmap) {
                cache.put(url, bitmap);
            }
        });
	//使用
	mImageLoader.get(IMAGE_URL, ImageLoader.getImageListener(mImageView,R.drawable.def_image, R.drawable.err_image));
	//get方法最终调用了这个方法，目前仍在mainThread
	public ImageContainer get(String requestUrl, ImageListener imageListener,
            int maxWidth, int maxHeight, ScaleType scaleType) {

        ......
        //getCacheKey是从LruCache来检索缓存
        final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight, scaleType);
        Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
        if (cachedBitmap != null) {//缓存存在直接回调
            ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
            imageListener.onResponse(container, true);
            return container;
        }

        //没有缓存，创建一个辅助类
        ImageContainer imageContainer =new ImageContainer(null, requestUrl, cacheKey, imageListener);
        //先回调接口，先暂时使用默认图像
        imageListener.onResponse(imageContainer, true);

        // 原理同RequestQueue的请求合并
        BatchedImageRequest request = mInFlightRequests.get(cacheKey);
        if (request != null) {//如果存在，保存回调接口的引用保证能够回调
            request.addContainer(imageContainer);
            return imageContainer;
        }

        //包装成ImageRequest投入到RequestQueue队列中
        Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType,
                cacheKey);
        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey,
                new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;
    }

    //如果没有缓存，是第一次请求，则通过这个方法来创建一个ImageRequest来复用一般的请求方法
    protected Request<Bitmap> makeImageRequest(String requestUrl, int maxWidth, int maxHeight,
            ScaleType scaleType, final String cacheKey) {
        return new ImageRequest(requestUrl, new Listener<Bitmap>() {
            @Override
            public void onResponse(Bitmap response) {
                onGetImageSuccess(cacheKey, response);
            }
        }, maxWidth, maxHeight, scaleType, Config.RGB_565, new ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
                onGetImageError(cacheKey, error);
            }
        });
    }
    
```

从上面的简略分析来看ImageLoader首先从LruCache中检索缓存是否存在，存在则直接返回，否则将该请求作为以作为一个新的ImageRequest来进入到一般的请求流程。同时还有一个点就是它会做一个请求的合并，防止浪费，但是响应还是会重复响应。总之ImageLoader一个重要的功能是在Volley的磁盘缓存上增加了一个简单的内存缓存。


