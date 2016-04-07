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
#####激进方案
通过将LoadedApk放进缓存的map中，是的在activity实例化的时候，getPackageInfo能够正常运作，返回插件的LoadedApk，然后得到插件的classLoader，从而能够将插件的类加载到JVM中，JVM正常反射就能new Activity对象。
#####保守方案
通过将插件中的类写进主App的classLoader中，使用实例化activity的时候就算传入主App的classLoader也能将插件中的类加载进JVM，从而JVM能够通过反射new Activity对象。
```Java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
```
就可以创建插件Activity对象了

##激进方案，通过LoadedApk放进缓存中，插件中的actiivty对象就能被创建了

###获取mPackages对象
```Java
// 先获取到当前的ActivityThread对象
Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
currentActivityThreadMethod.setAccessible(true);
Object currentActivityThread = currentActivityThreadMethod.invoke(null);

// 获取到 mPackages 这个静态成员变量, 这里缓存了dex包的信息
Field mPackagesField = activityThreadClass.getDeclaredField("mPackages");
mPackagesField.setAccessible(true);
Map mPackages = (Map) mPackagesField.get(currentActivityThread);
```

###下一步是去哪里来一个LoadedApk放进mPackages中
从上面可知，private LoadedApk getPackageInfo(...)会自动用系统的方式去生成一个LoadedApk，但由于是private函数，所以Google在改动的时候丝毫不用担心会影响到用户，考虑到日后的兼容性，放弃直接调用的方案
间接调用这个private LoadedApk getPackageInfo(...)的public函数有两个，
分别是
public LoadedApk getPackageInfo(...)
public LoadedApk getPackageInfoNoCheck(...)
从函数名称上可以看出，前者需要作一些验证，但两个函数都能拿到LoadedApk，所以选择后者。

###goto getPackageInfoNoCheck(...)
```Java
public final LoadedApk getPackageInfoNoCheck(ApplicationInfo ai,
            CompatibilityInfo compatInfo) {}
```
getPackageInfoNoCheck(..)需要两个参数，
1、ApplicationInfo
2、CompatibilityInfo，
由于目前我们只关心调用该函数，兼容性问题可以暂搁一边，第二个参数输入CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO。
然后目前剩下的就ApplicationInfo了

###ApplicationInfo
Information you can retrieve about a particular application. This corresponds to information collected from the AndroidManifest.xml’s <application> tag.
所以ApplicationInfo是从AndroidManifest.xml中解析出来的数据结构
两者的关系就像是一个是方便人类识别的文档，一个是方便系统识别的文档。
而从AndroidManifest.xml到ApplicationInfo，中间需要parser -- PackageParser。

###PackageParser
由于基本上Android每个版本上的PackageParser都被Google改动，所以PackageParser兼容性很差，所以基本上每个版本要调用不同的PackageParser去解析AndroidManifest.xml。
以API23为例，
PackageParser有一个方法,generateApplication
```Java
public static ApplicationInfo generateApplicationInfo(Package p, int flags,
   PackageUserState state){}
```
以下是调用generateApplicationInfo的反射代码
```Java
Class<?> packageParserClass = Class.forName("android.content.pm.PackageParser");
// 首先拿到我们得终极目标: generateApplicationInfo方法
// API 23 !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
// public static ApplicationInfo generateApplicationInfo(Package p, int flags,
//    PackageUserState state) {
// 其他Android版本不保证也是如此.
Class<?> packageParser$PackageClass = Class.forName("android.content.pm.PackageParser$Package");
Class<?> packageUserStateClass = Class.forName("android.content.pm.PackageUserState");
Method generateApplicationInfoMethod = packageParserClass.getDeclaredMethod("generateApplicationInfo",
        packageParser$PackageClass,
        int.class,
                packageUserStateClass);
```

###goto generateApplicationInfo
该函数有3个参数：
1、Package
2、Integer
3、PackageUserState

1、PackageParser.Package
Representation of a full package parsed from APK files on disk. A package consists of a single base APK, and zero or more split APKs.
代表的是被解析后的apk信息，就是通过PackageParser.parsePackage去解析得到apk信息

1.1、PackageParser.parserPackage
```Java
// 首先, 我们得创建出一个Package对象出来供这个方法调用
// 而这个需要得对象可以通过 android.content.pm.PackageParser#parsePackage 这个方法返回得 Package对象得字段获取得到
// 创建出一个PackageParser对象供使用
Object packageParser = packageParserClass.newInstance();
// 调用 PackageParser.parsePackage 解析apk的信息
Method parsePackageMethod = packageParserClass.getDeclaredMethod("parsePackage", File.class, int.class);

// 实际上是一个 android.content.pm.PackageParser.Package 对象
Object packageObj = parsePackageMethod.invoke(packageParser, apkFile, 0);
```
反射调用之后，就会得到第一个参数package

2、Integer
由于我们解析整个apk包，这里传0

3、PackageUserState
PackageUserState代表的是不同用户中的信息，Android是一个多用户多任务系统。
这里只需要默认即可
```Java
Object defaultPackageUserState = packageUserStateClass.newInstance();
```

至此，generateApplicationInfo的三个参数已准备好
```Java
ApplicationInfo applicationInfo = (ApplicationInfo) generateApplicationInfoMethod.invoke(packageParser,
        packageObj, 0, defaultPackageUserState);
//由于系统调用的generateApplicationInfo得到的ApplicationInfo并没有apk文件本身的信息，所以要设置下，建立目标apk与这个ApplicationInfo对象的关联。
String apkPath = apkFile.getPath();
applicationInfo.sourceDir = apkPath;
applicationInfo.publicSourceDir = apkPath;
```


###execute getPackageInfoNoCheck(...)
由于getPackageInfoNoCheck(...)的入参已得到，直接调用即可获得LoadedApk
```Java
// android.content.res.CompatibilityInfo
Class<?> compatibilityInfoClass = Class.forName("android.content.res.CompatibilityInfo");
Method getPackageInfoNoCheckMethod = activityThreadClass.getDeclaredMethod("getPackageInfoNoCheck", ApplicationInfo.class, compatibilityInfoClass);

Field defaultCompatibilityInfoField = compatibilityInfoClass.getDeclaredField("DEFAULT_COMPATIBILITY_INFO");
defaultCompatibilityInfoField.setAccessible(true);

Object defaultCompatibilityInfo = defaultCompatibilityInfoField.get(null);
ApplicationInfo applicationInfo = generateApplicationInfo(apkFile);

Object loadedApk = getPackageInfoNoCheckMethod.invoke(currentActivityThread, applicationInfo, defaultCompatibilityInfo);
```

###add LoadedApk to ActivityThread.mPackages
```Java
mPackages.put(applicationInfo.packageName, loadedApk);
```

###题外话1、为了日后对插件的类的加载作控制，自定义一个ClassLoader
```Java
public class CustomClassLoader extends DexClassLoader {

    public CustomClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent) {
        super(dexPath, optimizedDirectory, libraryPath, parent);
    }
}
```
目前只需要这样即可，不需要作任何额外处理
```Java
String odexPath = Utils.getPluginOptDexDir(applicationInfo.packageName).getPath();
String libDir = Utils.getPluginLibDir(applicationInfo.packageName).getPath();
ClassLoader classLoader = new CustomClassLoader(apkFile.getPath(), odexPath, libDir, ClassLoader.getSystemClassLoader());
Field mClassLoaderField = loadedApk.getClass().getDeclaredField("mClassLoader");
mClassLoaderField.setAccessible(true);
mClassLoaderField.set(loadedApk, classLoader);
```
替换loadedApk中的ClassLoader

###题外话2、稳定
```Java
// 由于是弱引用, 因此我们必须在某个地方存一份, 不然容易被GC; 那么就前功尽弃了.
sLoadedApk.put(applicationInfo.packageName, loadedApk);
WeakReference weakReference = new WeakReference(loadedApk);
mPackages.put(applicationInfo.packageName, weakReference);
```

###run
运行时报错了
```Java
04-05 02:49:53.742  11759-11759/com.weishu.upf.hook_classloader E/AndroidRuntime﹕ FATAL EXCEPTION: main
    Process: com.weishu.upf.hook_classloader, PID: 11759
    java.lang.RuntimeException: Unable to start activity ComponentInfo{com.weishu.upf.ams_pms_hook.app/com.weishu.upf.ams_pms_hook.app.MainActivity}: java.lang.RuntimeException: Unable to instantiate application android.app.Application: java.lang.IllegalStateException: Unable to get package info for com.weishu.upf.ams_pms_hook.app; is package not installed?
```
无法实例化application，由于插件未安装。
而application的实例化，代码也在performLaunchActivity中

###goto performLaunchActivity
```Java
try {
    java.lang.ClassLoader cl = getClassLoader();
    if (!mPackageName.equals("android")) {
        initializeJavaContextClassLoader();
    }
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    app = mActivityThread.mInstrumentation.newApplication(
            cl, appClass, appContext);
    appContext.setOuterContext(app);
} catch (Exception e) {
    if (!mActivityThread.mInstrumentation.onException(app, e)) {
        throw new RuntimeException(
            "Unable to instantiate application " + appClass
            + ": " + e.toString(), e);
    }
}
```
这里只能一行行代码检查了
1、getClassLoader()
该函数中没有throw任何错误，跳过
2、initializeJavaContextClassLoader();
```Java
private void initializeJavaContextClassLoader() {
    IPackageManager pm = ActivityThread.getPackageManager();
    android.content.pm.PackageInfo pi;
    try {
        pi = pm.getPackageInfo(mPackageName, 0, UserHandle.myUserId());
    } catch (RemoteException e) {
        throw new IllegalStateException("Unable to get package info for "
                + mPackageName + "; is system dying?", e);
    }
    if (pi == null) {
        throw new IllegalStateException("Unable to get package info for "
                + mPackageName + "; is package not installed?");
    }

    boolean sharedUserIdSet = (pi.sharedUserId != null);
    boolean processNameNotDefault =
        (pi.applicationInfo != null &&
         !mPackageName.equals(pi.applicationInfo.processName));
    boolean sharable = (sharedUserIdSet || processNameNotDefault);
    ClassLoader contextClassLoader =
        (sharable)
        ? new WarningContextClassLoader()
        : mClassLoader;
    Thread.currentThread().setContextClassLoader(contextClassLoader);
}
```

由于pm.getPackageInfo(...)返回null，所以throw IllegalStateException。
就是这么任性，走都走到这里，真的不回头

###hookPackageManager()
```Java
private static void hookPackageManager() throws Exception {

    // 这一步是因为 initializeJavaContextClassLoader 这个方法内部无意中检查了这个包是否在系统安装
    // 如果没有安装, 直接抛出异常, 这里需要临时Hook掉 PMS, 绕过这个检查.

    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
    Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
    currentActivityThreadMethod.setAccessible(true);
    Object currentActivityThread = currentActivityThreadMethod.invoke(null);

    // 获取ActivityThread里面原始的 sPackageManager
    Field sPackageManagerField = activityThreadClass.getDeclaredField("sPackageManager");
    sPackageManagerField.setAccessible(true);
    Object sPackageManager = sPackageManagerField.get(currentActivityThread);

    // 准备好代理对象, 用来替换原始的对象
    Class<?> iPackageManagerInterface = Class.forName("android.content.pm.IPackageManager");
    Object proxy = Proxy.newProxyInstance(iPackageManagerInterface.getClassLoader(),
            new Class<?>[] { iPackageManagerInterface },
            new IPackageManagerHookHandler(sPackageManager));

    // 1. 替换掉ActivityThread里面的 sPackageManager 字段
    sPackageManagerField.set(currentActivityThread, proxy);
}
```
这里真的不懂啊，膝盖碎了一地，有机会再了解hook.
###激进方案完

##保守方案
###复习一下private LoadedApk getPackageInfo(...)
```Java
private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
        ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
        boolean registerPackage) {
    final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
    synchronized (mResourcesManager) {
        WeakReference<LoadedApk> ref;
        if (differentUser) {
            // Caching not supported across users
            ref = null;
        } else if (includeCode) {
            ref = mPackages.get(aInfo.packageName);
        } else {
            ref = mResourcePackages.get(aInfo.packageName);
        }

        LoadedApk packageInfo = ref != null ? ref.get() : null;
        if (packageInfo == null || (packageInfo.mResources != null
                && !packageInfo.mResources.getAssets().isUpToDate())) {
            packageInfo =
                new LoadedApk(this, aInfo, compatInfo, baseLoader,
                        securityViolation, includeCode &&
                        (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

            // 略
    }
}
```
保守方案：
假设我们什么都不改动，调用插件的activity进行实例化的时候，getPackageInfo()会返回new LoadedApk(...).
此处再次引用activity实例化的代码：
```Java
java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
StrictMode.incrementExpectedActivityCount(activity.getClass());
r.intent.setExtrasClassLoader(cl);
```
可以看出，通过LoadedApk的getClassLoader()去获取类加载器，然后classLoader将目标类加载进JVM,最后通过反射新建了这个activity对象。
这时候问题就来了，我们的插件的类没有进行安装，没有出现在主APP的AndroidManifest.xml(ApplicationInfo)中，所以加载不到插件的类进来。
由于主角自带光环，所以轻易看出怎么样才可以让主App的classLoader能够加载没有在主LoadedApk中的类。
首先，主角告诉我们，
主LoadedApk所拥有的classLoader是通过系统的getClassLoader()得到的。

###goto LoadedApk.getClassLoader()
```Java
public ClassLoader getClassLoader() {
    synchronized (this) {
        if (mClassLoader != null) {
            return mClassLoader;
        }

        if (mIncludeCode && !mPackageName.equals("android")) {
            // 略...
            mClassLoader = ApplicationLoaders.getDefault().getClassLoader(zip, lib,
                    mBaseClassLoader);

            StrictMode.setThreadPolicy(oldPolicy);
        } else {
            if (mBaseClassLoader == null) {
                mClassLoader = ClassLoader.getSystemClassLoader();
            } else {
                mClassLoader = mBaseClassLoader;
            }
        }
        return mClassLoader;
    }
}
```
这里有两个ClassLoader，
一个是系统的，一个是主App的。
getSystemClassLoader()，系统级别的classLoader，不是不能加载未安装的类，是牵涉的太多了。
一只鸡，一头牛，能杀其一就可以通关，你挑哪个？
所以，目标是ApplicationLoaders.getDefault().getClassLoader();

###ApplicationLoaders.getDefault().getClassLoader(...)
```Java
public ClassLoader getClassLoader(String zip, String libPath, ClassLoader parent)
{

    ClassLoader baseParent = ClassLoader.getSystemClassLoader().getParent();

    synchronized (mLoaders) {
        if (parent == null) {
            parent = baseParent;
        }

        if (parent == baseParent) {
            ClassLoader loader = mLoaders.get(zip);
            if (loader != null) {
                return loader;
            }

            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
            PathClassLoader pathClassloader =
                new PathClassLoader(zip, libPath, parent);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            mLoaders.put(zip, pathClassloader);
            return pathClassloader;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, zip);
        PathClassLoader pathClassloader = new PathClassLoader(zip, parent);
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        return pathClassloader;
    }
}
```
可以看到classLoader都是PathClassLoader的实例。

###PathClassLoader.java
```Java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }

    public PathClassLoader(String dexPath, String libraryPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
}
```
继续向BaseDexClassLoader出发
BaseDexClassLoader中，主角看到，与classLoader相关的就只有defineClass与findClass方法。
然后主角确定与findClass有关

###goto BaseDexClassLoader.findClass(...)
```Java
public Class findClass(String name, List<Throwable> suppressed) {
   for (Element element : dexElements) {
       DexFile dex = element.dexFile;

       if (dex != null) {
           Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
           if (clazz != null) {
               return clazz;
           }
       }
   }
   if (dexElementsSuppressedExceptions != null) {
       suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
   }
   return null;
}
```
可以看到findClass最后都是通过寻找二进制文件将类加载进JVM，我们只需要将插件的dex放到dexElements中即可。
这样主App的classLoader在遍历dexElements的时候就能找到插件的类，然后加载进JVM。
```Java
public static void patchClassLoader(ClassLoader cl, File apkFile, File optDexFile)
        throws IllegalAccessException, NoSuchMethodException, IOException, InvocationTargetException, InstantiationException, NoSuchFieldException {
    // 获取 BaseDexClassLoader : pathList
    Field pathListField = DexClassLoader.class.getSuperclass().getDeclaredField("pathList");
    pathListField.setAccessible(true);
    Object pathListObj = pathListField.get(cl);

    // 获取 PathList: Element[] dexElements
    Field dexElementArray = pathListObj.getClass().getDeclaredField("dexElements");
    dexElementArray.setAccessible(true);
    Object[] dexElements = (Object[]) dexElementArray.get(pathListObj);

    // Element 类型
    Class<?> elementClass = dexElements.getClass().getComponentType();

    // 创建一个数组, 用来替换原始的数组
    Object[] newElements = (Object[]) Array.newInstance(elementClass, dexElements.length + 1);

    // 构造插件Element(File file, boolean isDirectory, File zip, DexFile dexFile) 这个构造函数
    Constructor<?> constructor = elementClass.getConstructor(File.class, boolean.class, File.class, DexFile.class);
    Object o = constructor.newInstance(apkFile, false, apkFile, DexFile.loadDex(apkFile.getCanonicalPath(), optDexFile.getAbsolutePath(), 0));

    Object[] toAddElementArray = new Object[] { o };
    // 把原始的elements复制进去
    System.arraycopy(dexElements, 0, newElements, 0, dexElements.length);
    // 插件的那个element复制进去
    System.arraycopy(toAddElementArray, 0, newElements, dexElements.length, toAddElementArray.length);

    // 替换
    dexElementArray.set(pathListObj, newElements);

}
```

##到此为止，已经写完了。
至于小结，还是看原文吧。
至于我，写完这篇文章已经消化了这篇技术文章。
