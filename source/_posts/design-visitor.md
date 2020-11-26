---
title: 设计模式-访问者模式
date: 2020-11-26 22:17:34
tags: 设计模式
---

> 访问者模式（Visitor Pattern）：将作用于某种数据结构中的各元素的操作分离出来封装成独立的类，使其在不改变数据结构的前提下可以添加作用于这些元素的新的操作，为数据结构中的每个元素提供多种访问方式。

公园中存在多个景点，也存在多个游客，不同的游客对同一个景点的评价可能不同。这种被访问对象具有稳定结构而访问方式多样的场景，一般采用访问者模式。

访问者模式包含角色：
- Vistor（抽象访问者）：定义一个访问具体元素的接口，为每个具体元素类对应一个访问操作 。
- ConcreteVisitor（具体访问者）：实现抽象访问者角色中声明的各个访问操作
- Element（抽象元素）：定义一个以抽象访问者为参数的accpet方法
- ConcreteElement（具体元素）：实现抽象元素的accpet方法
- ObjectStructure（对象结构）：对象结构是一个元素的集合，它用于存放元素对象，并且提供了遍历其内部元素的方法。

<!--more-->
### 实现
以游客游览公园为例
定义抽象元素景点
```java
public interface Scenic {

    void accpet(Visitor visitor);
}
```
实际景点
```java
public class ScenicA implements Scenic {
    @Override
    public void accpet(Visitor visitor) {
        visitor.visit(this);
    }

    public void operation(){
        System.out.println("荡秋千");
    }
}

public class ScenicB implements Scenic {
    @Override
    public void accpet(Visitor visitor) {
        visitor.visit(this);
    }

    public void operation(){
        System.out.println("跳广场舞");
    }
}
```

定义抽象访问者,并定义访问两个景点的方法。
```java
public interface Visitor {

    void visit(ScenicA scenicA);

    void visit(ScenicB scenicB);
}
```
实现抽象访问者，老人和小孩
```java
public class Child implements Visitor {
    @Override
    public void visit(ScenicA scenicA) {
        scenicA.operation();
        System.out.println("玩的很开心");
    }

    @Override
    public void visit(ScenicB scenicB) {
        scenicB.operation();
        System.out.println("好无聊");
    }
}

public class OldMan implements Visitor {
    @Override
    public void visit(ScenicA scenicA) {
        scenicA.operation();
        System.out.println("幼稚的游戏");
    }

    @Override
    public void visit(ScenicB scenicB) {
        scenicB.operation();
        System.out.println("锻炼身体");
    }
}
```

游览测试
```java
public class VisitorClient {

    public static void main(String[] args) {
        Park park = new Park();
        park.add(new ScenicA());
        park.add(new ScenicB());

        Visitor child = new Child();
        Visitor oldMan = new OldMan();

        park.accept(child);
        System.out.println("--------------");
        park.accept(oldMan);
    }
}
```

```
荡秋千
玩的很开心
跳广场舞
好无聊
--------------
荡秋千
幼稚的游戏
跳广场舞
锻炼身体
```

访问者模式用于元素结构稳定基本固定的场景，对需要经常修改元素的场景不适用。