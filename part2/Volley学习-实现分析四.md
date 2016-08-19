#Volley学习-实现分析四#
---

总的来说前面几篇已经差不多结束，最后一篇主要是关于网络请求相关的实现的分析，即Volley是如何利用HttpURLConnection来进行网络请求的

##一、网络请求的实现的分析##

从前面的分析来看网络请求最终是利用Volley类创建RequestQueue时传入的BasicNetwork的实现的，如下

```

	if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            stack = new HurlStack();
        } else {//这个基本可以忽略，主要也是由于SDK9一下的HTTPURLConnection有点问题，文档有相关的说明
            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
        }
    }
    Network network = new BasicNetwork(stack);
    RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);

    //相关类的声明是这样的
    public class BasicNetwork implements Network;
    public interface Network {
    	//该方法进行网络请求
    	public NetworkResponse performRequest(Request<?> request) throws VolleyError;
	}

	public class HurlStack implements HttpStack ;

	public interface HttpStack {
    /**
     * Performs an HTTP request with the given parameters.
     */
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError;

}

```

上面的代码基本列出了网络请求的类的层次结构，接着从NetworkDispatcher中进行网络请求的代码看起

```
	NetworkResponse networkResponse = mNetwork.performRequest(request);//也就是BasicNetwork的performRequest
	
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            Map<String, String> responseHeaders = Collections.emptyMap();
            try {
                ......//最终根据请求头和请求体来进行真正的请求，在HurlStack中实现
                httpResponse = mHttpStack.performRequest(request, headers);
                StatusLine statusLine = httpResponse.getStatusLine();
                int statusCode = statusLine.getStatusCode();
                responseHeaders = convertHeaders(httpResponse.getAllHeaders());

                // Handle cache validation.
                if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                	//304返回另说，从cache中获取数据
                }
                if (httpResponse.getEntity() != null) {
                   //从流中取数据
                  responseContents = entityToBytes(httpResponse.getEntity());
                } else {
                  responseContents = new byte[0];
                }
            }
        }
    }


    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
        ......
        URL parsedUrl = new URL(url);
        //这里创建了一个新的HttpURLConnection，并没有所谓的HTTP连接的复用
        HttpURLConnection connection = openConnection(parsedUrl, request);
        for (String headerName : map.keySet()) {
            connection.addRequestProperty(headerName, map.get(headerName));
        }
        setConnectionParametersForRequest(connection, request);
        ......
        //这里将HTTP请求返回的数据流存入到Response中，并在BasicNetWork中取出数据流
        if (hasResponseBody(request.getMethod(), responseStatus.getStatusCode())) {
            response.setEntity(entityFromConnection(connection));
        }
        ......
        return response;
    }

	//可以看到如果提供了SslSocketFactory的实现，则volley能够提供https的支持
    private HttpURLConnection openConnection(URL url, Request<?> request) throws IOException {
        HttpURLConnection connection = createConnection(url);

        int timeoutMs = request.getTimeoutMs();
        connection.setConnectTimeout(timeoutMs);
        connection.setReadTimeout(timeoutMs);
        connection.setUseCaches(false);
        connection.setDoInput(true);
        // use caller-provided custom SslSocketFactory, if any, for HTTPS
        if ("https".equals(url.getProtocol()) && mSslSocketFactory != null) {
            ((HttpsURLConnection)connection).setSSLSocketFactory(mSslSocketFactory);
        }

        return connection;
    }

    //接下来看BasicNetwork从流中读取数据的逻辑，其实只是想看一看字节连接池的实现
    private byte[] entityToBytes(HttpEntity entity) throws IOException, ServerError {
        PoolingByteArrayOutputStream bytes =
                new PoolingByteArrayOutputStream(mPool, (int) entity.getContentLength());
        byte[] buffer = null;
        try {
            InputStream in = entity.getContent();
            if (in == null) {
                throw new ServerError();
            }
            buffer = mPool.getBuf(1024);//获取缓存的字节数组
            int count;
            while ((count = in.read(buffer)) != -1) {
                bytes.write(buffer, 0, count);
            }
            return bytes.toByteArray();
        } finally {
            mPool.returnBuf(buffer);//返回使用的字节数据
            bytes.close();
        }
    }
```

从上面来看，网络请求最终使用了HttpURLConnection/HTTPS来进行，另一个就是**并没有所谓的HTTP连接池**，或者说我以前对HTTP连接池想法有误，以为和对象的连接池一样，是对同一个引用的服用，这里看来只是线程的并发。接着可以看出来在读取HTTP返回的数据时存在一个缓存的机制，主要是两个类PoolingByteArrayOutputStream、ByteArrayPool。
由entityToBytes中的调用开始看起

```
	//该类继承自ByteArrayOutputStream
	public PoolingByteArrayOutputStream(ByteArrayPool pool, int size) {
        mPool = pool;
        buf = mPool.getBuf(Math.max(size, DEFAULT_SIZE));
    }
    //选取一个合适的字节数据的缓存返回，没有则创建
    public synchronized byte[] getBuf(int len) {
        for (int i = 0; i < mBuffersBySize.size(); i++) {
            byte[] buf = mBuffersBySize.get(i);
            if (buf.length >= len) {
                mCurrentSize -= buf.length;
                mBuffersBySize.remove(i);
                mBuffersByLastUse.remove(buf);
                return buf;
            }
        }
        return new byte[len];
    }
    //归还字节数据的逻辑
    public synchronized void returnBuf(byte[] buf) {
        if (buf == null || buf.length > mSizeLimit) {
            return;
        }
        mBuffersByLastUse.add(buf);
        //二分搜索寻找合适的插入位置
        int pos = Collections.binarySearch(mBuffersBySize, buf, BUF_COMPARATOR);
        if (pos < 0) {
            pos = -pos - 1;
        }
        mBuffersBySize.add(pos, buf);
        mCurrentSize += buf.length;
        trim();
    }
```
就是说volley也是有一个内存方面的缓存工作，为了防止每次读取HTTP返回的数据时创建字节数据进行的缓存字节的频繁创建造成GC过多，所以volley创建了一个字节数据池来防止这些事。最后，关于volly相关的就先到这里了。
