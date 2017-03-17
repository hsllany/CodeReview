FragmentManager本身是一个抽象类，真正实现的是'FragmentManager.java'中的FragmentManagerImpl。

## 先简单看下Add Fragment操作。
在FragmentManagerImpl，通过FragmentManager的操作，是无法直接调用到的。

```
    public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (mAdded == null) {
            mAdded = new ArrayList<Fragment>();
        }
        if (DEBUG) Log.v(TAG, "add: " + fragment);
        makeActive(fragment);
        if (!fragment.mDetached) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
            mAdded.add(fragment);
            fragment.mAdded = true;
            fragment.mRemoving = false;
            if (fragment.mHasMenu && fragment.mMenuVisible) {
                mNeedMenuInvalidate = true;
            }
            if (moveToStateNow) {
                moveToState(fragment);
            }
        }
    }
```

# FragmentTransaction

我们知道，往FragmentManager中添加fragment，都是通过fragmentTransaction；但真正的FragmentTransaction只是一个抽象类，真正实现其的是BackStackRecord

# BackStackRecord

```
final class BackStackRecord extends FragmentTransaction implements
    FragmentManager.BackStackEntry, Runnable 
```

其首先是一个FragmentTransaction类型，其次是一个Runnable。真正执行添加、替换等操作的，均是在其run()方法中执行，由FragmentManager加到主线程的消息队列中执行。

此外，包含一个双向链表，用来串联操作，即支持FragmentTransaction的链式调用，其实每调用一个add等操作，都是在这个链表上添加新的BackStackRecord。

```
public BackStackRecord(FragmentManagerImpl manager) {
    mManager = manager;
}

...

Op mHead;
Op mTail;
```

## 链表节点类型：

```
static final class Op {

    // 双向链表，用于向后、向前遍历，对应commit()和popBackStack() 2种操作。
    Op next;
    Op prev;
    
    // cmd存储了是什么类型的command。
    // static final int OP_NULL = 0;
    // static final int OP_ADD = 1;
    // static final int OP_REPLACE = 2;
    // static final int OP_REMOVE = 3;
    // static final int OP_HIDE = 4;
    // static final int OP_SHOW = 5;
    // static final int OP_DETACH = 6;
    // static final int OP_ATTACH = 7;
    
    int cmd;
    
    // 存储了待操作的fragment
    Fragment fragment;
    
    // 定义了动画资源
    int enterAnim;
    int exitAnim;
    int popEnterAnim;
    int popExitAnim;
    
    // 这个后面会看到
    ArrayList<Fragment> removed;
    }
```

## add, attach, detach, show, hide, remove, replace
所有的FragmentTransition的add, attach, detach, hide, remove, replace, show均是在增加这个链表。

```
add -> doAddOp(), 对应类型OP_ADD
replace -> doAddOp(), 对应类型OP_REPLACE
attach -> addOp(), 对应类型OP_ATTACH
detach -> addOp(), 对应类型OP_DETACH
show -> addOp(), 对应类型OP_SHOW
hide -> addOp(), 对应类型OP_HIDE
remove -> addOp(), 对应类型OP_REMOVE
```

```
public FragmentTransaction add(Fragment fragment, String tag) {
        doAddOp(0, fragment, tag, OP_ADD);
        return this;
    }
    
public FragmentTransaction replace(int containerViewId, Fragment fragment, String tag) {
        if (containerViewId == 0) {
            throw new IllegalArgumentException("Must use non-zero containerViewId");
        }

        doAddOp(containerViewId, fragment, tag, OP_REPLACE);
        return this;
    }
    
public FragmentTransaction remove(Fragment fragment) {
        Op op = new Op();
        op.cmd = OP_REMOVE;
        op.fragment = fragment;
        addOp(op);

        return this;
    }
    
    ...
```

### doAddOp()
添加操作比较特殊，这个会留作以后分析。也是调用addOp();
```
    private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
        fragment.mFragmentManager = mManager;

        if (tag != null) {
            if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
                throw new IllegalStateException("Can't change tag of fragment "
                        + fragment + ": was " + fragment.mTag
                        + " now " + tag);
            }
            fragment.mTag = tag;
        }

        if (containerViewId != 0) {
            if (containerViewId == View.NO_ID) {
                throw new IllegalArgumentException("Can't add fragment "
                        + fragment + " with tag " + tag + " to container view with no id");
            }
            if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
                throw new IllegalStateException("Can't change container ID of fragment "
                        + fragment + ": was " + fragment.mFragmentId
                        + " now " + containerViewId);
            }
            fragment.mContainerId = fragment.mFragmentId = containerViewId;
        }

        Op op = new Op();
        op.cmd = opcmd;
        op.fragment = fragment;
        addOp(op);
    }
```

## commit
调用链：
```
commit() -> commitInternal(false) -> mManager.enqueueAction(this, allowStateLoss);
```

对于mManager的enqueueAction(this, allowStateLoss);
1. mManager中维护了一个所有待执行runnable的列表，该方法首先懒创建该列表，其次把this(BackStackRecord)作为Runnable加到这个mPenddingActions中。所有的操作其实都是放在BackStackRecord的run方法中。
2. 执行mExeCommit

```
public void enqueueAction(Runnable action, boolean allowStateLoss) {
        if (!allowStateLoss) {
            checkStateLoss();
        }
        synchronized (this) {
            if (mDestroyed || mHost == null) {
                throw new IllegalStateException("Activity has been destroyed");
            }
            if (mPendingActions == null) {
                mPendingActions = new ArrayList<Runnable>();
            }
            mPendingActions.add(action);
            
            // 这个size()==1，基本能保证这个Runnable被立即被加入到消息队列中去。如果在mExecCommit执行过程中，有新的Runnable被加入，则可由下面的exePendingActions()的循环处理
            if (mPendingActions.size() == 1) {
                mHost.getHandler().removeCallbacks(mExecCommit);
                mHost.getHandler().post(mExecCommit);
            }
        }
    }
```

### mExecCommit

Runnable，仅运行一个方法：execPendingActions();

### execPendingActions()

```
while (true) {
            int numActions;
            
            // 锁加在这里，只拿所有penddingAction
            synchronized (this) {
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
                mTmpActions[i].run();
                mTmpActions[i] = null;
            }
            mExecutingActions = false;
            didSomething = true;
        }
```

这里的锁，与enqueueAction中的锁，既保证了新加入的runnable被即时执行，也保证了可以持续加入之间的平衡，大家细细体会。

这里也可以看出，commit()并不是立即执行。

最后，回到了BackStackRecord的run方法中。

## BackStackRecord.run()

```
    public void run() {
        if (FragmentManagerImpl.DEBUG) {
            Log.v(TAG, "Run: " + this);
        }

        if (mAddToBackStack) {
            if (mIndex < 0) {
                throw new IllegalStateException("addToBackStack() called after commit()");
            }
        }

        bumpBackStackNesting(1);

        // 以下是对与其动画的处理。
        if (mManager.mCurState >= Fragment.CREATED) {
            // 建立2个array，存储对于每一个containerId的，最后进入的fragment，以及首个离开的fragment。出入动画，只对应最后进入和首个离开这个container的fragment
            SparseArray<Fragment> firstOutFragments = new SparseArray<Fragment>();
            SparseArray<Fragment> lastInFragments = new SparseArray<Fragment>();
            
            // 通过这个方法，遍历整个链表，对每个containerId对应的ViewGroup，计算出首个进入lastInFragment，及firstOutFragment，并分别存入SparseArray。
            calculateFragments(firstOutFragments, lastInFragments);
            beginTransition(firstOutFragments, lastInFragments, false);
        }

        // 下面遍历整个链表
        Op op = mHead;
        while (op != null) {
            switch (op.cmd) {
                
                // 遇到添加，直接添加，加入到mManager的mAdd List中。
                case OP_ADD: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.enterAnim;
                    // 这里才调用了文章开头所说的FragmentManagerImpl的addFragment方法
                    mManager.addFragment(f, false);
                }
                break;
                
                // 替换，要把替换的fragment(被删除的)，存入该backStackRecord的mRemoved中，为随后的回退做准备。
                case OP_REPLACE: {
                    Fragment f = op.fragment;
                    int containerId = f.mContainerId;
                    if (mManager.mAdded != null) {
                        for (int i = mManager.mAdded.size() - 1; i >= 0; i--) {
                            Fragment old = mManager.mAdded.get(i);
                            if (FragmentManagerImpl.DEBUG) {
                                Log.v(TAG,
                                        "OP_REPLACE: adding=" + f + " old=" + old);
                            }
                            if (old.mContainerId == containerId) {
                                if (old == f) {
                                    op.fragment = f = null;
                                } else {
                                    if (op.removed == null) {
                                        op.removed = new ArrayList<Fragment>();
                                    }
                                    
                                    //将被替换的，加入到removed list中。
                                    op.removed.add(old);
                                    old.mNextAnim = op.exitAnim;
                                    if (mAddToBackStack) {
                                        old.mBackStackNesting += 1;
                                        if (FragmentManagerImpl.DEBUG) {
                                            Log.v(TAG, "Bump nesting of "
                                                    + old + " to " + old.mBackStackNesting);
                                        }
                                    }
                                    mManager.removeFragment(old, mTransition, mTransitionStyle);
                                }
                            }
                        }
                    }
                    if (f != null) {
                        f.mNextAnim = op.enterAnim;
                        mManager.addFragment(f, false);
                    }
                }
                break;
                case OP_REMOVE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.exitAnim;
                    mManager.removeFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_HIDE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.exitAnim;
                    mManager.hideFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_SHOW: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.enterAnim;
                    mManager.showFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_DETACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.exitAnim;
                    mManager.detachFragment(f, mTransition, mTransitionStyle);
                }
                break;
                case OP_ATTACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.enterAnim;
                    mManager.attachFragment(f, mTransition, mTransitionStyle);
                }
                break;
                default: {
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
                }
            }

            op = op.next;
        }

        mManager.moveToState(mManager.mCurState, mTransition,
                mTransitionStyle, true);

        // 如果要可回退，则加入到mManager的
        if (mAddToBackStack) {
            mManager.addBackStackState(this);
        }
    }
```

这里设计巧妙之处在于，replace的操作，将所有被替换的fragment加入到了该backStackRecord的removed列表中，为后面回退做准备。

顺便看下FragmentManager中关于mBackStack的定义，以及addBackStackState()函数的代码。

```
...
    ArrayList<BackStackRecord> mBackStack;

...

    void addBackStackState(BackStackRecord state) {
        if (mBackStack == null) {
            mBackStack = new ArrayList<BackStackRecord>();
        }
        mBackStack.add(state);
        reportBackStackChanged();
    }
```

# FragmentManager.popBackStack()

同样是调用enqueueAction，不一定立即执行，而仅在恰当时期执行。

```
enqueueAction(new Runnable() {
            @Override public void run() {
                popBackStackState(mHost.getHandler(), null, -1, 0);
            }
        }, false);
```

其调用了一个函数popBackStackState(), 内部比较复杂，但最终都调用了BackStackRecord的popFromBackStack方法。

这个方法思路与run()类似，先处理动画，然后执行操作，只不过，操作都是反着执行了。

```
Op op = mTail;
        while (op != null) {
            switch (op.cmd) {
                //添加变删除
                case OP_ADD: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.popExitAnim;
                    mManager.removeFragment(f,
                            FragmentManagerImpl.reverseTransit(mTransition),
                            mTransitionStyle);
                }
                break;
                // 替换，先把被删除的从mRemove中加回来。再删除替换的
                case OP_REPLACE: {
                    Fragment f = op.fragment;
                    if (f != null) {
                        f.mNextAnim = op.popExitAnim;
                        mManager.removeFragment(f,
                                FragmentManagerImpl.reverseTransit(mTransition),
                                mTransitionStyle);
                    }
                    if (op.removed != null) {
                        for (int i = 0; i < op.removed.size(); i++) {
                            Fragment old = op.removed.get(i);
                            old.mNextAnim = op.popEnterAnim;
                            mManager.addFragment(old, false);
                        }
                    }
                }
                break;
                case OP_REMOVE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.popEnterAnim;
                    mManager.addFragment(f, false);
                }
                break;
                case OP_HIDE: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.popEnterAnim;
                    mManager.showFragment(f,
                            FragmentManagerImpl.reverseTransit(mTransition), mTransitionStyle);
                }
                break;
                case OP_SHOW: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.popExitAnim;
                    mManager.hideFragment(f,
                            FragmentManagerImpl.reverseTransit(mTransition), mTransitionStyle);
                }
                break;
                case OP_DETACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.popEnterAnim;
                    mManager.attachFragment(f,
                            FragmentManagerImpl.reverseTransit(mTransition), mTransitionStyle);
                }
                break;
                case OP_ATTACH: {
                    Fragment f = op.fragment;
                    f.mNextAnim = op.popExitAnim;
                    mManager.detachFragment(f,
                            FragmentManagerImpl.reverseTransit(mTransition), mTransitionStyle);
                }
                break;
                default: {
                    throw new IllegalArgumentException("Unknown cmd: " + op.cmd);
                }
            }

            op = op.prev;
        }
```

# addFragment

最后，搞清楚了调用链，我们再回过来分析addFragment这个方法。

```
    public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (mAdded == null) {
            mAdded = new ArrayList<Fragment>();
        }
        if (DEBUG) Log.v(TAG, "add: " + fragment);
        
        // makeActive，是将fragment加入到mActive列表中，
        makeActive(fragment);
        if (!fragment.mDetached) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
            mAdded.add(fragment);
            fragment.mAdded = true;
            fragment.mRemoving = false;
            if (fragment.mHasMenu && fragment.mMenuVisible) {
                mNeedMenuInvalidate = true;
            }
            
            // 重要的是这个moveToState方法，负责进行fragment的状态转移。
            if (moveToStateNow) {
                moveToState(fragment);
            }
        }
    }
```

## moveToState()

忽略一些细节来看代码

```
switch (f.mState) {
                // 初始化, 即创建fragment的第一个状态
                case Fragment.INITIALIZING:
                    ...
                    
                    // 将该fragment的host activiy 和 parent framgment更新，并更新其fragmentManager
                    f.mHost = mHost;
                    f.mParentFragment = mParent;
                    f.mFragmentManager = mParent != null
                            ? mParent.mChildFragmentManager : mHost.getFragmentManagerImpl();
                    
                    ...
                    // 调用onAttach
                    f.onAttach(mHost.getContext());
                    ...
                    
                    if (f.mParentFragment == null) {
                        mHost.onAttachFragment(f);
                    } else {
                        f.mParentFragment.onAttachFragment(f);
                    }

                    if (!f.mRetaining) {
                        f.performCreate(f.mSavedFragmentState);
                    } else {
                       ...
                    }
                    ...
                    if (f.mFromLayout) {
                        // For fragments that are part of the content view
                        // layout, we need to instantiate the view immediately
                        // and the inflater will take care of adding it.
                        
                        // fragment的performCreateView会调用fragment的onCreateView方法，返回View并将其保存在f的mView对象中。
                        f.mView = f.performCreateView(f.getLayoutInflater(
                                f.mSavedFragmentState), null, f.mSavedFragmentState);
                        if (f.mView != null) {
                            if (f.mHidden) f.mView.setVisibility(View.GONE);
                            f.onViewCreated(f.mView, f.mSavedFragmentState);
                        }
                    }
                    
                // INITIALIZING完了就是CREATED状态
                case Fragment.CREATED:
                    if (newState > Fragment.CREATED) {
                        ...
                                // 从该fragmentManager的mContainer中获取containerId的ViewGroup。
                                container = (ViewGroup) mContainer.onFindViewById(f.mContainerId);
                                
                            ...
                                    // 加入ViewGroup                
                                    container.addView(f.mView);
                                }
                                if (f.mHidden) f.mView.setVisibility(View.GONE);
                                f.onViewCreated(f.mView, f.mSavedFragmentState);
                            }
                        }

                        f.performActivityCreated(f.mSavedFragmentState);
                        if (f.mView != null) {
                            f.restoreViewState(f.mSavedFragmentState);
                        }
                        f.mSavedFragmentState = null;
                    }
                ...
```