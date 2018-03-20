---
title: read-fucking-source-code-arouter
date: 2018-03-20 14:28:52
categories: 
- read_fucking_source_code
tags: 
- read_fucking_source_code
- retrofit
---

本文主要分享Lipland一些核心代码和一些流程分析。

<!-- more -->

# Read fucking source code:Arouter


## 1、初始化
```
com.alibaba.android.arouter.launcher.ARouter#init
com.alibaba.android.arouter.launcher._ARouter#init
com.alibaba.android.arouter.core.LogisticsCenter#init

com.alibaba.android.arouter.core.LogisticsCenter#loadRouterMap #这个方法，通过arouter-register实现编译代码之后自动注册路由信息，摒弃了之前的版本扫描dex文件解析class文件进行注册，也避免了造成的性能问题
# [原理](https://github.com/luckybilly/AutoRegister)，[原理2](https://juejin.im/post/5a2b95b96fb9a045284669a9)
```

初始化之后，会把所有注册的路由信息处理成 路由信息映射，
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
进行跳转
com.alibaba.android.arouter.core.LogisticsCenter#completion

 public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }

        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    // Maybe its does't exist, or didn't load.
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  // Load route meta.
            if (null == groupMeta) {
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                // Load route and cache it into memory, then delete from metas.
                try {
                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] starts loading, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }

                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);
                    Warehouse.groupsIndex.remove(postcard.getGroup());

                    if (ARouter.debuggable()) {
                        logger.debug(TAG, String.format(Locale.getDefault(), "The group [%s] has already been loaded, trigger by [%s]", postcard.getGroup(), postcard.getPath()));
                    }
                } catch (Exception e) {
                    throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
                }

                completion(postcard);   // Reload
            }
        } else {
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());

            Uri rawUri = postcard.getUri();
            if (null != rawUri) {   // Try to set params into bundle.
                Map<String, String> resultMap = TextUtils.splitQueryParameters(rawUri);
                Map<String, Integer> paramsType = routeMeta.getParamsType();

                if (MapUtils.isNotEmpty(paramsType)) {
                    // Set value by its type, just for params which annotation by @Param
                    for (Map.Entry<String, Integer> params : paramsType.entrySet()) {
                        setValue(postcard,
                                params.getValue(),
                                params.getKey(),
                                resultMap.get(params.getKey()));
                    }

                    // Save params name which need auto inject.
                    postcard.getExtras().putStringArray(ARouter.AUTO_INJECT, paramsType.keySet().toArray(new String[]{}));
                }

                // Save raw uri
                postcard.withString(ARouter.RAW_URI, rawUri.toString());
            }

            switch (routeMeta.getType()) {
                case PROVIDER:  // if the route is provider, should find its instance
                    // Its provider, so it must implement IProvider
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { // There's no instance of this provider
                        IProvider provider;
                        try {
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                    postcard.setProvider(instance);
                    postcard.greenChannel();    // Provider should skip all of interceptors
                    break;
                case FRAGMENT:
                    postcard.greenChannel();    // Fragment needn't interceptors
                default:
                    break;
            }
        }
    }

```