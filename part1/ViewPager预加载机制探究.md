#ViewPager预加载机制探究#
---

##一、ViewPager以及相关类的作用##

* ViewPager:继承自viewGroup,该布局允许用户向左右滑动数据页
* PagerAdapter:该抽象类是ViewPager的适配器类，用于向ViewPager提供数据的内容UI，标题，直接子类为FragmentPagerAdapter、FragmentStatePagerAdapter
* PagerTitleStrip、PagerTabStrip:可以用于为ViewPager提供标题UI，而标题数据由Adapter提供，其中前者是后者的父类，其中他们在attachActivity时绑定父ViewGroup即ViewPager，并设置监听来获取标题信息

一个原始的例子如下
```
	//一个简单的Adapter的实现
	public class MainViewPagerAdapter extends FragmentPagerAdapter {
	    public static final String []mMainTitle={"社交","工具","欣赏"};
	    private Fragment[]mContFragment;
	    public MainViewPagerAdapter(FragmentManager fm) {
	        super(fm);
	        mContFragment=new Fragment[3];
	        //初始化几个Fragment，仅仅只有背景色不同
	        mContFragment[0]=new SocialFragment();
	        mContFragment[1]=new ToolsFragment();
	        mContFragment[2]=new PictureFragment();
	    }

	    @Override
	    public Fragment getItem(int position) {
	        Log.e("tag","Adapter getItem executed");
	        return mContFragment[position];
	    }

	    @Override
	    public int getCount() {
	        return mMainTitle.length;
	    }

	    @Override
	    public CharSequence getPageTitle(int position) {
	        super.getPageTitle(position);
	        return mMainTitle[position];//提供每页的标题
	    }
	}
	//将Adapter设置到ViewPager
	protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mViewPager= (ViewPager) findViewById(R.id.view_pager);
        MainViewPagerAdapter adapter=new MainViewPagerAdapter(getSupportFragmentManager());
        mViewPager.setAdapter(adapter);
    }

```

效果如下

![](https://github.com/getletCodes/StudyNotes/blob/master/part1/view_pager_demo.gif)

##二、PagerAdapter中基本方法介绍##

* 实现一个PagerAdapter需要实现的基本方法:
	* 抽象方法getCount():返回ViewPager数据页的数量
	* instantiateItem(ViewGroup, int)方法:该类是初始化方法，在这里应该初始化item，并加入到container中，同时注意的是其返回值的意义，ViewPager利用key-object的方法使用一个key来标识其引用的是哪一个item，一般的实线就是利用item本身作为key
	* isViewFromObject(View, Object)方法:用于判断当前的key与view是否是关联的
	* destroyItem(ViewGroup, int, Object)方法：在此处应将item从容器中移除
* FragmentPagerAdapter、FragmentStatePagerAdapter的区别:前者一般用于少量页面展示，它会使用FragmentManager持久化每个Fragment，而后者一般用于复杂、数量较多不定的Fragment页面展示，会使用回调机制来做一个页面的回收重建

##三、ViewPager的深入了解##

* ViewPager的onMeasure、onLayout流程:

	```

	@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 首先简单的默认设置为0或者根据容器的指定尺寸设置，此时无法知道子item的大小，只有添加到之后才能知道
        setMeasuredDimension(getDefaultSize(0, widthMeasureSpec),
                getDefaultSize(0, heightMeasureSpec));
        final int measuredWidth = getMeasuredWidth();
        final int maxGutterSize = measuredWidth / 10;
        mGutterSize = Math.min(maxGutterSize, mDefaultGutterSize);

        // 每个item的大小恰好是去掉padding的大小
        int childWidthSize = measuredWidth - getPaddingLeft() - getPaddingRight();
        int childHeightSize = getMeasuredHeight() - getPaddingTop() - getPaddingBottom();

        /*
         * 接下来是确保每个item都能都被正确测量，而且decor优先，同时下面的代码还直接认为decor不会相交
         */
        int size = getChildCount();
        for (int i = 0; i < size; ++i) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                if (lp != null && lp.isDecor) {//装饰View实现了ViewPager.DecorView注解
                	//中间的这些代码是用来测量每个装饰View的大小，然后减去装饰View的大小
                	//作为item的尺寸
                    if (consumeVertical) {
                        childHeightSize -= child.getMeasuredHeight();
                    } else if (consumeHorizontal) {
                        childWidthSize -= child.getMeasuredWidth();
                    }
                }
            }
        }
        //确定子item的尺寸
        mChildWidthMeasureSpec = MeasureSpec.makeMeasureSpec(childWidthSize, MeasureSpec.EXACTLY);
        mChildHeightMeasureSpec = MeasureSpec.makeMeasureSpec(childHeightSize, MeasureSpec.EXACTLY);

        // Make sure we have created all fragments that we need to have shown.
        mInLayout = true;
        //这个方法做的事情有点复杂，主要是与adapter相关，确定子item的绘制顺序，还有确定加载item的个数
        populate();
        mInLayout = false;

        // 在这里测量每个item的尺寸
        size = getChildCount();
        for (int i = 0; i < size; ++i) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
            	//只有当child没有被attach到ViewGroup时返回null
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                //可以知道的是ViewPager将修改View的高度，每个item的高度必然是相同的，而对于宽度则可以通过反射修改
                //lp.widthFactor(**一般是1**)的值使得ViewPager的item的宽不一样
                if (lp == null || !lp.isDecor) {
                    final int widthSpec = MeasureSpec.makeMeasureSpec(
                            (int) (childWidthSize * lp.widthFactor), MeasureSpec.EXACTLY);
                    child.measure(widthSpec, mChildHeightMeasureSpec);
                }
            }
        }
    }


    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int count = getChildCount();
        int width = r - l;
        int height = b - t;
        int paddingLeft = getPaddingLeft();
        int paddingTop = getPaddingTop();
        int paddingRight = getPaddingRight();
        int paddingBottom = getPaddingBottom();
        final int scrollX = getScrollX();

        int decorCount = 0;

        //第一步确定Decor View的位置
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
               	//这里的逻辑主要根据装饰view的gravity来放置其位置
                    childLeft += scrollX;
                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                    decorCount++;
                }
            }
        }

        final int childWidth = width - paddingLeft - paddingRight;
        // 确定 page的位置，可以看出来这里的逻辑很简单，根据padding值和page的位置来放置每个page
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                ItemInfo ii;
                if (!lp.isDecor && (ii = infoForChild(child)) != null) {
                    int loff = (int) (childWidth * ii.offset);
                    int childLeft = paddingLeft + loff;
                    int childTop = paddingTop;
                    if (lp.needsMeasure) {
                        //这里是一个测量的逻辑，如果没有测量过item
                    }

                    child.layout(childLeft, childTop,
                            childLeft + child.getMeasuredWidth(),
                            childTop + child.getMeasuredHeight());
                }
            }
        }
        mTopPageBounds = paddingTop;
        mBottomPageBounds = height - paddingBottom;
        mDecorChildCount = decorCount;
        //第一放置，滑动到指定位置
        if (mFirstLayout) {
            scrollToItem(mCurItem, false, 0, false);
        }
        mFirstLayout = false;
    }

	```

	**从上面两个方法来看,ViewPager会强制每个page item充满自身，而且强制每个page item的尺寸相同，但是如果我们能够在外部提供每个子page View的尺寸信息，我们仍然可以通过在外部使用ViewPager的LayoutParams变量来动态的让ViewPager来自适应子Pager 的尺寸。此外ViewPager中存在一个sortChildDrawingOrder()方法，ViewPager会根据每个item是否为decor view、距离的远近的来决定绘制顺序，通过覆写了ViewGroup的getChildDrawingOrder方法**

* 从ViewPager的setAdapter开始分析加载UI的机制:

	```

		public void setAdapter(PagerAdapter adapter) {
	        if (mAdapter != null) {
	            //如果之前adapter，
	        }

	        final PagerAdapter oldAdapter = mAdapter;
	        mAdapter = adapter;
	        mExpectedAdapterCount = 0;

	        if (mAdapter != null) {
	            if (mObserver == null) {
	                mObserver = new PagerObserver();
	            }
	            mAdapter.setViewPagerObserver(mObserver);//设置数据变化的监听
	            mPopulatePending = false;
	            final boolean wasFirstLayout = mFirstLayout;
	            mFirstLayout = true;
	            mExpectedAdapterCount = mAdapter.getCount();
	            if (mRestoredCurItem >= 0) {//与FragmentStatePagerAdapter的缓存机制有关
	                mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
	                //这里是设置当前显示的Item
	                setCurrentItemInternal(mRestoredCurItem, false, true);
	                mRestoredCurItem = -1;
	                mRestoredAdapterState = null;
	                mRestoredClassLoader = null;
	            } else if (!wasFirstLayout) {//标记ViewPager是否是第一次加载
	                populate();
	            } else {
	                requestLayout();//请求layout重绘
	            }
	        }
	        //最后如果设置的Adapter的监听且两个adapter不相同回调监听方法
	        if (mAdapterChangeListener != null && oldAdapter != adapter) {
	            mAdapterChangeListener.onAdapterChanged(oldAdapter, adapter);
	        }
	    }

	    /**
	    *在进行下一步之前，先看一下populate的实现，从名字来看它是填充的意思,其次是ItemInfo顾名思义它是关于page item信息的
	    *一个bean类，它包括item的key、位置、滑动、偏移、宽度系数的信息
	    **/
	    //populate方法中使用mCurItem参数调用了重载方法
	    void populate(int newCurrentItem) {
	        ItemInfo oldCurInfo = null;
	        if (mCurItem != newCurrentItem) {//与当前item比较，获取上一个item的ItemInfo
	            oldCurInfo = infoForPosition(mCurItem);
	            mCurItem = newCurrentItem;
	        }
	        //adapter为空则只是简单的调用sortChildDrawingOrder()方法对child进行绘制排序，然后return
	        //当用户手指抬起页面仍在滑动时做同样的时，没有attach到window直接返回

	        //空实现，只是表明ViewPager的内容变化，将会多次调用adapter的 instantiateItem、 destroyItem方法
	        mAdapter.startUpdate(this);
	        //一下部分代码为确定填充的item的position，默认是填充当前item的左右各一屏数据
	        final int pageLimit = mOffscreenPageLimit;
	        final int startPos = Math.max(0, mCurItem - pageLimit);
	        final int N = mAdapter.getCount();
	        final int endPos = Math.min(N-1, mCurItem + pageLimit);

	        //略...

	        // 搜索当前的mCurItem位置的page View的ItemInfo
	        int curIndex = -1;
	        ItemInfo curItem = null;
	        for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
	            final ItemInfo ii = mItems.get(curIndex);
	            if (ii.position >= mCurItem) {
	                if (ii.position == mCurItem) curItem = ii;
	                break;
	            }
	        }
	        //当前的mCurItem位置没有item对象则创建一个并加入到mItems集合中，curIndex是加在mItems中的索引
	        //mCurItem则是当前显示的page的序号
	        if (curItem == null && N > 0) {
	            curItem = addNewItem(mCurItem, curIndex);
	        }
	        //如果当前的ItemInfo获取的不为null,则是填重三屏数据或者根据offscreen的值来决定
	        if (curItem != null) {
	            float extraWidthLeft = 0.f;//从下面的分析来看，这个参数确定的是当前需要加载的item占的宽度比
	            int itemIndex = curIndex - 1;//由左边第一个ItemInfo开始
	            ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	            final int clientWidth = getClientWidth();//获取ViewPager的page宽度
	            //leftWidthNeeded需要额外加载的item所应占的宽度比，一旦extraWidthLeft>leftWidthNeeded停止加载
	            final float leftWidthNeeded = clientWidth <= 0 ? 0 :
	                    2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
	            //位置索引循环
	            for (int pos = mCurItem - 1; pos >= 0; pos--) {
	            	//如果已经加载的宽度大于需要且位置小于左侧开始位置
	                if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
	                    if (ii == null) {
	                        break;
	                    }
	                    if (pos == ii.position && !ii.scrolling) {//不需要展示的则移除
	                        mItems.remove(itemIndex);
	                        mAdapter.destroyItem(this, pos, ii.object);
	                        itemIndex--;
	                        curIndex--;
	                        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	                    }
	                } else if (ii != null && pos == ii.position) {//ItemInfo存在，使用的宽度增加
	                    extraWidthLeft += ii.widthFactor;
	                    itemIndex--;
	                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	                } else {
	                    ii = addNewItem(pos, itemIndex + 1);//如果左边第一个还没有ItemInfo,为当前位置添加一个
	                    extraWidthLeft += ii.widthFactor;
	                    curIndex++;
	                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
	                }
	            }

	            //接下来的则是从中间向右构建Iteminfo,逻辑与上面的从中间向左的类似
	            ....
	            //该方法的作用是计算每个page 的偏移量，存于offset中
	            calculatePageOffsets(curItem, curIndex, oldCurInfo);
	        }
	        //告知适配器当前的mCurItem为要显示的item
	        mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);

	        mAdapter.finishUpdate(this);//标记改变已经结束

	        //重新调用显示排序方法，更新child的LayoutParams参数
	    }

	    //这里接着看一下当ItemInfo缺失时，调用的addNewItem方法
	    ItemInfo addNewItem(int position, int index) {
	        ItemInfo ii = new ItemInfo();
	        ii.position = position;
	        //可以看到这里将ItemInfo与UI对象进行了绑定，即这里进行了预加载的操作
	        ii.object = mAdapter.instantiateItem(this, position);
	        ii.widthFactor = mAdapter.getPageWidth(position);
	        if (index < 0 || index >= mItems.size()) {
	            mItems.add(ii);
	        } else {
	            mItems.add(index, ii);
	        }
	        return ii;
	    }

	```

	**从上面的分析中我们就可以对ViewPager的预加载机制有一个了解，ViewPager会默认加载当前显示的Item以及预加载其左右的item，但是一个重要的问题就是，它在预加载时使用了两个参数一个是item的宽度比(或者说是宽度)、一个是mOffscreenPageLimit参数，但是我们仍然无法通过将后者设置为0来避免预加载机制，因为ViewPager只有在前者的数值到达预定目标才会去比较接下来的item的position是否已经超过了mOffscreenPageLimit，此外由宽度比这个参数的比较对象在populate方法中是一个局部变量，所以我们无法通过继承来修改预加载机制，所以只能从Fragment入手来做或者自定义ViewPager重写populate方法重新设置改逻辑**

	接着从onTouchEvent方法来看，populate(填充UI数据)、setCurrentItemInternal(设置当前page)是ViewPager的核心方法，尤其是前者，ViewPager严重依赖于这个方法

* ViewPager与FragmentStatePagerAdapter的回收缓存机制

	从ViewPager的数据存储、恢复的回调中可以看到的是它调用的Adapter的数据存储、恢复方法

	```

		@Override
	    public Parcelable onSaveInstanceState() {
	        Parcelable superState = super.onSaveInstanceState();
	        SavedState ss = new SavedState(superState);
	        ss.position = mCurItem;
	        if (mAdapter != null) {
	            ss.adapterState = mAdapter.saveState();//存储数据
	        }
	        return ss;
	    }

	    @Override
	    public void onRestoreInstanceState(Parcelable state) {
	        if (!(state instanceof SavedState)) {
	            super.onRestoreInstanceState(state);
	            return;
	        }

	        SavedState ss = (SavedState)state;
	        super.onRestoreInstanceState(ss.getSuperState());

	        if (mAdapter != null) {
	            mAdapter.restoreState(ss.adapterState, ss.loader);//恢复数据
	            setCurrentItemInternal(ss.position, false, true);
	        } else {
	            mRestoredCurItem = ss.position;
	            mRestoredAdapterState = ss.adapterState;
	            mRestoredClassLoader = ss.loader;
	        }
	    }
	    //再看FragmentStatePagerAdapter的destroy方法的实现
	    @Override
	    public void destroyItem(ViewGroup container, int position, Object object) {
	        Fragment fragment = (Fragment) object;

	        if (mCurTransaction == null) {
	            mCurTransaction = mFragmentManager.beginTransaction();
	        }
	        while (mSavedState.size() <= position) {
	            mSavedState.add(null);
	        }
	        mSavedState.set(position, fragment.isAdded()
	                ? mFragmentManager.saveFragmentInstanceState(fragment) : null);//这里存储了当前移除的Fragment的状态
	        mFragments.set(position, null);

	        mCurTransaction.remove(fragment);//移除Fragment
	    }
	    //在看UI对象的初始化方法
	    public Object instantiateItem(ViewGroup container, int position) {
	    	//如果当前Fragment仍然是持久化的则直接返回
	    	if (mFragments.size() > position) {
	            Fragment f = mFragments.get(position);
	            if (f != null) {
	                return f;
	            }
	        }
	        //...
	        //否则创建一个
	        Fragment fragment = getItem(position);
	       	//如果缓存中有Fragment的状态，则恢复状态到新创建的Fragment上
	        if (mSavedState.size() > position) {
	            Fragment.SavedState fss = mSavedState.get(position);
	            if (fss != null) {
	                fragment.setInitialSavedState(fss);
	            }
	        }

	        //...

	        return fragment;
	    }

	```

