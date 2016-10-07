#Android Transitions Framework入门#

##一、基本概念##

1、作用：Transitions Framework用于构建平滑的视图切换动画

2、核心类:

* Scene类:是对视图树状态的抽象，是视图组属性的集合，如其名是一个场景
* Transition类：切换动画的抽象

3、限制:

* SurfaceView、TextureView上无法正常作用
* 继承自AdapterView的控件无法正常作用，需要避免使用

##二、Scene的创建##

* 从资源文件创建Scene-如果视图组比较稳定，即没有较多的控件的添加和移除

>public static Scene getSceneForLayout (ViewGroup sceneRoot, int layoutId, Context context)

**参数说明:**

SceneRoot:Scene场景的根布局

layoutId:对应场景的布局文件的id，即这种方法只能对整个布局文件进行创建Scene

同时这种方式创建的Scene的布局视图组不能改变，如果改变需要重新创建

* 使用构造函数创建-这种适合用于需要动态创建View改变视图组的时候使用

>public Scene (ViewGroup sceneRoot, ViewGroup layout)

**参数说明:**

SceneRoot:Scene场景的根布局，每次场景进入时移除sceneRoot所有子View，将layout对象作为子View添加到sceneRoot中

layoutId:对应场景的View对象

此外我们还可以利用setEnterAction、setExitAction设置一些场景进入或者退出时伴随的一些其他操作


##三、Transition的创建##

* 由资源文件创建(res/transition)-使用TransitionInflater静态方法进行创建,类似动画的xml创建方式
* 使用对应的Transition的子类的构造方法动态创建

###相关的类###

1、transitionSet类:可以用于创建一个切换动画的集合，xml资源中也有相应的标签
2、TransitionListener接口：用于对Transition的生命周期进行监听


##四、Scene和Transiton的应用##

1、直接调用Scene的enter方法进行场景切换,这种方式使用默认的Transition

2、使用TransitionManager的go方法进行场景切换

3、不需要的场景Transition动画-这种方式用于动态的添加、删除View时添加动画，如下以一个淡入的方式添加一个View

```

	/**
     * @param parent 父ViewGroup,可以是非直接父View
     * @param child 子View
     */
    public void addViewWithTransition(ViewGroup parent,View child){
        Fade fade=new Fade(Fade.MODE_IN);
        TransitionManager.beginDelayedTransition(parent,fade);
        parent.addView(child);
    }

```

再接着是一个点击搜索切换视图的程序段

```
	
	public void click(View view){
        if(flag) {
            TransitionManager.beginDelayedTransition(mSceneRoot,mFade);//准备切换动画
            findViewById(R.id.scene_container1).setVisibility(View.GONE);
            LayoutInflater.from(this).inflate(R.layout.another_scene,mSceneRoot,true);
        }
        else{
            TransitionManager.beginDelayedTransition(mSceneRoot,mFade);
            findViewById(R.id.scene_container2).setVisibility(View.GONE);
            LayoutInflater.from(this).inflate(R.layout.a_scene,mSceneRoot,true);
        }
        flag=!flag;
    }

    //对应的动画效果，其中的fade动画效果，在TransitionManager中使用go方法做场景切换无法生效
    //根据setVisibility来启动
    <transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    	android:transitionOrdering="sequential"
    	android:duration="500">
    	<fade android:fadingMode="fade_out" android:duration="500" />
    	<changeBounds/>
    	<fade android:fadingMode="fade_in" android:duration="500"/>
	</transitionSet>

```

##五、作用目标设置##

在一中提到Transition的限制时提高部分控件无法适应与Transition FrameWork,所以当这些控件存在时需要避免对其产生效果,Transition类以及xml中中提供了相应API和标签来设置作用的对象或者设置需要剔除的对象

##六、自定义Transition##

主要是继承Transition方法，并实现captureStartValues、captureEndValues、createAnimator三个方法,以Visibility类为例

```
	@Override
    public void captureStartValues(TransitionValues transitionValues) {
        captureValues(transitionValues);//记录起始属性
    }

    @Override
    public void captureEndValues(TransitionValues transitionValues) {
        captureValues(transitionValues);//记录结束属性
    }
    //只关注需要操作和改变的属性
    private void captureValues(TransitionValues transitionValues) {
        int visibility = transitionValues.view.getVisibility();
        transitionValues.values.put(PROPNAME_VISIBILITY, visibility);
        transitionValues.values.put(PROPNAME_PARENT, transitionValues.view.getParent());
        int[] loc = new int[2];
        transitionValues.view.getLocationOnScreen(loc);
        transitionValues.values.put(PROPNAME_SCREEN_LOCATION, loc);
    }

```

有了View的起始数据之后就在createAnimator利用该数据构造属性动画返回即可，也就是说Transition底层也是使用的属性动画来实现的

##七、Activity中的Transition##

在Activity中可以使用Transition来共享元素，一个类似与SDK中sample的简化版例子如下

***1、在style中设置Window的transition***

```

	<item name="android:windowSharedElementEnterTransition">@transition/activity_trans</item>
    <item name="android:windowSharedElementExitTransition">@transition/activity_trans</item>

    //对应的transition资源文件
    <transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
    	<changeBounds/>
    	<changeImageTransform/>
	</transitionSet>

```

***2、跳转Activity前准备好共享元素***

```
	//点击事件
	public void click(View view){
        Intent intent = new Intent(this, DetailActivity.class);
        int index=1;
        //根据view确定index数据
        intent.putExtra(DetailActivity.EXTRA_ID, index);
        //传递共享元素，主要在于Pair中传的共享View和对应的tag
        ActivityOptionsCompat activityOptions = ActivityOptionsCompat.makeSceneTransitionAnimation(this,
                new Pair<View, String>(findViewById(R.id.imv1),DetailActivity.VIEW_IMAGE));
        //传递数据，跳转Activity
        ActivityCompat.startActivity(this, intent, activityOptions.toBundle());
    }

    //Activity的对应资源文件
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    	xmlns:tools="http://schemas.android.com/tools"
    	android:layout_width="match_parent"
    	android:layout_height="match_parent"
    	android:orientation="vertical">
    	<ImageView
        	android:id="@+id/imv1"
	        android:onClick="click"
	        android:layout_width="100dp"
	        android:layout_height="100dp"
	        android:src="@mipmap/a1"
	        />
	    <ImageView
	        android:id="@+id/imv2"
	        android:onClick="click"
	        android:layout_width="100dp"
	        android:layout_height="100dp"
	        android:src="@mipmap/a2"/>
	</LinearLayout>

```

***3、下一个Activity接收数据***

```

	protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.detail_layout);
        mImage= (ImageView) findViewById(R.id.imv);
        TextView tv= (TextView) findViewById(R.id.id_text);
        index=getIntent().getIntExtra(EXTRA_ID,1);//获取数据
        if(index==1){//首先加载小图，以便平滑过渡
            mImage.setImageResource(R.mipmap.a1_1);
        }
        else{
            mImage.setImageResource(R.mipmap.a2_2);
        }
        tv.setText("THIS IS IMAGE "+index);
        ViewCompat.setTransitionName(mImage, VIEW_IMAGE);//设置共享的元素tag，用于在Transition标识View
        loadImage();//设置Transition动画监听,加载数据
    }
    public void loadImage(){
        final Transition transition = getWindow().getSharedElementEnterTransition();
        transition.addListener(new Transition.TransitionListener() {
            //结束时加载完整数据
            public void onTransitionEnd(Transition transition) {
                if(index==1){
                    mImage.setImageResource(R.mipmap.a1);
                }
                else{
                    mImage.setImageResource(R.mipmap.a2);
                }
                transition.removeListener(this);
            }
            //..................
        });
    }

```

总的来说元素共享的步骤如下:添加window的style属性,transition资源文件->当前Activity通过options准备共享View的tag和View,通过静态方法启动Activity->启动的Activity接受数据，设置共享View的tag->根据实际在特定时机下加载数据

