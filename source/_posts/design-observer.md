---
title: 设计模式-观察者模式
date: 2020-11-09 16:52:05
tags: 设计模式
---

## 观察者模式
> 观察者模式定义对象之间的订阅关系，当被观察者行为状态发生变化，观察者则会被通知到并根据预设的逻辑产生相应的行为变化。也被称为发布订阅模式

在日常生活中经常会使用到这种模式,订阅杂志报纸，比如关注了某个博主，发博客时会想你推送信息。关注微信公众号，发送新文章会发推送。

观察者模式角色
- Subject（目标）：目标又称为主题，指被观察的对象，提供了一个用于保存观察者对象的聚集类和增加、删除观察者对象的方法，以及通知所有观察者的抽象方法
- ConcreteSubject（具体目标）：它实现抽象目标中的通知方法，当具体主题的内部状态发生改变时，通知所有订阅过的观察者对象
- Observer（抽象观察者）：它是一个抽象类或接口，它包含了一个更新自己的抽象方法，当接到具体主题的更改通知时被调用。
- ConcreteObserver（具体观察者）：实现抽象观察者中定义的抽象方法，以便在得到目标的更改通知时更新自身的状态。

<!--more-->
### JDK实现
JDK提供了一套观察者模式的类接口，Observer，Observable
> JDK9 中该系列接口已被废弃

- java.util.Observable:目标抽象类,包含一个观察者集合及增删观察者和改变状态通知观察者的方法

```java
public class Observable {
    private boolean changed = false; // 变化标识
    private Vector<Observer> obs; // 观察者集合

    /** Construct an Observable with zero Observers. */

    public Observable() {
        obs = new Vector<>();
    }

    /**
     * 添加观察者对象
     */
    public synchronized void addObserver(Observer o) {
        if (o == null)
            throw new NullPointerException();
        if (!obs.contains(o)) {
            obs.addElement(o);
        }
    }

    /**
     * 删除观察者对象
     */
    public synchronized void deleteObserver(Observer o) {
        obs.removeElement(o);
    }

    /**
     * 当被观察者发生变化，通知所有的观察者
     */
    public void notifyObservers() {
        notifyObservers(null);
    }

    /**
     * 当被观察者发生变化，通知所有观察者
     * @param   arg   any object.更新的对象信息
     */
    public void notifyObservers(Object arg) {
        /*
         * 临时缓存数组，相当于当前观察者状态的快照
         * a temporary array buffer, used as a snapshot of the state of
         * current Observers.
         */
        Object[] arrLocal;

        synchronized (this) {
            /* 
             * 同步代码块
             * 1）避免新添加观察者错过通知
             * 2）避免最近取消订阅的观察者错误通知
             */
            if (!changed)
                return;
            arrLocal = obs.toArray();
            clearChanged();
        }
        //遍历所有观察者并调用其update方法
        for (int i = arrLocal.length-1; i>=0; i--)
            ((Observer)arrLocal[i]).update(this, arg);
    }

    /**
     * 清理所以观察者
     */
    public synchronized void deleteObservers() {
        obs.removeAllElements();
    }

    /**
     * 改变被观察者状态
     */
    protected synchronized void setChanged() {
        changed = true;
    }

    /**
     * 恢复被观察者状态
     */
    protected synchronized void clearChanged() {
        changed = false;
    }

    /**
     * 获取当前状态
     */
    public synchronized boolean hasChanged() {
        return changed;
    }

    /**
     * 获取当前观察者个数.
     */
    public synchronized int countObservers() {
        return obs.size();
    }
}
```

- java.util.Observer:抽象观察者接口，具体观察者实现该接口，提供了更新观察者对象的方法

```java
public interface Observer {
    /**
     * 当订阅的被观察者发出通知执行该方法
     */
    void update(Observable o, Object arg);
}
```

场景：**智能家居**  
当主人下班回家时打开门（通知）开启空调电视，开灯。  
定义被观察者对象 主人Master
```java
public class Master extends Observable {

    public void openDoor(){
        System.out.println("回到家打开门");
        this.setChanged();
        this.notifyObservers();
    }
}
```

定义观察者灯电视空调等
```java
//电视
public class Television implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        System.out.println("打开电视");
    }
}

//灯
public class Light implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        System.out.println("打开灯");
    }
}

//空调
public class AirConditioner implements Observer {
    @Override
    public void update(Observable o, Object arg) {
        System.out.println("打开空调");
    }
}
```

场景测试
```java
public class Test {
    public static void main(String[] args) {
        //创建观察者
        Light light = new Light();
        Television television = new Television();
        AirConditioner airConditioner = new AirConditioner();
        //创建被观察者
        Master master = new Master();
        //添加观察者
        master.addObserver(television);
        master.addObserver(airConditioner);
        master.addObserver(light);
        // 行为改变，发布通知
        master.openDoor();
    }
}

```
```
回到家打开门
打开灯
打开空调
打开电视
```

不过在JDK9 已经不推荐这种写法
### Spring 应用

spring内部实现了一套事件监听机制，其就是观察者模式的实现。
包含三个角色，事件、事件发布者及事件监听者。
- **事件**：ApplicationEvent，继承自JDK的EventObject，所有事件将继承它，并通过source得到事件源。
- **事件发布者**：ApplicationEventPublisher及ApplicationEventMulticaster接口，使用这个接口，我们的Service就拥有了发布事件的能力。
- **事件订阅者**：ApplicationListener，继承自JDK的EventListener，所有监听器将继承它

ApplicationEventPublisher通过ApplicationEventMulticaster广播事件，事件监听者ApplicationListener监听到事件做出动作。


spring模式默认实现了以下事件
- ContextStartedEvent：ApplicationContext启动后的事件
- ContextStoppedEvent：ApplicationContext停止后触发的事件
- ContextRefreshedEvent：ApplicationContext刷新后的事件
- ContextClosedEvent：ApplicationContext关闭后触发的事件
