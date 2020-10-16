---
title: 设计模式-策略模式
date: 2020-10-16 11:47:02
tags: 设计模式
---
假设开发了一款旅游景点路线规划系统，这款系统核心就是地图线路规划,支持不同的出行方式，包括自行车，汽车公交及地铁等
```java
public class TourRouteSystem {

    public void planningTourRoute(String mode){

        if("bike".equals(mode)){
            System.out.println("骑行前往...");
        } else if("car".equals(mode)){
            System.out.println("使用汽车驾驶前往...");
        }else if("bus".equals(mode)){
            System.out.println("乘坐公交汽车前往...");
        }else if("subway".equals(mode)){
            System.out.println("乘坐地铁前往...");
        }else{
            System.out.println("步行前往...");
        }
    }
}
```
后期希望修改优化汽车出行的算法，则需要修改其中的逻辑。但是当多个同事开发修改这段代码，会使代码变得难以维护，这时可以引入策略模式。
<!--more-->
## 策略模式

> 策略模式：定义了一组算法，将每个算法都封装起来，并且使它们之间可以互换。
  
策略模式包含三个角色：
- Context（策略上下文）：主要维护了一个抽象策略类的引用实例，用于定义采用的策略
- Strategy(抽象策略类)：定义策略的具体业务方法，上下文调用定义的业务方法。
- ConcreteStrategy（具体策略类）：具体的策略，实现抽象策略类的定义方法。

### 实现
优化起初的路线规划系统，定义抽象策略`TourRouteStrategy`
```java
public interface TourRouteStrategy {
    
    void planningRoute();
}
```

定义一个策略上下文`TourRouteContext`
```java
public class TourRouteContext {
    private TourRouteStrategy strategy;

    public TourRouteContext(TourRouteStrategy tourRouteStrategy){
        this.strategy = tourRouteStrategy;
    }

    public void executeStrategy(){
        strategy.planningRoute();
    }
    
}
```

实现不同的策略
```java
public class BikeStrategy implements TourRouteStrategy {
    @Override
    public void planningRoute() {
        System.out.println("骑行前往...");
    }
}
```

```java
public class BusStrategy implements TourRouteStrategy {
    @Override
    public void planningRoute() {
        System.out.println("乘坐公交汽车前往...");
    }
}
```
```java
public class CarStrategy implements TourRouteStrategy {
    @Override
    public void planningRoute() {
        System.out.println("使用汽车驾驶前往...");
    }
}
```

修改系统代码
```java
public class TourRouteSystem {

    public void planningTourRoute(String mode){

        if("bike".equals(mode)){
            TourRouteContext tourRouteContext = new TourRouteContext(new BikeStrategy());
            tourRouteContext.executeStrategy();
        } else if("car".equals(mode)){
            TourRouteContext tourRouteContext = new TourRouteContext(new CarStrategy());
            tourRouteContext.executeStrategy();
        }else if("bus".equals(mode)){
            TourRouteContext tourRouteContext = new TourRouteContext(new BusStrategy());
            tourRouteContext.executeStrategy();
        }else if("subway".equals(mode)){
            TourRouteContext tourRouteContext = new TourRouteContext(new SubwayStrategy());
            tourRouteContext.executeStrategy();
        }else{
            TourRouteContext tourRouteContext = new TourRouteContext(new WalkStrategy());
            tourRouteContext.executeStrategy();
        }
    }
}

```

规划路线
```java
    public static void main(String[] args) {
        TourRouteSystem system = new TourRouteSystem();
        system.planningTourRoute("car");
        system.planningTourRoute("bus");
        system.planningTourRoute("subway");
    }
```
```
使用汽车驾驶前往...
乘坐公交汽车前往...
乘坐地铁前往...
```


策略模式可以将不同的业务逻辑与算法隔离开，降低了耦合性

