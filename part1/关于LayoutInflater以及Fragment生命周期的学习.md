# 关于LayoutInflater以及Fragment生命周期的学习
---
## 一、文档中关于LayoutInflater的解释

该类用于实例化布局到指定的对象，不会被直接使用，而是通过getLayoutInflater或者getSystemService(Context.LAYOUT_INFLATER_SERVICE)来获取一个标准的、关联了当前的context和设备的运行时配置的LayoutInflater实例，另外由于性能上的原因只会利用XmlPullParser 处理系统预处理的资源xml，而不能处理直接的文本

## 二、创建方法的比较

1、LayoutInflater.from(context)创建

这个方法是平常使用的方法

```
	//可以看出来它是与二中的方法是一样的
	public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }

```

2、getLayoutInflater创建

```
	//该方法最终是返回PhoneWindow的成员变量
	public LayoutInflater getLayoutInflater() {
        return mLayoutInflater;
    }
    //而该变量是在PhoneWindow的构造方法中创建的，经过搜索这个Context为appContext
    public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }

```

3、getSystemService(Context.LAYOUT_INFLATER_SERVICE)创建

```
	//可见三种方法是同一个方法，最终是利用该方法来创建
	public Object getSystemService(String name) {
        if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            if (mInflater == null) {
                mInflater = LayoutInflater.from(getBaseContext()).cloneInContext(this);
            }
            return mInflater;
        }
        return getBaseContext().getSystemService(name);
    }
    //最终返回实现者phoneLayoutInflater
    public LayoutInflater cloneInContext(Context newContext) {
        return new PhoneLayoutInflater(this, newContext);
    }

```

## 三、inflate方法解析

```
	//由此开始，当root为空时，第三个参数为false
	public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }

    //创建了一个XML解析器
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        ....
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    //
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;//传入的根View被赋值到result

            try {
                int type;
                ......
                final String name = parser.getName();

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    //由上面的注释可以明白当要inflate的布局的根标签使用了merge标签
                    //我们调用inflate方法传入的根View必须不为空且attach必须为true
                    rInflate(parser, root, inflaterContext, attrs, false);//初始化child，root成为这些View的根View
                } else {
                    // 根据标签来创建一个根View
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {//
                        
                        // 利用传入的root来生成LayoutParams
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }


                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);
                    //根据条件来决定是否让传入的View成为根View
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    //决定最后传回的View是什么
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } finally {
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }


            return result;
        }
    }
    //这个方法里初始化了各个子View，并且将其加入的到parent
    void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;//强转，所以这里可能会有出现类型强转的错误
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);//添加子View
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }

```

## 四、关于fragment的生命周期

先看官网上的关于生命周期的图片

Fragment的生命周期是这样的：

![](https://github.com/getletCodes/StudyNotes/blob/master/part1/fragment_lifecycle.png)

Activity 生命周期对片段生命周期的影响是这样的：

![](https://github.com/getletCodes/StudyNotes/blob/master/part1/activity_fragment_lifecycle.png)

接着关注它的onCreateView方法：文档中说到container参数是您的片段布局将插入到的父 ViewGroup（来自 Activity的布局），savedInstanceState参数是在恢复片段时，提供上一片段实例相关数据的 Bundle。同时一般来说在这里我们使用inflater时一般将attach这个boolean设置为false,文档中说到这是为了避免布局的嵌套、冗余(如果传入true,那么返回的View仍是container),最后文档中提到如果没有UI，
可以让其为Activity提供后台行为

最后要说的就是在实习的时候同事遇见的一个bug,也是这篇blog的原因。他的代码是在一个TabHost(有点古老)里面使用了几个Fragment来做页面切换，其中一个页面他用Handler+ViewPager来做轮播，**并且在onCreateView方法中进行轮播消息的发送**，出现的bug就是当切换几次页面时，图片轮播的那个Fragment的切换速度急剧的变快。我一开就是想到消息多发了，然后我就在onCreateView里面打了个Log来看，发现每次切换页面的时候这个方法都会调用，并且同一个Hanlder对象会在这里发消息(当时对Fragment不熟悉，以为Activity没有销毁，onCreateView只会调用一次)。所以我就建议他在onStop方法里面取消轮播消息，bug就这样解决了。解决之后我就在想这个问题，但是由于自己的疏忽，以为LayoutInflater有什么加载缓存机制，所以看了一下关于LayoutInflater的相关代码和Fragment的生命周期，原来就是当Fragment被移除后再加入就需要再走一遍onCreateView~onDestroyView的流程。另一个想法是每次在onCreateView中创建新的Handler来进行消息发送来避免消息过发，但是很快就否定了，虽然这种方法的确能够避免对ViewPager的过度更新，但是由Handler如果消息没有处理完就无法被回收，而轮播消息是循环发送的，所以Handler永远不会被回收，我的关于Hnadler原理的blog中有提到相关的引用持有，所以这种方法会造成内存泄露,Handler成为
**野指针**
