---
title: 设计模式-中介者模式
date: 2020-11-29 23:03:44
categories: 
- 设计模式
---

> 中介者模式（Mediator Pattern）:用一个中介对象（中介者）来封装一系列的对象交互，中介者使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

在系统中经常会遇到复杂的对象关联关系，导致系统复杂难以维护，而中介者模式就是为了减少对象之间混乱无序的依赖关系。例如房产中介对买方和卖方的沟通，资源飞机场控制塔台控制飞机的起飞降落，交警指挥道路交通，房产中介、控制塔台和交警都属于中介者。

中介者模式包含的角色：
- Mediator（抽象中介者）：定义同事对象间的通信接口
- ConcreteMediator（具体中介者）：实现抽象中介者定义的接口，维护了与各同时对象的关系
- Colleague（抽象同事类）：定义同事类的抽象方法
- ConcreteColleague（具体同时类）：实现抽象同时类定义的抽象方法

<!--more-->
## 实现
以聊天软件为例，聊天软件就是中介者，而用户就属于同事类。 

抽象中介类，抽象聊天软件InstantMessenger
```java
public abstract class InstantMessenger {

    public List<User> users = new ArrayList<>();

    public void registerUser(User user){
        users.add(user);
    }

    public abstract void replay(String sender,String receiver,String message);
}

```
具体实现微信
```java
public class WeChat extends InstantMessenger {

    @Override
    public void replay(String sender, String receiver, String message) {
        User user = this.users.stream().filter(el -> el.getUserName().equals(receiver)).findFirst().orElse(null);

        user.receiveMessage(sender,message);
    }
}

```
定义抽象同事，聊天软件用户
```java
public abstract class User {

    public InstantMessenger instantMessenger;

    public User(InstantMessenger instantMessenger,String userName) {
        this.instantMessenger = instantMessenger;
        this.userName = userName;
    }
    public abstract void sendMessage(String userName, String message);

    public abstract void receiveMessage(String sender,String message);

    public String userName;

    public String getUserName() {
        return userName;
    }
}
```

创建微信用户类
```java
public class WeChat extends InstantMessenger {

    @Override
    public void replay(String sender, String receiver, String message) {
        User user = this.users.stream().filter(el -> el.getUserName().equals(receiver)).findFirst().orElse(null);

        user.receiveMessage(sender,message);
    }
}
```

测试
```java
public class Test {
    public static void main(String[] args) {

        InstantMessenger instantMessenger = new WeChat();
        User userA = new WeChatUser(instantMessenger,"马云");
        instantMessenger.registerUser(userA);
        User userB = new WeChatUser(instantMessenger,"马化腾");
        instantMessenger.registerUser(userB);

        userA.sendMessage("马化腾","小马周末出来喝杯酒");

        userB.sendMessage("马云","不好意思，最近得加班，下次吧");

        userA.sendMessage("马化腾","那下次吧，bye");

        userB.sendMessage("马云","行，下次一定，再见");
    }
}
```

```
马化腾：小马周末出来喝杯酒
马云：不好意思，最近得加班，下次吧
马化腾：那下次吧，bye
马云：行，下次一定，再见
```
中介者模式中，中介者角色承担了较多的责任，所以一旦这个中介者对象出现了问题，整个系统将会受到重大的影响。


