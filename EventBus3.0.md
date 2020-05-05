#### EventBus

##### 介绍

EventBus是Android和Java的一个发布和订阅事件总线第三方库。

[官网](http://greenrobot.org/eventbus/)

Features（特点）：

1. 简化组件间通信
   1. 分离事件发送者和接收者
   2. 能够很好的在Activity，Fragment和后台线程之间协作
   3. 避免复杂和容易出错的依赖关系和生命周期问题
2. 代码简洁
3. 反应迅速
4. 体积小（60k jar）
5. 被大量的app实践使用过
6. 具有发送线程，订阅优先级等特性

##### 使用示例

1. 创建要传递的事件实体

   ```java
   public static class MessageEvent { 
       /* Additional fields if needed */
   }
   ```

   

2. 订阅者：声明和注释订阅方法，指定一个线程模式（MAIN, POSTING, MAIN_ORDERED, BACKGROUND, ASYNC）

   ```java
   @Subscribe(threadMode = ThreadMode.MAIN)  
   public void onMessageEvent(MessageEvent event) {
       /* Do something */
   };
   ```

   订阅者进行注册及取消注册。例如在Android上，Activity和Fragment通常应该根据它们的生命周期进行注册

   ```java
   @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
   
    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
   ```

   

3. 发布事件

   ```java
    EventBus.getDefault().post(new MessageEvent());
   ```

##### 前提

EventBus是基于观察者模式扩展而来的，不了解观察者模式的可以先提前了解，再来阅读这个解析文章。

##### EventBus.getDefault().register(this)

首先从注册订阅者开始分析

```java
    /** Convenience singleton for apps using a process-wide EventBus instance. */
    public static EventBus getDefault() {
        EventBus instance = defaultInstance;
        if (instance == null) {
            synchronized (EventBus.class) {
                instance = EventBus.defaultInstance;
                if (instance == null) {
                    instance = EventBus.defaultInstance = new EventBus();// 分析点1
                }
            }
        }
        return instance;
    }
```

在getDefault()中使用双重校验加锁的单例模式创建EventBus实例

接着看下分析点1处的EventBus的构造方法实现

```java
    public EventBus() {
        this(DEFAULT_BUILDER);// 分析点2
    }

    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();// 分析点3
        typesBySubscriber = new HashMap<>();// 分析点4
        stickyEvents = new ConcurrentHashMap<>();// 分析点5
        mainThreadSupport = builder.getMainThreadSupport();// 分析点6
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);// 分析点7
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;// 分析点8
    }

```

在构造方法的**分析点2**中可以看出,它调用了另一个有参构造方法,其中的参数是一个类型为 EventBuilder 的 DEFAULT_BUILDER 对象,这里明显构建者模式, DEFAULT_BUILDER 对象中包括了许多 EventBus 的默认设置.猜测可以通过 EventBuilder 类型来构建自定义的EventBus对象.继续分析 DEFAULT_BUILDER 可知

```java
public class EventBusBuilder {
    ...
    EventBusBuilder() {
        // DEFAULT_BUILDER没有进行什么特殊的处理,仅仅是实例自身
    }
    ...
}
```

回到EventBus的有参构造方法,**分析点3** 定义为`private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;`是一个key为Class,value为Subscription的线性表.而Subscription是什么呢？

```java

/**
 Subscription是一个订阅信息对象,有两个字段: Object(用于保存订阅者,即EventBus.getDefault().register(this)中的this对象), SubscriberMethod (用于保存订阅方法,即被@Subcribe注解的方法)
**/
final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

    Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }

    @Override
    public boolean equals(Object other) {
        if (other instanceof Subscription) {
            Subscription otherSubscription = (Subscription) other;
            return subscriber == otherSubscription.subscriber
                    && subscriberMethod.equals(otherSubscription.subscriberMethod);
        } else {
            return false;
        }
    }

    @Override
    public int hashCode() {
        return subscriber.hashCode() + subscriberMethod.methodString.hashCode();
    }
}


/**
SubscriberMethod 可以看到它的字段包括 Method 方法本身, ThreadMode 订阅方法的执行线程, eventType 订阅方法的参数的类型, priority 优先级, sticky 是否是黏性事件。
**/
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }

    @Override
    public boolean equals(Object other) {
        if (other == this) {
            return true;
        } else if (other instanceof SubscriberMethod) {
            checkMethodString();
            SubscriberMethod otherSubscriberMethod = (SubscriberMethod)other;
            otherSubscriberMethod.checkMethodString();
            // Don't use method.equals because of http://code.google.com/p/android/issues/detail?id=7811#c6
            return methodString.equals(otherSubscriberMethod.methodString);
        } else {
            return false;
        }
    }

    private synchronized void checkMethodString() {
        if (methodString == null) {
            // Method.toString has more overhead, just take relevant parts of the method
            StringBuilder builder = new StringBuilder(64);
            builder.append(method.getDeclaringClass().getName());
            builder.append('#').append(method.getName());
            builder.append('(').append(eventType.getName());
            methodString = builder.toString();
        }
    }

    @Override
    public int hashCode() {
        return method.hashCode();
    }
}
```

接着看**分析点4**定义为`private final Map<Object, List<Class<?>>> typesBySubscriber;`是一个key为 Object , value为Class的集合.

**分析点5**定义为`private final Map<Class<?>, Object> stickyEvents`是一个用于黏性事件处理的字段。

**分析点6**新建了三个不同类型的事件发送器：

1. mainThreadPoster:实际是 HandlerPoster 类型的实例对象, HandlerPoster 继承Handler, Looper是 Looper.getMainLooper ,这里通过Handler机制在主线程中回调**订阅方法**
2. backgroundPoster:实际是实现了 Runnable 接口的对象,将方法加入后台队列中,最后通过线程池去执行,由于添加了synchronized关键字并使用了 executorRunning 这个boolean类型的字段进行控制,线程池保证了同一时间只有一个任务会被线程池执行
3. asyncPoster:实际是实现了 Runnable 接口的对象,与backgroundPoster类似,区别在于没有字段和synchronized关键字的控制,同一时间可以有多个任务被线程池执行

```java
/**
mainThreadPoster
**/
public class HandlerPoster extends Handler implements Poster {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
        super(looper);// 此处的Looper为Looper.getMainLooper()
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                handlerActive = true;
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // Check again, this time in synchronized
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}


/**
backgroundPoster
**/
final class BackgroundPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    private volatile boolean executorRunning;

    BackgroundPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            queue.enqueue(pendingPost);
            if (!executorRunning) {
                executorRunning = true;
                eventBus.getExecutorService().execute(this);
            }
        }
    }

    @Override
    public void run() {
        try {
            try {
                while (true) {
                    PendingPost pendingPost = queue.poll(1000);
                    if (pendingPost == null) {
                        synchronized (this) {
                            // Check again, this time in synchronized
                            pendingPost = queue.poll();
                            if (pendingPost == null) {
                                executorRunning = false;
                                return;
                            }
                        }
                    }
                    eventBus.invokeSubscriber(pendingPost);
                }
            } catch (InterruptedException e) {
                eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
            }
        } finally {
            executorRunning = false;
        }
    }

}


/**
asyncPoster
**/
class AsyncPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }

}
```

接着**分析点7**新建了 SubscriberMethodFinder 对象,根据命名可以知道这是一个查找订阅对象中的订阅方法的类,这个文章的后面会分析

而**分析点8**是一个线程池，它是在 DEFAULT_BUILDER 实例自身的时候，通过Executor的newCachedThreadPool()方法创建，它是一个有则用、无则创建、无数量上限的线程池。也就是上面backgroundPoster,asyncPoster会用到的线程池.

上面是EventBus的getDefault()方法接下来终于可以分析**register(this)**的逻辑

```java
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);// 分析点9
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);// 分析点10
            }
        }
    }
```

需要关注的地方有两个,**分析点9**处根据代码上下文和方法名可以知道是找出**订阅类中的所有订阅方法并得到一个订阅方法的List**.而**分析点10**显然是遍历这个列表再进行操作.这里先追踪分析点9

```java
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();// 一个以class为key,List<SubscriberMethod>为value的hashmap
    ...
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // 先从hashmap取,没有则证明该Class没有订阅过
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
	    // ignoreGeneratedIndex这个字段是用来判断是否使用APT生成代码优化寻找订阅方法的过程,默认为false
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            // 所以一般调用的是 findUsingInfo 这个方法
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

	
	
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        /* prepareFindState()是获取一个FindState实例,不继续跟踪改变该方法。而FindState是 SubscriberMethodFinder 的内部类，这个方法主要做一个初始化、回收对象等工作。*/
        FindState findState = prepareFindState();
        // initForSubscriber(Class clazz)将订阅对象的Class用于初始化FindState
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 首次初始化时,subscriberInfo 为null,所以此处获得的是null值,此处太简单也省略了追踪
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                // 最终调用该方法获取订阅方法集合
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        // 分析点9.1
        return getMethodsAndRelease(findState);
    }

	// SubscriberMethodFinder的内部类
    static class FindState {
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        final StringBuilder methodKeyBuilder = new StringBuilder(128);

        Class<?> subscriberClass;
        Class<?> clazz;
        boolean skipSuperClasses;
        SubscriberInfo subscriberInfo;

        void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            // 这里可以知道初始化时subscriberInfo为null值
            subscriberInfo = null;
        }
        
        ...
    }

	
	// 重点方法
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            try {
                methods = findState.clazz.getMethods();
            } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
                String msg = "Could not inspect methods of " + findState.clazz.getName();
                if (ignoreGeneratedIndex) {
                    msg += ". Please consider using EventBus annotation processor to avoid reflection.";
                } else {
                    msg += ". Please make this class visible to EventBus annotation processor to avoid reflection.";
                }
                throw new EventBusException(msg, error);
            }
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    // 重点1
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        // 重点2
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            // 重点3
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

看上面的代码注释可知重点在于`findUsingReflectionInSingleClass`这个方法，它做了什么呢

1. 通过反射获取了订阅对象的类的所有声明方法（有点绕口）
2. 遍历所有声明方法获得以@Subscribe作为注解的方法
3. 检查上面获得的方法的入参是不是只有一个
4. `checkadd`根据方法和方法的入参检查是否已经记录过同类型入参的订阅方法
5. 如果满足上面的条件则新建一个**SubscriberMethod 对象**加入到**findState.subscriberMethods**这个集合中

这时候分析回到**findUsingInfo**方法中,它会通过 findState.moveToSuperclass(); 继续去检查父类的class,当然这个循环会终结在系统的class,也就是不检查系统的class.最后我们要看的就是分析点9.1**getMethodsAndRelease(findState)**方法

```java
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

很简单,构建了一个新的`List<SubscriberMethod>`进行返回,回收findState对象.*关于FindState对象的缓存池处理不是重点.所以跳过分析.*

终于回到了**分析点9**处获得了一个关于订阅对象的订阅方法集合`List<SubscriberMethod>`,接着就是**分析点10**的遍历集合中的subscribe(subscriber, subscriberMethod);方法.

```java
	private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        // 构建一个Subscription对象,该对象持有订阅对象以及订阅集合的引用
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        /* 从subscriptionsByEventType 中获取一个eventType(即订阅方法的入参类型)为key的CopyOnWriteArrayList<Subscription>集合,没有则创建,并放入subscriptionsByEventType中*/
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
	    // 判断订阅方法优先级并进行排序
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        // 对typesBySubscriber集合进行添加
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
	    // 判断该订阅方法是否是对sticky事件进行的订阅,最后会分析,粘性事件会在订阅之后立刻被分发
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

至此`EventBus.getDefault().register(this)`分析结束,流程经历了**EventBus的单例创建(包括了一些集合的初始化)**,**通过反射分析订阅对象的class对象获取订阅方法集合**,再把**订阅方法集合的字段信息填充到EventBus的集合中**.

##### EventBus.getDefault().post(new MessageEvent())

```java
    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };


	public void post(Object event) {
        // 分析点1
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);// 分析点2
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }


    /** For ThreadLocal, much faster to set (and get multiple values). */
    final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
```

**分析点1**中 currentPostingThreadState 是一个 ThreadLocal 对象,存储了线程相关的变量 PostingThreadState 实例,而PostingThreadState 相对于一个实体类,存储一些字段信息,其中最重要的是一个eventQueue线性表.而传入的event正是加入到了这个eventQueue中.

经过状态和线性表长度判断后调用了postSingleEvent也就是**分析点2**

```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        /* 取出event的class类型后,对eventInheritance 标志位判断，它默认为true，如果设为 true 的话，它会在发射事件的时候判断是否需要发射父类事件*/
        if (eventInheritance) {
            // 取出 Event 及其父类和接口的 class 列表
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                // 分析点3
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }


	// 分析点3
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            // 根据event的class类型获取订阅了该event的订阅对象和订阅方法的封装类 Subscription 的线性表集合
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted;
                try {
                    // 分析点4 遍历subscriptions调用postToSubscription方法
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }

	
	// 分析点4
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
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
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

重要的方法已经注释了,接下来看下**分析点3** postSingleEventForEventType 方法,它做的就是根据event的类型获取**订阅了该类型的订阅者集合**然后**遍历调用postToSubscription方法(也就是分析点4)**

**分析点4**根据@Subscribe注释的threadmode值,判断如何调用订阅方法

1. POSTING：执行 invokeSubscriber() 方法，内部**直接采用反射调用**.
2. MAIN：判断当前是否是UI 线程，**是则直接反射调用，否则调用mainThreadPoster的enqueue()方法(即把当前的方法加入到队列之中，然后通过 handler 去发送一个消息，在 handler 的 handleMessage 中去执行方法,分析过了)**。
3. MAIN_ORDERED：**确保是顺序执行的**,与Main类似。
4. BACKGROUND：判断当前是否在 UI 线程，**不是则直接反射调用，否则通过backgroundPoster的enqueue()方法将方法加入到后台的一个队列，最后通过线程池去执行(已分析,可以查看上文回顾).**
5. ASYNC：逻辑实现类似于BACKGROUND，**将任务加入到后台的一个队列，最终由线程池去调用，即Executors的newCachedThreadPool()方法创建的线程池（已分析,可以查看上文回顾）。**

##### EventBus.getDefault().unregister(this)

```java
    public synchronized void unregister(Object subscriber) {
    	// 取出该订阅者(object)订阅的事件的class集合
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                // 分析点1 遍历集合调用unsubscribeByEventType(subscriber, eventType)
                unsubscribeByEventType(subscriber, eventType);
            }
            // 删除订阅者和订阅类型的集合元素
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }


    /** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        // 取出订阅了事件类型的所有订阅对象和订阅方法的封装类 Subscription 的集合
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                // 遍历集合 当该封装类是需要取消注册的订阅者时,从Subscription集合中剔除
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```

unregister的流程相对简单只是**对EventBus中的一些集合中需要取消注册的元素的剔除.**

##### EventBus.getDefault.postSticky(new MessageEvent())

最后来看看粘性事件

```java
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            // 加入集合中
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        // 分析点1
        post(event);
    }
```

**分析点1**的post(event)走的是正常的post流程,也就是上面已经分析过的流程.那么如何**确保粘性事件会被事件发送后才订阅的订阅者接收**,这其实是在订阅者register流程中实现的.还记得register流程最后调用的 subscribe 方法中的分析吗?

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    ...
        
    // 判断该订阅方法是否是对sticky事件进行的订阅,最后会分析,粘性事件会在订阅之后立刻被分发
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            // 分析点2
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    // 分析点3
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```
**分析点2**遍历 stickyEvents 集合,如果订阅者的订阅方法的入参类型和 stickyEvents 集合中某个元素相同,则调用 checkPostStickyEventToSubscription 方法.

```java
    private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            // post流程中最后的调用方法
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```

`checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent)` 也就是**分析点3**最后也是调用post流程中的最后的调用方法(已分析).

##### 总结

自此EventBus3.0的简单使用的流程也就走通了,核心还是**通过反射获取订阅者注解的订阅方法**,**根据注解的参数以及订阅方法的参数实现事件分发后订阅者的订阅操作**.当然还有一些优先级和粘性事件的设计,总的来说这是一个设计得很棒的第三方库.第三方库源码的详细分析终究只是走一遍设计的流程.**了解设计的思想才是重点,细节则不必太过纠结.**