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

###2.EventBus和系统广播

Android系统的广播和开源库EventBus其实也是观察者模式的一种实现，可以做到不同组件之间相互通信。EventBus能够实现的功能系统广播都可以实现，EventBus的优点在于代码量少用起来方便，但是不能实现不同进程间的通信，比如监听系统时间变化等，这个只能用广播实现。

###3.EventBus的使用

1.下载EventBus库（https://github.com/greenrobot/EventBus.git） 放在工程目录libs中

2.在Activity中的OnCreate()中注册在OnDestory()方法中注销

3.需要事件分发时EventBus.getDefault().post(object)

4.在EventBus中的观察者通常有四种订阅函数（就是某件事情发生被调用的方法）

 * onEvent:如果使用onEvent作为订阅函数，那么该事件在哪个线程发布出来的，onEvent就会在这个线程中运行，也就是说发布事件和接收事件线程在同一个线程。使用这个方法时，在onEvent方法中不能执行耗时操作，如果执行耗时操作容易导致事件分发延迟

 * onEventMainThread:如果使用onEventMainThread作为订阅函数，那么不论事件是在哪个线程中发布出来的，onEventMainThread都会在UI线程中执行，接收事件就会在UI线程中运行，这个在Android中是非常有用的，因为在Android中只能在UI线程中更新UI，所以在onEvnetMainThread方法中是不能执行耗时操作的。这个用的比较多。

 * onEvnetBackground:如果使用onEventBackgrond作为订阅函数，那么如果事件是在UI线程中发布出来的，那么onEventBackground就会在子线程中运行，如果事件本来就是子线程中发布出来的，那么onEventBackground函数直接在该子线程中执行

 * onEventAsync：使用这个函数作为订阅函数，那么无论事件在哪个线程发布，都会创建新的子线程在执行onEventAsync