---
title: 设计模式-责任链模式
date: 2020-10-27 18:43:02
tags: 设计模式
---

## 责任链模式
>职责链模式(Chain of Responsibility  Pattern)：避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。

例如一个员工请假审批系统，一到三天组长审批，三到五天主管审批，五到十天经理审批，十五天以上老板审批。这个审批流程就形成一条责任链，而链上的每一个对象都是请求处理者，实现了请求与审批处理者的解耦。
<!--more-->
责任链模式主要包括两个角色

- Handler（抽象处理者）：声明了具体处理者处理请求的对象，并维护了下一处理者对象引用，通过该引用处理者形成一条链。
- ConcreteHandler（具体处理类）：抽象处理类的具体实现，实现了抽象处理者中定义的抽象请求处理方法。处理前先判断是否可以处理当前请求。同时可以将请求转发给下一处理者

### 实现
以审批流程为实例  

定义审批请求类
```java
public class ApprovalRequest {

    private String approvalPerson;

    private int approvalDays;

    public ApprovalRequest(String approvalPerson, int approvalDays) {
        this.approvalPerson = approvalPerson;
        this.approvalDays = approvalDays;
    }

    public String getApprovalPerson() {
        return approvalPerson;
    }

    public int getApprovalDays() {
        return approvalDays;
    }
}
```

定义抽象审批处理类
```java
public abstract class ApprovalHandler {

    protected ApprovalHandler nextHandler;

    protected int maxDays;

    public ApprovalHandler(int maxDay) {
        this.maxDays = maxDay;
    }

    public abstract void approveRequest(ApprovalRequest request);
    
    public ApprovalHandler setNextHandler(ApprovalHandler nextHandler){
        this.nextHandler= nextHandler;
        return this.nextHandler;
    }
}
```

审批组长
```java
public class GroupLeaderHandler extends ApprovalHandler {

    public GroupLeaderHandler(int maxDays) {
        super(maxDays);
    }

    @Override
    public void approveRequest(ApprovalRequest request) {
        int approvalDays = request.getApprovalDays();
        String approvalPerson = request.getApprovalPerson();

        if(approvalDays<=maxDays){
            System.out.println("组长审批通过，批准"+approvalPerson+"请假"+approvalDays+"天");
        }else{
            nextHandler.approveRequest(request);
        }
    }
}
```

审批经理
```java
public class ManagerHandler extends ApprovalHandler {
    public ManagerHandler(int maxDays) {
        super(maxDays);
    }

    @Override
    public void approveRequest(ApprovalRequest request) {
        int approvalDays = request.getApprovalDays();
        String approvalPerson = request.getApprovalPerson();
        if(approvalDays <= maxDays){
            System.out.println("经理审批通过，批准"+approvalPerson+"请假"+approvalDays+"天");
        }else{
            nextHandler.approveRequest(request);
        }
    }
}
```
审批老板
```java
public class BossHandler extends ApprovalHandler {
    public BossHandler(int maxDays) {
        super(maxDays);
    }

    @Override
    public void approveRequest(ApprovalRequest request) {
        int approvalDays = request.getApprovalDays();
        String approvalPerson = request.getApprovalPerson();

        System.out.println("老板审批通过，批准"+approvalPerson+"请假"+approvalDays+"天");

    }
}
```

发起审批，责任链由客户端负责构建
```java
public class ApprovalProcess {

    public static void main(String[] args) {

        ApprovalRequest request = new ApprovalRequest("张三",4);
        GroupLeaderHandler groupLeaderHandler = new GroupLeaderHandler(3);
        groupLeaderHandler.setNextHandler(new ManagerHandler(5)).setNextHandler(new BossHandler(10));
        groupLeaderHandler.approveRequest(request);
    }
}

```

```
经理审批通过，批准张三请假4天
```

责任链模式使得客户端与处理类解耦，由客户端负责链的创建更具有灵活性