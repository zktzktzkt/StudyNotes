#Android Tranning中volley教程阅读笔记#
---

##一、Volley是什么##

volley是一个http库，它能够让android app的网络访问更加简单、快速

##二、Volley的优点和缺点##

+ 网络请求的自动调度
+ 多个并发的网络连接
+ 磁盘和内存级的标准Http缓存
+ 支持设置请求的优先级
+ 提供撤销请求的API
+ 可以很方便的进行定制
+ 确保能够简单、正确的将异步网络数据填充到UI中
+ 提供调试和追踪工具
+ 由于Volley在解析时会将数据全部存储在内存中，所以它不适合下载、处理流数据，可以利用DownloadManager来完成

##三、发送一个简单的请求##

简单来说使用volley发送网络请求就是建立RequestQueue，然后将Request对象传给它即可。其中RequestQueue是用来管理工作线程，发送请求、读写缓存、解析数据等耗时任务都在这里进行，而且RequestQuene保证将解析的响应结果返回到主线程。

官网的工作原理图如下

![](https://github.com/getletCodes/StudyNotes/blob/master/part2/volley-request.png)

取消请求可以调用cancel方法，一旦调用，volley保证你的监听回调不会被调用，一般在onStop方法中调用和Activity的生命周期进行联动，此外为了更好的使用这个方法，可以为每个请求加上一个tag，然后可以利用这个tag来取消一个域的请求

##四、配置RequestQueue##

如果需要经常使用网络，建议使用了单例来创建RequestQueue，这样就可以在app的整个生命周期中存在。实现一个ReuqestQueue单例一般有两种方法，一是继承Application类在这里创建，一是写一个单独的单例类，文档推荐使用后者，这样更加模块化。其次一个关键的概念是RquestQueue必须通过Application context来创建而不是Activity context，这是为了保证RequestQueue生命周期可以和app一样长，而不会因为Activity的重建而重建

其次RequestQueue需要两件东西来保证正常工作：一个用于请求的网络、一个缓存来处理缓存。这两个在Volley toolbox都有标准实现：DiskBasedCache利用内存索引提供了响应到文件的一对一缓存，BasicNetwork可以根据你的HTTP客户端提供网络传输


##五、发送基本的请求##

根据返回的内容的差异有三种标准的请求实现：*StringRequest*、*ImageRequest*、*JsonObjectRequest、JsonArrayRequest*

**特别的关于图片的请求Volley提供了一下几个类**

ImageRequest-可以根据指定的URL来获取图片的请求类，而且它还在构造函数中提供图片尺寸设置的相关方法，最重要的一点它保证解码等耗时操作在工作线程中完成

ImageLoader-一个用来加载和缓存图片的帮助类，它相当于大量的ImageRequest的集合，比如加载ListView的缩略图。并且它在Volley的标准缓存之上提供了一个内存缓存。同时ImageLoader会做请求合并，比如每个响应向同一个View设置bitmap(*应该没翻译错吧*)

NetworkImageView-建立于ImageLoader之上用于在利用url来设置图片的情况下替代ImageView,当它从View树中分离时会取消请求。


```

	public class MySingleton {
    	private static MySingleton mInstance;
    	private RequestQueue mRequestQueue;
    	private ImageLoader mImageLoader;
    	private static Context mCtx;

    	private MySingleton(Context context) {
        	mCtx = context;
        	mRequestQueue = getRequestQueue();

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
    	}

    	public static synchronized MySingleton getInstance(Context context) {
        	if (mInstance == null) {
        	    mInstance = new MySingleton(context);
        	}
        	return mInstance;
    	}

	    public RequestQueue getRequestQueue() {
	        if (mRequestQueue == null) {
	            mRequestQueue = Volley.newRequestQueue(mCtx.getApplicationContext());
	        }
	        return mRequestQueue;
	    }

	    public <T> void addToRequestQueue(Request<T> req) {
	        getRequestQueue().add(req);
	    }

	    public ImageLoader getImageLoader() {
	        return mImageLoader;
	    }
	}

	public class LruBitmapCache extends LruCache<String, Bitmap>
	        implements ImageCache {

	    public LruBitmapCache(int maxSize) {
	        super(maxSize);
	    }

	    public LruBitmapCache(Context ctx) {
	        this(getCacheSize(ctx));
	    }

	    @Override
	    protected int sizeOf(String key, Bitmap value) {
	        return value.getRowBytes() * value.getHeight();
	    }

	    @Override
	    public Bitmap getBitmap(String url) {
	        return get(url);
	    }

	    @Override
	    public void putBitmap(String url, Bitmap bitmap) {
	        put(url, bitmap);
	    }

	    // Returns a cache size equal to approximately three screens worth of images.
	    public static int getCacheSize(Context ctx) {
	        final DisplayMetrics displayMetrics = ctx.getResources().
	                getDisplayMetrics();
	        final int screenWidth = displayMetrics.widthPixels;
	        final int screenHeight = displayMetrics.heightPixels;
	        // 4 bytes per pixel
	        final int screenBytes = screenWidth * screenHeight * 4;

	        return screenBytes * 3;
	    }
	}

```

使用案例

```

	RequestQueue mRequestQueue; // assume this exists.
	ImageLoader mImageLoader = new ImageLoader(mRequestQueue, new LruBitmapCache(
	            LruBitmapCache.getCacheSize()));

```

##六、实现一个自定义的Request##

实现自定义Request的基本要求：继承Request类，实现parseNetworkResponse()、deliverResponse()方法，其中前者的参数包含了HTTP的响应头、状态码的数据。

```

	public class GsonRequest<T> extends Request<T> {
	    private final Gson gson = new Gson();
	    private final Class<T> clazz;
	    private final Map<String, String> headers;
	    private final Listener<T> listener;

	    /**
	     * Make a GET request and return a parsed object from JSON.
	     *
	     * @param url URL of the request to make
	     * @param clazz Relevant class object, for Gson's reflection
	     * @param headers Map of request headers
	     */
	    public GsonRequest(String url, Class<T> clazz, Map<String, String> headers,
	            Listener<T> listener, ErrorListener errorListener) {
	        super(Method.GET, url, errorListener);
	        this.clazz = clazz;
	        this.headers = headers;
	        this.listener = listener;
	    }

	    @Override
	    public Map<String, String> getHeaders() throws AuthFailureError {
	        return headers != null ? headers : super.getHeaders();
	    }

	    @Override
	    protected void deliverResponse(T response) {
	        listener.onResponse(response);
	    }

	    @Override
	    protected Response<T> parseNetworkResponse(NetworkResponse response) {
	        try {
	            String json = new String(
	                    response.data,
	                    HttpHeaderParser.parseCharset(response.headers));
	            return Response.success(
	                    gson.fromJson(json, clazz),
	                    HttpHeaderParser.parseCacheHeaders(response));
	        } catch (UnsupportedEncodingException e) {
	            return Response.error(new ParseError(e));
	        } catch (JsonSyntaxException e) {
	            return Response.error(new ParseError(e));
	        }
	    }
	}

```