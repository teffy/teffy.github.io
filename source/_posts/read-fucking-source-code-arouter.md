---
title: read-fucking-source-code-arouter
date: 2018-03-20 14:28:52
categories: 
- read_fucking_source_code
tags: 
- read_fucking_source_code
- Arouter
---

本文主要分享Arouter一些核心代码和一些流程分析。

<!-- more -->

Read fucking source code:Arouter


## 1、初始化
```
com.alibaba.android.arouter.launcher.ARouter#init
com.alibaba.android.arouter.launcher._ARouter#init
com.alibaba.android.arouter.core.LogisticsCenter#init

com.alibaba.android.arouter.core.LogisticsCenter#loadRouterMap 
```
loadRouterMap这个方法，通过arouter-register实现编译代码之后自动注册路由信息，摒弃了之前的版本扫描dex文件解析class文件进行注册，也避免了造成的性能问题，[原理](https://github.com/luckybilly/AutoRegister)，[原理2](https://juejin.im/post/5a2b95b96fb9a045284669a9)
初始化之后，会把所有注册的路由信息处理成 路由信息映射
```
com.alibaba.android.arouter.core.Warehouse
class Warehouse {
    // Cache route and metas
    static Map<String, Class<? extends IRouteGroup>> groupsIndex = new HashMap<>();
    static Map<String, RouteMeta> routes = new HashMap<>();

    // Cache provider
    static Map<Class, IProvider> providers = new HashMap<>();
    static Map<String, RouteMeta> providersIndex = new HashMap<>();

    // Cache interceptor
    static Map<Integer, Class<? extends IInterceptor>> interceptorsIndex = new UniqueKeyTreeMap<>("More than one interceptors use same priority [%s]");
    static List<IInterceptor> interceptors = new ArrayList<>();

    static void clear() {
        routes.clear();
        groupsIndex.clear();
        providers.clear();
        providersIndex.clear();
        interceptors.clear();
        interceptorsIndex.clear();
    }
}
```
RouteMeta 就是路由信息
```
public class RouteMeta {
    private RouteType type;         // Type of route
    private Element rawType;        // Raw type of route
    private Class<?> destination;   // Destination
    private String path;            // Path of route
    private String group;           // Group of route
    private int priority = -1;      // The smaller the number, the higher the priority
    private int extra;              // Extra data
    private Map<String, Integer> paramsType;  // Param type

    public enum RouteType {
    ACTIVITY(0, "android.app.Activity"),
    SERVICE(1, "android.app.Service"),
    PROVIDER(2, "com.alibaba.android.arouter.facade.template.IProvider"),
    CONTENT_PROVIDER(-1, "android.app.ContentProvider"),
    BOARDCAST(-1, ""),
    METHOD(-1, ""),
    FRAGMENT(-1, "android.app.Fragment"),
    UNKNOWN(-1, "Unknown route type");
```
## 2、inject
把Uri上带的数据参数，给跳转目标中的class中使用com.alibaba.android.arouter.facade.annotation.Autowired 标记的成员变量赋值
例如：
```
public class Test1Activity extends AppCompatActivity {

    @Autowired
    String name = "jack";

    @Autowired
    int age = 10;

    @Autowired
    int height = 175;

    @Autowired(name = "boy")
    boolean girl;
```
下面是通过gradle自动生成的代码
```
public class Test1Activity$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    Test1Activity substitute = (Test1Activity)target;
    substitute.name = substitute.getIntent().getStringExtra("name");
    substitute.age = substitute.getIntent().getIntExtra("age", substitute.age);
    substitute.height = substitute.getIntent().getIntExtra("height", substitute.height);
    substitute.girl = substitute.getIntent().getBooleanExtra("boy", substitute.girl);
    substitute.ch = substitute.getIntent().getCharExtra("ch", substitute.ch);
    substitute.fl = substitute.getIntent().getFloatExtra("fl", substitute.fl);
    substitute.dou = substitute.getIntent().getDoubleExtra("dou", substitute.dou);
    substitute.pac = substitute.getIntent().getParcelableExtra("pac");
    if (null != serializationService) {
      substitute.obj = serializationService.parseObject(substitute.getIntent().getStringExtra("obj"), new com.alibaba.android.arouter.facade.model.TypeWrapper<TestObj>(){}.getType());
    } else {
      Log.e("ARouter::", "You want automatic inject the field 'obj' in class 'Test1Activity' , then you should implement 'SerializationService' to support object auto inject!");
    }
    if (null != serializationService) {
      substitute.objList = serializationService.parseObject(substitute.getIntent().getStringExtra("objList"), new com.alibaba.android.arouter.facade.model.TypeWrapper<List<TestObj>>(){}.getType());
    } else {
      Log.e("ARouter::", "You want automatic inject the field 'objList' in class 'Test1Activity' , then you should implement 'SerializationService' to support object auto inject!");
    }
    if (null != serializationService) {
      substitute.map = serializationService.parseObject(substitute.getIntent().getStringExtra("map"), new com.alibaba.android.arouter.facade.model.TypeWrapper<Map<String, List<TestObj>>>(){}.getType());
    } else {
      Log.e("ARouter::", "You want automatic inject the field 'map' in class 'Test1Activity' , then you should implement 'SerializationService' to support object auto inject!");
    }
    substitute.url = substitute.getIntent().getStringExtra("url");
    substitute.helloService = ARouter.getInstance().navigation(HelloService.class);
  }
}
```

## 3、路由跳转
通过下面的代码进行跳转
```
 ARouter.getInstance()
                        .build("/kotlin/test")
                        .withString("name", "老王")
                        .withInt("age", 23)
                        .navigation();
```
build 和with方法组成Postcard（明信片）
```
com.alibaba.android.arouter.launcher.ARouter#navigation(java.lang.Class<? extends T>)
com.alibaba.android.arouter.launcher._ARouter#navigation(java.lang.Class<? extends T>)
    protected <T> T navigation(Class<? extends T> service) {
        try {
            Postcard postcard = LogisticsCenter.buildProvider(service.getName());

            // Compatible 1.0.5 compiler sdk.
            if (null == postcard) { // No service, or this service in old version.
                postcard = LogisticsCenter.buildProvider(service.getSimpleName());
            }

            LogisticsCenter.completion(postcard);
            return (T) postcard.getProvider();
        } catch (NoRouteFoundException ex) {
            logger.warning(Consts.TAG, ex.getMessage());
            return null;
        }
    }
```
进行跳转
```
com.alibaba.android.arouter.facade.Postcard#navigation()
com.alibaba.android.arouter.launcher.ARouter#navigation(Context mContext, Postcard postcard, int requestCode, NavigationCallback callback)
com.alibaba.android.arouter.launcher._ARouter# navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) 在这个方法中会处理拦截器的逻辑
```
最终到
```
com.alibaba.android.arouter.launcher._ARouter#_navigation
  private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Navigation in main looper.
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        if (requestCode > 0) {  // Need start for result
                            ActivityCompat.startActivityForResult((Activity) currentContext, intent, requestCode, postcard.getOptionsBundle());
                        } else {
                            ActivityCompat.startActivity(currentContext, intent, postcard.getOptionsBundle());
                        }

                        if ((-1 != postcard.getEnterAnim() && -1 != postcard.getExitAnim()) && currentContext instanceof Activity) {    // Old version.
                            ((Activity) currentContext).overridePendingTransition(postcard.getEnterAnim(), postcard.getExitAnim());
                        }

                        if (null != callback) { // Navigation over.
                            callback.onArrival(postcard);
                        }
                    }
                });

                break;
            case PROVIDER:
                return postcard.getProvider();
            case BOARDCAST:
            case CONTENT_PROVIDER:
            case FRAGMENT:
                Class fragmentMeta = postcard.getDestination();
                try {
                    Object instance = fragmentMeta.getConstructor().newInstance();
                    if (instance instanceof Fragment) {
                        ((Fragment) instance).setArguments(postcard.getExtras());
                    } else if (instance instanceof android.support.v4.app.Fragment) {
                        ((android.support.v4.app.Fragment) instance).setArguments(postcard.getExtras());
                    }

                    return instance;
                } catch (Exception ex) {
                    logger.error(Consts.TAG, "Fetch fragment instance error, " + TextUtils.formatStackTrace(ex.getStackTrace()));
                }
            case METHOD:
            case SERVICE:
            default:
                return null;
        }
        return null;
```