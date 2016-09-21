#Picasso的使用与实现分析三#
---

接上篇继续，至此我们已经走完了一遍加载的流程，了解了Picasso的图片加载逻辑，最后对于前面的分析做一个总结

##四、加载逻辑的总结##

* 主要类的作用说明:
	* Picasso类：主要是用于初始化、请求的消息的发送、回调结果的处理,即提供了请求发送的接口以及请求回调的接口，属于一个管理类
	* RequestCreator、Request、Action：这三个类是图片请求的抽象，它们之前对图片请求的描述程度是一个递进关系,RequestCreator更多的是一个**请求的创建类**它包含一些请求的次要元素-占位图片、错误图片的信息、缓存策略、网络策略等，接着Request则是一个真正的请求的抽象，包含了一个请求需要的资源位置、实施的变换等绝大部分信息，从下面的构造函数也可以看出来

	```

	return new Request(uri, resourceId, stableKey, transformations, targetWidth, targetHeight,
          centerCrop, centerInside, onlyScaleDown, rotationDegrees, rotationPivotX, rotationPivotY,
          hasRotationPivot, config, priority);

	```

	而最后的Action则进一步描述了请求的需要的信息包括缓存策略、网络策略等，同样还是从构造方法来看

	```

	Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);

	```

	所以它们之间对于请求的描述是一个递进的关系
	* Target接口：请求加载事件过程的监听
	* BitmapHunter：该类是下载任务的核心管理类，也是Picasso线程池的实际的任务封装了类(实现了Runnable接口)，它封装了任务下载、解析图像流、任务回送UI的逻辑
	* RequestHandler类：图片下载任务的处理类，该类是图片加载任务处理过程的抽像，使得Picasso扩展加载不同资源的能够友好的扩展，它的的具体实现类NetworkRequestHandler、FileRequestHandler等分别拥有处理网络图片的加载、本地文件加载的能力
	* Downloader接口：下载图片的行为的抽象，UrlConnectionDownloader其的一个实现该类利用HTTPUrlConnection实现了图片的下载以及**磁盘缓存**
	* Cache接口：Picasso内存缓存的行为标准，LruCache为实现类
	* Dispatcher类：该类的作用如其名-**Picasso的事件分发的核心类**,它的作用就是一个中转站，利用Handler的消息机制实现了关于请求的发送、请求的暂停、请求的完成等等一系列消息的处理

* 缓存逻辑总结：我们可以知道Picasso本身没有实现任何关于磁盘缓存的逻辑，它的缓存依赖与第三方(okHttp)或者SDK中的HTTPURLConnection的HTTP缓存，对于4.0以上的版本则使用了HTTPResponseCache类来缓存HTTP响应的实体，内存缓存则是利用常见的LruCache来实现,所以Picasso的缓存逻辑还是比较简单

* 加载逻辑梳理：
	首先Picasso类初始化必须组件(任务处理类RequestHandler、任务下载类Downloader、缓存实现类Cache、事件分发类Dispatcher、线程池PicassoExecutorService等)----->请求的预处理(RequestCreator),这里存在缓存检查的逻辑(以及其他)，没有则继续准备创建请求类Request、Action---->Picasso调用Dispatcher开始发送请求提交的消息，同时提交时还有一个请求检查的逻辑，如果有相同的请求存在则取消之前的---->BitmapHunter上场，请求消息的处理是将BitmapHunter、请求Action、请求处理类RequestHandler绑定，然后送到线程池中执行---->BitmapHunter开始加载图片，缓存图片，图片加载完成后由Dispatcher发送加载完成消息，同时这里加载图片之前还会先检查一次缓存---->Picasso接收结果，利用其**主线程HANDLER(Picasso利用了类似Volley的方法，使用Looper.mainLooper来创建Handler从而利用该Handler将事件调度到主线程中)**来将结果发送到主线程中并回调监听

##五、自问自答##

* 内存缓存空间大小的怎么改变；

	系统默认通过Utils的calculateMemoryCacheSize方法来计算LruCache大小，我们想要改变大小可以自己利用Picasso的Builder类来配置自定义的LruCache大小,如下
	
	```

	LruCache lruCache=new LruCache(2000);
    Picasso.Builder builder=new Picasso.Builder(this);
    builder.memoryCache(lruCache);
    Picasso picasso=builder.build();
    Picasso.setSingletonInstance(picasso);//一定要调

	```

* 磁盘缓存的空间以及大小如何改变:

	由于Picasso的磁盘缓存目录是由Utils类通过函数直接返回的所以无法改变，对于磁盘缓存的大小Picasso定义了一个范围以及一个Utils中的calculateDiskCacheSize方法来计算可用大小，它被用在android4.0以上的磁盘缓存策略上,同样改变不了

	```

	private static final int MIN_DISK_CACHE_SIZE = 5 * 1024 * 1024; // 5MB
  	private static final int MAX_DISK_CACHE_SIZE = 50 * 1024 * 1024; // 50MB

  	static Object install(Context context) throws IOException {
      File cacheDir = Utils.createDefaultCacheDir(context);
      HttpResponseCache cache = HttpResponseCache.getInstalled();
      if (cache == null) {
        long maxSize = Utils.calculateDiskCacheSize(cacheDir);//获取最大的可用缓存的2%
        cache = HttpResponseCache.install(cacheDir, maxSize);
      }
      return cache;
    }

  	```

 * 如果只想下载图片怎么做
 	1、使用get方法:不要从主线程调用，会阻塞线程

 	```

 	Picasso.with(this).load(url).get();//用到Action实现类GetAction

 	```

 	2、使用target作为into的参数，在加载成功后获取图片引用

 	```

 	    Picasso.with(this).load(url).into(new Target() {
      @Override
      public void onBitmapLoaded(Bitmap bitmap, Picasso.LoadedFrom from) {
      }

      @Override
      public void onBitmapFailed(Drawable errorDrawable) {
      }

      @Override
      public void onPrepareLoad(Drawable placeHolderDrawable) {
      }
    });

    额外提一下Picasso.with(this).load(url).fetch();的用法，该方法只是下载存缓存而不返回任何引用(FetchAction)

 	```
 * Picasso会开几条线程
 	一条Dispatcher的Handler使用的线程+数条图片加载线程，后者由网络情况决定且Picasso注册了一个网络变化的广播接收器---连接wifi等4条线程、4G三条线程，3G，2G依次递减一条

 * 请求如何取消
 	首先取消的API有如下几个，见下图

 	![](https://github.com/getletCodes/StudyNotes/blob/master/part2/picasso_cancacel.png)

 	以ImageView为参数为例来查看取消的逻辑

 	```

 	private void cancelExistingRequest(Object target) {
    	checkMain();//检查线程，必须在主线程中调用
    	Action action = targetToAction.remove(target);//取消目标ImageView和请求的关联
    	if (action != null) {
      		action.cancel();//取消标记
      		dispatcher.dispatchCancel(action);//发送取消事件
    	}
    	if (target instanceof ImageView) {//从下面变量的命名来看应该是延时任务的取消
      		ImageView targetImageView = (ImageView) target;
      		DeferredRequestCreator deferredRequestCreator =targetToDeferredRequestCreator.remove(targetImageView);
      		if (deferredRequestCreator != null) {
        		deferredRequestCreator.cancel();
      		}
    	}
  	}

  	//dispatcher事件接着进一步取消Action与一些请求管理类的联系并设置标记取消任务
  	void performCancel(Action action) {
	    String key = action.getKey();
	    BitmapHunter hunter = hunterMap.get(key);
	    if (hunter != null) {
	      hunter.detach(action);//取消关联
	      if (hunter.cancel()) {//取消hunter的future任务
	        hunterMap.remove(key);
	        //...
	      }
	    }

	    if (pausedTags.contains(action.getTag())) {//移除处于暂停状态的请求
	      pausedActions.remove(action.getTarget());
	      //...
	    }

	    Action remove = failedActions.remove(action.getTarget());//移除处于失败状态的请求
	    //...
  	}

 	```

 	在对请求做了取消标记后取消的任务就算完成了，此外在最后准备设置结果时还有一个检查是否取消的逻辑

* 什么时候任务变为延迟任务？

	对于这个问题首先要从DeferredRequestCreator类入手，它的类声明以及构造方法如下

	```

	class DeferredRequestCreator implements ViewTreeObserver.OnPreDrawListener 
	DeferredRequestCreator(RequestCreator creator, ImageView target, Callback callback) {
    	this.creator = creator;
    	this.target = new WeakReference<ImageView>(target);
    	this.callback = callback;
    	target.getViewTreeObserver().addOnPreDrawListener(this);//注册自身为目标ImageView的监听
  	}

	```

	可以看到它实现了OnPreDrawListener接口，并在设置自身为目标ImageView的视图树的绘制监听，然后在加载流程的into方法中我们可以看到如下代码

	```

	if (deferred) {//defered值当我们使用fit时被设置
      int width = target.getWidth();
      int height = target.getHeight();
      if (width == 0 || height == 0) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        //创建了延时任务，当该View将要被绘制时采取发起任务加载
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      data.resize(width, height);
    }

	```

* 最后回调时任务为什么会有批处理，BitmapHunter不是每个请求对应一个?
	在Dispatcher的performSubmit方法中提交BitmapHunter任务之前，有如下代码

	```

	//即Dispatcher事件派发之前会存请求对应BitmapHunter于一个map中，每次提交之前会检查是否已经有了请求，如
	//果有则做一个请求合并的工作
	BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {
      hunter.attach(action);
      return;
    }

	```

* 取消任务中还有一个failedActions的remove的调用，它的作用是什么？

	对于这个问题要回到BitmapHunter的run方法对于下载结果不正确的处理，注意到一下两个异常的处理，这种类型的错误会加入到failedActions集合中在适当的时候(网络发生变化的时候)再次重试下载
	
	```

	 catch (NetworkRequestHandler.ContentLengthException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    } catch (IOException e) {
      exception = e;
      dispatcher.dispatchRetry(this);
    }

	```

最后提一下提一下Picasso实施transformation的时机是在BitmapHunter的hunt方法中，及下载完成之后，回调到主线程之前，以及Picasso自身配有Stats类用于统计图片加载相关的数据(缓存大小之类的)。

本次分析基于最新的Picasso2.5.