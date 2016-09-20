#Picasso的使用与实现分析一#
---

##一、Picasso官网对Picasso的介绍##

* 引言:对于需要加载大量图片的来说，Picasso可以让开发者仅用一行代码集成图片加载功能,同时它还解决了很多常见的Androiod图片加载的一些错误，比如:
	* 对于处于adapter中的ImageView,当它被回收时自动取消请求
	* 使用尽量少的内存来实现复杂的图片转换
	* 自动的内存、磁盘缓存
* 特性:
	* Adapter中的复用能够被自动检测到，能够依此取消之前的请求
	* 能够将图片转换到合适的尺寸依此适应布局同时减少内存的使用
	* 下载时和错误状态的默认占位图片的设置
	* 支持加载资源文件、assets文件、文件、内容提供者
	* 调试指示-可以使用setIndicatorsEnabled(true)来在图片源上增加一个色带,不同的颜色表示不同的加载源

##二、Picasso的Sampler如何Picasso##

Picasso的Sample示例中一共演示了五种情况下Picasso的使用:
* 位于Adapter中的使用

	```

	Picasso.with(context).load(url).placeholder(R.drawable.placeholder).error(R.drawable.error).fit().tag(context).into(view);//一行代码链式加载

	```

* 从图库中加载

	```
	//加载的源可以是从图库返回的Bitmap对象image
	private void loadImage() {
    	animator.setDisplayedChild(1);
    	Picasso.with(this).load(image).fit().centerInside().into(imageView, new EmptyCallback() {
      	@Override 
      	public void onSuccess() {
        	animator.setDisplayedChild(0);
      	}
    	});
  	}

	```

* 从内容提供者的Url加载

	```

	@Override 
	public void bindView(View view, Context context, Cursor cursor) {
	    Uri contactUri = Contacts.getLookupUri(cursor.getLong(ContactsQuery.ID),
	        cursor.getString(ContactsQuery.LOOKUP_KEY));
	    ViewHolder holder = (ViewHolder) view.getTag();
	    holder.text1.setText(cursor.getString(ContactsQuery.DISPLAY_NAME));
	    holder.icon.assignContactUri(contactUri);
	    //同样是一行链式加载
	    Picasso.with(context).load(contactUri).placeholder(R.drawable.contact_picture_placeholder)
	        .tag(context).into(holder.icon);
  	}

	```

* 列表->详情式
	
	```

	//位于ListView的Item中时使用resizeDimen来加载小图，点击之后才加载大图
	Picasso.with(context).load(url).placeholder(R.drawable.placeholder).error(R.drawable.error)
        .resizeDimen(R.dimen.list_detail_image_size, R.dimen.list_detail_image_size)
        .centerInside().tag(context).into(holder.image);

	```

* 通知中的使用
	
	```

	//可以看出支持RemoteView的图片加载设置，只不过需要指定Id而已
	icasso.with(activity).load(Data.URLS[new Random().nextInt(Data.URLS.length)]) .resizeDimen(R.dimen.notification_icon_width_height,R.dimen.notification_icon_width_height).into(remoteViews, R.id.photo, NOTIFICATION_ID, notification);

	```

##三、实现分析##

以Picasso的Sample的第一种情况来分析整个的加载流程

```

Picasso.with(context).load(url).placeholder(R.drawable.placeholder).error(R.drawable.error).fit().tag(context).into(view);

```

* Picasso的创建分析(with(context)):

	Picasso类的采用了Builder模式来创建实例，它的构造方法是的访问是包内的，所有我们无法通过构造方法创建实例。只能通过with方法或者通过其内部类Builder类来创建,不过前者也是通过后者来实现，创建方法如下
	
	```

	//可以看到这里对于我们没有配置的对象，采用了Picasso的默认实现，所以这里就是我们可以自定义的几个东西
	//具体这里的变量的含义在留待之后的分析
	public Picasso build() {
      Context context = this.context;
      if (downloader == null) {
        downloader = Utils.createDefaultDownloader(context);
      }
      if (cache == null) {
        cache = new LruCache(context);
      }
      if (service == null) {
        service = new PicassoExecutorService();
      }
      if (transformer == null) {
        transformer = RequestTransformer.IDENTITY;
      }
      Stats stats = new Stats(cache);
      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);
      return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
          defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
  	}

	```

	从上面的代码来看Picasso实例的创建需要相当多的参数，一时之间我们根本无法理解这些参数的作用，所以我们接着看加载的流程往下看。

* load(url)方法逻辑:

```
	
	//该方法返回的RequestCreator用于创建一个图像下载请求
	public RequestCreator load(@Nullable String path) {
    	if (path == null) {
      		return new RequestCreator(this, null, 0);//返回uri为空的请求创建类
    	}
    	if (path.trim().length() == 0) {
      		throw new IllegalArgumentException("Path must not be empty.");
    	}
    	return load(Uri.parse(path));//return new RequestCreator(this, uri, 0);
  	}

  	//RequestCreator的创建如下
  	RequestCreator(Picasso picasso, Uri uri, int resourceId) {
    	if (picasso.shutdown) {//已经关闭
      	throw new IllegalStateException(
          	"Picasso instance already shut down. Cannot submit new requests.");
    	}
    	this.picasso = picasso;//持有picasso的实例引用
    	this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);//创建对应的Builder构造器
  	}

  	//Builder构造器的创建如下
  	Builder(Uri uri, int resourceId, Bitmap.Config bitmapConfig) {
      this.uri = uri;
      this.resourceId = resourceId;
      this.config = bitmapConfig;
    }

```

* placeholder、error、fit、tag方法:
	
```
	//可以看出来申明了不需要Holder、错误的id、Holder已存在三种情况是非法的,error资源的设置、与之类似
	public RequestCreator placeholder(int placeholderResId) {
	    if (!setPlaceholder) {
	      throw new IllegalStateException("Already explicitly declared as no placeholder.");
	    }
	    if (placeholderResId == 0) {
	      throw new IllegalArgumentException("Placeholder image resource invalid.");
	    }
	    if (placeholderDrawable != null) {
	      throw new IllegalStateException("Placeholder image already set.");
	    }
	    this.placeholderResId = placeholderResId;
	    return this;
  	}

	//fit仅仅设置了一个标记位
	public RequestCreator fit() {
    	deferred = true;
    	return this;
  	}
  	//tag同理只是先持有了该引用
  	public RequestCreator tag(Object tag) {
    	if (tag == null) {
      	throw new IllegalArgumentException("Tag invalid.");
    	}
    	if (this.tag != null) {
      	throw new IllegalStateException("Tag already set.");
    	}
    	this.tag = tag;
    	return this;
  	}

```

* into(view)方法:

```

  //经过一级跳转到该方法，注意该方法的逻辑都在主线程中执行
  public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
    checkMain();//检查是否位于主线程，不是则直接抛异常

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }

    if (!data.hasImage()) {//uri != null || resourceId != 0，已加载，停止请求
      picasso.cancelRequest(target);
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }

    if (deferred) {//是否设置了类似于fit的自适应
      if (data.hasSize()) {//如果有width、height指定了值，不允许自适应
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      int width = target.getWidth();
      int height = target.getHeight();
      //根据目标ImageView的尺寸、是否要重绘决定是否取消请求
      if (width == 0 || height == 0 || target.isLayoutRequested()) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());//先设置占位符
        }
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      data.resize(width, height);//设置目标值
    }

    Request request = createRequest(started);//创建请求
    String requestKey = createKey(request);//创建索引键

    if (shouldReadFromMemoryCache(memoryPolicy)) {//是否可以从缓存取
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);//获取缓存
      if (bitmap != null) {//已缓存则可以停止请求
        picasso.cancelRequest(target);
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
        if (picasso.loggingEnabled) {
          log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
        }
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }

    if (setPlaceholder) {//设置placeholder
      setPlaceholder(target, getPlaceholderDrawable());
    }

    Action action =
        new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
            errorDrawable, requestKey, tag, callback, noFade);

    picasso.enqueueAndSubmit(action);//提交请求
  }

```

从上面的代码来看有三种情况不会发送请求：当要构建的请求的url为空且resourceId(仅在加载本地资源时不为空)为空、如果目标的ImageView不可见或者将不可见这不会立即请求、缓存命中不会请求

接着分别追一下这三种情况的流程

* 当要构建的请求的url为空且resourceId为空

```

	if (!data.hasImage()) {
      picasso.cancelRequest(target);//首先取消请求
      if (setPlaceholder) {//如果设置占位图
        setPlaceholder(target, getPlaceholderDrawable());//设置占位符
      }
      return;
    }
    //简单的设置了占位图到ImageView上,是动画则播放
    static void setPlaceholder(ImageView target, Drawable placeholderDrawable) {
    	target.setImageDrawable(placeholderDrawable);
    	if (target.getDrawable() instanceof AnimationDrawable) {
      		((AnimationDrawable) target.getDrawable()).start();
    	}
  	}

```

* 如果目标的ImageView不可见或者将不可见这不会立即请求

```

	if (width == 0 || height == 0 || target.isLayoutRequested()) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());//设置占位图片
        }
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
    }
    //defer方法如下，可以知道这里只是将请求暂存，至于怎么用的先不看
    void defer(ImageView view, DeferredRequestCreator request) {
    if (targetToDeferredRequestCreator.containsKey(view)) {
      cancelExistingRequest(view);//如果已经有了一个defer request则先取消
    }
    //Map<ImageView, DeferredRequestCreator>将请求存到一个map中
    targetToDeferredRequestCreator.put(view, request);
  }

```

* 缓存命中不会请求

```
	//这里的逻辑就是检索你缓存->设置图片->回调监听，比较简单，所以我们主要关注着这里的缓存的检索
	if (shouldReadFromMemoryCache(memoryPolicy)) {//检查是否可以从缓存中读
      Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);//去缓存
      if (bitmap != null) {
        picasso.cancelRequest(target);//取消所有相关请求
        setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);//设置图片
        if (callback != null) {
          callback.onSuccess();//回调监听
        }
        return;
      }
    }

```
先从方法shouldReadFromMemoryCache(memoryPolicy)开始，通过搜索memoryPolicy的使用位置我们可以了解到该变量的设置是通过.memoryPolicy(MemoryPolicy.NO_CACHE,MemoryPolicy.NO_STORE)方法来做的，可以看到与之相关的是一个枚举类如下,它指示了Picasso的请求的缓存缓存策略

```

public enum MemoryPolicy {

  //不需要从内存缓存中读
  NO_CACHE(1 << 0),
  //不存储最终的结果，这个一般用于一次性请求，以免LruCache删除其他的Bitmap
  NO_STORE(1 << 1);

  final int index;

  private MemoryPolicy(int index) {
    this.index = index;
  }
}

```

接着是缓存的检索如下

```

Bitmap quickMemoryCacheCheck(String key) {
    Bitmap cached = cache.get(key);//获取缓存
    //下面的消息发送只是Stats对数据的一个统计
    if (cached != null) {
      stats.dispatchCacheHit();//发送命中消息，内部相关计数器+1
    } else {
      stats.dispatchCacheMiss();//发送未命中消息，内部相关计数器+1
    }
    return cached;
}

```

对于三个例外过程看完后再接着看一个新的请求是如何进行的,首先先是请求的创建和索引key的创建如下

```

	Request request = createRequest(started);
    String requestKey = createKey(request);

    //请求对象的创建过程如下
    private Request createRequest(long started) {
	    int id = nextId.getAndIncrement();

	    Request request = data.build();//利用Request的Builder对象创建
	    request.id = id;//唯一id
	    request.started = started;//开始时间
	    //.....
	    Request transformed = picasso.transformRequest(request);//默认的返回值是request本身
	    if (transformed != request) {//如果上述的返回不是同一个则改变时间戳和id
	      transformed.id = id;
	      transformed.started = started;
	    }

	    return transformed;//返回RRequest对象
  	}
  	//Request的Builder对象的build代码如下
  	public Request build() {
      //...检查一些矛盾的设置并抛出相应的异常
      
      if (priority == null) {
        priority = Priority.NORMAL;
      }
      //可以看出来给予了一个请求的所有信息：url、id、尺寸、等等
      return new Request(uri, resourceId, stableKey, transformations, targetWidth, targetHeight,
          centerCrop, centerInside, onlyScaleDown, rotationDegrees, rotationPivotX, rotationPivotY,
          hasRotationPivot, config, priority);
    }

    //创建索引key的方法如下,根据Request请求来创建key
    static String createKey(Request data) {
    	String result = createKey(data, MAIN_THREAD_KEY_BUILDER);
    	MAIN_THREAD_KEY_BUILDER.setLength(0);
    	return result;
  	}
  	//可以看出来key的组成有Request的成员参数决定
	static String createKey(Request data, StringBuilder builder) {
	    if (data.stableKey != null) {//是否存在stabkey
	      builder.ensureCapacity(data.stableKey.length() + KEY_PADDING);
	      builder.append(data.stableKey);
	    } else if (data.uri != null) {
	      String path = data.uri.toString();
	      builder.ensureCapacity(path.length() + KEY_PADDING);
	      builder.append(path);
	    } else {
	      builder.ensureCapacity(KEY_PADDING);
	      builder.append(data.resourceId);
	    }
	    builder.append(KEY_SEPARATOR);
	    //接下来是根据一些变换信息来补充键
	    if (data.rotationDegrees != 0) {
	      builder.append("rotation:").append(data.rotationDegrees);
	      if (data.hasRotationPivot) {
	        builder.append('@').append(data.rotationPivotX).append('x').append(data.rotationPivotY);
	      }
	      builder.append(KEY_SEPARATOR);
	    }
	    if (data.hasSize()) {
	      builder.append("resize:").append(data.targetWidth).append('x').append(data.targetHeight);
	      builder.append(KEY_SEPARATOR);
	    }
	    if (data.centerCrop) {
	      builder.append("centerCrop").append(KEY_SEPARATOR);
	    } else if (data.centerInside) {
	      builder.append("centerInside").append(KEY_SEPARATOR);
	    }

	    if (data.transformations != null) {
	      //noinspection ForLoopReplaceableByForEach
	      for (int i = 0, count = data.transformations.size(); i < count; i++) {
	        builder.append(data.transformations.get(i).key());
	        builder.append(KEY_SEPARATOR);
	      }
	    }
	    return builder.toString();
	}

```

