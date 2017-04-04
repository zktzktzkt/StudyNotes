# View的测量以及绘制流程简要分析一
---

## 一、onMeasure、onLayout、onDraw调用栈的截图

![](https://github.com/getletCodes/StudyNotes/blob/master/part1/picture/onmeasure.png) ![](https://github.com/getletCodes/StudyNotes/blob/master/part1/picture/onLayout.png) ![](https://github.com/getletCodes/StudyNotes/blob/master/part1/picture/ondraw.png)

一个very long的调用栈，不过从这三幅图可以看到绘制起点ViewRootImpl类开始到视图树最下端的调用流程

**再简要看一下根布局创建**,从setContentView开始看首先该方法调用到的是PhoneWindow的setContentView方法，部分代码如下

```

	@Override
    public void setContentView(int layoutResID) {
    	//ViewGroup mContentParent;放置内容的View
        if (mContentParent == null) {
            installDecor();
        }
        //....if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);//将资源布局填充到内容区域中
        }
    }


```

可以看到当contentParent为空,见名知意即内容的父View为空调用installDecor方法,部分代码如下

```

	private void installDecor() {
        //DecorView mDecor;Window的第一层View,继承自FrameLayout
        if (mDecor == null) {
            mDecor = generateDecor(-1);//生成一个DecorView
            //...
        } else {
            mDecor.setWindow(this);//绑定Window
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);//生成ContentView
            //设置标题之类的、或者一些图标之类的

        }
    }


	//虽然是创建contentParent的方法，但大部分工作都在根据window主题配置UI属性和填充DecorView的
	//布局
	protected ViewGroup generateLayout(DecorView decor) {
        // 根据xml(com.android.internal.R.styleable.Window)
        //中配置的Window的主题设置Window风格,例如:R.styleable.Window_windowSoftInputMode

        //......各种属性的读取，根据feature来选择根目录资源文件(例如:R.layout.screen_title;)
        
        //这里将之前选择的资源文件进行加载,从各个资源文件的布局来看一般分为标题区、内容区
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
        //contentParent为内容去，实际上是从DecorView中findView
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);

        return contentParent;
    }

```

以上分析也就是网上给出的Activity的布局结构

## 二、onMeasure的调用流程

这里主要分析onMeasure中两个参数MeasureSpec的生成规则以及ViewGroup如何测量子View

首先我想先提出一个前提，即我们在将View加入到ViewGroup中或者通过setContentView等等之类将View放到视图树中时，***每个View的LayoutParams实例已经创建好了***(文档对LayoutParams的说明：LayoutParams are used by views to tell their parents how they want to be laid out---View使用LayoutParams实例告诉它的父View自身希望如何布局),之所以提这个前提是因为MeasureSpec的创建依赖于LayoutParams实例

我们由从上面的onMeasure的方法调用栈开始看，可以追到的第一个调用了测量的方法是performTraversals方法(一个代码+注释一共801行的方法),只看关键部分如下

```

	private void performTraversals() {

        //以下三种情况会执行测量逻辑
        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            final Resources res = mView.getContext().getResources();
            // 测量根View,WindowManager.LayoutParams lp;
            windowSizeMayChange |= measureHierarchy(host, lp, res,
                    desiredWindowWidth, desiredWindowHeight);//开始测量
        }

        if (mApplyInsetsRequested) {
   			//...
            dispatchApplyInsets(host);
            if (mLayoutRequested) {
            	//....
                windowSizeMayChange |= measureHierarchy(host, lp,
                        mView.getContext().getResources(),
                        desiredWindowWidth, desiredWindowHeight);//开始测量
            }
        }

        if (!mStopped || mReportNextDraw) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged) {
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
                     // Ask host how big it wants to be
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);//开始测量

                    // Implementation of weights from WindowManager.LayoutParams
                    // We just grow the dimensions as needed and re-measure if
                    // needs be
                    int width = host.getMeasuredWidth();
                    int height = host.getMeasuredHeight();
                    boolean measureAgain = false;

                    if (lp.horizontalWeight > 0.0f) {
                        width += (int) ((mWidth - width) * lp.horizontalWeight);
                        childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
                                MeasureSpec.EXACTLY);
                        measureAgain = true;
                    }
                    if (lp.verticalWeight > 0.0f) {
                        height += (int) ((mHeight - height) * lp.verticalWeight);
                        childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
                                MeasureSpec.EXACTLY);
                        measureAgain = true;
                    }

                    if (measureAgain) {
                        if (DEBUG_LAYOUT) Log.v(TAG,
                                "And hey let's measure once more: width=" + width
                                + " height=" + height);
                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);//开始测量
                    }

                    layoutRequested = true;
                }
            }
        }

        //执行布局相关的逻辑
        if (didLayout) {
            performLayout(lp, desiredWindowWidth, desiredWindowHeight);//开始layout
            //....
        }
        //执行绘制逻辑
        if (!cancelDraw && !newSurface) {
            if (!skipDraw || mReportNextDraw) {
                if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                    for (int i = 0; i < mPendingTransitions.size(); ++i) {
                        mPendingTransitions.get(i).startChangingAnimations();
                    }
                    mPendingTransitions.clear();
                }
                performDraw();//开始绘制
            }
        }
    }

```

从ViewRootImpl的performTraversals方法中可以了解到的是这里执行了一个View从测量、布局到绘制的三大流程，我们这里只关注测量、布局、绘制的流程，对于不同情况下的测量的调用等不分析(主要也是还看不懂)

### 开始看测量流程，先看measureHierarchy方法的部分

```
	
	private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
 		//......
        boolean goodMeasure = false;
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            //做根View的尺寸调整
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }

            if (baseSize != 0 && desiredWindowWidth > baseSize) {
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);//根据模式和尺寸生成MeasureSpec
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                //下面再根据测量的结果决定是否需要拉伸根View
            }
        }
        //再测量
        if (!goodMeasure) {
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }

        return windowSizeMayChange;
    }

```

从上面的注释以及代码的逻辑里可以了解到的是这里是在为根布局的测量生成两个MeasureSpec实例，借助于WidowManager.LayoutParms以及承载View的Window等信息，然后调用performMeasure方法开始从根布局开始测量的流程,那么接着看performMeasure方法

```
	//mView即被绑定到ViewRootImpl中的DecorView
	private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);//根View的测量方法调用
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }

```

可以看到就一行核心代码-调用DecorView的measure(即View的measure)进行顶层View的尺寸测量，接着调用栈看measure方法

```

	//可以看到该方法是final的
	public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
		//第一步根据背景来调整MeasureSpec,这里我只看到InsetsDrawable存在返回，InsetsDrawable使得背景可以小于控件
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        //生成缓存尺寸的key
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

		//尺寸改变或者强制layout,则准备开始测量
        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;//清flag
            resolveRtlPropertiesIfNeeded();
            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);//取缓存
            if (cacheIndex < 0 || sIgnoreMeasureCache) {//没有缓存则直接测量
                onMeasure(widthMeasureSpec, heightMeasureSpec);//调用onMeasure进行测量
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);//直接设置
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
            //没有调用setDismension会抛异常...

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
        //保存数据，存缓存
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;
        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

```

上面的分析可以看到的是DecorView在执行onMeasure进行测量之前的会做一些背景尺寸的适配以及尺寸变化的检查等等,如果MeasureSpec没有发生变化着不会去测量,当然如果PFLAG_FORCE_LAYOUT设置了则会测量测量，接着再看DecorView的onMeasure方法，再看之前我们把View的默认测量逻辑分析一下

```

	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }

    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST://如果存在确定值则取MeasureSpec的值
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
    //根据背景以及控件的最小值来设置值
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }

```

上面View的onMeasure的方法可以清楚的看到控件大小的设置的逻辑，如果MeasureSpec.UNSPECIFIED模式则会根据取背景和minWidth的最大值，如果MeasureSpec提供了准确的值则直接使用

看完View的onMeasure方法在回过来看DecorView的onMeasure的测量方法，由于它继承自FrameLayout故以此为例分析*ViewGroup如何测量子View以及自身*

```

 	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        final DisplayMetrics metrics = getContext().getResources().getDisplayMetrics();
        final boolean isPortrait = metrics.widthPixels < metrics.heightPixels;//判断是否为竖屏

        final int widthMode = getMode(widthMeasureSpec);//取模式
        final int heightMode = getMode(heightMeasureSpec);

        boolean fixedWidth = false;
        //分别构造宽、高的MeasureSpec
        if (widthMode == AT_MOST) {//铺满父布局
            final TypedValue tvw = isPortrait ? mFixedWidthMinor : mFixedWidthMajor;
            if (tvw != null && tvw.type != TypedValue.TYPE_NULL) {
                //....计算宽度
                if (w > 0) {//如果得到了宽度则直接构造EXACTLY模式的MeasureSpec
                    final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
                    widthMeasureSpec = MeasureSpec.makeMeasureSpec(Math.min(w, widthSize), EXACTLY);
                    fixedWidth = true;
                }
            }
        }

        if (heightMode == AT_MOST) {//如果得到了高度则直接构造EXACTLY模式的MeasureSpec
            final TypedValue tvh = isPortrait ? mFixedHeightMajor : mFixedHeightMinor;
            if (tvh != null && tvh.type != TypedValue.TYPE_NULL) {
                //计算高度
                if (h > 0) {
                    final int heightSize = MeasureSpec.getSize(heightMeasureSpec);
                    heightMeasureSpec = MeasureSpec.makeMeasureSpec(Math.min(h, heightSize), EXACTLY);
                }
            }
        }
        //一下应该同View的Measure对Insets的处理，同样是对背景的尺寸处理
        getOutsets(mOutsets);
        if (mOutsets.top > 0 || mOutsets.bottom > 0) {
            int mode = MeasureSpec.getMode(heightMeasureSpec);
            if (mode != MeasureSpec.UNSPECIFIED) {
                int height = MeasureSpec.getSize(heightMeasureSpec);
                heightMeasureSpec = MeasureSpec.makeMeasureSpec(height + mOutsets.top + mOutsets.bottom, mode);
            }
        }
        if (mOutsets.left > 0 || mOutsets.right > 0) {
            int mode = MeasureSpec.getMode(widthMeasureSpec);
            if (mode != MeasureSpec.UNSPECIFIED) {
                int width = MeasureSpec.getSize(widthMeasureSpec);
                widthMeasureSpec = MeasureSpec.makeMeasureSpec(width + mOutsets.left + mOutsets.right, mode);
            }
        }

        super.onMeasure(widthMeasureSpec, heightMeasureSpec);//调用FrameLayout的onMeasure方法
        //......根据测量结果决定是够需要再测一次
        if (measure) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        }
    }

```

其实上面的一些没有正确测量重新测量的逻辑不需要过多关注，值得关注的是它根据上一级传过来的MeasureSpec来传给下一级的MeasureSpec的值,接着看FrameLayout的onMeasure方法

```

	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        final boolean measureMatchParentChildren =MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {//只测非GONE的View或者设置了全测
            	//测子View
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //记录最大的宽高
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {//宽或高需要充满屏幕
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        //....根据前景以及padding就算控件的宽高
        //根据MeasureSpec的模式以及测量得到的尺寸进行设置
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        count = mMatchParentChildren.size();
        if (count > 1) {//需要铺满布局的子View重新调整以及测量
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {//宽度需要铺满
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);//设置宽度为父布局宽度去掉padding以及margin后的
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {//不需要
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                //....类似以宽度，重新生成高的MeasureSpec

                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);//使用新的MesureSpec测量
            }
        }
    }

```

从上面的代码可以看到FrameLayout对子View的测量流程包括两层，首先是一般测量利用measureChildWithMargins，其次对一般测量中需要进行布局充满的View进行重新计算生成MeasureSpec,measureChildWithMargins代码如下

```

	protected void measureChildWithMargins(View child,int parentWidthMeasureSpec, int widthUsed,int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();//支持margin的LayoutParams
        //根据LayputParams来生成子View的MeasureSpec
        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }

```
上面的方法就是利用LayoutParams以及padding来生成子View测量需要的MeasureSpec由getChildMeasureSpec来生成，接着看该方法生成MeasureSpec的生成规则

```
	
	//注意这里的spec是要生成MeasureSpec的子View的父布局的MeasureSpec，padding是留白,
	public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);//父布局的尺寸以及模式
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);//去padding

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {//根据父布局的模式来决定子View的MeasureSpec
        case MeasureSpec.EXACTLY://父布局为定值
            if (childDimension >= 0) {//子View的尺寸确定
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {//子View希望充满父布局-定值
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {//子View不确定，则限制子View的最大宽度
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.AT_MOST://父布局有最大值
            if (childDimension >= 0) {//子View定值
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {//加限制不超过父布局
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {//同上
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;
        case MeasureSpec.UNSPECIFIED://父布局由子布局决定
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {//子View不定值
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }

```

上面的MeasureSpec的生成过程可以了解到子View的MeasureSpec由自身尺寸以及父布局的MeasureSpec共同来决定,我们在XML中设定的宽高规则不一定就是最终的我们在测量时获取的MeasureSpec中的规则

### Measure总结

首先整个测量过程中MeasureSpec是一个核心的作用,View以及ViewGroup通过层次的measure调用来完成整个视图树的测量(performTraversals->(measureHierarchy)->performMeasure->Measure->onMeasure->Measure->onMeasure->......),而每个MeasureSpec的生成规则主要由**父布局的MeasureSpec以及子View的LayoutParams共同决定**。所以当我们在onMeasure中测量时需要自己根据内容或者需求来自己处理MeasureSpec.UNSPECIFIED、MeasureSpec.AT_MOST的情况，这种情况下MeasureSpec中的尺寸值只是表示一个约束。






