---
title: 设计模式-适配器模式
date: 2020-10-13 17:14:18
tags: 设计模式
---

## 适配器模式

在日常生活中，适配器处处可见，比如电源适配器，将220V电压转换为电器可用的电压。亦或者插头适配器，可以将三孔插座转两孔的插口适配器。  


适配器模式就是将一个类的接口转换成客户希望的另外一个接口。使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。别名为包装器（Wrapper）
<!--more-->
适配器模式包含三个角色：
- Target(目标抽象类)：目标抽象类定义客户所需的接口，可以是一个抽象类或接口，也可以是一个具体的类。
- Adapter（适配器类）: 适配器可以调用另外一个接口，作为一个转换器，对Adaptee和Target进行适配，适配器类是适配器模式的核心。
- Adaptee（适配者类）：适配者即被适配的角色，它定义了已存在的接口，这个接口需要适配。一般是一个具体的类，包含了客户希望使用的业务方法，在某些情况下可能没有适配者类的源代码。


适配器模式分为**类适配器**和**对象适配器**。；

### 对象适配器模式
在对象适配器模式中，适配器与适配者之间是关联关系

现在我们将三孔插座转换为两孔插座，需要一个插孔适配器。    
定义要使用的两孔插口 Target
```
public class TwoHoleSocket {

    public void useTwoHole(){
        System.out.println("使用两孔插头");
    }
}
```
定义要适配的三孔插口 Adaptee
```
public class ThreeHoleSocket {

    public void useThreeHole(){
        System.out.println("使用三孔插头");
    }
}
```
定义插口转换器 Adapter
```
public class HoleSocketAdapter extends TwoHoleSocket {

    public ThreeHoleSocket threeHoleSocket;

    public HoleSocketAdapter(ThreeHoleSocket threeHoleSocket) {
        this.threeHoleSocket = threeHoleSocket;
    }

    @Override
    public void useTwoHole() {
        this.threeHoleSocket.useThreeHole();
    }
}
```


```
    public static void main(String[] args) {
        TwoHoleSocket holeSocketAdapter = new HoleSocketAdapter(new ThreeHoleSocket());
        holeSocketAdapter.useTwoHole();
    }
```
### 类适配器模式
在类适配器模式中，适配器与适配者之间是继承（或实现）关系,

```
public interface TwoHoleSocket {

    void useTwoHole();
}

```

适配器继承适配者类实现目标类（目标类必须为接口类型，因为java不支持多继承）
```
public class HoleSocketAdapter extends ThreeHoleSocket implements TwoHoleSocket {

    @Override
    public void useTwoHole() {
        useThreeHole();
    }

}
```

```
    public static void main(String[] args) {
        TwoHoleSocket holeSocketAdapter = new HoleSocketAdapter();
        holeSocketAdapter.useTwoHole();
    }
```

适配器模式属于补偿模式，专门用来在系统后期扩展、修改时使用，但要注意不要过度使用适配器模式。如果系统过多使用适配器模式，会使系统变得凌乱，比如本来调用的A接口方法内部却适配为B接口方法。当系统大量使用适配器模式时应考虑重构代码。



