# View的测量以及绘制流程简要分析二
---
## layout流程分析

同measure的起点一样,layout的流程也是从ViewRootImpl的performTraversals中调用performLayout开始

```

	private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;

        final View host = mView;
        //...

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
            //传入根布局的矩形区域进行layout
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;
            int numViewsRequestingLayout = mLayoutRequesters.size();
            //如果在layout过程中有View调用了requestLayout方法，则该View会被加入mLayoutRequesters集合中
            if (numViewsRequestingLayout > 0) {
                //检查Request标记删选合理的View
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    mHandlingLayoutInLayoutRequest = true;

                    //刷新layout，重新测量、布局
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        view.requestLayout();
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);//重新测量
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());//layout

                    mHandlingLayoutInLayoutRequest = false;
                    //如果还有则留到下次刷新时再处理
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }

```

上面代码表示正常逻辑下从DecorView的layout方法开始进行布局，传入的参数也就是measure中测量到的尺寸信息,而对于layout时子View调用request方法导致的重新测量、布局则不讨论.接着看DecorView的layout方法，不过DecorView并没有layout方法，这是由于其父类ViewGroup中将layout方法声明为final，所以DecorView的layout方法即ViewGroup的layout方法,如下

```

    //可以了解执行Transition动画时，不做layout
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }

```

layout流程中上面ViewGroup的核心代码就一行直接调用View的layout方法,View的layout代码如下

```

    public void layout(int l, int t, int r, int b) {
        //部分情况需要重新测量
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        //分配自身布局空间，并检查是否产生了变化、取消缓存
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);//调用onLayout方法要求进行布局
            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            //layout完成后的回调，onLayoutChange
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
    }

```

由代码可知View在layout方法中分配了可以用于layout的区域后便调用onLayout方法，将设置子View的布局权移交给onLayout方法，而该方法在View类里是一个空实现，对应DecorView这里就是FrameLayout的实现，以此为例说明ViewGroup的布局子View的过程,FrameLayout的onLayout如下

```

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();
        //首先计算可用空间，注意在这里坐标变换为子View使用的相对与父View的相对坐标
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;

                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }

                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
                //以下根据Grivaty计算每个子View的位置计算

                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }

                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }
                //子View分配确定自身空间，如果ViewGroup递归布局其子View
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }

```

由上面的代码可以知道如果没有Gravity属性的设置，layout的方法将及其简单，就是只需要考虑一些padding值然后调用子View的layout方法让其分配自身空间，或者进一步布局自己的子View的位置

## layout流程总结

与onMeasure的测量过程相比较，由于measure提供了尺寸信息，layout流程要相对简单一点：一个View在layout方法中确定自身可用空间->在onLayout方法中使用相对坐标确定自己的子View的相对位置，这里的复杂性则依据ViewGroup的复杂性，比如对于规则较简单的FrameLayout的比较简单

## View绘制流程分析

同理由performTraversals中调用的performLayout开始看

```

    private void performDraw() {
        //...

        final boolean fullRedrawNeeded = mFullRedrawNeeded;
        mFullRedrawNeeded = false;

        mIsDrawing = true;
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "draw");
        try {
            draw(fullRedrawNeeded);
        } finally {
            mIsDrawing = false;
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        //...关于硬件渲染、窗口绘制之类的略
    }

```

上面的代码几乎直接调用了ViewRootImpl的draw方法,直截取关键的代码如下

```

    private void draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;//同样也是在Surface上绘图
        //....

        mAttachInfo.mTreeObserver.dispatchOnDraw();//通知绘制开始
        
        //...
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty)) {
            return;
        /...

    }

```

上面的代码可以知道View的本身也是在SurfaceView上被绘制，其次是绘制之前的额一些回调以及其他的一些操作，最后通过drawSoftware开始绘制流程

```
    
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        final Canvas canvas;//draw方法的Canvas参数声明
        try {
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;

            canvas = mSurface.lockCanvas(dirty);//Canvas参数的创建,只重绘dirty区域
            //...

            //mDensity = context.getResources().getDisplayMetrics().densityDpi;
            canvas.setDensity(mDensity);//设置密度
        } 
        //....

        t//...

        mView.draw(canvas);//开始从根View开始绘图
        //...
        return true;
    }

```

接着上述流程，进入DecorView的draw方法,即View的draw方法

```

    @CallSuper
    public void draw(Canvas canvas) {
        //.......

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);//绘制背景
        }

        // skip step 2 & 5 if possible (common case)第二步到到第五步可能跳过常见情况
        //...这里是一个完整的绘制流程，但是根据注释一般情况跳过

        //以下是一个完整绘制流程

        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;

        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;

        // Step 2, save the canvas' layers
        //......计算绘制区域

        saveCount = canvas.getSaveCount();//返回当前层的编号

        int solidColor = getSolidColor();//一般返回0，即canvas将创建新图层
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // Step 3, draw the content
        if (!dirtyOpaque) 
            onDraw(canvas);//绘制内容，有上面的判断，一般在新图层中绘制再合并

        // Step 4, draw the children
        dispatchDraw(canvas);//绘制子View,ViewGroup会在该方法中调用drawChild方法调用子View的draw方法

        // Step 5, draw the fade effect and restore layers
        //......绘制渐变效果

        canvas.restoreToCount(saveCount);//恢复合并图层

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);//绘制装饰
    }

```

draw方法开始处的注释对draw的步骤有一个很清楚的描述如下:

#### Draw traversal performs several drawing steps which must be executed in the appropriate order

1. Draw the background-绘制背景
2. If necessary, save the canvas' layers to prepare for fading-有需要时，保存图层
3. Draw view's content-绘制View的内容
4. Draw children-绘制子View
5. If necessary, draw the fading edges and restore layers-有需要时，绘制fading边缘，绘制图层
6. Draw decorations (scrollbars for instance)-绘制装饰，比如滚动条

所以绘制的流程就是：背景->自身内容->绘制子View->绘制fade效果->绘制装饰View,其中中间的四个步骤通常情况下在新的图层绘制

## 总结

View的测量、布局、绘制我觉得最重要的部分就是measur的过程，这个过程一个重要的类就是MeasureSpec，后面的布局过程依赖与它的结果.三个过程的方法调用流程如下

measure:performTraversals->(measureHierarchy)->performMeasure->Measure->onMeasure->Measure->onMeasure->......)

layout:performTraversals->performLayout->layout->(onLayout->layoput->.....)

onDraw:performTraversals->performDraw->draw(ViewRootImpl)->drawSoftware->draw(View)->onDraw->(dispatchDraw->draw->.......)



