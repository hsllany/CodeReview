http://blog.csdn.net/hohohong/article/details/54616053

- WindowSession每个进程只有一个，保存在WindowManagerGlobal的静态变量中。实现类是com.android.server.wm.Session.java，APP获得是BP端；
- ViewRootImpl首先调用WindowSession的addToDisplay()方法。将IWindow传入，这里IWindow的BN端在APP测。

# WindowManager.LayoutParams

## type

type应用测是TYPE_APPLICATION = 2;

分为三档：
1-99： TYPE_APPLICATION
1000 - 1999: TYPE_SUB_APPLICATION
2000 - 2999: SYSTEM_APPLICATION

# WindowManagerService.addWindow

mTokenMap，以appToken为键，存储了WindowToken。
其在addWindow方法中，判断了token；（有的dialog不能使用Application的context打开，也是因为这里的判断没有过）

其回为每个window建立一个WindowState对象。

