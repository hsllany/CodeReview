## ActivityThread.main()

```
public static void main(String[] args) {
    ...
    
    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }
	...
    Looper.loop();
	...
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

## ActivityThread.attach(false)

```
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    
    // @: system == false
    
    if (!system) {
    
    // @: will go this branch
    
        ViewRootImpl.addFirstDrawHandler(new Runnable() {
            @Override
            public void run() {
                ensureJitEnabled();
            }
        });
		
		...
		
		//@: communicate with ActivityManagerService through binder
		
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            // Ignore
        }
        
        ...
        
    } else {
        ...
       	try {
            mInstrumentation = new Instrumentation();
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate Application():" + e.toString(), e);
        }
    }

    // add dropbox logging to libcore
    DropBox.setReporter(new DropBoxReporter());

    ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
        @Override
        public void onConfigurationChanged(Configuration newConfig) {
            synchronized (mResourcesManager) {
                // We need to apply this change to the resources
                // immediately, because upon returning the view
                // hierarchy will be informed about it.
                if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                    // This actually changed the resources!  Tell
                    // everyone about it.
                    if (mPendingConfiguration == null ||
                            mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                        mPendingConfiguration = newConfig;

                        sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                    }
                }
            }
        }
        @Override
        public void onLowMemory() {
        }
        @Override
        public void onTrimMemory(int level) {
        }
    });
}
```

## ActivityManagerService.attachApplication

```
@Override
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
```

## ActivityManagerService.attachApplicationLocked

```
private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

	// @: find the app record
	// @: usually we will find it, because when launching this process, app will be added.
	 
    // Find the application record that is being attached...  either via
    // the pid if we are running in multiple processes, or just pull the
    // next app record if we are emulating process with anonymous threads.
    ProcessRecord app;
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }

    if (app == null) {
        if (pid > 0 && pid != MY_PID) {
            Process.killProcessQuiet(pid);
            //TODO: killProcessGroup(app.info.uid, pid);
        } else {
            try {
                thread.scheduleExit();
            } catch (Exception e) {
                // Ignore exceptions.
            }
        }
        return false;
        ...
        
        @: exception happens, will kill the process
    }

    // If this application record is still attached to a previous
    // process, clean it up now.
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }

    // Tell the process all about itself.

    ...
    
    final String processName = app.processName;
    try {
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        ...
        return false;
    }

    app.makeActive(thread, mProcessStats);
    app.curAdj = app.setAdj = -100;
    app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
    app.forcingToForeground = null;
    updateProcessForegroundLocked(app, false, false);
    app.hasShownUi = false;
    app.debugging = false;
    app.cached = false;
    app.killedByAm = false;

    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

    boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
    //@: normalMode == true;
	...	    
    List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;
	...
	
    try {
    	//@: doing some preparation for debug
        ...
        
        //@: ODEX?
        
        ensurePackageDexOpt(app.instrumentationInfo != null
                ? app.instrumentationInfo.packageName
                : app.info.packageName);
        if (app.instrumentationClass != null) {
            ensurePackageDexOpt(app.instrumentationClass.getPackageName());
        }
        
        ...
        
        ApplicationInfo appInfo = app.instrumentationInfo != null
                ? app.instrumentationInfo : app.info;
        app.compat = compatibilityInfoForPackageLocked(appInfo);
        if (profileFd != null) {
            profileFd = profileFd.dup();
        }
        ProfilerInfo profilerInfo = profileFile == null ? null
                : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
                
        //@@@-> ActivityThread.bindApplication
        
        thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                isRestrictedBackupMode || !normalMode, app.persistent,
                new Configuration(mConfiguration), app.compat,
                getCommonServicesLocked(app.isolated),
                mCoreSettingsObserver.getCoreSettingsLocked());
        updateLruProcessLocked(app, false, null);
        app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
    } catch (Exception e) {
        ...
        return false;
    }

    // Remove this record from the list of starting applications.
    mPersistentStartingProcesses.remove(app);
    ...
    mProcessesOnHold.remove(app);

    boolean badApp = false;
    boolean didSomething = false;

    // See if the top visible activity is waiting to run in this process...
    if (normalMode) {
        try {
        
        //@: launch the activity?
        //@@@-> mStackSupervisor.attachApplicationLocked
        
        if (mStackSupervisor.attachApplicationLocked(app)) {
                didSomething = true;
            }
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
            badApp = true;
        }
    }

    // Find any services that should be running in this process...
    if (!badApp) {
        try {
            didSomething |= mServices.attachApplicationLocked(app, processName);
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
            badApp = true;
        }
    }

    // Check if a next-broadcast receiver is in this process...
    if (!badApp && isPendingBroadcastProcessLocked(pid)) {
        try {
            didSomething |= sendPendingBroadcastsLocked(app);
        } catch (Exception e) {
            // If the app died trying to launch the receiver we declare it 'bad'
            Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
            badApp = true;
        }
    }

    // Check whether the next backup agent is in this process...
    if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
        ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
        try {
            thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                    compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                    mBackupTarget.backupMode);
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
            badApp = true;
        }
    }

    if (badApp) {
        app.kill("error during init", true);
        handleAppDiedLocked(app, false, true);
        return false;
    }

    if (!didSomething) {
        updateOomAdjLocked();
    }

    return true;
}
```

## ActivityThread.bindApplication

```
public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableOpenGlTrace, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo, Map<String, IBinder> services,
                Bundle coreSettings) {

     if (services != null) {
         // Setup the service cache in the ServiceManager
         ServiceManager.initServiceCache(services);
     }

     setCoreSettings(coreSettings);

     /*
      * Two possible indications that this package could be
      * sharing its runtime with other packages:
      *
      * 1.) the sharedUserId attribute is set in the manifest,
      *     indicating a request to share a VM with other
      *     packages with the same sharedUserId.
      *
      * 2.) the application element of the manifest has an
      *     attribute specifying a non-default process name,
      *     indicating the desire to run in another packages VM.
      *
      * If sharing is enabled we do not have a unique application
      * in a process and therefore cannot rely on the package
      * name inside the runtime.
      */
     IPackageManager pm = getPackageManager();
     android.content.pm.PackageInfo pi = null;
     try {
         pi = pm.getPackageInfo(appInfo.packageName, 0, UserHandle.myUserId());
     } catch (RemoteException e) {
     }
     if (pi != null) {
         boolean sharedUserIdSet = (pi.sharedUserId != null);
         boolean processNameNotDefault =
         (pi.applicationInfo != null &&
          !appInfo.packageName.equals(pi.applicationInfo.processName));
         boolean sharable = (sharedUserIdSet || processNameNotDefault);

         // Tell the VMRuntime about the application, unless it is shared
         // inside a process.
         if (!sharable) {
             VMRuntime.registerAppInfo(appInfo.packageName, appInfo.dataDir,
                                     appInfo.processName);
         }
     }

     AppBindData data = new AppBindData();
     data.processName = processName;
     data.appInfo = appInfo;
     data.providers = providers;
     data.instrumentationName = instrumentationName;
     data.instrumentationArgs = instrumentationArgs;
     data.instrumentationWatcher = instrumentationWatcher;
     data.instrumentationUiAutomationConnection = instrumentationUiConnection;
     data.debugMode = debugMode;
     data.enableOpenGlTrace = enableOpenGlTrace;
     data.restrictedBackupMode = isRestrictedBackupMode;
     data.persistent = persistent;
     data.config = config;
     data.compatInfo = compatInfo;
     data.initProfilerInfo = profilerInfo;
     sendMessage(H.BIND_APPLICATION, data);
 }
```

## ActivityThread.handleBindApplication

```

private void handleBindApplication(AppBindData data) {
        mBoundApplication = data;
        mConfiguration = new Configuration(data.config);
        mCompatConfiguration = new Configuration(data.config);

        mProfiler = new Profiler();
        if (data.initProfilerInfo != null) {
            mProfiler.profileFile = data.initProfilerInfo.profileFile;
            mProfiler.profileFd = data.initProfilerInfo.profileFd;
            mProfiler.samplingInterval = data.initProfilerInfo.samplingInterval;
            mProfiler.autoStopProfiler = data.initProfilerInfo.autoStopProfiler;
        }

        // send up app name; do this *before* waiting for debugger
        Process.setArgV0(data.processName);
        
        if (data.persistent) {
            // Persistent processes on low-memory devices do not get to
            // use hardware accelerated drawing, since this can add too much
            // overhead to the process.
            if (!ActivityManager.isHighEndGfx()) {
                HardwareRenderer.disable(false);
            }
        }
        
        ...

        // If the app is Honeycomb MR1 or earlier, switch its AsyncTask
        // implementation to use the pool executor.  Normally, we use the
        // serialized executor as the default. This has to happen in the
        // main thread so the main looper is set right.
        if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
            AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
        }

        Message.updateCheckRecycle(data.appInfo.targetSdkVersion);

        /*
         * Before spawning a new process, reset the time zone to be the system time zone.
         * This needs to be done because the system time zone could have changed after the
         * the spawning of this process. Without doing this this process would have the incorrect
         * system time zone.
         */
        TimeZone.setDefault(null);

        /*
         * Initialize the default locale in this process for the reasons we set the time zone.
         */
        Locale.setDefault(data.config.locale);

        /*
         * Update the system configuration since its preloaded and might not
         * reflect configuration changes. The configuration object passed
         * in AppBindData can be safely assumed to be up to date
         */
        mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
        mCurDefaultDisplayDpi = data.config.densityDpi;
        applyCompatConfiguration(mCurDefaultDisplayDpi);

        data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

        /**
         * Switch this process to density compatibility mode if needed.
         */
        if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
                == 0) {
            mDensityCompatMode = true;
            Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
        }
        updateDefaultDensity();

        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        if (!Process.isIsolated()) {
            final File cacheDir = appContext.getCacheDir();

            if (cacheDir != null) {
                // Provide a usable directory for temporary files
                System.setProperty("java.io.tmpdir", cacheDir.getAbsolutePath());
            } else {
                Log.v(TAG, "Unable to initialize \"java.io.tmpdir\" property due to missing cache directory");
            }

            // Use codeCacheDir to store generated/compiled graphics code
            final File codeCacheDir = appContext.getCodeCacheDir();
            if (codeCacheDir != null) {
                setupGraphicsSupport(data.info, codeCacheDir);
            } else {
                Log.e(TAG, "Unable to setupGraphicsSupport due to missing code-cache directory");
            }
        }


        final boolean is24Hr = "24".equals(mCoreSettings.getString(Settings.System.TIME_12_24));
        DateFormat.set24HourTimePref(is24Hr);

        View.mDebugViewAttributes =
                mCoreSettings.getInt(Settings.Global.DEBUG_VIEW_ATTRIBUTES, 0) != 0;

        /**
         * For system applications on userdebug/eng builds, log stack
         * traces of disk and network access to dropbox for analysis.
         */
        if ((data.appInfo.flags &
             (ApplicationInfo.FLAG_SYSTEM |
              ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {
            StrictMode.conditionallyEnableDebugLogging();
        }

        /**
         * For apps targetting SDK Honeycomb or later, we don't allow
         * network usage on the main event loop / UI thread.
         *
         * Note to those grepping:  this is what ultimately throws
         * NetworkOnMainThreadException ...
         */
        if (data.appInfo.targetSdkVersion > 9) {
            StrictMode.enableDeathOnNetwork();
        }

        NetworkSecurityPolicy.getInstance().setCleartextTrafficPermitted(
                (data.appInfo.flags & ApplicationInfo.FLAG_USES_CLEARTEXT_TRAFFIC) != 0);

        ...

        // Enable OpenGL tracing if required
        if (data.enableOpenGlTrace) {
            GLUtils.setTracingLevel(1);
        }

        ...

        /**
         * Initialize the default http proxy in this process for the reasons we set the time zone.
         */
        IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
        if (b != null) {
            // In pre-boot mode (doing initial launch to collect password), not
            // all system is up.  This includes the connectivity service, so don't
            // crash if we can't get it.
            IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
            try {
                final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
                Proxy.setHttpProxySystemProperty(proxyInfo);
            } catch (RemoteException e) {}
        }

		//@: create instrumentation object defined in Manifest
		
        if (data.instrumentationName != null) {
            InstrumentationInfo ii = null;
            try {
                ii = appContext.getPackageManager().
                    getInstrumentationInfo(data.instrumentationName, 0);
            } catch (PackageManager.NameNotFoundException e) {
            }
            if (ii == null) {
                throw new RuntimeException(
                    "Unable to find instrumentation info for: "
                    + data.instrumentationName);
            }

            mInstrumentationPackageName = ii.packageName;
            mInstrumentationAppDir = ii.sourceDir;
            mInstrumentationSplitAppDirs = ii.splitSourceDirs;
            mInstrumentationLibDir = ii.nativeLibraryDir;
            mInstrumentedAppDir = data.info.getAppDir();
            mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
            mInstrumentedLibDir = data.info.getLibDir();

            ApplicationInfo instrApp = new ApplicationInfo();
            instrApp.packageName = ii.packageName;
            instrApp.sourceDir = ii.sourceDir;
            instrApp.publicSourceDir = ii.publicSourceDir;
            instrApp.splitSourceDirs = ii.splitSourceDirs;
            instrApp.splitPublicSourceDirs = ii.splitPublicSourceDirs;
            instrApp.dataDir = ii.dataDir;
            instrApp.nativeLibraryDir = ii.nativeLibraryDir;
            LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
            ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

            try {
                java.lang.ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            mInstrumentation.init(this, instrContext, appContext,
                   new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
                   data.instrumentationUiAutomationConnection);

            if (mProfiler.profileFile != null && !ii.handleProfiling
                    && mProfiler.profileFd == null) {
                mProfiler.handlingProfiling = true;
                File file = new File(mProfiler.profileFile);
                file.getParentFile().mkdirs();
                Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
            }

        } else {
            mInstrumentation = new Instrumentation();
        }

        if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
            dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
        } else {
            // Small heap, clamp to the current growth limit and let the heap release
            // pages after the growth limit to the non growth limit capacity. b/18387825
            dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
        }

        // Allow disk access during application and provider setup. This could
        // block processing ordered broadcasts, but later processing would
        // probably end up doing the same disk access.
        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        
        //@: here create the Application object
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                List<ProviderInfo> providers = data.providers;
                if (providers != null) {
                    installContentProviders(app, providers);
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
            	mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
            	//@: call application.onCreate
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
```

