##EventBus使用以及源码分析

###1.观察者模式

观察者模式定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。这个主题对象在状态上发生变化时，会通知所有观察者对象，使它们能够自动更新自己。因为观察者模式非常常见，所以在jdk中已经帮我们实现了观察者模式，只要继承相对应的类（Observable、Observer）就可以快速地实现观察者模式。

```
public class Observable {
    private boolean changed = false;
    private Vector obs = new Vector();
    public Observable() {
    }
    public synchronized void addObserver(Observer var1) {
        if(var1 == null) {
            throw new NullPointerException();
        } else {
            if(!this.obs.contains(var1)) {
                this.obs.addElement(var1);
            }
        }
    }

    public synchronized void deleteObserver(Observer var1) {
        this.obs.removeElement(var1);
    }
    public void notifyObservers() {
        this.notifyObservers((Object)null);
    }
    public void notifyObservers(Object var1) {
        Object[] var2;
        synchronized(this) {
            if(!this.changed) {
                return;
            }
            var2 = this.obs.toArray();
            this.clearChanged();
        }
        for(int var3 = var2.length - 1; var3 >= 0; --var3) {
            ((Observer)var2[var3]).update(this, var1);
        }
    }
    public synchronized void deleteObservers() {
        this.obs.removeAllElements();
    }
    protected synchronized void setChanged() {
        this.changed = true;
    }
    protected synchronized void clearChanged() {
        this.changed = false;
    }
    public synchronized boolean hasChanged() {
        return this.changed;
    }
    public synchronized int countObservers() {
        return this.obs.size();
    }
}

```

```

public interface Observer {

    /**

     * This method is called whenever the observed object is changed. An

     * application calls an <tt>Observable</tt> object's

     * <code>notifyObservers</code> method to have all the object's

     * observers notified of the change.

     *

     * @param   o     the observable object.

     * @param   arg   an argument passed to the <code>notifyObservers</code>

     *                 method.

     */

    void update(Observable o, Object arg);

}

```

###2.java反射机制
1.定义：在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。
2.EventBus中用的比较多的有
* getDeclaredMethods()//获取类或者接口的所有方法，不包括继承的方法
* method.getName()//获取方法名，用来过滤出onEvent开头的方法
* method.getModifiers()//方法返回int类型值表示该字段的修饰符，EventBus中用于过滤掉非  public、static、abstract方法
PUBLIC: 1
PRIVATE: 2
PROTECTED: 4
STATIC: 8
FINAL: 16
SYNCHRONIZED: 32
VOLATILE: 64
TRANSIENT: 128
NATIVE: 256
INTERFACE: 512
ABSTRACT: 1024
STRICT: 2048
* method.getParameterTypes()//获取方法的参数名，用作key，后面会讲到
* method.invoke(subscription.subscriber, new Object[]{event})//执行对应的方法，在EventBus中就是执行 onEvent（）、onEventMainThread（）、onEvnetBackground（）、onEventAsync（）方法


###3.EventBus和系统广播

Android系统的广播和开源库EventBus其实也是观察者模式的一种实现，可以做到不同组件之间相互通信。EventBus能够实现的功能系统广播都可以实现，EventBus的优点在于代码量少用起来方便，但是不能实现不同进程间的通信，比如监听系统时间变化等，这个只能用广播实现。

###4.EventBus的使用

1.下载EventBus库（https://github.com/greenrobot/EventBus.git） 放在工程目录libs中

2.在Activity中的OnCreate()中注册在OnDestory()方法中注销

3.需要事件分发时EventBus.getDefault().post(object)

4.在EventBus中的观察者通常有四种订阅函数（就是某件事情发生被调用的方法）

 * onEvent:如果使用onEvent作为订阅函数，那么该事件在哪个线程发布出来的，onEvent就会在这个线程中运行，也就是说发布事件和接收事件线程在同一个线程。使用这个方法时，在onEvent方法中不能执行耗时操作，如果执行耗时操作容易导致事件分发延迟

 * onEventMainThread:如果使用onEventMainThread作为订阅函数，那么不论事件是在哪个线程中发布出来的，onEventMainThread都会在UI线程中执行，接收事件就会在UI线程中运行，这个在Android中是非常有用的，因为在Android中只能在UI线程中更新UI，所以在onEvnetMainThread方法中是不能执行耗时操作的。这个用的比较多。

 * onEvnetBackground:如果使用onEventBackgrond作为订阅函数，那么如果事件是在UI线程中发布出来的，那么onEventBackground就会在子线程中运行，如果事件本来就是子线程中发布出来的，那么onEventBackground函数直接在该子线程中执行

 * onEventAsync：使用这个函数作为订阅函数，那么无论事件在哪个线程发布，都会创建新的子线程在执行onEventAsync

###5.源码分析
1.源码中一些重要的方法
* getDefault 单例模式
* register(Object subscriber, boolean sticky, int priority) 所有的注册方法最好都会调用这个方法
 ``` 
 private synchronized void register(Object subscriber, boolean sticky, int priority) {
        //获取subscriber类中声明过的方法
        List subscriberMethods=this.subscriberMethodFinder.findSubscriberMethods(subscriber.getClass());
        Iterator var5 = subscriberMethods.iterator();
        while(var5.hasNext()) {
            SubscriberMethod subscriberMethod = (SubscriberMethod)var5.next();
            this.subscribe(subscriber, subscriberMethod, sticky, priority);
        }
    }
 ```
* List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) 获取类中的订阅方法
```
 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        String key = subscriberClass.getName();//类明作为map的key
        Map clazz = methodCache;
        List subscriberMethods;
        synchronized(methodCache) {
            subscriberMethods = (List)methodCache.get(key);
        }

        if(subscriberMethods != null) {
            return subscriberMethods;
        } else {
            ArrayList subscriberMethods1 = new ArrayList();
            Class clazz1 = subscriberClass;
            HashMap eventTypesFound = new HashMap();

            for(StringBuilder methodKeyBuilder = new StringBuilder(); clazz1 != null; clazz1 = clazz1.getSuperclass()) {
                String name = clazz1.getName();
                //过滤掉系统类
                if(name.startsWith("java.") || name.startsWith("javax.") || name.startsWith("android.")) {
                    break;
                }

                try {
                    Method[] th = clazz1.getDeclaredMethods();//利用反射获取所有方法
                    this.filterSubscriberMethods(subscriberMethods1, eventTypesFound, methodKeyBuilder, th);//过滤出onEvent开头的方法 
                } catch (Throwable var13) {
                    var13.printStackTrace();
                    Method[] methods = subscriberClass.getMethods();
                    subscriberMethods1.clear();
                    eventTypesFound.clear();
                    this.filterSubscriberMethods(subscriberMethods1, eventTypesFound, methodKeyBuilder, methods);
                    break;
                }
            }

            if(subscriberMethods1.isEmpty()) {
                throw new EventBusException("Subscriber " + subscriberClass + " has no public methods called " + "onEvent");
            } else {
                Map name1 = methodCache;
                synchronized(methodCache) {
                    methodCache.put(key, subscriberMethods1);//存入缓存
                    return subscriberMethods1;
                }
            }
        }
    }
```
*     private void filterSubscriberMethods(List<SubscriberMethod> subscriberMethods, HashMap<String, Class> eventTypesFound, StringBuilder methodKeyBuilder, Method[] methods) 过滤出onEvent开头的订阅方法
```
private void filterSubscriberMethods(List<SubscriberMethod> subscriberMethods, HashMap<String, Class> eventTypesFound, StringBuilder methodKeyBuilder, Method[] methods) {
        Method[] var5 = methods;
        int var6 = methods.length;
        for(int var7 = 0; var7 < var6; ++var7) {
            Method method = var5[var7];
            String methodName = method.getName();
            if(methodName.startsWith("onEvent")) {//onEvent开头的函数
                int modifiers = method.getModifiers();
                Class methodClass = method.getDeclaringClass();
                if((modifiers & 1) != 0 && (modifiers & 5192) == 0) {
                    Class[] parameterTypes = method.getParameterTypes();
                    if(parameterTypes.length == 1) {//onEvent方法只能有一个参数，和用法符合
                        ThreadMode threadMode = this.getThreadMode(methodClass, method, methodName);
                        if(threadMode != null) {
                            Class eventType = parameterTypes[0];
                            methodKeyBuilder.setLength(0);
                            methodKeyBuilder.append(methodName);
                            methodKeyBuilder.append('>').append(eventType.getName());//拼接方法名和参数名
                            String methodKey = methodKeyBuilder.toString();
                            Class methodClassOld = (Class)eventTypesFound.put(methodKey, methodClass);
                            if(methodClassOld != null && !methodClassOld.isAssignableFrom(methodClass)) {
                                eventTypesFound.put(methodKey, methodClassOld);
                            } else {
                                subscriberMethods.add(new SubscriberMethod(method, threadMode, eventType));
                            }
                        }
                    }
                } else if(!this.skipMethodVerificationForClasses.containsKey(methodClass)) {
                    Log.d(EventBus.TAG, "Skipping method (not public, static or abstract): " + methodClass + "." + methodName);
                }
            }
        }

    }
```
* private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) 订阅
```
 private void subscribe(Object subscriber, SubscriberMethod subscriberMethod, boolean sticky, int priority) {
        Class eventType = subscriberMethod.eventType;
        //通过订阅事件类型，找到所有的订阅（Subscription）,订阅中包含了订阅者，订阅方法
        CopyOnWriteArrayList subscriptions = (CopyOnWriteArrayList)this.subscriptionsByEventType.get(eventType);
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod, priority);//创建一个新的订阅  
        //将新建的订阅加入到这个事件类型对应的所有订阅列表 
        if(subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList();
            this.subscriptionsByEventType.put(eventType, subscriptions);
        } else if(subscriptions.contains(newSubscription)) {//如果有订阅列表，检查是否已经加入过 
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event " + eventType);
        }

        int size = subscriptions.size();

        for(int subscribedEvents = 0; subscribedEvents <= size; ++subscribedEvents) {
            if(subscribedEvents == size || newSubscription.priority > ((Subscription)subscriptions.get(subscribedEvents)).priority) {
                subscriptions.add(subscribedEvents, newSubscription);
                break;
            }
        }

        Object var15 = (List)this.typesBySubscriber.get(subscriber);
        if(var15 == null) {
            var15 = new ArrayList();
            this.typesBySubscriber.put(subscriber, var15);
        }

        ((List)var15).add(eventType);
        if(sticky) {
            if(this.eventInheritance) {
                Set stickyEvent = this.stickyEvents.entrySet();
                Iterator var11 = stickyEvent.iterator();

                while(var11.hasNext()) {
                    Entry entry = (Entry)var11.next();
                    Class candidateEventType = (Class)entry.getKey();
                    if(eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent1 = entry.getValue();
                        this.checkPostStickyEventToSubscription(newSubscription, stickyEvent1);
                    }
                }
            } else {
                Object var16 = this.stickyEvents.get(eventType);
                this.checkPostStickyEventToSubscription(newSubscription, var16);
            }
        }

    }
```
* private boolean postSingleEventForEventType(Object event, EventBus.PostingThreadState postingState, Class<?> eventClass) 分发事件,所有的post方法最后都会调用这个方法
```
 private boolean postSingleEventForEventType(Object event, EventBus.PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList subscriptions;
        synchronized(this) {
            subscriptions = (CopyOnWriteArrayList)this.subscriptionsByEventType.get(eventClass);//获取对应的订阅方法
        }
        if(subscriptions != null && !subscriptions.isEmpty()) {
            Iterator var5 = subscriptions.iterator();
            while(var5.hasNext()) {
                Subscription subscription = (Subscription)var5.next();
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    this.postToSubscription(subscription, event, postingState.isMainThread);//对每个订阅调用该方法  
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if(aborted) {
                    break;
                }
            }
            return true;
        } else {
            return false;
        }
    }
```
* private void postToSubscription(Subscription subscription, Object event, boolean isMainThread)
```
 private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        //第一个参数就是传入的订阅，第二个参数就是对于的分发事件，第三个参数非常关键：是否在主线程
        switch (subscription.subscriberMethod.threadMode) {
        //这个threadMode是根据onEvent,onEventMainThread,onEventBackground,onEventAsync决定的
        case PostThread:
            //利用反射直接调用
            invokeSubscriber(subscription, event);
            break;
        case MainThread:
            if (isMainThread) {
                //如果直接在主线程，那么直接在本现场中调用订阅函数
                invokeSubscriber(subscription, event);
            } else {
                //如果不在主线程，那么通过handler实现在主线程中执行
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case BackgroundThread:
            if (isMainThread) {
                //如果主线程，创建一个runnable丢入线程池中
                backgroundPoster.enqueue(subscription, event);
            } else {
                //如果子线程，利用反射直接调用
                invokeSubscriber(subscription, event);
            }
            break;
        case Async:
            //不论什么线程，直接丢入线程池
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }

    }
```
* public synchronized void unregister(Object subscriber)解绑订阅
```
   public synchronized void unregister(Object subscriber) {
        List subscribedTypes = (List)this.typesBySubscriber.get(subscriber);//
        if(subscribedTypes != null) {
            Iterator var3 = subscribedTypes.iterator();
            while(var3.hasNext()) {
                Class eventType = (Class)var3.next();
                this.unsubscribeByEventType(subscriber, eventType);
            }
            this.typesBySubscriber.remove(subscriber);//remove解除绑定
        } else {
            Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```
