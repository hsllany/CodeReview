AppToken
====


AppToken是Acitivity、ActivityRecord、ActivityClientRecord，甚至是WindowState之间联系的纽带。

```
Activity.mToken
ActivityRecord.appToken
ActivityClientRecord.token
```

ActivityThread存储在mActivities中，以IBinder为键，值为ActivityClientRecord

在ActivityManagerService中，appToken通过其弱引用找到对应的activityRecord

WindowManagerService中，通过token找到对应的WindowToken。存储在:

```
  /**
     * Mapping from a token IBinder to a WindowToken object.
     */
    final HashMap<IBinder, WindowToken> mTokenMap = new HashMap<>();
```

# 定义及初始化
初始化在ActivityRecord的构造方法中：

```
ActivityRecord(ActivityManagerService _service, ProcessRecord _caller,
            int _launchedFromUid, String _launchedFromPackage, Intent _intent, String _resolvedType,
            ActivityInfo aInfo, Configuration _configuration,
            ActivityRecord _resultTo, String _resultWho, int _reqCode,
            boolean _componentSpecified, ActivityStackSupervisor supervisor,
            ActivityContainer container, Bundle options) {
		...
        appToken = new Token(this);
        ...
```

其定义在ActivityRecord中：

```
static class Token extends IApplicationToken.Stub {
        final WeakReference<ActivityRecord> weakActivity;

        Token(ActivityRecord activity) {
            weakActivity = new WeakReference<ActivityRecord>(activity);
        }
```

本质上是一个Binder，ActivityManagerService维系着各应用的栈，栈中存储的ActivityClient，其持有该Binder的服务端，而传递给ActivityThread, Activity的是其BP端。

在BN端，Token持有了一个弱引用，指向对应的ActivityRecord，这样，每次client端调用ActivityManagerService的系统方法，ActivityManagerService就能根据这个弱引用快速的找到对应的ActivityRecord.

```
ActivityRecord r = ActivityRecord.isInStackLocked(token);
以及
final ActivityRecord r = ActivityRecord.forToken(token);
```


