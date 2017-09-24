# Fragment简析
---
从Android引入Fragment开始，在Android中使用的场景也越来越多，但是我们会发现Fragment相对于View来说实在坑有点多，这篇主要对Fragment做一个较深一点的分析(太深估计也不太行)，主要讨论Fragment与Activity的生命周期如何联动、Activity如何管理Fragment等话题

## 一、Fragment基本使用以及过程分析

从Fragment创建添加到Activity开始看

```
	FragmentManager manager=getSupportFragmentManager();//这里最终获得一个FragmentManagerImpl类型的对象
    FragmentTransaction fragmentTransaction=manager.beginTransaction();//开启事务，return new BackStackRecord(this)
    MyFragment myFragment=new MyFragment();
    fragmentTransaction.add(R.id.container,myFragment,"tag");//添加Fragment到Activity
    fragmentTransaction.addToBackStack(null);
    fragmentTransaction.commit();//提交事务
```

简单的说要想添加一个Fragment到FragmentActivity中，我们需要首先获取它的FragmentManager，然后需要开启Fragment事务，接着添加Fragment，最后提交事务让操作生效,一步步来看

首先FragmentManager什么时候被创建

```
	//获取FragmentManager，其中FragmentActivity中成员变量mFragments类型为FragmentController
	public FragmentManager getSupportFragmentManager() {
        return mFragments.getSupportFragmentManager();
    }
    //FragmentController的成员变量mHost数据类型为FragmentHostCallback
    public FragmentManager getSupportFragmentManager() {
        return mHost.getFragmentManagerImpl();
    }
    //最终返回位于FragmentHostCallBack类中的FragmentManagerImpl的实现对象
    FragmentManagerImpl getFragmentManagerImpl() {
        return mFragmentManager;
    }
```

从FragmentManager的获取可以看到Activity不通过FragmentManager来管理，而是通过FragmentController来管理，所以再看FragmentController的创建过程如下

```
    final FragmentController mFragments = FragmentController.createController(new HostCallbacks());//变量声明时创建管理类
    //HostCallback类声明以及构造方法如下,文档中
    class HostCallbacks extends FragmentHostCallback<FragmentActivity> {
        public HostCallbacks() {
            super(FragmentActivity.this);
        }
    }
    //FramentController的创建过程如下
    public static final FragmentController createController(FragmentHostCallback<?> callbacks) {
        return new FragmentController(callbacks);
    }
    private FragmentController(FragmentHostCallback<?> callbacks) {
        mHost = callbacks;//将HostCallback与之关联
    }

```

从代码来看Activity管理Fragment与FragmentController、HostCallbacks、FragmentManager这三个类息息相关，文档对它们的说明如下

HostCallbacks：继承自FragmentHostCallback，后者是对Fragment的宿主的封装类(任何对象都可能成为Fragment的宿主)，典型的宿主是Activity

FragmentController:主要集成FragmentManager的功能，向Fragment的宿主提供管理Fragment的能力

FragmentManager:其实现类FragmentManagerImpl文档说明其为Fragment的容器,充当管理Fragment的角色

next step 看事务的开启

```
	//直接new 了一个BackStackRecord
    public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }
```

对于BackStackRecord对象(继承自FragmentTransaction)文档的解释是Entry of an operation on the fragment back stack-fragment返回栈的操作条目，仔细看这个类我们可以发现这个定义了相当多的动作，比如OP_ADD、OP_REPLACE等Fragment的添加、替换等操作，我认为该类是对命令的封装，代表这对于Fragment的一系列操作。同时我们也可以知道一个事务的操作都是被记录在BackStackRecord中，即一个BackStackRecord对象代表着一个事务。

next step-BackStackRecord的add方法添加Fragment，也就是一系列在BackStackRecord上的操作

```
    public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
        doAddOp(containerViewId, fragment, tag, OP_ADD);//OP_ADD表示添加Fragment的操作
        return this;
    }

    private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        final Class fragmentClass = fragment.getClass();
        final int modifiers = fragmentClass.getModifiers();
        //检查Fragment的类声明，不能是匿名类、非共有类、如果是内部类必须是静态的，代码略
        fragment.mFragmentManager = mManager;//这里我们创建的Fragment和FragmentManager进行了绑定
        //如果Fragment已经又tag则不能再添加tag，代码略

        if (containerViewId != 0) {
            //可以看到这种添加方式使得fragment的contanierId和fragmentId成为一个，宿主id
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }
        Op op = new Op();//Fragment的操作类
        op.cmd = opcmd;
        op.fragment = fragment;
        addOp(op);//插入到操作的链表中
    }
```

从上述代码看当我们添加一个Fragment到Activity中时，Fragment不会被立即进入生命周期方法的调用，而是先将Fragment与Activity的FragmentManager进行绑定，然后为添加Fragment中这个操作生成了一个命令对象Op,添加时使用数字来表示对应的操作

next step-可选步骤添加回退栈

```
	只是简单的置回退状态为true，保存传入的字符串
	public FragmentTransaction addToBackStack(String name) {
        //....
        mAddToBackStack = true;
        mName = name;
        return this;
    }
```

final step-提交事务

```
    public int commit() {
        return commitInternal(false);
    }
    int commitInternal(boolean allowStateLoss) {
    	//可以看到事务不能提交两次
        if (mCommitted) throw new IllegalStateException("commit already called");
        mCommitted = true;
        if (mAddToBackStack) {
            mIndex = mManager.allocBackStackIndex(this);
        } else {
            mIndex = -1;
        }
        mManager.enqueueAction(this, allowStateLoss);//将事务提交
        return mIndex;
    }
    //如文档所解释将一个任务加入到队列中，同时可以看到BackStackRecord实现了Runnable接口
    public void enqueueAction(Runnable action, boolean allowStateLoss) {
        if (!allowStateLoss) {//首先需要检查Activity状态，以决定是否可以提交
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<Runnable>();
            }
            mPendingActions.add(action);//添加任务
            if (mPendingActions.size() == 1) {//如果只有当前一个任务
                mHost.getHandler().removeCallbacks(mExecCommit);//移除任务队列中的任务，猜测是为了避免死循环
                mHost.getHandler().post(mExecCommit);//重新post一个
            }
        }
    }
    //该方法必是mExecCommit对象执行的方法，须从主线程来调用
    public boolean execPendingActions() {
        //检查事务是否重复提交，以及调用线程

        boolean didSomething = false;

        while (true) {
            int numActions;

            synchronized (this) {
            	//以下取出当前FragmentManager管理的Fragment的事务的操作
                if (mPendingActions == null || mPendingActions.size() == 0) {
                    break;
                }
                numActions = mPendingActions.size();
                if (mTmpActions == null || mTmpActions.length < numActions) {
                    mTmpActions = new Runnable[numActions];
                }
                mPendingActions.toArray(mTmpActions);
                mPendingActions.clear();
                mHost.getHandler().removeCallbacks(mExecCommit);
            }

            mExecutingActions = true;
            for (int i=0; i<numActions; i++) {
                mTmpActions[i].run();//执行BackStackRecord的run方法
                mTmpActions[i] = null;
            }
            mExecutingActions = false;
            didSomething = true;
        }
        doPendingDeferredStart();//该方法内部逻辑根据mHavePendingDeferredStart的值决定是否执行，默认为false不执行
        return didSomething;
    }

	//BackStackRecord的run方法逻辑如下
    public void run() {
        if (mAddToBackStack) {
            if (mIndex < 0) {
                throw new IllegalStateException("addToBackStack() called after commit()");
            }
        }

        bumpBackStackNesting(1);
        //....关于场景的操作略
        Op op = mHead;
        while (op != null) {//根据事务的不同操作来执行
            int enterAnim = state != null ? 0 : op.enterAnim;
            int exitAnim = state != null ? 0 : op.exitAnim;
            switch (op.cmd) {
                case OP_ADD: {
                    Fragment f = op.fragment;
                    f.mNextAnim = enterAnim;
                    mManager.addFragment(f, false);//添加Fragment到管理类中
                } break;
                //其他事务的处理
            }
            op = op.next;//执行事务中的其他操作
        }
        //切换状态
        mManager.moveToState(mManager.mCurState, transition, transitionStyle, true);

        if (mAddToBackStack) {
            mManager.addBackStackState(this);
        }
    }

    //Fragment生命周期切换的核心方法
    void moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive) {
        // Fragments that are not currently added will sit in the onCreate() state.
        if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
            newState = Fragment.CREATED;
        }
        if (f.mRemoving && newState > f.mState) {
            // While removing a fragment, we can't change it to a higher state.
            newState = f.mState;
        }
        // Defer start if requested; don't allow it to move to STARTED or higher
        // if it's not already started.
        if (f.mDeferStart && f.mState < Fragment.STARTED && newState > Fragment.STOPPED) {
            newState = Fragment.STOPPED;
        }
        if (f.mState < newState) {
            //...略
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (f.mSavedFragmentState != null) {//首先检查是否为恢复状态
                        f.mSavedFragmentState.setClassLoader(mHost.getContext().getClassLoader());
                        f.mSavedViewState = f.mSavedFragmentState.getSparseParcelableArray(
                                FragmentManagerImpl.VIEW_STATE_TAG);
                        f.mTarget = getFragment(f.mSavedFragmentState,
                                FragmentManagerImpl.TARGET_STATE_TAG);
                        if (f.mTarget != null) {
                            f.mTargetRequestCode = f.mSavedFragmentState.getInt(
                                    FragmentManagerImpl.TARGET_REQUEST_CODE_STATE_TAG, 0);
                        }
                        f.mUserVisibleHint = f.mSavedFragmentState.getBoolean(
                                FragmentManagerImpl.USER_VISIBLE_HINT_TAG, true);
                        if (!f.mUserVisibleHint) {
                            f.mDeferStart = true;
                            if (newState > Fragment.STOPPED) {
                                newState = Fragment.STOPPED;
                            }
                        }
                    }
                    f.mHost = mHost;//这里为Fragment绑定Host
                    f.mParentFragment = mParent;
                    f.mFragmentManager = mParent != null
                            ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();
                    f.mCalled = false;
                    f.onAttach(mHost.getContext());//Attch方法调用
                    if (!f.mCalled) {
                        throw new SuperNotCalledException("Fragment " + f
                                + " did not call through to super.onAttach()");
                    }
                    if (f.mParentFragment == null) {
                        mHost.onAttachFragment(f);
                    } else {
                        f.mParentFragment.onAttachFragment(f);
                    }

                    if (!f.mRetaining) {
                        f.performCreate(f.mSavedFragmentState);//onCreate方法调用
                    } else {
                        f.restoreChildFragmentState(f.mSavedFragmentState);//恢复子Fragment状态
                        f.mState = Fragment.CREATED;
                    }
                    f.mRetaining = false;
                    if (f.mFromLayout) {//初始化时从XML中初始化则立即初始化布局
                        f.mView = f.performCreateView(f.getLayoutInflater(
                                f.mSavedFragmentState), null, f.mSavedFragmentState);//这里进去可看看到onCreateView方法被调用
                        if (f.mView != null) {
                            f.mInnerView = f.mView;
                            if (Build.VERSION.SDK_INT >= 11) {
                                ViewCompat.setSaveFromParentEnabled(f.mView, false);
                            } else {
                                f.mView = NoSaveStateFrameLayout.wrap(f.mView);
                            }
                            if (f.mHidden) f.mView.setVisibility(View.GONE);
                            f.onViewCreated(f.mView, f.mSavedFragmentState);
                        } else {
                            f.mInnerView = null;
                        }
                    }
                case Fragment.CREATED:
                    if (newState > Fragment.CREATED) {
                        if (DEBUG) Log.v(TAG, "moveto ACTIVITY_CREATED: " + f);
                        if (!f.mFromLayout) {
                            ViewGroup container = null;
                            if (f.mContainerId != 0) {
                                if (f.mContainerId == View.NO_ID) {
                                    throwException(new IllegalArgumentException(
                                            "Cannot create fragment "
                                                    + f
                                                    + " for a container view with no id"));
                                }
                                //找宿主的View，其中mContainer在FragmentActivity中的onCreate中attach
                                container = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                                if (container == null && !f.mRestored) {
                                    String resName;
                                    try {
                                        resName = f.getResources().getResourceName(f.mContainerId);
                                    } catch (NotFoundException e) {
                                        resName = "unknown";
                                    }
                                    throwException(new IllegalArgumentException(
                                            "No view found for id 0x"
                                            + Integer.toHexString(f.mContainerId) + " ("
                                            + resName
                                            + ") for fragment " + f));
                                }
                            }
                            f.mContainer = container;
                            f.mView = f.performCreateView(f.getLayoutInflater(
                                    f.mSavedFragmentState), container, f.mSavedFragmentState);
                            if (f.mView != null) {
                                f.mInnerView = f.mView;
                                if (Build.VERSION.SDK_INT >= 11) {
                                    ViewCompat.setSaveFromParentEnabled(f.mView, false);
                                } else {
                                    f.mView = NoSaveStateFrameLayout.wrap(f.mView);
                                }
                                if (container != null) {
                                    Animation anim = loadAnimation(f, transit, true,
                                            transitionStyle);
                                    if (anim != null) {
                                        setHWLayerAnimListenerIfAlpha(f.mView, anim);
                                        f.mView.startAnimation(anim);
                                    }
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) f.mView.setVisibility(View.GONE);
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                            } else {
                                f.mInnerView = null;
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                //...一系列生命周期方法的调用
            }
        } else if (f.mState > newState) {
            //这里是从显示到隐藏的生命周期逻辑，略
            }
        }

        if (f.mState != newState) {
            Log.w(TAG, "moveToState: Fragment state for " + f + " not updated inline; "
                    + "expected state " + newState + " found " + f.mState);
            f.mState = newState;
        }
    }
```

上面的一系列分析来看，我们在启动一个事务去操作Fragment时，实际上当时只是记录一组操作，我们提交时才会真正去做操作，同时Fragment的状态切换的核心函数是moveToState，Fragment的生命周期方法在这里连续执行到指定周期。

## 二、Fragment如何与Activity的生命周期联动

从上面来看Fragment生命周期即状态切换主要依靠moveToState来，所以这里一个猜想就是Activity在其生命周期方法中做Fragment的状态切换

首先看FragmentActivity的onCreate方法

```
protected void onCreate(@Nullable Bundle savedInstanceState) {
        mFragments.attachHost(null /*parent*/);//首先FragmentController绑定相应的host

        super.onCreate(savedInstanceState);

        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            mFragments.restoreLoaderNonConfig(nc.loaders);
        }
        if (savedInstanceState != null) {//如果是Activity回收重新恢复的情况
            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
            //重新恢复Fragment的状态
            mFragments.restoreAllState(p, nc != null ? nc.fragments : null);
            //....略
        }
        //...略
        mFragments.dispatchCreate();
    }
```

接着在看onResume以及onStop方法

```
	protected void onResume() {
        super.onResume();
        mHandler.sendEmptyMessage(MSG_RESUME_PENDING);//这里最终会调用Fragment的生命周期切换方法
        mResumed = true;
        mFragments.execPendingActions();
    }

    protected void onStop() {
        super.onStop();
        mStopped = true;
        mHandler.sendEmptyMessage(MSG_REALLY_STOPPED);
        mFragments.dispatchStop();//这里是Fragment的状态切换
    }
```

所以这里就很简单，Acitivity在自己的生命周期内通过Fragment的管理对象去切换Fragment的状态，从而使得Activity与Fragment生命周期的联动

## 三、Fragment的恢复过程

这里主要看一个Activity被销毁之后Fragment如何恢复

主要看销毁之前会存哪些数据

```
	protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        Parcelable p = mFragments.saveAllState();//存储数据
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
        //略...
    }
    //数据存储操作
    Parcelable saveAllState() {
        // 执行某些未完成的任务
        execPendingActions();
        //...
        if (mActive == null || mActive.size() <= 0) {
            return null;
        }
        // First collect all active fragments.
        int N = mActive.size();
        FragmentState[] active = new FragmentState[N];
        boolean haveFragments = false;
        for (int i=0; i<N; i++) {
            Fragment f = mActive.get(i);
            if (f != null) {
                //.....

                haveFragments = true;

                FragmentState fs = new FragmentState(f);//这里创建的State类保存了Fragment的类名、View的id等
                active[i] = fs;
                if (f.mState > Fragment.INITIALIZING && fs.mSavedFragmentState == null) {
                    fs.mSavedFragmentState = saveFragmentBasicState(f);//这里做关于Fragment的View的状态恢复
                    //....

                } else {
                    fs.mSavedFragmentState = f.mSavedFragmentState;
                }
                //...
            }
        }

        //...

        int[] added = null;
        BackStackState[] backStack = null;
        //BackStack状态保存...
        FragmentManagerState fms = new FragmentManagerState();
        fms.mActive = active;//保存Fragment状态
        fms.mAdded = added;
        fms.mBackStack = backStack;
        return fms;
    }
```

上面的方法可以看出主要会存Fragment的状态信息、回退栈、View信息之类的关键数据，然后相应的会在onCreate的方法中恢复，所以很多时候如果我们不处理关于Activity的回收恢复的逻辑，当我们使用的Fragment的时候每次只是在Activity中去直接new 一个Fragment很有可能我们的Activity恢复之后会有多个同样的Fragment同时存在的情况,同时由于恢复跳过一些生命周期方法则host attach不到则会出现一些错误