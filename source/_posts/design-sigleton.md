---
title: 设计模式-单例模式
date: 2020-09-15 16:04:39
categories: 
- 设计模式
---

## 单例模式
该设计模式属于创建型模式，这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。即一个类只有一个实例化对象的实现模式。
<!-- more -->

### 实现方式
单例模式具有多种实现方式，以下为各种实现
#### 饿汉式
饿汉式即在初始化类的同时就创建该对象，但是这样会浪费内存
```
public class HungerSingleton {

    private static final HungerSingleton instance= new HungerSingleton();

    private HungerSingleton(){}

    public static HungerSingleton getInstance(){
        return instance;
    }
}
```

#### 懒汉式
懒汉式是在饿汉式的方式上做了懒加载处理,避免类加载时的内存占用。但是这样会出现多线程的线程安全问题

```
public class LazySingleton {
    
    private static LazySingleton instance;

    private LazySingleton(){
    }

    public static LazySingleton getInstance(){
        if(null==instance){
            instance=new LazySingleton();
        }
        return instance;
    }
}
```
#### 同步懒汉式
这种方式可以在多线程中保证线程安全同时保证懒加载，但是会出现线程阻塞的性能问题
```
public class SynchronizationLazySingleton {
    
    private static SynchronizationLazySingleton instance;
    
    private SynchronizationLazySingleton(){}
    
    public static synchronized SynchronizationLazySingleton getInstance(){
        if(null==instance){
            instance=new SynchronizationLazySingleton();
        }
        return instance;
    }
}
```

#### 双重检查式
双重检查是在执行同步前先进行检查，能保证线程安全且提高性能，volatile保证代码有序性，避免jvm指令重排序出现的问题

```
public class DoubleCheckSingleton {
    private static volatile DoubleCheckSingleton instance;

    private DoubleCheckSingleton(){}

    public static DoubleCheckSingleton getInstance(){
        //获取锁前校验，已创建不再争抢锁
        if(null == instance){
            synchronized (DoubleCheckSingleton.class){
                if(null== instance){
                    instance=new DoubleCheckSingleton();
                }
            }
        }
        return instance;
    }
}
```

#### 静态内部类式
这种方式利用了 classloader 机制来保证初始化 instance 时只有一个线程，即保证了线程安全性，而内部类只有在显式调用时才会被加载，即保证了懒加载
```
public class StaticInnerSingleton {

    private StaticInnerSingleton(){}

    private static class SingletonHolder{
        private static final StaticInnerSingleton instance = new StaticInnerSingleton();
    }

    public static StaticInnerSingleton getInstance(){
        return SingletonHolder.instance;
    }
}
```

#### 枚举式
不仅能避免多线程同步问题，而且还自动支持序列化机制，防止反序列化重新创建新的对象，绝对防止多次实例化。

```
public class EnumSingleton {

    private EnumSingleton(){}

    private enum SingletonHolder{
        INSTANCE;
        private EnumSingleton instance;
        SingletonHolder(){
            instance = new EnumSingleton();
        }
        private EnumSingleton getInstance(){
            return instance;
        }
    }

    public static EnumSingleton getInstance(){
        return SingletonHolder.INSTANCE.getInstance();
    }
    
}
```





