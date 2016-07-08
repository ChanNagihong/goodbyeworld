#### Xposed初体验  
##### 允许应用通过权限检查   
```Java  
public class HookSystemPermission implements IXposedHookZygoteInit {

    /**
    *  记录上次本应用的uid，不用每次去查
    */
    private static int mLastFoundUid = Integer.MIN_VALUE;

    @Override
    public void initZygote(StartupParam startupParam) throws Throwable {
        XC_MethodHook methodHook = new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                if (param.args[0].equals(Manifest.permission.WRITE_SECURE_SETTINGS)) {
                    String name = AndroidAppHelper.currentApplication().getPackageManager().getNameForUid((int) param.args[2]);
                    int thisUid = (int) param.args[2];
                    if (mLastFoundUid != Integer.MIN_VALUE) {
                        if (mLastFoundUid == thisUid) {
                            /**
                            *  返回0表示允许
                            */
                            param.setResult(0);
                        }
                    } else {
                        if (name.equals("--packagename--")) {
                            /**
                            *  返回0表示允许
                            */
                            mLastFoundUid = thisUid;
                            param.setResult(0);
                        }
                    }
                }
            }
        };
        Object[] checkPermissionArgs = new Object[]{String.class, int.class, int.class, methodHook};
        XposedHelpers.findAndHookMethod("android.app.ContextImpl", ClassLoader.getSystemClassLoader(), "checkPermission", checkPermissionArgs);
        checkPermissionArgs = new Object[]{String.class, int.class, int.class, IBinder.class, methodHook};
        XposedHelpers.findAndHookMethod("android.app.ContextImpl", ClassLoader.getSystemClassLoader(), "checkPermission", checkPermissionArgs);
    }
}  
```  
##### 监听音量加减键(在屏幕关闭时，是不会触发监听的)  
```Java  
public class HookPhysicalButtons implements IXposedHookZygoteInit {

    @Override
    public void initZygote(StartupParam startupParam) throws Throwable {
        XC_MethodHook methodHook = new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
            }

            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                if(null == param.args || param.args.length < 1) {
                    return;
                }
                KeyEvent keyEvent = (KeyEvent) param.args[0];
                if(keyEvent.getAction() != KeyEvent.ACTION_UP) {
                    return;
                }
                if(keyEvent.getKeyCode() == KeyEvent.KEYCODE_VOLUME_UP) {
                    // todo...
                } else if(keyEvent.getKeyCode() == KeyEvent.KEYCODE_VOLUME_DOWN) {
                    // todo...
                }
            }
        };
        Class keyEventClass = XposedHelpers.findClass("android.view.KeyEvent", ClassLoader.getSystemClassLoader());
        Object[] objs = new Object[]{keyEventClass, int.class, boolean.class, methodHook};
        XposedHelpers.findAndHookMethod("com.android.internal.policy.impl.PhoneWindowManager", ClassLoader.getSystemClassLoader(), "interceptKeyBeforeQueueing", objs);

    }
}
```
