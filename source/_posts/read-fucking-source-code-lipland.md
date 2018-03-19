---
title: read-fucking-source-code-lipland
date: 2018-03-14 15:20:08
categories: 
- read_fucking_source_code
tags: 
- read_fucking_source_code
- retrofit
---


本文主要分享Lipland一些核心代码和一些流程分析。

<!-- more -->
# Read fucking source code:Lipland
## 关键技术说明
插件化的技术，大量的使用了hook技术，而hook过程就是利用java的反射和动态代理，替换原有类中的属性对象或者 hook 原有类中的方法，一般情况下，如果属性对象是非final Class，可以直接写一个类继承目标Class，然后重写一些关键方法的实现过程；如果属性对象是接口或者一个final的Class，就需要通过动态代理去hook 那些方法。

第一种，替换属性对象，拿这个项目中的hook ActivityThread的Instrumentation为例：
通过ActivityThread对象和class反射根据fieldName获取Instrumentation Field，强转为Instrumentation，然后把Instrumentation的所有属性都通过反射clone赋值给我们自己写的InstrumentationHacker，然后把InstrumentationHacker再通过反射重新设置回给ActivityThread，这样就把Instrumentation替换成InstrumentationHacker。

第二种，动态代理劫持方法,自己写一个InvocationHandler的实现类，在invoke方法中，通过方法名字的过滤，加上自己的逻辑处理，可以直接阻断原方法执行的逻辑，替换为自己的逻辑处理，简单的实例如下面的代码，我把MathOp接口中的sum方法劫持，把参数加大10
```
   public void addition_isCorrect() throws Exception {
        MathOpImpl m = new MathOpImpl();
        MathOp mi = (MathOp) Proxy.newProxyInstance(
                MathOpImpl.class.getClassLoader(),
                new Class<?>[]{MathOp.class},
                new MInvokeHandler(m));
        System.out.println(mi.sum(10, 2));
        assertEquals(4, 2 + 2);
    }

    class MInvokeHandler implements InvocationHandler {
        MathOp mi;
        public MInvokeHandler(MathOp mi) {
            this.mi = mi;
        }
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object result = null;
            System.out.println("before");
            if (method.getName().contains("sum")) {
                int a = (int) args[0];
                a += 10;
                args[0] = a;
                result = method.invoke(mi, args);
            } else {
                result = method.invoke(mi, args);
            }
            System.out.println("after");
            return result;
        }
    }

    interface MathOp {
        int sum(int a, int b);
        int max(int a, int b);
    }

    class MathOpImpl implements MathOp {
        public int sum(int a, int b) {
            return a + b;
        }
        @Override
        public int max(int a, int b) {
            return a > b ? a : b;
        }
    }
```
拿这个项目中的hook ActivityThread的ActivityManager的关键方法startService为例，关键InvokeHandler代码：
```
	@Override
	public Object invoke(Object obj, Method method, Object[] args)
			throws Throwable {
		if(hookHandler != null){
			if(!hookHandler.onBefore(origin, method, args))
					return hookHandler.result;
		}
		Object result = null;
		Throwable thr = null;
		try{
			if(hookHandler != null){
				result = hookHandler.invoke(origin,method, args);
			}else{
				result = method.invoke(origin, args);
			}
		}catch(Throwable th){
			if(th instanceof InvocationTargetException){  
                thr = ((InvocationTargetException) th).getTargetException();  
            }else{  
                //doXXX()  
    			thr = th;
            }  
		}
		try{
			if(hookHandler != null){
				result = hookHandler.onAfter(origin, method, args, result,thr);
			}
		}catch(Exception e){
			Log.e(TAG, e);
		}
		if(thr != null)
			throw thr;
		return result;
	}
```

实现流程
1.插件初始化
```
    public static boolean init(Application app){
        PluginHelper.app = app;
        if (!isPluginsSupport()) {
            return false;
        }
        //安装插件管理器，必须的步骤
        PluginManager.setup(app);
        PluginManager pluginManager = PluginManager.getInstance();
        //开启调试模式，调试模式下，打开日志。
        //不验证插件签名
        pluginManager.setDebug(true);
        pluginManager.setVerifySign(false);
        //配置插件管理器，可选
        configure();
        return true;
    }
```

1.1 PluginManager初始化，pm.init(); ，然后 调用HostApplicationProxy.hook(app);
```
private static boolean setup(final Application app,final boolean startPluginProcess,boolean installDefaultPlugin,boolean forceInstallDefaultPlugin){
		HostGlobal.init(app);
		// 4.0以下版本不支持
		if (android.os.Build.VERSION.SDK_INT < VERSION_CODES.ICE_CREAM_SANDWICH) {
			return false;
		}
		if(!inited){
			final PluginManager pm = PluginManager.getInstance();
			pm.init();
			HostApplicationProxy.hook(app);
			pm.runInBackground(new Runnable() {
				@Override
				public void run() {
					NetworkManager.getInstance(app);
					app.registerComponentCallbacks(new ComponentCallbacks() {、
						@Override
						public void onLowMemory() {
						}
						@Override
						public void onConfigurationChanged(Configuration newConfig) {
							// 同步插件中资源的onConfigurationChanged事件
							Map<String, Plugin> plugins = PluginManager.getInstance().getPlugins();
							for (Plugin p : plugins.values()) {
								p.getRes().updateConfiguration(newConfig, app.getResources().getDisplayMetrics());
							}
						}
					});
					if(pm.isMainProcess() && startPluginProcess){
						PluginHelper.startPluginProcess(null);
					}

					if(pm.isPluginProcess()) {

//					if (installDefaultPlugin)
//						pm.installDefaultPlugins(forceInstallDefaultPlugin);
						pm.resetProxyActivitiesStatus();
//						//加载有loadOnAppStarted标志的插件，该标志表示在应用启动时在后台将插件加载到内存
//						//有这个标志的插件，启动相对会快一点，但是在没有启动时也会占用部分内存。
//						pm.loadPluginOnAppStarted();
							}
//					if(pm.isPluginProcess() && startUpdate){
//						pm.startUpdate();
//					}
				}
			});
			inited = true;
		}
		return true;
	}
```
```
static void hook(final Application app){
		activityThread = ActivityThread.currentActivityThread();
		injectInstrumentation(activityThread);
		/**
		 * 挂钩系统回调
		 */
//		SystemCallbackHacker.hook(activityThread);
		/**
		 * 挂钩ActivityManager
		 */
//		hookActivityManager();
		//策略一
		//默认主进程不挂钩PackageManager
		//也可以在启动完毕后，延迟挂钩，这样可以忽略性能影响
//		if(!HostGlobal.isMainProcess()){
//			hookPackageManager();
//		}
		//策略二
		//主进程延迟500毫秒挂钩，等程序启动起来了再说
		Runnable run = new Runnable() {
			@Override
			public void run() {
				hookActivityManager();
				IClipboardHacker.hook();
				ITelephonyHacker.hook();
				IMountServiceHacker.hook();
				INotificationManagerHacker.hook();
//				IAccessibilityManagerHacker.hook();
			}
		};
		initService(app);
		hookPackageManager(app);
		if(!HostGlobal.isMainProcess()){
			run.run();
		}else{
			new Handler().postDelayed(run, 500);
		}
//		run.run();
//		hookAppOpsManager();
```

1.2 PluginManager#init 
1.2.1、初始化，用Application注册了加载插件的监听
1.2.2、初始化InstallManager，ActivityThread
1.2.3com.qihoo.plugin.install.InstallManager#InstallManager  初始化
1.2.3.1、初始化com.qihoo.plugin.update.UpdateManager，com.qihoo.plugin.update.UpdateManager#init
1.2.3.2、com.qihoo.plugin.install.InstallManager#initData

1.3.1 hook ActivityThread的Instrumentation，替换为InstrumentationHacker,用来劫持startactivity一系列方法，启动代理Activity；
1.3.2 hook ActivityThread的ActivityManager中的关键方法
1.3.2.1 劫持startServic、bindService方法，在方法之前，经过一些判断，调用PluginManager.startService、bindService方法启动代理service；
1.3.2.2 劫持stopService、stopServiceToken、unbindService方法，同样的在方法之前经过一些判断，调用pluginManager.stopService、stopServiceToken、unbindService来处理代理启动的service;
1.3.2.3 劫持getIntentSender、getRunningAppProcesses、checkPermission、checkPermissionWithToken等方法
1.3.3 hook其他的一些点
```
IClipboardHacker.hook();
ITelephonyHacker.hook();
IMountServiceHacker.hook();
INotificationManagerHacker.hook();
```
1.3.4 hook PackageManager
1.3.4.1 hook getPackageInfo方法用来处理插件的包信息，如果是宿主包名走原生逻辑，如果是插件，调用pluginManager.getInstallManager()来获取插件包信息
1.3.4.2 hook getActivityInfo 方法 处理Activity的信息
1.3.4.3 hook getApplicationInfo 方法 处理插件的 Application的信息
1.3.4.4 hook queryIntentActivities 方法 处理查询activity的信息，扩展查询插件包里的activity

1.4 启动插件进程 PluginHelper.startPluginProcess(null);
```
pm.runInBackground(new Runnable() {
				@Override
				public void run() {
					NetworkManager.getInstance(app);
					app.registerComponentCallbacks(new ComponentCallbacks() {
						@Override
						public void onLowMemory() {

						}
						@Override
						public void onConfigurationChanged(Configuration newConfig) {
							// 同步插件中资源的onConfigurationChanged事件
							Map<String, Plugin> plugins = PluginManager.getInstance().getPlugins();
							for (Plugin p : plugins.values()) {
								p.getRes().updateConfiguration(newConfig, app.getResources().getDisplayMetrics());
							}
						}
					});
					if(pm.isMainProcess() && startPluginProcess){
						PluginHelper.startPluginProcess(null);
					}
```

1.4.1 启动 PluginProcessStartup这个service，在service内部去加载插件
```
pluginManager.installDefaultPlugins(false);->InstallManager#install(PluginInfo,syncPreproccess)
pluginManager.handlePendingInstallPlugin();->UpdateManager#installPlugin->InstallManager#install(PluginInfo,syncPreproccess) 
pluginManager.preproccess();
pluginManager.loadPluginOnAppStarted();
```
pluginManager.installDefaultPlugins(false);内部去解析xml，然后copy 插件apk到插件目录，以及一些检测插件安全，然后预加载一次apk，具体就是DexClassLoader加载一次，把宿主的classloader传过来，用于加载类，重写loadClass方法，在调用super.loadClass找不到类的时候，尝试使用宿主的classloader去加载类 clazz = hostClassLoader.loadClass(className);
```
public class DexClassLoaderEx extends BaseDexClassLoader {
	
	private final static String TAG = DexClassLoaderEx.class.getSimpleName();

	private ClassLoader hostClassLoader;
	private Context context;
	private String tag;
	
	private Map<String,Map<String,LibInfo>> libs;

	public DexClassLoaderEx(String tag,Map<String,Map<String,LibInfo>> libs,Context context,String dexPath, String optimizedDirectory,
			String libraryPath, ClassLoader parent, ClassLoader hostLoader) {
		super(dexPath, new File(optimizedDirectory), libraryPath, parent);
		
		this.hostClassLoader = hostLoader;
		this.context = context;
		this.tag = tag;
		this.libs = libs;
	}

	public Class<?> loadClassOrig(String className) throws ClassNotFoundException {
		return super.loadClass(className);
	}

	@Override
	public String findLibrary(String name) {
		// TODO Auto-generated method stub
		if(libs != null){
			if(libs.containsKey(tag)){
				Map<String,LibInfo> pLibs = libs.get(tag);
				if(pLibs.containsKey(name))
					name = pLibs.get(name).mappingName;
			}
		}
		String filename = super.findLibrary(name);
		if(filename == null)
			return context.getApplicationInfo().nativeLibraryDir+"/lib"+name+".so";
		else
			return filename;
	}
	
	@TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
	@Override
	public Class<?> loadClass(String className) throws ClassNotFoundException {
		Log.d(TAG, "loadClass [" + className + "]");
		
		Class<?> clazz = null;
		try {
			try {
				clazz = super.loadClass(className);
			} catch (ClassNotFoundException e) {
				Log.d(TAG, "Class not found [" + className
						+ "],try to find from the host class loader");
				clazz = hostClassLoader.loadClass(className);
			}
		} catch (java.lang.IllegalAccessError err) {
			Log.e(TAG, "className= [" + className + "],cl=" + this + ",parent="
					+ this.getParent());
			Log.e(TAG, "className= [" + className + "],hostClassLoader="
					+ hostClassLoader + ",parent" + hostClassLoader.getParent());
			throw err;
		}
		return clazz;
	}

}
```
1.4.2
pluginManager.loadPluginOnAppStarted();->PluginManager#load(String, IPluginLoadListener) 加载插件
处理插件apk的所有四大组件信息，native 库，classloader，Resources
```
DexClassLoaderEx classLoader = new DexClassLoaderEx(tag,libs,application,
				apkPath, Config.getDexWorkPath(tag),
				soPath, application.getClassLoader().getParent(),
				application.getClassLoader());
```
处理Resources，PluginManager#loadResources(Context, String, boolean)，利用反射new一个AssetManager,然后把apk的目录通过反射调用addAssetPath方法加进去，然后new 一个Resources
```
Object assetMag;
		Class<?> class_AssetManager = Class
				.forName("android.content.res.AssetManager");
		assetMag = class_AssetManager.newInstance();
		Method method_addAssetPath = class_AssetManager.getDeclaredMethod(
				"addAssetPath", String.class);
		method_addAssetPath.invoke(assetMag, path);

		// 5.0系统单独处理WebView
//		if (useWebView && Build.VERSION.SDK_INT >= 21) {
//
//			try {
//				Class<?> cls = Class.forName("android.webkit.WebViewFactory");
//				Method getProviderMethod = cls.getDeclaredMethod("getProvider",
//						new Class[] {});
//				getProviderMethod.setAccessible(true);
//
//				getProviderMethod.invoke(cls);
//				PackageInfo pi = (PackageInfo) RefUtil.getFieldValue(cls,
//						"sPackageInfo");
//				// String webViewAssetPath = pi.applicationInfo.sourceDir;
//				// String packageName = pi.packageName;
//				// application.getPackageManager().getPackageInfo(packageName,
//				// 0);
//				// Context webViewContext =
//				// application.createPackageContext(packageName,
//				// Context.CONTEXT_INCLUDE_CODE |
//				// Context.CONTEXT_IGNORE_SECURITY);
//				method_addAssetPath.invoke(assetMag,
//						pi.applicationInfo.sourceDir);
//			} catch (Exception e) {
//				e.printStackTrace();
//			}
//		}

		Resources res = application.getResources();
		return new Resources((AssetManager) assetMag,res.getDisplayMetrics(), res.getConfiguration())
```
1.3 检查插件更新 PluginManager#startUpdate(boolean, java.lang.Class<? extends com.qihoo.plugin.update.UpdateFilter>)
2 打开插件中的页面
2.1 正常调用 startActivity，然后在InstrumentationHacker中，已经hook了actiivty启动的相关方法，主要功能是，判断是否是插件apk中的activity，如果是调用pluginManager.makeActivityIntent创建代理activity ProxyActivity的Intent，会把插件中Activity中信息放在Intent中，最终通过反射调用Android原生的Instrumentation 的execStartActivity方法，把目标为Activity的Intent包装成代理Activity-ProxyActivity的Intent
2.2 hook newActivity方法，完全重写->createPluginActivity
```
private Activity createPluginActivity(Intent intent, String proxyClassName) {
		CodeTraceTS.begin("createPluginActivity");
		Activity targetActivity = null;
		PluginManager pluginManager = PluginManager.getInstance();
		String pTag = null;
		String targetClassName = null;
		try {
			targetClassName = 
			intent.getStringExtra(PluginManager.KEY_TARGET_CLASS_NAME);
			pTag = intent.getStringExtra(PluginManager.KEY_PLUGIN_TAG);

		} catch (Exception e) {
			Log.e(TAG, e);
			PluginManager.getInstance().postCrash(e);
		}
		if (pTag == null) {
			Log.e(TAG, "createPluginActivity()::error,pTag=" + pTag);
		}
		Log.i(TAG, "createPluginActivity()::pTag=" + pTag);
		com.qihoo.plugin.bean.Plugin plugin = pluginManager.getPlugin(pTag);
		Log.i(TAG, "createPluginActivity()::plugin=" + plugin);

		if (plugin == null) {
			Log.i(TAG, "createPluginActivity()::Plugin is not loaded in the "
					+ (TextUtils.isEmpty(pluginManager.getName()) ? "main"
							: "[" + pluginManager.getName() + "]")
					+ " process,tag=" + pTag);
			plugin = pluginManager.load(pTag);
			if (plugin == null)
				return null;
		}
		Log.i(TAG, "createPluginActivity()::targetClassName=" + targetClassName);
		ActivityInfo ai = plugin.findActivity(targetClassName);
//		if(ai.launchMode==ActivityInfo.LAUNCH_SINGLE_TASK
//				|| ai.launchMode==ActivityInfo.LAUNCH_SINGLE_INSTANCE){
//			PluginContextInfo pci = getPluginContextInfo(pTag, targetClassName);
//			if (pci != null) {
//				Activity act = (Activity)pci.context;
//				RefUtil.setFieldValue(act, "mBase", null);
//				RefUtil.setFieldValue(act.getFragmentManager(), "mActivity", null);
//				singleActivityCache.put(act, act);
//				act.setIntent(intent);
//				return (Activity)pci.context;
//			}
//		}
		if (targetClassName != null) {
			Class<?> clz = null;
			try {
				clz = plugin.getCl().loadClass(targetClassName);
			} catch (Exception e) {
				Log.e(TAG, e);
				Log.e(TAG, "createPluginActivity()::Didn't find class "
						+ targetClassName);
				PluginManager.getInstance().postCrash(e);
				return null;
			}
			try {
				targetActivity = (Activity) clz.newInstance();
				Log.i(TAG, "createPluginActivity()::targetActivity="
						+ targetActivity);
			} catch (Error e) {
				Log.e(TAG, "createPluginActivity()::clz.newInstance(),error,e="
						+ e);
			} catch (Exception e) {
				Log.e(TAG,
						"createPluginActivity()::clz.newInstance(),Exception,e="
								+ e);
			}
			// 缓存在当前正在运行的activity中
			pluginContexts.put(targetActivity, new PluginContextInfo(
					targetActivity, plugin, proxyClassName, ai, intent));
			TimeStatistics.getOrNewStartTimeInfo(pTag).activity_newActivity = CodeTraceTS.end("createPluginActivity").time();
			return targetActivity;
		}
		return null;
	}
```
通过Intent获取插件中的目标Activity的classname，和KEY_PLUGIN_TAG，然后通过pluginManager根据KEY_PLUGIN_TAG获取相关的Plugin插件信息和插件中的目标Activity的类名，插件信息如下：
```
public class Plugin {

	private String tag;
	private String path;
	private String srcPath;

	private Resources res;
	private DexClassLoaderEx cl;
	private ActivityInfo[] activityInfo;
	private IPlugin callback;
	private Application pluginApplication;
	private PackageInfo packageInfo;
	private PluginPackage pluginPackage;
	private LoadedApk loadedApk;
	private List<WeakReference<Activity>> activities;
```
通过插件的classloader去加载出插件中目标Activity的Class，通过反射newInstance new出插件中的目标Activity，这样就把ProxyActivity替换成为真正的目标Activity
2.3 那Activity的生命周期呢，其实还是在InstrumentationHacker中，hook了Activity的生命周期的回调方法，如callActivityOnCreate，callActivityOnStart等方法，