---
title: 设计模式-建造者模式
date: 2020-10-23 18:31:39
categories: 
- 设计模式
---

## 建造者模式
> 建造者模式（Builder Pattern）使用多个简单的对象一步一步构建成一个复杂的对象，使得同样的构建过程可以创建不同的表示

建造者模式一步一步创建一个复杂的对象，它允许用户只通过指定复杂对象的类型和内容就可以构建它们，用户不需要知道内部的具体构建细节。例如电脑是由CPU、主板、硬盘、内存条、电源、显示器、鼠标及键盘等组装成的，可以根据不同需求组装不同配置的电脑，而不需要关心组装细节。

建造者模式主要包括以下几个角色
- Builder（抽象建造者）：定义创建产品对象构建各个组件的抽象方法
- ConcreteBuilder（具体建造者）：实现Builder构建产品对象部件的抽象方法，定义并明确所创建的复杂对象
- Product（产品对象）：被构建的复杂对象，包含多个组成部件
- Director（指挥者）：指挥者又称为导演类，它负责安排复杂对象的建造次序，指挥者与抽象建造者之间存在关联关系

<!--more-->

### 实现
假如你现在想要组装一台电脑，需要构建不同的组件，比如CPU、主板及硬盘等等，而你又不懂如何组装，找到电脑老板提出自己的需求，老板然后让店里的技术人员帮你组装一台电脑。

定义抽象建造者
```java
public interface ComputerBuilder {

    void setCPU(String CPU);

    void setDisk(String Disk);

}
```
定义电脑建造类
```java
public class ComputerProductBuilder implements ComputerBuilder {

    private String CPU;

    private String Disk;

    @Override
    public void setCPU(String CPU) {
        this.CPU = CPU;
    }

    @Override
    public void setDisk(String Disk) {
        this.Disk = Disk;
    }

    public ComputerProduct getResult(){
        return new ComputerProduct(CPU,Disk);
    }

}
```
定义电脑说明书建造类
```
public class ComputerManualBuilder implements ComputerBuilder {

    private String CPU;

    private String Disk;

    @Override
    public void setCPU(String CPU) {
        this.CPU = CPU;
    }

    @Override
    public void setDisk(String Disk) {
        this.Disk = Disk;
    }

    public ComputerManual getResult(){
        return new ComputerManual(CPU,Disk);
    }

}
```
定义电脑产品
```java
public class ComputerProduct {

    private String CPU;

    private String Disk;

    public ComputerProduct(String CPU, String disk) {
        this.CPU = CPU;
        Disk = disk;
    }

    public String getCPU() {
        return CPU;
    }

    public void setCPU(String CPU) {
        this.CPU = CPU;
    }

    public String getDisk() {
        return Disk;
    }

    public void setDisk(String disk) {
        Disk = disk;
    }

    @Override
    public String toString() {
        return "ComputerProduct{" +
                "CPU='" + CPU + '\'' +
                ", Disk='" + Disk + '\'' +
                '}';
    }
}
```
定义电脑说明书类
```java
public class ComputerManual {

    public String CPU;

    public String Disk;



    public String getCPU() {
        return CPU;
    }

    public void setCPU(String CPU) {
        this.CPU = CPU;
    }

    public String getDisk() {
        return Disk;
    }

    public void setDisk(String disk) {
        Disk = disk;
    }

    @Override
    public String toString() {
        return "电脑说明书{" +
                "CPU='" + CPU + '\'' +
                ", Disk='" + Disk + '\'' +
                '}';
    }
}
```

定义指挥类
```java
public class Director {

    public void buildComputer(ComputerBuilder builder){
        builder.setCPU("AMD");
        builder.setDisk("512G");
    }
}
```

构建测试
```java
    public static void main(String[] args) {

        Director director = new Director();
        //建造电脑
        ComputerProductBuilder computerProductBuilder = new ComputerProductBuilder();
        director.buildComputer(computerProductBuilder);
        ComputerProduct computer = computerProductBuilder.getResult();
        System.out.println(computer);

        //建造电脑说明书
        ComputerManualBuilder computerManualBuilder = new ComputerManualBuilder();
        director.buildComputer(computerManualBuilder);
        ComputerManual manual = computerManualBuilder.getResult();
        System.out.println(manual);
    }
```
打印
```
电脑产品{CPU='AMD', Disk='512G'}
电脑说明书{CPU='AMD', Disk='512G'}
```

建造者模式隐藏了创建复杂对象的过程，不需要关心其具体的构建细节，同样的构建过程可以创建不同的对象

### 简化建造者模式-链式编程
在传统的建造者模式中，存在指挥者对象，其主要是关联建造类与被创建对象关系，在其建造方法中调用建造者对象的部件构造与装配方法，从而完成创建对象。
其实该对象可以省略，建造者通过链式编程的方式构建复杂对象。

假设你想要组装一个不同配置的电脑，有些配置需要而有些可能不需要，所以要不同的组装方式（构造方法）
```java
public class Computer {

    private String CPU;

    private String disk;

    private String displayScreen;

    private String mouse;

    private String keyboard;
    
    
    public Computer(String CPU, String disk) {
        this.CPU = CPU;
        this.disk = disk;
    }

    public Computer(String CPU, String disk, String displayScreen) {
        this.CPU = CPU;
        this.disk = disk;
        this.displayScreen = displayScreen;
    }

    public Computer(String CPU, String disk, String displayScreen, String mouse, String keyboard) {
        this.CPU = CPU;
        this.disk = disk;
        this.displayScreen = displayScreen;
        this.mouse = mouse;
        this.keyboard = keyboard;
    }

}

```

这时我们通过静态内部类的方式简化构造
```java
public class Computer {

    private String CPU;
    private String disk;
    private String displayScreen;
    private String mouse;
    private String keyboard;

    private Computer(){}
    private Computer(Builder builder){
        this.CPU=builder.CPU;
        this.keyboard=builder.keyboard;
        this.displayScreen=builder.displayScreen;
    }
    public static class Builder{
        private String CPU;
        private String disk;
        private String displayScreen;
        private String mouse;
        private String keyboard;

        public Builder(String cpu,String disk){
            this.CPU = cpu;
            this.disk = disk;
        }
        public Builder keyboard(String keyboard){
            this.keyboard = keyboard;
            return this;
        }

        public Builder displayScreen(String displayScreen){
            this.displayScreen = displayScreen;
            return this;
        }

        public Builder mouse(String mouse){
            this.mouse = mouse;
            return this;
        }

        public Computer build(){
            return new Computer(this);
        }

    }
    
}
```

构造不同的电脑
```java
    public static void main(String[] args) throws InterruptedException {
        
        Computer computerA = new Computer.Builder("CPU", "硬盘")
                .displayScreen("显示屏")
                .build();

        Computer computerB = new Computer.Builder("CPU", "硬盘")
                .displayScreen("显示屏").keyboard("键盘").mouse("鼠标")
                .build();
    }
```

通过不同的属性构造不同的产品


