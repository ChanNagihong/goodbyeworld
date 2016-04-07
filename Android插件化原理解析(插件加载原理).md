原文网址：http://weishu.me/2016/04/05/understand-plugin-framework-classloader/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io

###创建Activity类对象的过程
```Java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
activity = mInstrumentation.newActivity(
        cl, component.getClassName(), r.intent);
StrictMode.incrementExpectedActivityCount(activity.getClass());
r.intent.setExtrasClassLoader(cl);
```
这是正常情况下创建Activity类对象的过程(系统通过ClassLoader加载了需要的Activity类并通过反射构造函数创建Activity对象)
在插件化中，由于代码存在独立于原进程的文件之中，classLoader不能直接加载。因此必须hook，才能加载。

###ClassLoader机制
>Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校检、转换解析和初始化的，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。
与那些在编译时进行链连接工作的语言不同，在Java语言里面，类型的加载、连接和初始化都是在程序运行期间完成的，这种策略虽然会令类加载时稍微增加一些性能开销，但是会为Java应用程序提供高度的灵活性，Java里天生可以同代拓展的语言特性就是依赖运行期动态加载和动态链接这个特点实现的。例如，如果编写一个面相接口的应用程序，可以等到运行时在制定实际的实现类；用户可以通过Java与定义的和自定义的类加载器，让一个本地的应用程序可以在运行时从网络或其他地方加载一个二进制流作为代码的一部分，这种组装应用程序的方式目前已经广泛应用于Java程序之中。从最基础的Applet，JSP到复杂的OSGi技术，都使用了Java语言运行期类加载的特性。
####1、class to java
####2、on RunTime
####3、高度灵活性，如执行刚从网络加载的一个二进制流转换过来的java代码

###插件化方案
将插件的dex或者apk文件植入到合适的DexClassLoader，借助这个ClassLoader完成类的加载

###ClassLoader入手
在Activity创建对象过程代码当中可以看到
```Java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
```
而r.packageInfo是一个LoadedApk类的对象。

###LoadedApk
官方文档
Local state maintained about a currently loaded apk.
就是已经被加载到jvm中的apk文件。
一个App程序运行所需的信息，如res、Activity，Service等所有代码都从这个Apk文件获取。

###解决了从r.packageInfo再到getClassLoader()的问题，理论上就可以加载插件的代码了
####从performLaunchActivity上溯，辗转到handleLaunchActivity, case LAUNCH_ACTIVITY，找到r.packageInfo的来源：
```Java
final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
r.packageInfo = getPackageInfoNoCheck(
        r.activityInfo.applicationInfo, r.compatInfo);
handleLaunchActivity(r, null);
```
####goto getPackageInfoNoCheck
```Java
public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,
        CompatibilityInfo compatInfo) {
    return getPackageInfo(ai, compatInfo, null, false, true, false);
}
```

###goto getPackageInfo
```Java
private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
        ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
        boolean registerPackage) {
        // 获取userid信息
        final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
        synchronized (mResourcesManager) {
        // 尝试获取缓存信息
        // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!激进方案
        WeakReference<LoadedApk> ref;
        if (differentUser) {
            // Caching not supported across users
            ref = null;
        } else if (includeCode) {
            //if not differentUser, sharing codes and resources
            ref = mPackages.get(aInfo.packageName);
        } else {
            //if not differentUser, sharing resources only
            ref = mResourcePackages.get(aInfo.packageName);
        }

        LoadedApk packageInfo = ref != null ? ref.get() : null;
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
                // 缓存没有命中，直接new
                // !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!保守方案
            packageInfo =
                new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode &&
                        (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

        // 省略。。更新缓存
        return packageInfo;
    }
}
```

###由此可见，可有两个方案
*激进方案
通过将LoadedApk放进缓存的map中，自然会返回ref
*保守方案
通过new LoadedApk，就能得到ClassLoader,
```Java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
```
就可以创建插件Activity对象了
