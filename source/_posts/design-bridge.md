---
title: 设计模式-桥接模式
date: 2020-11-11 18:55:05
categories: 
- 设计模式
---

## 桥接模式
> 将抽象部分与它的实现部分分离，使它们都可以独立地变化.

假设咖啡机器可以生产咖啡，后面增加生产大杯小杯中杯的功能需求，对机器进行了改造。后面有人不喜欢喝咖啡要喝开水或者可乐牛奶，又要对机器进行改造。如果后面又扩展了其他饮品或者其他尺寸，机器就会越来越臃肿。 其实饮品和大小中杯就属于不同的维度，同时这两个维度都可以进行扩展，我们就可以通过桥接模式将这两个维度连接起来

桥接模式包含角色：  
- Abstraction（抽象类）：维护了一个Implementor的关联关系，一般为抽象类
- RefinedAbstraction（扩充实现类）：扩充由Abstraction定义的接口，实现其中的抽象方法
- Implementor（实现类接口）：提供了一些基本方法，Abstraction通过与其的关联关系，可以调用其基本方法，使用关联关系替代继承关系
- ConcreteImplementor（具体实现类）：实现Implementor接口，为Abstraction抽象类提供基本方法具体实现
<!--more-->

### 实现
以刚刚的饮料与尺寸作为案例
定义饮料抽象类
```java
public abstract class AbstractDrink {
    public CupSize cupSize;

    public AbstractDrink(CupSize cupSize) {
        this.cupSize = cupSize;
    }

    public abstract void getDrink();
}
```

定义尺寸实现类接口
```java
public interface CupSize {

     void getSize();
}
```

扩展饮料实现类
```java
//咖啡
public class Coffee extends AbstractDrink {
    public Coffee(CupSize cupSize) {
        super(cupSize);
    }

    @Override
    public void getDrink() {
        System.out.println("咖啡");
        this.cupSize.getSize();
    }
}
//牛奶
public class Milk extends AbstractDrink {
    public Milk(CupSize cupSize) {
        super(cupSize);
    }

    @Override
    public void getDrink() {
        System.out.println("牛奶");
        this.cupSize.getSize();
    }
}
```

具体杯尺寸实现类
```java
//小杯
public class SmallSize implements CupSize {
    @Override
    public void getSize() {
        System.out.println("小杯");
    }
}

//中杯
public class MediumCupSize implements CupSize {
    @Override
    public void getSize() {
        System.out.println("中杯");
    }
}

//大杯
public class BigCupSize implements CupSize{

    @Override
    public void getSize() {
        System.out.println("大杯");
    }
}
```


组合不同的饮料
```java
public class Test {

    public static void main(String[] args) {

        AbstractDrink milk = new Milk(new SmallSize());
        milk.getDrink();

        AbstractDrink coffee = new Coffee(new BigCupSize());
        coffee.getDrink();
    }
}
```
```
牛奶
小杯
咖啡
大杯
```

桥接模式将每个维度抽取为独立的类层次，通过继承实现扩展对应层次的实现。其每个维度相互独立便于扩展，降低了耦合








