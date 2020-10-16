---
title: 设计模式-装饰器模式
date: 2020-10-16 09:18:06
tags: 设计模式
---

## 装饰器模式
装饰器模式可以在不改变对象行为能力的基础上为其添加新的行为能力，使其能够适应不同场景下的需求，如生活中，天气热了可以穿短袖，天冷了穿羽绒服，下雨了穿雨衣，睡觉了穿睡衣。不同的衣服就相当于不同的装饰器(Decorator)。

<!--more-->
装饰器包含四个角色：
- Component（抽象构件）：具体构件和抽象装饰类的共同父类，声明了具体构建的业务方法。
- ConcreteComponent（具体构件）：抽象构建的子类，实现抽象构件的方法，装饰器为其提供额外的功能呢
- Decorator（抽象装饰器）：抽象装饰器定义了为被装饰者增加功能的业务方法，维护了一个指向被装饰对象的引用，通过该引用可以调用装饰之前构件对象的方法，并通过其子类扩展该方法，以达到装饰的目的。
- ConcreteDecorator（具体装饰器）：定义了可动态添加到构件的额外行为。 具体装饰器会重写抽象装饰类的方法， 并在调用父类方法之前或之后进行额外的行为。


### 实现

定义一个抽象构件 人
```java
public interface Person {
    void operation();
}
```
实现抽象构件，具体构件 学生
```java
public class Student implements Person {

    @Override
    public void operation() {
        System.out.println("去上学");
    }
}
```

定义抽象装饰器实现抽象构件,衣服装饰器根据不同天气场景穿着不同衣服
```java
public class DressDecorator implements Person {

    protected Person person;

    public DressDecorator(Person person){
        this.person=person;
    }

    @Override
    public void operation() {
        person.operation();
    }
}
```
具体装饰器，添加额外行为
```java
public class TShirtDecorator extends DressDecorator{
    public TShirtDecorator(Person person) {
        super(person);
    }

    @Override
    public void operation() {
        dressTShirt();
        super.operation();
    }

    public void dressTShirt(){
        System.out.println("夏天穿短袖");
    }
}
```

```java
public class RainCoatDecorator extends DressDecorator {
    public RainCoatDecorator(Person person) {
        super(person);
    }

    @Override
    public void operation() {
        dressRainCoat();
        super.operation();
    }

    public void dressRainCoat(){
        System.out.println("雨天穿雨衣");
    }
}
```
模拟测试
```java
    public static void main(String[] args) {

        Person student = new Student();
        student.operation();
        Person tShirtStudent= new TShirtDecorator(student);
        tShirtStudent.operation();
        RainCoatDecorator rainCoatStudent = new RainCoatDecorator(student);
        rainCoatStudent.operation();
    }

```

```
上学
夏天穿短袖
上学
下雨天穿着雨衣
上学
```






