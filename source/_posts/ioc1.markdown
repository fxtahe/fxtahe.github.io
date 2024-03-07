# Spring IOC

Spring 中的核心概念IOC（Inverse Of Control）常被称作IOC容器，它的目标是让开发人员不需要手动创建对象，对Bean的创建维护操作交由Spring控制完成，提供给开发人员开箱即用的舒适体验，因此得名IOC控制反转。

Spring为实现IOC设计了一批复杂且独立的接口和类，例如核心接口 BeanFactory ，Bean的元数据定义 BeanDefinition和Spring应用上下文ApplicationContext等等，而为了更好的理解IOC则需要充分了解这些接口和类设计的底层逻辑，并通过梳理这些概念设计的关系了解IOC 容器的运作原理，可以帮助开发人员更加透明清晰的使用Spring。

> 代码示例中 spring 版本：5.3.31

## 核心容器 BeanFactory

BeanFactory 是IOC容器的核心接口，定义了对于bean的一些基本操作方法，比如通过bean名称获取实例，是否单例，获取bean类型等，而该接口的默认实现类为`DefaultListableBeanFactory`，该类不仅实现了`BeanFactory`接口，同时集成实现了各种接口及抽象类，对BeanFactory核心接口进行了扩展增强
![BeanFactory](https://gitee.com/fxtahe/img/raw/master/spring/DefaultListableBeanFactory.png)

下面简单描述了各接口的作用

* AliasRegistry：通用的别名注册接口
* BeanDefinitionRegistry：注册BeanDefinition （bean元数据类，下文介绍）接口，定义对BeanDefinition的基本操作
* SimpleAliasRegistry: 对AliasRegistry 接口的简单实现类
* SingletonBeanRegistry: 单例Bean注册接口，定义对单例bean的注册获取方法
* **DefaultSingletonBeanRegistry：SingletonBeanRegistry的默认实现类，是bean 注册存储的真实容器类，而面试中常提到的三级缓存便是该类的属性**
* FactoryBeanRegistrySupport: 针对FactoryBean 的处理定义类
* ListableBeanFactory:BeanFactory的扩展接口，可以通过类型或者注解等方式获取符合条件的一组bean
* HierarchicalBeanFactory：对BeanFactory 的层级结构扩展接口，支持父BeanFactory
* **ConfigurableBeanFactory:扩展对BeanFactory 即插即用的配置与特定的访问方式，例如设置父容器，设置加载Bean 的ClassLoader,BeanPostProcessor 等**
* AutowireCapableBeanFactory：主要用于集成其他框架，自动装配充并不由Spring管理生命周期并已存在的实例Bean。
* ConfigurableListableBeanFactory：支持分析和修改BeanDefinition，以及预实例化单例bean
* **AbstractBeanFactory：BeanFactory 的核心实现，完整实现了ConfigurableBeanFactory，及父子容器的功能，并定义了getBeanDefinition（根据beanName 获取BeanDefinition）和createBean（根据beanname,BeanDefinition，构造参数获取实例bean）的模板方法供子类去实现**
* AbstractAutowireCapableBeanFactory：AutowireCapableBeanFactory接口定义操作的实现抽象类，并提供了AbstractBeanFactory createBean方法的默认实现
* **DefaultListableBeanFactory:Spring BeanFactory 的默认实现类，并实现了BeanDefinitionRegistry BeanDefinition操作方法**

BeanFactory 子接口根据不同场景扩展了不同的接口方法，例如HierarchicalBeanFactory 支持父子级的容器，ListableBeanFactory 提供类型获取一组Bean的接口方法（注意：如果子类同时为HierarchicalBeanFactory 类型则不支持从父级获取 Bean 实例），SingletonBeanRegistry 接口定义了对单例Bean的注册方式等。

通过简单了解 Spring BeanFactory 接口设计，会发现IOC容器根据不同的责任分配划分不同的接口，每个接口分别定义了各自对bean或者相关数据的操作方法，遵从单一职责原则，使得Spring在功能扩展中可以更加灵活多变。

### BeanDefinition

上文中提及了`BeanDefinition`，它是spring中 Bean 相关的一个非常重要的概念，顾名思义 `BeanDefinition` 就是提供给Spring容器的各种Bean定义描述信息，主要包括Bean名称，构造参数，作用域与初始方法、销毁方法等。
以一个Spring 中通过xml 文件注册Bean实例方式举例

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="tom" class="io.fxtahe.ioc.bean.User" lazy-init="true"  init-method="init" destroy-method="destroy">
        <property name="id" value="1"/>
        <property name="userName" value="tom"/>
    </bean>
</beans>
```

文件中标识了bean的名称，指定加载类，加载方式为懒加载并指定了bean的初始化方法和销毁方法，这些信息首先都会转换成`BeanDefintion`存储在Spring的容器中，然后通过这些数据实现Bean的实例化创建。BeanDefinition 支持父子关系，子BeanDefinition可以继承和重写父BeanDefnition定义的属性配置，实现配置复用。

当然 BeanDefinition 的配置方式不仅仅局限于xml文件配置，还有其他例如通过注解、Properties文件 、Groovy DSL等等方式，也可以根据需求自定义配置方式。不管是通过哪种方式，其最终目的都是将些可识别的配置信息转换为`BeanDefinition`。
而将配置信息转换为Bean Definition的过程一般需要通过`BeanDefinitionReader` 的组件协助完成。以一个代码示例

```java
    DefaultListableBeanFactory df = new DefaultListableBeanFactory();
    BeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(df);
    beanDefinitionReader.loadBeanDefinitions(new ClassPathResource("beans.xml"));
    User tom = df.getBean("tom", User.class);
```

![BeanDefinitionReader](https://gitee.com/fxtahe/img/raw/master/spring/BeanDefinitionReader.png)

> *Resource 类型的配置文件读取组件一般实现`BeanDefinitionReader`接口，比如XmlBeanDefinitionReader。而AnnotatedBeanDefinitionReader、classPathBeanDefinitionScanner 则用于注解方式的bean配置方式，一般是配合ApplicationContext使用。*

了解上述信息后注册Bean的流程也变得清晰起来，`BeanDefinitionReader`将配置信息转换为 BeanDefinition 然后通过`BeanDefinitionRegistry`组件注册到容器工厂中，在实例化Bean时从容器中根据名称获取BeanDenifition，针对各种属性配置去创建实例Bean。

![BeanDefinition处理流程](https://gitee.com/fxtahe/img/raw/master/spring/bddr.png)

### Bean 的生命周期回调

Spring 针对Bean的定义、创建、初始化与销毁等阶段提供了一系列的生命周期接口，Spring通过实现这些接口完成了对Bean的生命周期管理，开发人员也可以通过实现这些接口完成对Bean的定制化处理，下面介绍下Spring针对 Bean 的生命周期设计的接口扩展。

#### Bean 的创建与销毁

`InitializingBean` 接口提供了一个初始化回调方法，该方法在bean在容器初始化阶段被执行调用。

```java
    void afterPropertiesSet() throws Exception;
```

而在Bean的销毁阶段Spring 同样提供了回调接口`DisposableBean`，允许在Bean的销毁阶段执行销毁逻辑

```java
 void destroy() throws Exception;
```

Spring在bean的定义时可以指定`init-method`和`destroy-method`用于Bean的初始化和销毁逻辑，而且同样支持添加了`@PostConstruct`和`@PreDestroy`注解的方法。而这些方法的执行遵循以下顺序

* 1.添加了`@PostConstruct`注解的方法
* 2.`InitializingBean#afterPropertiesSet`初始化回调方法
* 3.定制化的`init-method`
* 4.添加了`@PreDestroy`注解的方法
* 5.`PreDestroy#destroy`销毁回调方法
* 6.定制化的`destroy-method`

#### Aware接口

`Aware`是一个顶级的标记接，它标识容器需要通过回调方法设置指定的对象，一般为框架定义的对象实例。Spring提供了一系列的子接口获取不同的内容，以下为常见的`Aware`接口

* `BeanNameAware`：设置beanName。
* `BeanClassLoaderAware`:获取实例化当前Bean的`ClassLoader`
* `BeanFactoryAware`:获取当前的Bean工厂
* `EnvironmentAware`:获取组件运行的`Environment`实例
* `ApplicationContextAware`:获取`ApplicationContext`实例

通过实现上述接口IOC容器便会执行相应的回调方法，获取容器中的各种组件，例如很多项目中会通过实现`ApplicationContextAware`接口去实现获取容器实例的静态工具类。

#### BeanPostProcessor

BeanPostprocessor是spring 提供的一个很重要Bean的定制化扩展接口，不仅Spring内部也提供了许多的重要的`BeanPostProcessor`以实现对Bean 管理与扩展，例如依赖注入与AOP实现等，而且开发人员可以通过实现该接口提供定制化的实例化或者依赖处理逻辑，同时Spring在此接口的基础上设计了不同的生命周期回调接口，针对Bean在不同生命周期阶段都可以实现定制化的设计开发。
该接口主要包含了两个回调方法

```java
public interface BeanPostProcessor {

 @Nullable
 default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  return bean;
 }

 @Nullable
 default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
  return bean;
 }

}
```

这两个方法在 Bean 初始化回调前后执行，一般特指InitializingBean#afterPropertiesSet 或者 定制的 init-method 方法。

Spring在`BeanPostProcessor`接口基础上根据Bean的生命周期添加了其他扩展子接口,下面分别介绍下

![image](https://gitee.com/fxtahe/img/raw/master/spring/MergedBeanDefinitionPostProcessor.png)

> SmartInstantiationAwareBeanPostProcessor 是Spring框架内部的接口继承类，主要用于对Bean 的类型推断、构造器选择，更多的是用在Spring AOP 中。

##### MergedBeanDefinitionPostProcessor

该接口是IOC容器在处理`BeanDefinition`阶段提供的后置处理器，开发人员可以通过实现其回调方法进行对`BeanDefinition`进行修改或者在实例化前对元数据进行解析缓存预处理。该接口也提供了两个扩展方法

```java
 void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
```

处理`BeanDefinition` Bean 元数据后置处理方法，通常用于在Bean实例化前对Bean 元数据进行解析缓存，例如 `AutowiredAnnotationBeanPostProcessor`与`CommonAnnotationBeanPostProcessor`在对添加`@Autowired`、`@Value` 与`@Resource`等注解的属性解析缓存，在填充数据时直接获取缓存信息即可。

```java
 default void resetBeanDefinition(String beanName) {}
```

 `BeanDefinition`在被重置时执行的回调，可用去清理元数据缓存

##### InstantiationAwareBeanPostProcessor
>
> 注意：Instantiation（实例化），指bean的创建过程。Initialization （初始化），指bean的初始化过程，这时bean已创建但未完全装配完成。

Bean实例化后置处理器，该接口在`BeanPostProcessor`的基础上额外扩展了三个回调方法，分别在Bean的实例化前后与属性填充阶段执行。

```java
 default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
  return null;
 }
```

`postProcessBeforeInstantiation`在 Bean 执行实例化前回调，可以通过该方法返回一个代理对象替换真实要获取的对象，例如Spring AOP组件类`AbstractAutoProxyCreator`实现了该方法进行代理。

```java
 default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
  return true;
 }
```

`postProcessAfterInstantiation`在Bean 已创建但是属性未完成数据填充阶段执行回调。

```java
 default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
   throws BeansException {
  return null;
 }
```

`postProcessProperties`可实现对Bean的属性填充或者替换修改，Spring内部实现类`AutowiredAnnotationBeanPostProcessor`与`CommonAnnotationBeanPostProcessor` 对添加了 `@Autowired`、`@Value` 与`@Resource`等注解的属性进行数据填充。

> *该接口其实还有一个`postProcessPropertyValues`回调方法，其作用与`postProcessProperties`方法一致，并且被标注未废弃。所以一般推荐使用`postProcessProperties`方法*

##### DestructionAwareBeanPostProcessor

与`InstantiationAwareBeanPostProcessor`实例化相对应，`DestructionAwareBeanPostProcessor`是 Bean 在销毁阶段的后置处理器接口，用于在Bean的销毁阶段回调执行。

```java
 void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
```

`postProcessBeforeDestruction` Bean在销毁阶段进行回调，spring中对`@PreDestroy`注解方法的回调就是通过子类`CommonAnnotationBeanPostProcessor`实现。

```java
 default boolean requiresDestruction(Object bean) {
  return true;
 }
```

`requiresDestruction` 判断给定 Bean 实例是否需要执行销毁后置回调方法。

#### 总结

不同的接口与回调在Bean的生命周期中有着不同作用，Spring容器通过实现部分内部接口对Bean完成了生命周期管理，而开发人员也可以实现接口或者相关的生命周期回调方法对容器进行扩展。
相关回调的执行链路如下图所示
![生命周期](https://gitee.com/fxtahe/img/raw/master/spring/lifecycle.png)

## Bean 的诞生
>
> IOC容器普遍意义下指的是单例Bean singletonBean，对于原型或者其他作用域的Bean的创建和使用不做介绍，源码解析中删减了不太重要的部分。

在上文代码示例中

```
User tom = df.getBean("tom", User.class);
```

通过`getBean`方法即可轻松获取指定名称和类型的实例，开发人员最简单的想法就是**通过反射创建实例并放到容器中**，但是这看似简单的操作下却充满了复杂的场景与代码逻辑，例如Bean元数据的解析缓存、Bean生命周期回调、依赖注入`DI`、AOP代理等，这一系列的工作在IOC容器是如何运作？接下来让我们透过源码一起探索门后的秘密。

![secret](https://gitee.com/fxtahe/img/raw/master/spring/pawel-czerwinski-lL2pdNaHqaQ-unsplash.jpg)

### Bean创建的核心流程

Spring中创建Bean的逻辑错综复杂，对于那些旁枝末叶简单了解即可，点到即止，将重心放到核心流程，对主要的脉络关系进行梳理，留下一个清晰的印象。假如对每个分支逻辑都事无巨细的研究分析，可能会陷入到Spring那庞大的逻辑沼泽中。

#### AbstractBeanFactory#doGetBean

`DefaultBeanFactory#getBean`方法调用的是抽象父类 `AbstractBeanFactory#doGetBean`方法，代码如下

```java
 protected <T> T doGetBean(
   String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
   throws BeansException {
        // bean 名称转换，通过别名获取规范名称
        // 删除 FactoryBean bean 名称 '&' 前缀
  String beanName = transformedBeanName(name);
  Object beanInstance;

  // 检查单例池中是否已存在，可获取提前暴露的实例（循环引用）
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
            // 假如为FactoryBean类型，则通过 getObject 方法获取其真实类型
   beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }

  else {
   // 检查原型实例是否已经在创建
   if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
   }

   // 假如bean definition 在父容器中，从父容器中获取实例
   BeanFactory parentBeanFactory = getParentBeanFactory();
   if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                ...
   }

   if (!typeCheckOnly) {
    markBeanAsCreated(beanName);
   }

   try {
    // 获取BeanDefinition
    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    checkMergedBeanDefinition(mbd, beanName, args);

    // 保证优先初始化当前实例通过 dependsOn 指定的依赖实例
    String[] dependsOn = mbd.getDependsOn();
    if (dependsOn != null) {
     for (String dep : dependsOn) {
         // 禁止循环指定 dependsOn
      if (isDependent(beanName, dep)) {
       throw new BeanCreationException(mbd.getResourceDescription(), beanName,
         "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
      }
      // 缓存依赖关系
      registerDependentBean(dep, beanName);
      try {
          // 优先创建依赖实例
       getBean(dep);
      }
      catch (NoSuchBeanDefinitionException ex) {
       throw new BeanCreationException(mbd.getResourceDescription(), beanName,
         "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
      }
     }
    }

    // 创建单例bean
    if (mbd.isSingleton()) {
     // 调用ObjectFactory匿名类getObject方法创建实例，并缓存在单例池中
     sharedInstance = getSingleton(beanName, () -> {
      try {
          // 创建bean实例，抽象模板方法
       return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
       // 销毁 bean 实例
       destroySingleton(beanName);
       throw ex;
      }
     });
     // 假如为FactoryBean类型，则通过 getObject 方法获取其真实类型
     beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
                // 原型实例
    else if (mbd.isPrototype()) {...}
                // 其他作用域实例
    else {...}
   }
   catch (BeansException ex) {
                ...
   }
   finally {
    beanCreation.end();
   }
  }
        // 根据类型进行适配转换
  return adaptBeanInstance(name, beanInstance, requiredType);
 }
```

> FactoryBean 是创建Bean的工厂Bean，而BeanFactory 是Bean的容器，两者有本质上的区别。FactoryBean主要用于那些创建过程需要复杂的配置属性或者涉及其他组件bean构建的实例bean。

简要概述下方法逻辑

* 首先检查单例池是否已经缓存实例
* 假如存在父容器并且bean definition 存在于父容器中，从父容器中获取实例
* 优先初始化当前实例指定的依赖实例
* 创建实例并缓存到单例池中

最终创建Bean的方法`createBean`为AbstractBeanFactory定义的抽象模板方法，`AbstractAutowireCapableBeanFactory`继承实现了该方法。

#### AbstractAutowireCapableBeanFactory#createBean

该方法用于创建实例，填充属性数据并支持对BeanPostProcessors的扩展。

```java
 protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
   throws BeanCreationException {

  RootBeanDefinition mbdToUse = mbd;

  // 获取 bean Class
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
  if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
   mbdToUse = new RootBeanDefinition(mbd);
   mbdToUse.setBeanClass(resolvedClass);
  }

  // 预处理覆盖方法
  try {
   mbdToUse.prepareMethodOverrides();
  }
  catch (BeanDefinitionValidationException ex) {...}

  try {
   // 扩展：InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation
   Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
   if (bean != null) {
    return bean;
   }
  }
  catch (Throwable ex) {...}

  try {
      // 真实创建实例bean
   Object beanInstance = doCreateBean(beanName, mbdToUse, args);
   return beanInstance;
  }
  ...
 }
```

在`createBean`方法触发了上文中介绍的扩展方法`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`，实现该方法返回实例直接返回进行覆盖实例。 与 getBean 方法类似，创建实例通过 `doCreateBean`方法实现。

#### AbstractAutowireCapableBeanFactory#doCreateBean

```java
    protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
            throws BeanCreationException {

        // Instantiate the bean.
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        }
        if (instanceWrapper == null) {
            // 选择合适的构造器创建实例，返回 BeanWrapper 包装类
            instanceWrapper = createBeanInstance(beanName, mbd, args);
        }
        Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            mbd.resolvedTargetType = beanType;
        }

        synchronized (mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                    // 扩展点：应用 MergedBeanDefinitionPostProcessor 扩展方法
                    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable ex) {...}
                mbd.postProcessed = true;
            }
        }

        // 循环依赖 提前暴露对象实例
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            //添加到单例工厂bean 缓存中
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        }

        // 初始化 bean 实例
        Object exposedObject = bean;
        try {
            // 填充 bean 实例属性
            populateBean(beanName, mbd, instanceWrapper);
            //执行bean的初始化方法
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        } catch (Throwable ex) {...}
        
        // 检查循环依赖后实例改变场景问题
        if (earlySingletonExposure) {
            Object earlySingletonReference = getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                    //获取依赖于当前实例的bean
                    String[] dependentBeans = getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                    for (String dependentBean : dependentBeans) {
                        // 依赖于当前实例的其他实例还未创建结束添加到集合中
                        if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }
                    // 存在循环依赖 但是 bean 发生了改变，违反单例特性导致报错
                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName,...);
                    }
                }
            }
        }

        
        try {
            // 实例存在销毁方法则注册回调
            registerDisposableBeanIfNecessary(beanName, bean, mbd);
        } catch (BeanDefinitionValidationException ex) {...}

        return exposedObject;
    }
```

> BeanWrapper ：Bean的包装类，它提供了对Java bean的分析操作能力，诸如属性设置修改，获取属性类型信息等
> 循环依赖：A 依赖 B时，B又依赖A 出现循环依赖。

该方法是IOC容器创建实例的核心方法，主要包含对象创建、属性填充以及初始化回调等 比如`applyMergedBeanDefinitionPostProcessors` 方法即是`MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition` 处理，同时还针对**循环依赖**这一特殊场景进行了特殊处理。

#### AbstractAutowireCapableBeanFactory#populateBean

```java
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
        
        // 扩展点：InstantiationAwareBeanPostProcessors#postProcessAfterInstantiation 
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
                if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }

        PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

        int resolvedAutowireMode = mbd.getResolvedAutowireMode();
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
            // 根据名称注入属性
            if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
                autowireByName(beanName, mbd, bw, newPvs);
            }
            // 根据类型注入属性
            if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
                autowireByType(beanName, mbd, bw, newPvs);
            }
            pvs = newPvs;
        }
        // 检查是否存在 InstantiationAwareBeanPostProcessor
        boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
        boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

        PropertyDescriptor[] filteredPds = null;
        if (hasInstAwareBpps) {
            if (pvs == null) {
                pvs = mbd.getPropertyValues();
            }
            for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
                // 扩展点：InstantiationAwareBeanPostProcessor#postProcessProperties
                PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    // 扩展点：InstantiationAwareBeanPostProcessors#postProcessPropertyValues （标记废弃）
                    pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }
        if (needsDepCheck) {
            if (filteredPds == null) {
                filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }

        if (pvs != null) {
            //设置属性
            applyPropertyValues(beanName, mbd, bw, pvs);
        }
    }
```

该方法的作用就是填充实例属性，比如通过名称或者类型进行注入，执行`InstantiationAwareBeanPostProcessors`的`postProcessAfterInstantiation`与`postProcessProperties`回调方法。 正如上文中介绍的Spring内部实现类`AutowiredAnnotationBeanPostProcessor`与`CommonAnnotationBeanPostProcessor` 通过`postProcessProperties`回调方法实现了 `@Autowired`、`@Value` 与`@Resource`等注解的属性的数据填充能力

#### AbstractAutowireCapableBeanFactory#initializeBean

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                invokeAwareMethods(beanName, bean);
                return null;
            }, getAccessControlContext());
        } else {
            // 扩展点：Aware接口回调
            // 1.BeanNameAware#setBeanName 设置beanName
            // 2.BeanClassLoaderAware#setBeanClassLoader 设置 classLoader
            // 3.BeanFactory#setBeanFactory 设置BeanFactory
            invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            // 扩展点：BeanPostProcessor#postProcessBeforeInitialization
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        }

        try {
            // 扩展点：初始化回调
            // 1.InitializingBean#afterPropertiesSet
            // 2.定制化 init-method
            invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable ex) {
            throw new BeanCreationException(
                    (mbd != null ? mbd.getResourceDescription() : null),
                    beanName, "Invocation of init method failed", ex);
        }
        if (mbd == null || !mbd.isSynthetic()) {
            // 扩展点：BeanPostProcessor#postProcessAfterInitialization
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
```

该方法主要是执行bean实例的生命周期回调方法，完成bean的初始化：

* 1.aware相关接口
* 2.BeanPostProcessor#postProcessBeforeInitialization 前置回调
* 3.初始化生命周期回调
* 4.BeanPostProcessor#postProcessAfterInitialization


