#WebView文档阅读笔记#
---

##一、基本使用##

WebView本身是继承自AbsoluteLayout一般用于加载HTML文档或者一个远程的url，加载的方式则是直接调用loadUrl即可，不过需要注意的是，默认情况下WebView对一个链接的处理是使用浏览器来打开(实际上根据ActivityManager选择能够处理的App)，如果需要改变这一行为则需要一个WebViewClient对象来提供行为的确认,之所以说是行为的确认，是因为WebViewClicent类中包含了很多询问方法，比如对url的处理方式、资源加载询问，此外该类也是一个监听器，对于页面的加载、缩放等等事件均有相应的回掉(该类被回调基本上是因为某些会影响渲染内容的事件,比如错误)，下面是一个demo

```
	protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        webView= (WebView) findViewById(R.id.wv_show_html);
        webView.setWebViewClient(new MyClient());
        webView.loadUrl("http://www.baidu.com");

    }
    class MyClient extends WebViewClient{
    	/**该方法是将要加载url的连接时被回调，返回false表示WebView处理，否则为app处理，这个例子一直返回false
    	则表示该WebView实际上已经起了一个简单的浏览器的作用,值得注意的是对于POST请求不回掉*/
        @Override
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            return false;
        }
    }

```

另外对于类似与浏览器的页面导航功能，WebView也提供了相应的API,例如回退与前进的方法goBack、goForward

##二、与Js进行交互##

默认情况下WebView禁止使用Js，可以通过WebView的getSettings方法获取WebView的配置对象WebSettings的实例，并设置js开启即可。

###1、JS调用Android代码###

首先准备好一个类定义好相关方法，然后webView的addJavascriptInterface添加js接口即可，例如:

```

	protected void onCreate(Bundle savedInstanceState) {
        
        WebSettings settings=webView.getSettings();
        settings.setJavaScriptEnabled(true);
        settings.setBuiltInZoomControls(true);//启用默认的缩放
        //该方法为js文件提供了一个名为Android的类作为接口，来映射到WvInterface,传入的对象务必要确定执行对应方法时环境的保证
        webView.addJavascriptInterface(new WvInterface(this),"Android");
        StringBuilder stringBuilder=new StringBuilder();
        ....
        webView.loadData(stringBuilder.toString(),"text/html","utf-8");

    }
    class WvInterface{
        private Context context;
        public WvInterface(Context context){
            this.context=context;
        }
        @JavascriptInterface
        public void showToast(String text){
            Toast.makeText(context,text,Toast.LENGTH_SHORT).show();
        }

    }
    //js文件是这样的
    <script type="text/javascript">
	function toast(){
		Android.showToast("hello ,I am javascript");
	}
</script>

```

此外值得注意的一点是SDK17以上的版本的WebView的内核有所改变，所以需要加上@JavascriptInterface注解在对应的方法上来确保正确工作

###2、从Android中调用JS###

对与SDK19以上的版本可以使用evaluateJavascript方法，而对于低版本则可以使用loadurl方法，例如：

```

	//利用loadUrl方法，可以再做一个js到java的回调来带会返回值
	String call = "javascript:msg(\"" + "content" + "\")";
	webView.loadUrl(call);

	/**高版本的方法，至于为什么在onPageFinished方法中调用，则是因为文档中有提到该方法是异步调用
	*当前已展示页面的js方法，此外刚方法必须在UI线程中调用，其回调也在UI线程中
	**/
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            webView.evaluateJavascript("test(\"he\",1)", new ValueCallback<String>() {
                @Override
                public void onReceiveValue(String value) {
                    Toast.makeText(MainActivity.this,value,Toast.LENGTH_SHORT).show();
                }
            });
        }
    }

```

##三、如何调试WebView中的js##

对于SDK17一下的版本可以用console下的系列打印日志来进行调试，不过这需要WebView配合WebChromeClient类(该类的回调常是因为事件会影响浏览器UI，可以通过定制该类来操作cookies)来使用,例如

```
	webView.setWebChromeClient(new WebChromeClient(){
		//对与HTML中的console的打印方法会产生此回调，此外还有一个类似方法是支持更高版本，此方法支持到SDK8
        @Override
        public boolean onConsoleMessage(ConsoleMessage consoleMessage) {
            Log.e("tag","hello from "+consoleMessage);
            return super.onConsoleMessage(consoleMessage);
       	}
    );

```

对于更高版本则可以通过chrome浏览器来进行调试，参考链接

![](https://developers.google.com/web/tools/chrome-devtools/debug/remote-debugging/remote-debugging?utm_source=dcc&utm_medium=redirect&utm_campaign=2016q3)

最后对于CSS/HTML的尺寸信息低版本的为屏幕实际尺寸，而高版本的则是CSS的尺寸单位