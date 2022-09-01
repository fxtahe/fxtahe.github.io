---
title: 设计模式-代理模式
date: 2020-09-16 22:51:19
categories: 
- 设计模式
---

## 代理模式
  
在某些情况下，一个客户类不想或者不能直接引用一个委托对象，而代理类对象可以在客户类和委托对象之间起到中介的作用，其特征是代理类和委托类实现相同的接口。  
同时通过代理对象可以扩展委托对象的功能，而不用直接去修改委托对象，比如通过代理模式为项目添加日志，缓存功能等。  
代理模式在java有着广泛的应用，比如常见的Spring AOP就使用了动态代理
<!--more-->

## 实现
代理模式主要包含三个角色，即抽象主题角色(Subject)、委托类角色(被代理角色，Proxied)以及代理类角色(Proxy)

抽象主题角色可以为抽象类或者接口，委托类和代理类继承或者实现抽象主题，客户类访问代理类继而访问委托类。通过代理类增强委托类的功能。

代理模式又分为**静态代理**与**动态代理**

### 静态代理

静态就是在程序运行前就已经存在代理类的字节码文件，代理类和委托类的关系在运行前就已经确定了。

定义一个抽象主题
```java
public interface BusinessService {

    void doBusiness();

}
```

委托类
```java
public class BusinessServiceImpl implements BusinessService{
    @Override
    public void doBusiness() {
        System.out.println("do Business");
    }
}
```

定义代理类

```java
public class BusinessServiceProxy implements BusinessService {

    private BusinessService businessService;

    public BusinessServiceProxy(BusinessService businessService) {
        this.businessService = businessService;
    }

    @Override
    public void doBusiness() {
        System.out.println("before do Business");
        businessService.doBusiness();
        System.out.println("after do Business");
    }
}
```
客户端静态代理调用
```java
public static void main(String[] args) {
    BusinessService businessService = new BusinessServiceImpl();
    BusinessServiceProxy businessServiceProxy = new BusinessServiceProxy(businessService);
    businessServiceProxy.doBusiness();
}
```
```
before do Business
do business
after do Business
```
静态代理可以为委托类扩展功能符合开闭原则，但是每个委托类都需要创建一个代理类，接口调整过程中代理类也需要做出相应修改

### 动态代理
动态代理是通过反射机制进行动态确定代理类和委托类关系，通过运行时动态创建代理类去对委托类进行代理增强扩展。Java可以通过JDK与CGLIB的方式实现动态代理


#### JDK动态代理

创建动态代理
```java

public class DynamicBusinessServiceProxy implements InvocationHandler {

    private Object object;

    public DynamicBusinessServiceProxy(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before do business");
        Object result = method.invoke(object, args);
        System.out.println("after do business");
        return result;
    }
}

```

动态创建代理类
```java
    public static void main(String[] args) {
        BusinessService businessService = new BusinessServiceImpl();
        BusinessService dynamicBusinessService = (BusinessService) Proxy.newProxyInstance(
                BusinessService.class.getClassLoader(),
                new Class[]{BusinessService.class},
                new DynamicBusinessServiceProxy(businessService));
        dynamicBusinessService.doBusiness();
    }
```

- InvocationHandler: JDK动态代理处理类必须实现的接口
- Proxy.newProxyInstance() 代理创建接收三个参数
    -  ClassLoader loader 目标类的类加载器，加载目标类字节码
    -  Class<?>[] interfaces 目标类实现的接口类
    -  InvocationHandler h 代理处理类，用于代理执行目标类



#### CGLIB 动态代理
JDK动态代理要求委托类必须通过接口的方式去定义委托类的方法，对于没有实现接口的类则无能为力，而CGLIB则更加的灵活。  

**CGLIB其原理是通过字节码技术为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。**

定义一个被代理类
```
public class Person {

    public void eat(){
        System.out.println("eat...");
    }
    
}
```

创建CGLIB代理拦截器
```
public class CglibPersonProxy implements MethodInterceptor {

    public Object getProxy(Class<?> clz){

        Enhancer enhancer = new Enhancer();
        //设置被代理类
        enhancer.setSuperclass(clz);
        //执行回调
        enhancer.setCallback(this);

        return enhancer.create();

    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before...");
        //执行父类方法
        Object result = methodProxy.invokeSuper(o, objects);

        System.out.println("after...");
        return result;
    }
}

```

客户端执行

```java
    public static void main(String[] args) {
        
        CglibPersonProxy cglibPersonProxy = new CglibPersonProxy();

        Person personProxy = (Person) cglibPersonProxy.getProxy(Person.class);

        personProxy.eat();

    }
```

CGLIB 动态代理相比与JDK方式更加的灵活，Spring中的AOP即采用了该种方式，完成日志和鉴权的处理。