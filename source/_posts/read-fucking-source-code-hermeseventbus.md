---
title: read_fucking_source_code_hermeseventbus
date: 2018-03-05 19:06:07
categories: 
- read_fucking_source_code
tags: 
- read_fucking_source_code
- hermeseventbus
---

本文主要分享Hermeseventbus和Eventbus一些核心代码和一些流程分析。

<!-- more -->
# Read fucking source code:hermeseventbus
Hermeseventbus和Eventbus区别在于Hermeseventbus利用Hermes这个Android IPC库来在多进程直接传递发送出来的event。
接下来就先分析一下Eventbus的原理和流程，然后再来分析Hermeseventbus的原理和流程。

## Eventbus分析
原理就是通过反射拿到注册的订阅者的class中有Subscribe注解的方法和相关信息，并缓存起来以便下次订阅的时候不用重新再反射一遍，然后把订阅者放进订阅集合中，有事件发送的时候直接调用mehtod.invoke方法调用订阅方法
###1、注册
```
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
注册过程通过subscriberMethodFinder去获取subscriber中用注解定义的方法和一些其他信息，如下
```
final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;
```
然后通过subscribe(subscriber, subscriberMethod);方法把订阅者的信息放进要观察的集合中
1.1 先从METHOD_CACHE中通过subscriberClass找是否有换成，然后通过findUsingReflection->findUsingReflectionInSingleClass
```
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```
1.2findUsingReflectionInSingleClass通过反射分析Subscribe注解，来确定SubscriberMethod信息。
```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```
###2、发送event
post->postSingleEvent->postSingleEventForEventType->postToSubscription->invokeSubscriber or org.greenrobot.eventbus.HandlerPoster#enqueue->invokeSubscriber
```
    /** Posts the given event to the event bus. */
    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```
```
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```
postToSubscription方法通过该时间类型所对应注册的方法的线程模型，来决定是直接执行invokeSubscriber还是把事件放进对应的PendingPostQueue中去，PendingPostQueue是一个双向链表
```
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
invokeSubscriber就直接拿之前已经解析出来的method调用method.invoke方法
3、unregister就简单了，只是从订阅者集合中删除了这个订阅者

## Hermeseventbus分析
其实Hermeseventbus的核心原理是Hermes这个库，其他方面就是对多进程IPC传递数据的封装。还是一点一点深入。
原理HermesEventBus的[github](https://github.com/elemers/HermesEventBus)上已经分享过了。如下
```
事件收发是基于EventBus，IPC通信是基于Hermes。Hermes是一个简单易用的Android IPC库。

本库首先选一个进程作为主进程，将其他进程作为子进程。

每次一个event被发送都会经过以下四步：

1、使用Hermes库将event传递给主进程。

2、主进程使用EventBus在主进程内部发送event。

3、主进程使用Hermes库将event传递给所有的子进程。

4、每个子进程使用EventBus在子进程内部发送event。
```


