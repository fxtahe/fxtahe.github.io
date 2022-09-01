---
title: 设计模式-状态模式
date: 2020-11-16 22:20:37
categories: 
- 设计模式
---

## 状态模式
> 状态模式（State Pattern）当一个对象内在状态改变时允许其改变行为，这个对象看起来像改变了其类。

状态模式允许修改对象状态，使其可在多个状态间转换，比如博客系统，一篇博客可能存在草稿、审阅、发布等状态，不同状态可任意转换。

状态模式角色：
- State（抽象状态类）：接口或抽象类，负责对象状态定义，并且封装环境角色以实现状态切换。
- ConcreteState（具体状态类）：抽象状态类子类，实现特定于状态的方法。
- Context（环境类）：定义客户端需要的接口，并且负责具体状态的切换。

<!--more-->
## 实现
以发布博客文章为例，文章包括草稿、审阅、发布等状态。
定义抽象状态类
```java
public interface BlogState {

    void handle();
}

```
具体状态类，实现状态方法
```java
public class NewState implements BlogState {

    @Override
    public void handle() {
        System.out.println("新建状态");
    }
}

public class CheckState implements BlogState {

    @Override
    public void handle() {
        System.out.println("审阅状态");
    }
}

public class PublishState implements BlogState {

    @Override
    public void handle() {
        System.out.println("发布状态");
    }
}
```
定义状态类，并进行相应的状态操作
```java
public class BlogContext {

    private BlogState state;

    public BlogContext() {
        this.state = new NewState();
    }

    public void setState(BlogState state) {
        this.state = state;
    }

    public BlogState getState() {
        return state;
    }

    public void handle() {
        state.handle();
    }
}
```
测试
```java
public class Test {

    public static void main(String[] args) {

        BlogContext blogContext = new BlogContext();
        blogContext.setState(new CheckState());
        blogContext.handle();
        blogContext.setState(new PublishState());
        blogContext.handle();
    }
}
```
```
审阅状态
发布状态
```

状态模式将一个对象的状态从该对象中分离出来，封装到专门的状态类中，使得对象状态可以灵活变化。