Loader的机制比较复杂，会在研读过程中不断更新。

# Loader
Loader是所有Loader的基类。直接子类为AsyncTaskLoader.

继承关系：

Loader < AsyncTaskLoader < CursorLoader

# AsyncTaskLoader

AsyncTaskLoader 为一个抽象类，其内部提供了LoadTask（继承自AsyncTask）来执行具体的操作。

Loader的使用方式为，首先在activity中声明相应代码：

```
// 第3步，实现LoaderCallbacks接口
public class FeedActivity extends AppCompatActivity implements LoaderManager.LoaderCallbacks<Profile> {
    private static final String TAG = FeedActivity.class.getName();


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 第一步，获取LoaderManager
        // 第二步，调用initLoader
        getLoaderManager().initLoader(0, null, this);

    }

    // 第4步，在onCreateLoader中返回自定义Loader，或CursorLoader.
    @Override
    public Loader onCreateLoader(int id, Bundle args) {
        return new ProfileLoader(this);
    }

    // 第5步，实现onLoadFinish接口，会在Loader加载完毕后调用
    @Override
    public void onLoadFinished(Loader loader, Profile data) {
        Log.d(TAG, "onLoadFinished");

        Log.d(TAG, data.toString());
    }

    // 第6步，实现onLoaderReset接口，会在Loader重载时调用
    @Override
    public void onLoaderReset(Loader loader) {
        Log.d(TAG, "onLoadReset");
    }
}
```

然后实现Loader
```
public class ProfileLoader extends AsyncTaskLoader<Profile> {
    public ProfileLoader(Context context) {
        super(context);
    }
    
    @Override
    public void onStartLoading(){
        this.forceLoad();
    }


    @Override
    public Profile loadInBackground() {
        // do some loading heavy work here
        return new Profile();
    }
}
```


下面简单就上述代码进行分析。

## 1 getLoaderManager()

首先LoaderManager(android.app.LoaderManager)仅只是一个抽象类，具体实现为LoaderManagerImpl，和LoaderManager同处一个文件中。

Activity对象在初始化时，会维护一个代理对象FragmentController mFragments（see android.app.FragmentController）用来和FragmentManager进行交互，其扮演的角色为Fragment Host。在启动时，FragmentController会传入一个FragmentHostCallback的对象，用来提供所有的所需对象。其中，在FragmentHostCallback中，有一个还有一个
```LoaderManagerImpl getLoaderManagerImpl()```的方法，即时我们所需要的LoaderManager.(其调用的是```LoaderManagerImpl getLoaderManager(String who, boolean started, boolean create)```方法，会将该LoaderManager（标签为“root")存入FragmentHostCallback的mAllLoaderManagers对象中，其后所有fragment的LoaderManager也会存入mAllLoaderManager中)

```
LoaderManagerImpl getLoaderManagerImpl() {
        if (mLoaderManager != null) {
            return mLoaderManager;
        }
        mCheckedForLoaderManager = true;
        mLoaderManager = getLoaderManager("(root)", mLoadersStarted, true /*create*/);
        return mLoaderManager;
    }
```


activity中调用链为 
```
getLoaderManager() -> FragmentController.getLoaderManager() -> FragmentHostCallback.getLoaderManagerImpl();
```

也即最终存储LoaderManagerImpl对象的，为fragmentHostCallback；但进一步分析Activity源码发现，其内部继承了一个内部类HostCallback（继承自FragmentHostCallback），所以activity间接持有了该LoaderManager.

此外，getLoaderManagerImpl()中，mLoaderManager为lasy加载，所以我们如果不在activity中调用getLoaderManager，那么LoaderManager是不会被实例化的。


## 2 initLoader()

### 2.1 LoadInfo

一个LoaderManager中可以有多个不同的Loader，这些Loader被包装成LoaderInfo对象，存储在LoaderManager的mLoaders。LoaderInfo是LoaderManager的内部类，存储了Loader以及其的一些状态信息。上面Activity中的LoaderCallbacks，实际是被该LoaderInfo所持有，并管理相应的回调。

```
final SparseArray<LoaderInfo> mLoaders = new SparseArray<LoaderInfo>(0);

```

### 2.2 初始化Loader


在调用initLoader时，外部需要把整形id传入，这个id的目的，也就是为了标记具体的Loader。

```
public <D> Loader<D> initLoader(int id, Bundle args, LoaderManager.LoaderCallbacks<D> callback) {
        if (mCreatingLoader) {
            throw new IllegalStateException("Called while creating a loader");
        }
        
        // 首先查找id对应的Loader
        LoaderInfo info = mLoaders.get(id);
        
        if (DEBUG) Log.v(TAG, "initLoader in " + this + ": args=" + args);

        // 如果找不到，则创建Loader
        if (info == null) {
            // Loader doesn't already exist; create.
            // onCreateLoader内部，就调用了callBack的onCreateLoader方法。
            info = createAndInstallLoader(id, args,  (LoaderManager.LoaderCallbacks<Object>)callback);
            if (DEBUG) Log.v(TAG, "  Created new loader " + info);
        } else {
            if (DEBUG) Log.v(TAG, "  Re-using existing loader " + info);
            info.mCallbacks = (LoaderManager.LoaderCallbacks<Object>)callback;
        }
        
        if (info.mHaveData && mStarted) {
            // If the loader has already generated its data, report it now.
            info.callOnLoadFinished(info.mLoader, info.mData);
        }
        
        return (Loader<D>)info.mLoader;
    }
```

## 3 Loader如何被启动

要分析Loader如何被启动前，我们需要知道，Loader中的哪个方法是负责启动操作的。

拿AsyncTaskLoader为例，有一个loadInBackground()方法是需要开发者实现的，这个方法和AsyncTask的doInBackground()很类似，但LoaderManager并不是直接调用这个方法去执行load操作。

### 3.1 AsyncTaskLoader的启动与执行过程

Loader中有一个startLoading()方法，我们猜想这个是启动Loading过程的方法。来看调用链：

Loader.startLoading() -> Loader.onStartLoading()

Loader的onStartLoading为空，我们来看AsyncTaskLoader的onStartLoading()，发现竟然没有被实现。

看来，如果我们要自定义AsyncTaskLoader，还必须做点别的操作。

Loader中有另外一个方法，是forceLoad()，强制更新的意思。所以我们需要在自定义的AsyncTaskLoader的onStartLoading()方法中，自己实现forceLoad()方法，否则自定义的AsyncTaskLoader是无法启动操作的。

再来看AsyncTaskLoader的forceLoad()方法:

```
@Override
    protected void onForceLoad() {
        super.onForceLoad();
        
        // 首先取消load
        cancelLoad();
        
        // 实例化一个LoadTask
        mTask = new LoadTask();
        if (DEBUG) Log.v(TAG, "Preparing load: mTask=" + mTask);
        
        // 执行executePendingTask()
        executePendingTask();
    }
```

上文中提到，LoadTask实际是一个AsyncTask，在其doInBackground()中

```
protected D doInBackground(Void... params) {
    if (DEBUG) Log.v(TAG, this + " >>> doInBackground");
    try {
        D data = AsyncTaskLoader.this.onLoadInBackground();
        if (DEBUG) Log.v(TAG, this + "  <<< doInBackground");
        return data;
    } catch (OperationCanceledException ex) {
        if (!isCancelled()) {
            // onLoadInBackground threw a canceled exception spuriously.
            // This is problematic because it means that the LoaderManager did not
            // cancel the Loader itself and still expects to receive a result.
            // Additionally, the Loader's own state will not have been updated to
            // reflect the fact that the task was being canceled.
            // So we treat this case as an unhandled exception.
            throw ex;
        }
        if (DEBUG) Log.v(TAG, this + "  <<< doInBackground (was canceled)", ex);
            return null;
        }
    }
}
```

调用了该Loader的onLoadInBackground()。

再来看executePendingTask():

```
    void executePendingTask() {
        if (mCancellingTask == null && mTask != null) {
            if (mTask.waiting) {
                mTask.waiting = false;
                mHandler.removeCallbacks(mTask);
            }
            if (mUpdateThrottle > 0) {
                long now = SystemClock.uptimeMillis();
                if (now < (mLastLoadCompleteTime+mUpdateThrottle)) {
                    // Not yet time to do another load.
                    if (DEBUG) Log.v(TAG, "Waiting until "
                            + (mLastLoadCompleteTime+mUpdateThrottle)
                            + " to execute: " + mTask);
                    mTask.waiting = true;
                    mHandler.postAtTime(mTask, mLastLoadCompleteTime+mUpdateThrottle);
                    return;
                }
            }
            if (DEBUG) Log.v(TAG, "Executing: " + mTask);
            
            //这里的mExecutor如果在AsyncTaskLoader构造方法中传入，如果没有指定，则会被赋值为AsyncTask.THREAD_POOL_EXECUTOR。
            mTask.executeOnExecutor(mExecutor, (Void[]) null);
        }
    }
```

最终调用了mTask的executeOnExecutor来执行的。

至此，我们明白了，AsyncTaskLoader需要通过startLoading()（需要手动在onStartLoading()中调用forceLoading()），然后其内部新建了一个AsyncTask执行操作的。

### 3.2 Activity中的start Loader

我们知道，Loader的启动会被Activity自动维护，我们需要找到activity中的调用链，哪一步启动了这些Loader的。

明白了AsyncTaskLoader的执行过程，我们来看，activity中是哪一步调用到了startLoading()这个方法。

在activity的onStart()方法中，找到了```mFragments.doLoaderStart();```代码。最终通过FragmentHostCallback调用了LoaderManager的doStart()方法。

```
    void doStart() {
        if (DEBUG) Log.v(TAG, "Starting in " + this);
        if (mStarted) {
            RuntimeException e = new RuntimeException("here");
            e.fillInStackTrace();
            Log.w(TAG, "Called doStart when already started: " + this, e);
            return;
        }
        
        mStarted = true;

        // Call out to sub classes so they can start their loaders
        // Let the existing loaders know that we want to be notified when a load is complete
        for (int i = mLoaders.size()-1; i >= 0; i--) {
            // 逐一从mLoaders中启动Loader
            mLoaders.valueAt(i).start();
        }
    }
```

但启动Loader的过程并不是直接调用Loader的startLoading()方法，而是通过LoaderInfo的start()方法，其中会做一些判断，例如如果已经启动过，就不在启动了。

注意上文提到过，所有的LoaderCallback均是由LoaderInfo所持有，但执行具体操作的是Loader，2者如何交互呢？下面代码中也可发现线索。

LoadInfo:
```
    void start() {
            if (mRetaining && mRetainingStarted) {
                // Our owner is started, but we were being retained from a
                // previous instance in the started state...  so there is really
                // nothing to do here, since the loaders are still started.
                mStarted = true;
                return;
            }

            if (mStarted) {
                // If loader already started, don't restart.
                return;
            }

            mStarted = true;
            
            if (DEBUG) Log.v(TAG, "  Starting: " + this);
            if (mLoader == null && mCallbacks != null) {
               mLoader = mCallbacks.onCreateLoader(mId, mArgs);
            }
            if (mLoader != null) {
                if (mLoader.getClass().isMemberClass()
                        && !Modifier.isStatic(mLoader.getClass().getModifiers())) {
                    throw new IllegalArgumentException(
                            "Object returned from onCreateLoader must not be a non-static inner member class: "
                            + mLoader);
                }
                if (!mListenerRegistered) {
                
                    // 将LoaderInfo作为OnLoadCompleteListener注册到Loader中，这样Loader在执行完毕后，可回调到LoaderInfo的相关代码。
                    mLoader.registerListener(mId, this);
                    mLoader.registerOnLoadCanceledListener(this);
                    mListenerRegistered = true;
                }
                mLoader.startLoading();
            }
        }
```



所以activity在onStart()阶段启动了所有的Loader的start方法，让加载得以执行。

这里还需要注意一个细节，就是我们在调用LoaderManager的restartLoader()方法时（有时比如数据源更新了，我们需要重新手动load，就会调用这个方法），同样会最后调用到Loader的installLoader()方法（同见initLoader()的createAndInstallLoader())，该方法中有个判断：

```
    mLoaders.put(info.mId, info);
    if (mStarted) {
        // The activity will start all existing loaders in it's onStart(),
        // so only start them here if we're past that point of the activitiy's
        // life cycle
        info.start();
    }
```
即，如果该activity已经start过了，那么Loader的start()方法会在此执行；否则，交由activity的onStart方法去启动Loader.

### 3.3 Loader的retain

在activity的performStop()函数中（即执行stop操作），会调用到FragmentHostCallback的doLoaderStop方法，其中根据activity是否销毁，调用LoaderManager的retain或stop()操作。

如果activity不销毁，则LoaderManager会处于retain状态。

待activity恢复时，会对LoaderManager进行恢复操作（finishRetain)，具体如下:

此外，在Activity的performStart()方法中，有一行```mFragments.reportLoaderStart();```代码，调用了FragmentHostCallback的reportLoaderStart()方法：

```

    void reportLoaderStart() {
        if (mAllLoaderManagers != null) {
            final int N = mAllLoaderManagers.size();
            LoaderManagerImpl loaders[] = new LoaderManagerImpl[N];
            for (int i=N-1; i>=0; i--) {
                // 倒序排序，因为activity的LoaderManager，作为第一个LoaderManager，是存在mAllLoaderManagers的第一个，其后的fragment如果也初始化了LoaderManager，那么也会依次存储。恢复时需要倒过来恢复，先恢复Fragment的LoaderManager，然后是Activity的。
                loaders[i] = (LoaderManagerImpl) mAllLoaderManagers.valueAt(i);
            }
            for (int i=0; i<N; i++) {
                LoaderManagerImpl lm = loaders[i];
                lm.finishRetain();
                lm.doReportStart();
            }
        }
    }
    
```

LoaderManager的finishRetail()，通知所有LoaderInfo finishRetain()。

```
        void finishRetain() {
            if (mRetaining) {
                if (DEBUG) Log.v(TAG, "  Finished Retaining: " + this);
                mRetaining = false;
                if (mStarted != mRetainingStarted) {
                    if (!mStarted) {
                        // This loader was retained in a started state, but
                        // at the end of retaining everything our owner is
                        // no longer started...  so make it stop.
                        stop();
                    }
                }
            }

            if (mStarted && mHaveData && !mReportNextStart) {
                // This loader has retained its data, either completely across
                // a configuration change or just whatever the last data set
                // was after being restarted from a stop, and now at the point of
                // finishing the retain we find we remain started, have
                // our data, and the owner has a new callback...  so
                // let's deliver the data now.
                callOnLoadFinished(mLoader, mData);
            }
        }
        
```

其中，mHaveData是用来通知是否有新数据的。

简单来说，就是如果Loader中有新数据返回，如果该Loader未被retain(即activity未onStop())，那么LoaderManager.LoaderCallbacks的onLoadFinished会被执行；如果在Loader异步请求数据过程中，Loader被retain了（activity被stop()了），那么在下一次start时，会触发LoaderCallbacks的onLoadFinished。这也达到了Loader的管理目的，会帮助你自动管理数据加载过程。


