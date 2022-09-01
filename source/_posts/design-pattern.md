---
title: 设计模式
date: 2020-11-30 23:05:06
categories: 
- 设计模式
---

 
 > 设计模式是工程设计中积累出经验，是软件开发过程的总结。项目中合理地运用设计模式可以完美地解决很多问题，每种模式在现实中都有相应的原理来与之对应，每种模式都描述了一个在我们周围不断重复发生的问题，以及该问题的核心解决方案，这也是设计模式能被广泛应用的原因。



### 设计模式的六大原则

设计模式遵循六大原则：开闭原则、里氏代换原则、依赖倒转原则、接口隔离原则、迪米特法则及合成复用原则

<!--more-->


- 开闭原则（Open Close Principle）：对扩展开放，对修改关闭。程序尽量支持扩展而避免去修改原有代码
- 里氏代换原则（Liskov Substitution Principle）：任何基类可以出现的地方，子类一定可以出现
- 依赖倒转原则（Dependence Inversion Principle）：针对接口编程，依赖于抽象而不依赖于具体
- 接口隔离原则（Interface Segregation Principle）：使用多个专门的接口，而不使用单一的总接口，降低类之间的耦合度
- 迪米特法则（Demeter Principle）： 一个实体应当尽量少地与其他实体之间发生相互作用，使得系统功能模块相对独立。
- 合成复用原则（Composite Reuse Principle, CRP）：尽量使用合成/聚合的方式，而不是使用继承。


### 设计模式类型

- 创建型模式

工厂模式（Factory Pattern）  
抽象工厂模式（Abstract Factory Pattern）  
单例模式（Singleton Pattern）  
建造者模式（Builder Pattern）  
原型模式（Prototype Pattern）  

- 结构型模式

适配器模式（Adapter Pattern）  
桥接模式（Bridge Pattern）  
组合模式（Composite Pattern）  
装饰器模式（Decorator Pattern）  
外观模式（Facade Pattern）  
享元模式（Flyweight Pattern）  
代理模式（Proxy Pattern）  

- 行为型模式

责任链模式（Chain of Responsibility Pattern）  
命令模式（Command Pattern）  
解释器模式（Interpreter Pattern）  
迭代器模式（Iterator Pattern）  
中介者模式（Mediator Pattern）  
备忘录模式（Memento Pattern）  
观察者模式（Observer Pattern）  
状态模式（State Pattern）  
策略模式（Strategy Pattern）  
模板方法模式（Template Method Pattern）  
访问者模式（Visitor Pattern）  
