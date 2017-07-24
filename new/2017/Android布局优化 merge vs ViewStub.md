# Android布局优化 merge vs ViewStub

最近开始用Android Studio的Android Lint优化部分代码,然后Lint提示了有一部分UI的xml的可以将FrameLayout标签换成merge标签,由于以前我自己做一些布局优化merge用的很少,一般ViewStub用的比较多,这篇正好就稍微系统一点的看下merge、ViewStub的工作原理,主要从LayoutInflater、ViewStub的源码来分析,算是基础的一个巩固吧.

## ViewStub工作原理

### 1.LayoutInflate ViewStub标签的处理方式

首先ViewStub其实一个继承自View的一个View组件,所以其实它和其他的自定义View实际上并没有什么本质的区别,而之所以它能够起到懒加载的优化作用主要有以下两点

#### ViewStub使用方式

首先在谈ViewStub使用方式如何完成优化之前,先说一下Android如果通过XML来创建View,我们知道在开发Android UI的时候我们一般是利用XML来开发UI,系统通过LayoutInflater这个类来解析XML文件,并根据不同的标签来执行不同的逻辑,比如对于View 组件来说一般是反射创建一个View组件的实例-**注意这里LayoutInflater是解析到一个View组件的标签才会反射创建View,View才会被绘制到界面上.**

那回过来再说ViewStub的使用方法与懒加载的关系:我们在使用ViewStub的时候一般是因为页面的某一个部分View不需要在显示的时候立即创建、显示,而只是在特定条件下才被触发显示,比如一些错误页,这种页面可以是一个简单的View、也可以是一个更复杂的页面,而对于这种页面我们要ViewStub进行优化的话,ViewStub要求我们单独创建一个XML文件,然后通过android:layout属性声明ViewStub要加载的layout资源文件的id,也就是说ViewStub要懒加载的UI布局对于ViewStub**仅仅只是一个id属性,而不是类似ViewGoup设置子View直接在xml文件中书写View标签,**,所以当LayoutInflater解析到ViewStub时,它只会反射创建出ViewStub实例,LayoutInflater根本就不知道ViewStub中其实包含了更丰富的UI元素,LayoutInflater只知道ViewStub有一个layout id属性,所以ViewStub这种通过id引用layout资源形式可能节省LayoutInflater解析XMK、反射创建View的时间,从而完成了优化的过程.

LayoutInflater解析到自定义View并反射创建的代码如下

```
void rInflate(XmlPullParser parser, View parent, Context context, AttributeSet attrs, boolean finishInflate) 
                                                                            throws XmlPullParserException, IOException {
        final int depth = parser.getDepth();//这里获取当前节点的深度
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
                //此处反射创建自定义View,
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);//这里会递归调用rInflater方法解析创建子View
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }

```

从上面的代码可以看到LyoutInflater会递归的解析XML,而对于ViewStub来说它和其他View一样同样还是会进一步调用调用rInflater方法递归去看是否有子XML节点需要解析,只不过递归之后会因为depth无法进入循环,直接跳到方法结尾执行parent.onFinishInflate()就结束了.

#### ViewStub自身优化

+ 绘制相关优化

这个不多说,看代码

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(0, 0);
}
@Override
public void draw(Canvas canvas) {
}
@Override
protected void dispatchDraw(Canvas canvas) {
}
```

这个代码估计一目了然,ViewStub重写关于测量、绘制的相关方法,而且并没有调用父类方法,并将尺寸设为0,尽可能的减少了ViewStub本身作为一个View的性能开销

+ ViewStub的一次性

这一点主要说的是ViewStub在初始化懒加载的界面之后,ViewStub本身就没有任何作用了,ViewStub初始化时会将要加载的View组件直接attach到其父View上,然后将自己从父View中移除.代码如下

```
public View inflate() {
        final ViewParent viewParent = getParent();
        if (viewParent != null && viewParent instanceof ViewGroup) {
            if (mLayoutResource != 0) {
                final ViewGroup parent = (ViewGroup) viewParent;
                final LayoutInflater factory;
                if (mInflater != null) {
                    factory = mInflater;
                } else {
                    factory = LayoutInflater.from(mContext);
                }
                //这里使用LayoutInflater解析XML创建对应的View实例
                final View view = factory.inflate(mLayoutResource, parent, false);
                if (mInflatedId != NO_ID) {
                    view.setId(mInflatedId);
                }
                final int index = parent.indexOfChild(this);//寻找ViewStub
                parent.removeViewInLayout(this);//将自身移除

                final ViewGroup.LayoutParams layoutParams = getLayoutParams();
                //将懒加载页面直接添加到父View上
                if (layoutParams != null) {
                    parent.addView(view, index, layoutParams);
                } else {
                    parent.addView(view, index);
                }
                mInflatedViewRef = new WeakReference<View>(view);//后期ViewStub可以通过该参数来改变懒加载页面的可见性
                if (mInflateListener != null) {
                    mInflateListener.onInflate(this, view);
                }
                return view;
            } else {
                throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
            }
        } else {
            throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
        }
    }
```

## merge工作原理

首先看下merge标签与ViewStub的区别-前者主要作用在于减少UI层级而后者而在于懒加载,更重要的是**merge只是一个标签,而ViewStub是一个View组件**.merge的工作原理主要要从LayoutInflate的代码中看起,核心代码如下

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    //略...
    if (TAG_MERGE.equals(name)) {
        if (root == null || !attachToRoot) {
            throw new InflateException("<merge /> can be used only with a valid "
                    + "ViewGroup root and attachToRoot=true");
        }
        rInflate(parser, root, inflaterContext, attrs, false);
    }
    //略... 
}
```

可以看见遇见merge标签之后,LayoutInflater会继续调用rInflate继续解析子节点,但是不同于上面inflate解析到View组件是它传给rInflate中父View的参数是**merge的父节点**,所以merge标签实际上直接被忽略LayoutInflater忽略了,从而达到了优化效果.