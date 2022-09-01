---
title: 设计模式-命令模式
date: 2020-11-16 15:55:20
categories: 
- 设计模式
---

## 命令模式
> 命令模式(Command Pattern)：将请求转换为一个包含与请求相关的所有信息的独立对象。 该转换让你能根据不同的请求将方法参数化、 延迟请求执行或将其放入队列中， 且能实现可撤销操作。

在日常生活中，我们再饭店吃饭时通过服务员点餐下单，服务员将点的菜品告诉厨师。我们还可以通过服务员点饮料酒水，服务员通知柜台订单的饮料酒水，这就属于命令模式。命令模式中命令发送者不用关心命令接收者如何接收执行命令，将发送者和接收者独立开来。

命令模式包含角色：  
- Command（抽象命令类）：抽象类或者接口，声明了执行命令的操作。
- ConcreteCommand（具体命令类）：抽象命令子类，实现了执行命令的具体操作，绑定了具体的接收者对象。
- Invoker（调用者）：请求发送者，通过命令对象执行请求。
- Receiver（接收者）：接收者执行与请求相关的操作，它具体实现对请求的业务处理。
<!--more-->
### 实现
以去饭店吃饭为例，客户就是调用者，厨师或者柜台就是接收者，而订单就是命令，定义抽象命令类 `OrderCommmand`

- 抽象命令类

```java
public interface OrderCommand {

    void execute();
}
```

- 具体实现类:菜、酒水饮料

```java
public class DrinkOrderCommand implements OrderCommand {

    private DrinkManager drinkManager;

    public DrinkOrderCommand(DrinkManager drinkManager) {
        this.drinkManager = drinkManager;
    }

    @Override
    public void execute() {
        drinkManager.manageDrink();
    }
}
```
```java
public class MealOrderCommand implements OrderCommand {

    private Chef chef;

    public MealOrderCommand(Chef chef) {
        this.chef = chef;
    }

    @Override
    public void execute() {
        chef.cook();
    }
}
```

- 调用者：顾客

```java
public class Customer {

    private ArrayList<OrderCommand> orderCommands = new ArrayList<>();

    public void addCommand(OrderCommand orderCommand){
        orderCommands.add(orderCommand);
    }

    public void call(){
        orderCommands.forEach(OrderCommand::execute);
    }
}
```

- 接收者：厨师、酒水管理

```java
public class Chef {

    public void cook(){
        System.out.println("提供菜品");
    }
}

public class DrinkManager {

    public void manageDrink(){
        System.out.println("提供酒水");
    }
}
```

场景测试
```java
public class Test {

    public static void main(String[] args) {

        Customer customer = new Customer();
        Chef chef = new Chef();
        DrinkManager drinkManager = new DrinkManager();
        customer.addCommand(new MealOrderCommand(chef));
        customer.addCommand(new DrinkOrderCommand(drinkManager));
        customer.call();
    }
}

```
```
提供菜品
提供酒水
```

命令模式的本质是对请求进行封装，一个请求对应于一个命令，将发出命令的责任和执行命令的责任分割开。