# Picasso的使用与实现分析二
---

接上文创建完Request和key后会检索缓存是否存在这个已经讨论过，然后接着就是Request实质是如何利用网络链接来下载图片的过程

```
	//接上文创建完Request和key后没有缓存的代码如下
	if (setPlaceholder) {//首先设置占位图片
      setPlaceholder(target, getPlaceholderDrawable());
    }

    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);//由目标、缓存策略、请求key等创建一个action

    picasso.enqueueAndSubmit(action);//提交一个Action

    //提交之前首先检查是否已经有请求，有则先取消再加入,注意这些方法目前仍在主线程中
    void enqueueAndSubmit(Action action) {
    	Object target = action.getTarget();
    	if (target != null && targetToAction.get(target) != action) {
      		cancelExistingRequest(target);
      		targetToAction.put(target, action);
    	}
    	submit(action);
  	}
  	//最终请求由一个disptacher来处理
  	void submit(Action action) {
    	dispatcher.dispatchSubmit(action);
  	}
  	//通过搜索可以发现submit中的dispatcher的创建信息如下
  	Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);

```

对于Dispatcher的创建中各个参数的创建信息如下

* service:service = new PicassoExecutorService();
* HANDLER:static final Handler HANDLER = new Handler(Looper.getMainLooper()) {//...};可以知道该Handler是利用主线程的Looper来做的
* downloader:downloader = Utils.createDefaultDownloader(context);//Downloader
* cache:cache = new LruCache(context);
* stats：Stats stats = new Stats(cache);

接着再看dispatcher.dispatchSubmit(action);方法的处理

```
	//仅仅是发送了一条Msg
	void dispatchSubmit(Action action) {
    	handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  	}

```

显然我们需要找到这条消息的处理者，所以我们先找上面发送消息的Handler的创建

```

	//发送信息的Handler的创建如下
	this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
	//对应的Handler类如下
	private static class DispatcherHandler extends Handler {
    private final Dispatcher dispatcher;

    public DispatcherHandler(Looper looper, Dispatcher dispatcher) {
      super(looper);
      this.dispatcher = dispatcher;
    }
    //下面是一系列的消息处理以及dispatcher类的perform系列方法的调用，我们只看提交任务的处理
    @Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        case REQUEST_SUBMIT: {//调价任务的消息处理如下
          Action action = (Action) msg.obj;
          dispatcher.performSubmit(action);
          break;
        }
        //其他的处理...
      }
    }
  }

```
可以看出来过程中利用了Handler的不同消息来调用不同的perform函数，但是仍然没有发起网络请求来下载图像,所以再接着看对于REQUEST_SUBMIT消息的处理

```

void performSubmit(Action action, boolean dismissFailed) {
    if (pausedTags.contains(action.getTag())) {//首先检查请求是否已经暂停
      pausedActions.put(action.getTarget(), action);
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),
            "because tag '" + action.getTag() + "' is paused");
      }
      return;
    }

    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {//由下面提交请求的代码来看，一个请求与一个BitmapHUnter关联
      hunter.attach(action);//如果之前存在请求则直接关联之前的BitmapHunter
      return;
    }

    if (service.isShutdown()) {//如果线程池已经关闭则不再加载
      if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
      }
      return;
    }
    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    hunter.future = service.submit(hunter);//提交了一个请求
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {
      failedActions.remove(action.getTarget());
    }

    if (action.getPicasso().loggingEnabled) {
      log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
    }
}

```
上面的代码我们终于看到了任务提交到线程池中但是实际上我们还是没有看到下载的过程，在这之前首先看一下forRequest的创建

```
//在这里我们看到了Picasso创建时出啊关键的几个Handler被用在了BitmapHunter的创建上
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats,
      Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();
    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
      RequestHandler requestHandler = requestHandlers.get(i);
      if (requestHandler.canHandleRequest(request)) {//根据请求的资源的类型来选择Handler的处理对象
        return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
      }
    }

    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
  }

```

最后我们看hunter.future = service.submit(hunter);发生了什么

```
	//将BitmapHunter任务交与线程池执行
  public Future<?> submit(Runnable task) {
    PicassoFutureTask ftask = new PicassoFutureTask((BitmapHunter) task);
    execute(ftask);
    return ftask;
  }

```

由上面对于任务的处理我们可以断定BitmapHunter实现了Runnable接口,图片的下载逻辑应该也就在其run方法中

```

public void run() {
    try {
      //.....
      result = hunt();//hunt:打猎 这里开启下载逻辑

      if (result == null) {
        dispatcher.dispatchFailed(this);
      } else {
        dispatcher.dispatchComplete(this);//下载成功的回调
      }
    } catch (Downloader.ResponseException e) {
      if (!e.localCacheOnly || e.responseCode != 504) {
        exception = e;
      }
      //各种异常以及处理...
    } finally {
      Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
  }

  //下载图片的处理
  Bitmap hunt() throws IOException {
    Bitmap bitmap = null;
    if (shouldReadFromMemoryCache(memoryPolicy)) {
      //仍然是检查缓存，命中则stats更新统计数、返回图片
    }
    //网络规则的确定
    data.networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;

    //***这里的RequestHandler即Picasso构造方法创建的Handler***
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
      loadedFrom = result.getLoadedFrom();//获得数据来源
      exifRotation = result.getExifOrientation();

      bitmap = result.getBitmap();
      if (bitmap == null) {//图片没有创建则先从流中创建
        InputStream is = result.getStream();
        try {
          bitmap = decodeStream(is, data);
        } finally {
          Utils.closeQuietly(is);
        }
      }
    }

    if (bitmap != null) {
      if (picasso.loggingEnabled) {
        log(OWNER_HUNTER, VERB_DECODED, data.logId());
      }
      stats.dispatchBitmapDecoded(bitmap);//数据统计
      if (data.needsTransformation() || exifRotation != 0) {
        synchronized (DECODE_LOCK) {
          if (data.needsMatrixTransform() || exifRotation != 0) {
          	//实施图片的变换，比如尺寸的自适应等等
            bitmap = transformResult(data, bitmap, exifRotation);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
            }
          }
          //如果有自定义的变换则开始实施
          if (data.hasCustomTransformations()) {
            bitmap = applyCustomTransformations(data.transformations, bitmap);
            if (picasso.loggingEnabled) {
              log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
            }
          }
        }
        if (bitmap != null) {
          stats.dispatchBitmapTransformed(bitmap);//统计数据
        }
      }
    }

    return bitmap;//返回结果
  }

```

通过上面的分析我们在这里就可以了解之前Picasso构造方法中创建的各个Hnadler的用处,原来Picasoo对对应的资源类型使用对应的Handler来加载数据，即Handler是加载数据的逻辑的管理者，以从网络加载数据的Handler的load方法为例

```

@Override public Result load(Request request, int networkPolicy) throws IOException {
    Response response = downloader.load(request.uri, request.networkPolicy);//使用download下载图像
    if (response == null) {
      return null;
    }

    Picasso.LoadedFrom loadedFrom = response.cached ? DISK : NETWORK;//确定数据来源

    Bitmap bitmap = response.getBitmap();
    if (bitmap != null) {
      return new Result(bitmap, loadedFrom);
    }

    InputStream is = response.getInputStream();
    if (is == null) {
      return null;
    }
    ///这里注释解释到是个bug，当请求被重试时有时会出现响应结果为0的情况
    if (loadedFrom == DISK && response.getContentLength() == 0) {
      Utils.closeQuietly(is);
      throw new ContentLengthException("Received response with 0 content-length header.");
    }
    if (loadedFrom == NETWORK && response.getContentLength() > 0) {
      stats.dispatchDownloadFinished(response.getContentLength());//数据统计
    }
    return new Result(is, loadedFrom);//返回数据尚未解析数据流
  }

```
接着再看downloader是如何下载图片的，我们从之前的分析知道downloader的创建是使用Utils类来创建的,如下

```
//可以看到它的默认优先是使用okhttp来下载
static Downloader createDefaultDownloader(Context context) {
    if (SDK_INT >= GINGERBREAD) {
      try {
        Class.forName("okhttp3.OkHttpClient");
        return OkHttp3DownloaderCreator.create(context);
      } catch (ClassNotFoundException ignored) {
      }
      try {
        Class.forName("com.squareup.okhttp.OkHttpClient");
        return OkHttpDownloaderCreator.create(context);
      } catch (ClassNotFoundException ignored) {
      }
    }
    return new UrlConnectionDownloader(context);
}

//默认的UrlConnectionDownloader下载器使用HTTPURLConnection来下载图片，同时我们也可以看到Picasso的磁盘
//缓存逻辑-利用HTTP自身的缓存逻辑，我们通过查看Picasso的类可以了解到Picasso没有与磁盘缓存相关的实现类
@Override public Response load(@NonNull Uri uri, int networkPolicy) throws IOException {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
      installCacheIfNeeded(context);
    }

    HttpURLConnection connection = openConnection(uri);
    connection.setUseCaches(true);

    if (networkPolicy != 0) {
      String headerValue;

      if (NetworkPolicy.isOfflineOnly(networkPolicy)) {
        headerValue = FORCE_CACHE;
      } else {
        StringBuilder builder = CACHE_HEADER_BUILDER.get();
        builder.setLength(0);

        if (!NetworkPolicy.shouldReadFromDiskCache(networkPolicy)) {
          builder.append("no-cache");
        }
        if (!NetworkPolicy.shouldWriteToDiskCache(networkPolicy)) {
          if (builder.length() > 0) {
            builder.append(',');
          }
          builder.append("no-store");
        }

        headerValue = builder.toString();
      }

      connection.setRequestProperty("Cache-Control", headerValue);
    }

    int responseCode = connection.getResponseCode();
    if (responseCode >= 300) {
      connection.disconnect();
      throw new ResponseException(responseCode + " " + connection.getResponseMessage(),
          networkPolicy, responseCode);
    }

    long contentLength = connection.getHeaderFieldInt("Content-Length", -1);
    boolean fromCache = parseResponseSourceHeader(connection.getHeaderField(RESPONSE_SOURCE));

    return new Response(connection.getInputStream(), fromCache, contentLength);
  }

```

至此一张图片的加载逻辑已经完成，接着就是再回到run方法中分析图片是如何被回调到主线程并设置到ImageView中

```

dispatcher.dispatchComplete(this);

//DispatcherHandler发送完成消息
void dispatchComplete(BitmapHunter hunter) {
    handler.sendMessage(handler.obtainMessage(HUNTER_COMPLETE, hunter));
}

//目标消息的处理
case HUNTER_COMPLETE: {
    BitmapHunter hunter = (BitmapHunter) msg.obj;
    dispatcher.performComplete(hunter);
    break;
}
void performComplete(BitmapHunter hunter) {
    if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {//是否需要写入缓存
      cache.set(hunter.getKey(), hunter.getResult());//存缓存
    }
    hunterMap.remove(hunter.getKey());//请求关联解除
    batch(hunter);
  }
  //批量处理已完成的BitmapHunter
  private void batch(BitmapHunter hunter) {
    if (hunter.isCancelled()) {
      return;
    }
    batch.add(hunter);//添加到集合中
    if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
      handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    }
  }
  //HUNTER_DELAY_NEXT_BATCH消息的处理如下
  void performBatchComplete() {
    List<BitmapHunter> copy = new ArrayList<BitmapHunter>(batch);
    batch.clear();
    mainThreadHandler.sendMessage(mainThreadHandler.obtainMessage(HUNTER_BATCH_COMPLETE, copy));
    logBatch(copy);
  }
  //HUNTER_BATCH_COMPLETE消息处理如下，其是由在Picasso中的创建的Handler(mainLooper)处理
  case HUNTER_BATCH_COMPLETE: {
          @SuppressWarnings("unchecked") List<BitmapHunter> batch = (List<BitmapHunter>) msg.obj;
          //noinspection ForLoopReplaceableByForEach
          for (int i = 0, n = batch.size(); i < n; i++) {
            BitmapHunter hunter = batch.get(i);
            hunter.picasso.complete(hunter);//对于每个请求回调
          }
          break;
        }
//回调方法如下
void complete(BitmapHunter hunter) {
    Action single = hunter.getAction();
    List<Action> joined = hunter.getActions();
    boolean hasMultiple = joined != null && !joined.isEmpty();
    boolean shouldDeliver = single != null || hasMultiple;
    if (!shouldDeliver) {
      return;
    }
    Uri uri = hunter.getData().uri;
    Exception exception = hunter.getException();
    Bitmap result = hunter.getResult();//获取图片及加载来源的信息调试使用
    LoadedFrom from = hunter.getLoadedFrom();

    if (single != null) {
      deliverAction(result, from, single);//发送Action设置图片到目标ImageView
    }

    if (hasMultiple) {//当前的BitmapHunter绑定了多个Action
      for (int i = 0, n = joined.size(); i < n; i++) {
        Action join = joined.get(i);
        deliverAction(result, from, join);
      }
    }

    if (listener != null && exception != null) {
      listener.onImageLoadFailed(this, uri, exception);
    }
  }
  //对于Action的检查
  private void deliverAction(Bitmap result, LoadedFrom from, Action action) {
    if (action.isCancelled()) {//是否已取消
      return;
    }
    if (!action.willReplay()) {
      targetToAction.remove(action.getTarget());
    }
    if (result != null) {
      if (from == null) {
        throw new AssertionError("LoadedFrom cannot be null.");
      }
      action.complete(result, from);//设置图片,对应的Action实例为ImageViewAction
  	}
  }
  //complete实现如下
  public void complete(Bitmap result, Picasso.LoadedFrom from) {
    //...
    ImageView target = this.target.get();
    if (target == null) {
      return;
    }
    Context context = picasso.context;
    boolean indicatorsEnabled = picasso.indicatorsEnabled;
    //设置图片到目标上，PicassoDrawable是一个封装设置图片的逻辑
    PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);
    if (callback != null) {
      callback.onSuccess();
    }
  }

```
