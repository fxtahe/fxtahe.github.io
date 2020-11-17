---
title: 设计模式-迭代器模式
date: 2020-11-17 23:07:19
tags: 设计模式
---

## 迭代器模式
> 迭代器模式（Iterator Pattern）：提供一种方法顺序访问一个聚合对象中各个元素，而又不暴露该对象的内部。

迭代器模式是开发中经常使用的模式，比如通过迭代器遍历集合数据。

迭代器模式包含角色：
- Iterator（抽象迭代器）：定义了访问和遍历元素的接口，声明了用于遍历数据元素的方法
- ConcreteIterator（具体迭代器）：实现了抽象迭代器接口，完成对聚合对象的遍历，并且记录当前遍历元素位置。
- Aggregate（抽象聚合类）：存储和管理元素对象，并声明创建迭代器方法
- ConcreteAggregate（具体聚合类）：抽象聚合类子类，实现创建迭代器方法
<!--more-->
### 实现
定义抽象迭代器
```java
public interface Iterator {

    boolean hasNext();

    Object next();
}
```
定义抽象聚合类
```java
public interface Aggregate {

    Iterator iterator();

    boolean add(String item);
}
```
具体迭代器
```java
public class ConcreteIterator implements Iterator {

    private String[] names;

    private int index;

    public ConcreteIterator(String[] names){
        this.names = names;
        this.index = Math.max(names.length-1, 0);
    }

    @Override
    public boolean hasNext() {
        return index>=0;
    }

    @Override
    public Object next() {
        String oldValue = names[index];
        names[index]=null;
        index--;
        return oldValue;
    }
}
```
具体聚合类
```java
public class ConcreteAggregate implements Aggregate {

    String[] names = new String[]{};
    
    private int size=0;

    @Override
    public boolean add(String name){
        names = Arrays.copyOf(names, size+1);
        names[size] = name;
        size++;
        return true;
    }


    @Override
    public Iterator iterator() {
        return new ConcreteIterator(names);
    }

    public boolean isEmpty(){
        return names.length ==0;
    }
}
```
测试
```java
public class Test {

    public static void main(String[] args) {
        Aggregate collection = new ConcreteAggregate();
        collection.add("java");
        collection.add("php");
        collection.add("go");
        collection.add("python");
        Iterator iterator = collection.iterator();
        while (iterator.hasNext()){
            String next = (String) iterator.next();
            System.out.println(next);
        }
    }
}
```
```
python
go
php
java
```

