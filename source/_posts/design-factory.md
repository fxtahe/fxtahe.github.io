---
title: 设计模式-工厂模式
date: 2020-09-23 10:58:11
categories: 
- 设计模式
---

## 工厂模式
> “Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.”(在基类中定义创建对象的一个接口，让子类决定实例化哪个类。工厂方法让一个类的实例化延迟到子类中进行。)


在工厂模式中，我们在创建对象时不会对客户端暴露创建逻辑，并且是通过使用一个共同的接口来指向新创建的对象。我们只关注创建工厂对象。通过给工厂对象传递不同参数来实现获得不同的子类。

工厂模式又分为**简单工厂模式，工厂方法模式、抽象工厂模式**
<!--more-->

### 简单工厂模式
定义一个工厂类，它可以根据参数的不同返回不同类的实例，被创建的实例通常都具有共同的父类。因为在简单工厂模式中用于创建实例的方法是静态(static)方法，因此简单工厂模式又被称为静态工厂方法(Static Factory Method)模式，它属于类创建型模式。

简单工厂模式包含三个角色:  
- Factory:工厂角色即工厂类，负责创建所有产品实例
- Product:产品实例的抽象父类，定义了产品的基本属性方法
- ConcreteProduct:具体产品实例，抽象产品的实现，由工厂实现。

定义抽象类手机Phone及其基本方法
```java
public interface Phone {

    void call(String phoneNo);

    void sendMessage(String message);
}
```

创建实现类MiPhone和HuaweiPhone

```java

public class HuaweiPhone implements Phone{

    @Override
    public void call(String phoneNo) {
        System.out.println("华为手机打电话:"+phoneNo);
    }

    @Override
    public void sendMessage(String message) {
        System.out.println("华为手机发消息:"+message);
    }
}

```

```
public class MiPhone implements Phone{
    @Override
    public void call(String phoneNo) {
        System.out.println("小米手机打电话:"+phoneNo);
    }

    @Override
    public void sendMessage(String message) {
        System.out.println("小米手机发消息:"+message);
    }
}
```
创建静态工厂类，通过不同参数生产不同手机
```java
public class PhoneFactory {

    public static Phone createPhone(String phoneName){
        if("Mi".equals(phoneName)){
            return new MiPhone();
        }
        if("Huawei".equals(phoneName)){
            return new HuaweiPhone();
        }
        return null;
    }
}
```

```
    public static void main(String[] args) {
        Phone mi = PhoneFactory.createPhone("Mi");
        mi.call("13666666666");
    }
```
简单工厂模式核心是工厂类，通过不同的参数创建不同的实例。其屏蔽了实例创建的过程只需要通过既定参数获取实例，但是缺点也很明显，工厂类集合了所有产品的创建逻辑，添加新的产品也要修改工厂类。
### 工厂方法模式
工厂方法模式中，核心的工厂类不再负责所有的产品的创建，而是将具体创建的工作交给子类去做。该核心类成为一个抽象工厂角色，仅负责给出具体工厂子类必须实现的接口，而不接触哪一个产品类应当被实例化这种细节。

工厂方法模式具备四个角色
- Factory:抽象工厂，定义工厂类的创建产品实例的方法
- ConcreteFactory：抽象工厂的实现类，实现了创建产品的方法
- Product:产品实例的抽象父类，定义了产品的基本属性方法
- ConcreteProduct:具体产品实例，抽象产品的实现，由具体工厂实现，不同产品对应不同的工厂实现类。

定义手机抽象工厂类PhoneFactory
```java
public interface PhoneFactory {

    Phone createPhone();
}
```
针对不同的手机创建不同的工厂类
```
public class MiPhoneFactory implements PhoneFactory {
    @Override
    public Phone createPhone() {
        return new MiPhone();
    }
}
```

```
public class HuaweiPhoneFactory implements PhoneFactory {
    @Override
    public Phone createPhone() {
        return new HuaweiPhone();
    }
}
```

通过不同的工厂类实现不同手机产品实例

```
public static void main(String[] args) {

    HuaweiPhoneFactory huaweiPhoneFactory = new HuaweiPhoneFactory();
    Phone phone = huaweiPhoneFactory.createPhone();
    phone.sendMessage("hello world");
}
```

工厂方法模式与简单工厂模式不同的是其工厂实现类与产品类对应，在扩展新的产品时添加新的工厂类与产品类即可，不会破坏既有的逻辑，缺点时一种产品只能有一个工厂生产，在已知产品数量的情况下推荐使用简单工厂模式。

### 抽象工厂模式
> 提供一个创建一系列相关或相互依赖对象的接口，而无需指定它们具体的类。

抽象工厂模式相较于工厂方法模式，具体工厂类不仅仅生产一种产品而是负责创建一系列产品(产品族)，是工厂方法模式的升级版。  
抽象工厂模式包含两个新概念：
- **产品族**，是指不同产品等级结构中，功能相关的产品组成的家族。

- **产品等级结构**, 即产品的继承结构，继承（或实现）同一抽象产品的具体产品属于同一产品等级结构。

比如上例中的华为和小米是两个产品族，继承或实现抽象手机类的华为手机和小米手机就属于同一产品等级，而如果华为电脑和小米电脑继承或实现抽象电脑类型则为另一产品等级

抽象工厂模式同样具备四个角色
- AbstractFactory 抽象工厂，定义创建一族产品的方法，每个方法对应一种产品
- ConcreteFactory 实现抽象工厂定义生产族产品的方法，生产一组具体产品，构成一个产品族
- AbstractProduct 抽象产品，为产品等级的抽象，比如电脑或者手机抽象产品
- Concrete Product 具体产品，为抽象产品的具体实现。


新增电脑产品等级
```
public interface Computer {

    void surfing();

    void playGame();
}
```

具体实现产品
```
//小米电脑
public class MiComputer implements Computer {

    @Override
    public void surfing() {
        System.out.println("小米电脑上网冲浪");
    }

    @Override
    public void playGame() {
        System.out.println("小米电脑打游戏");
    }
}
//华为电脑
public class HuaweiComputer implements Computer {
    @Override
    public void surfing() {
        System.out.println("华为电脑上网冲浪");
    }

    @Override
    public void playGame() {
        System.out.println("华为电脑打游戏");
    }
}

```

定义抽象工厂类
```java
public interface EleFactory {
    
    Phone createPhone();
    Computer createComputer();
}
```


具体的工厂类
```java
//华为工厂
public class HuaweiFactory implements EleFactory {
    @Override
    public Phone createPhone() {
        return new HuaweiPhone();
    }

    @Override
    public Computer createComputer() {
        return new HuaweiComputer();
    }
}
//小米工厂
public class MiFactory implements EleFactory {
    @Override
    public Phone createPhone() {
        return new MiPhone();
    }

    @Override
    public Computer createComputer() {
        return new MiComputer();
    }
}
```
生产产品，测试
```java
    public static void main(String[] args) {
        EleFactory miFactory = new MiFactory();
        Phone miPhone = miFactory.createPhone();
        Computer miComputer = miFactory.createComputer();
        miPhone.sendMessage("hello world Mi");
        miComputer.playGame();

        EleFactory huaweiFactory = new HuaweiFactory();
        Phone huaweiPhone = huaweiFactory.createPhone();
        Computer huaweiComputer = huaweiFactory.createComputer();
        huaweiPhone.sendMessage("hello world Huawei");
        huaweiComputer.playGame();
    }
```
```
小米手机发消息:hello world Mi
小米电脑打游戏
华为手机发消息:hello world Huawei
华为电脑打游戏
```

抽象工厂模式在工厂方法模式的基础上，增添了产品族和产品等级结构的概念，是产品工厂模式的更深一步的抽象。可以更加灵活的添加某一产品族，  
但是新增产品等级即在新增产品的过程中，比如添加pad平板产品则需要去修改抽象工厂与所有抽象实现类类，不符合开闭原则。



