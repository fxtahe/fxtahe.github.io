---
title: 设计模式-解释器模式
date: 2020-11-30 22:00:25
categories: 
- 设计模式
---

> 解释器模式（Interpreter Pattern）定义一个语言的文法，并且建立一个解释器来解释该语言中的句子，这里的“语言”是指使用规定格式和语法的代码

解释器模式在编译器中语义分析过程中常用的一种模式，它用于描述如何使用面向对象语言构成一个简单的语言解释器。通过定义语法规则，然后解析其真实含义。

解释器包含角色：
- AbstractExpression（抽象表达式）：在抽象表达式中声明了抽象的解释操作，它是所有终结符表达式和非终结符表达式的公共父类
- TerminalExpression（终结表达式）：实现与文法中的终结符相关联的解释操作，一个句子中的每一个终结符需要该类的一个实例
- NonterminalExpression（非终结表达式）：非终结解释器
对文法中的规则的解释操作。
- Context（环境）：环境角色
包含解释器之外的一些全局信息

在加减运算的语法中，比如a+b a,b就是终结符号，+-等就属于非终结表达式
<!--more-->
### 实现
通过解释器模式实现一个简单的计算器
```java
public abstract class AbstractExpression {

    public  abstract void interpret(Context ctx);
}
```
```java
class TerminalExpression extends  AbstractExpression {

    @Override
    public  void interpret(Context ctx) {
        //终结符表达式的解释操作
    }

}
```
```java
public class NonterminalExpression extends AbstractExpression {

    private  AbstractExpression left;
    private  AbstractExpression right;
    public  NonterminalExpression(AbstractExpression left,AbstractExpression right) {
        this.left=left;
        this.right=right;
    }
    @Override
    public void interpret(Context ctx) {
    //递归调用每一个组成部分的interpret()方法

    //在递归调用时指定组成部分的连接方式，即非终结符的功能
    }
}
```
```java
public class Context {
    private HashMap<String,String> map = new HashMap();

    public void assign(String key, String value) {
        map.put(key,value);
    }


    public String  lookup(String key) {
        return map.get(key);
    }
}

``` 

解释器模式主要适用于语法解析的使用场景，一般较少使用
