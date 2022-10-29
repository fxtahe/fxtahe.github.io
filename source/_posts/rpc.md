---
title: 如何实现一个Rpc 框架
date: 2022-10-29 20:47:02
tags: rpc
---

先贴一下代码地址 <https://github.com/fxtahe/fx-rpc> 代码中参考了dubbo 与sofa-rpc的一些实现逻辑，大部分功能都没有经过充分测试，仅仅是作为一个学习的玩具项目。

## 关于rpc

远程服务调用（Remote Procedure Call，RPC），就是通过网络请求远程服务，而不需要了解底层网络技术的协议。
举个例子，我们需要在订单系统中查询商品信息，我们在电商系统中发布一个接口服务，在订单系统中定义相同的接口去调用这个方法，就可以获取商品服务的信息。

rpc的目的就是让计算机能够跟调用本地方法一样去调用远程方法。
如果想了解更多的可以看下周志明大佬在《凤凰架构》中[远程服务调用](https://icyfenix.cn/architect-perspective/general-architecture/api-style/rpc.html)章节的描述，比较清晰明了的介绍了rpc并且解释了rpc相关的争论。

<!-- more -->
### rpc调用过程

[![远程调用](https://s1.ax1x.com/2022/09/28/xeowan.png)](https://imgse.com/i/xeowan)

1）调用本地服务
2）client stub 封装方法签名与参数，并进行序列化，通过socket发送到远端
3）远程服务socket接收请求，server stub反序列化数据，并根据参数调用方法
4）server stub 封装返回值并进行序列化，通过socket响应客户端
5）本地socket接收响应，client stub 反序列化响应数据
6）完成一次Rpc调用

通过上面的流程可以看出rpc调用需要涉及到**动态代理、序列化、通信协议、网络传输**等技术。
而rpc框架封装了rpc的调用实现，并且需要提供额外的能力，比如**服务发现、负载均衡、异常重试、健康检查、服务路由**等功能。
开源界有很多的优秀的rpc框架项目，比如阿里dubbo、谷歌grcp、百度brpc、蚂蚁sofa-rpc等。

## 实现一个Rpc框架

### 架构与功能设计

[![架构设计](https://s1.ax1x.com/2022/09/28/xmPfPK.png)](https://imgse.com/i/xmPfPK)

架构上使用常见的注册中心服务消费模型，服务提供者将服务注册到注册中心，然后客户端通过注册中心订阅服务，调用远程服务完成一次rpc调用。 当服务提供方发生变更会注册中心通知服务订阅者，刷新服务列表。

框架的功能设计与调用流程，如下图

[![功能设](https://s1.ax1x.com/2022/09/30/xuf55t.md.png)](https://imgse.com/i/xuf55t)

框架包含以下功能点

*   服务注册发现
*   服务路由与负载均衡
*   异常重试
*   服务超时
*   异步调用
*   序列化与反序列化
*   协议编码与解码
*   网络传输

下面逐个章节介绍各个功能的实现与设计，讲解各个功能模块的设计实现

### 服务注册发现

服务注册发现模块是一个相对比较独立的模块，该功能模块默认实现了Zookeeper注册中心作为功能模块的核心，zookeeper提供了基本的服务注册订阅通知与服务的健康检查能力。并在此基础上封装提供了客户端缓存、故障转移、重连恢复等能力。

[![注册中心.png](https://s1.ax1x.com/2022/10/06/x1MRit.md.png)](https://imgse.com/i/x1MRit)

*   服务注册：注册本地服务到zookeeper并进行客户端缓存`registers`。
*   服务发现与订阅：拉取远程服务，如果订阅服务则进行客户端缓存向zookeeper注册监听，并异步刷新到本地文件中，当服务变更异步刷新备份文件与`subscribers`缓存。
*   客户端缓存：缓存本地服务注册与远端服务订阅信息，当拉取服务信息时订阅该服务则直接从缓存获取服务信息，减少与zookeeper的io交互，通过对zookeeper的服务节点监听保持缓存一致性。假如未订阅该服务则直接从zookeeper 拉取服务，结合了服务信息推送与拉取两种方式。
*   重连恢复：当zookeeper宕机后重连，则将客户端缓存的订阅信息与本地服务注册信息进行重新注册订阅。
*   故障转移：当开启故障转移后会优先加载备份文件中的服务信息到缓存中，可以在注册中心连接失败的情况中通过上次的文件缓存服务信息取请求远程服务，尽量避免因注册中心连接失败导致客户端不可用。

### 动态代理

上面介绍rpc时，阐述了**rpc的目的就是让计算机能够跟调用本地方法一样去调用远程方法。**
以上面商品系统与订单系统交互的例子说明下。

*   定义商品查询接口

```java
public class ProductService{
    Product queryProduct(String productId); 
}
```

*   商品系统发布服务

```java
public class ProductServiceImpl implements ProductService{
    Product queryProduct(String productId){
        //query
        return product;
    }
}
```

*   订单系统引入相同的接口，并调用

```java
    ProductService productService  = Consumer.getRefer(ProductService.class);
    Product product = productService.queryProduct(1024);
```

客户端调用接口屏蔽底层的数据封装，网络传输以及响应处理。实现这一切就需要通过动态代理完成，在进行rpc服务调用时，会为接口创建一个代理类，代理类方法封装了rpc请求的各种要素，这样我们就可以像调用本地方法一样调用远程服务。

通过java 默认动态代理`Proxy`与`InvocationHandler`类实现，讲解下实现
```
        ProductService productService = Proxy.newProxyInstance(ProductService.class.getClassLoader(), new Class[]{ProductService.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) {
                //封装数据 网络传输
                Client client = new Clinet(method,args);
                return client.rpc();
            }
        });
```

上面`Proxy.newProxyInstance`会为接口动态生成一个代理类，代理方法真实的执行逻辑则是由`InvocationHandler invoke`实现。大致的代理类如下，希望查看完整的class文件信息，可以通过`        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");`输出本地

```java
public final class $Proxy0 extends Proxy implements ProductService{
    
    private Method m1;
    private InvocationHandler h;
        
    public $Proxy0(InvocationHandler h) {
        super(var1);
    }
    public Product queryProduct(String productId){
        return h.invoke(this,m1,new Object[]{productId});
    }
}
```

rpc调用过程中的数据传输，网络通信都可以封装到 `InvocationHandler`类中，通过动态类屏蔽了底层的实现。

当然除了java默认实现，还有许多三方框架都可以实现动态代理，比如 Javassist、CgliB、ByteBuddy等，其实现原理都是通过操作字节码构建动态类最终完成底层操作的屏蔽。源码框架实现目前仅实现了jdk与javassist 两种动态代理方式。

### 负载均衡
当我们的服务提供者无法满足消费者的访问量，便会部署多个服务节点形成集群，然后通过负载均衡算法选择服务节点，共同分摊请求压力，保证服务的高可用。
服务新版本发布时，我们希望先通过少量的请求去做验证，在验证没有问题后逐渐提高服务处理请求的权重，完成新服务验证上线，这个过程被称为流量切分。
[![负载均衡.png](https://s1.ax1x.com/2022/10/11/xtXEKx.png)](https://imgse.com/i/xtXEKx)

常见的负载均衡算法
- 轮询：将请求循环顺序的分发给服务节点
- hash：根据请求标识hash，比如客户端ip、用户标识等，然后对服务节点数取模获取服务节点
- 随机：完全随机选取服务节点
- 加权: 为服务节点配置权重，在随机或者轮询基础上权重越大越容易被选中，上面提到的流量切分可以通过该算法实现。

以上都是基本的负载均衡算法，但是简单的轮询或者随机分发请求，而各个服务器状态不一致导致器处理请求能力不同，所以在负载均衡时需要考虑服务端状态，比如服务端当前连接数，服务响应耗时，响应异常，服务注册时间，健康状态等。通过这些指标对服务节点计算服务权重然后再分发请求，实现自适应的负载均衡，从而提高集群的可用性。

以服务注册时间为例，当服务注册时间过短，服务可能处于服务预热阶段比如缓存加载等，导致无法处理大量请求，这时就需要减少对给服务节点的请求分发，将请求分配给那些已经平稳运行的节点。
假设服务权重标准时100，服务注册时间小于10分钟则指定权重25，小于20分钟的权重50，以此类推超过40分钟则权重为100，然后根据权重在做随机分配。

```java
public class WeightRandomLoadBalance implements LoadBalance{

    @Override
    public ServiceInstance select(List<ServiceInstance> serviceInstances) {
        int totalWeight = 0;
        int[] weights = new int[serviceInstances.size()];
        for(int i=0;i<serviceInstances.size();i++) {
            ServiceInstance serviceInstance = serviceInstances.get(i);
            //根据注册时间计算权重
            int weight = getWeight(serviceInstance);
            totalWeight +=weight;
            weights[i] = totalWeight;
        }
        ThreadLocalRandom threadLocalRandom = ThreadLocalRandom.current();
        int index = threadLocalRandom.nextInt(totalWeight);
        //逐级比较，当大于当前随机值则选中
        for(int i=0;i<weights.length;i++){
            int weight = weights[i];
            if(weight>index){
                return serviceInstances.get(i);
            }
        }
        return serviceInstances.get(ThreadLocalRandom.current().nextInt(serviceInstances.size()));
    }
    
    private static final int upThresholdTime = 10*60*1000;
    private static final int defaultThresholdWeight = 25;
    
    public int getWeight(ServiceInstance serviceInstance){
        long registrationTime = serviceInstance.getRegistrationTime();
        long currentTime = System.currentTimeMillis();
        long upTime = currentTime - registrationTime;
        if(upTime < 0){
            return 1;
        }else{
            //计算注册时间的权重
            return Math.min((int)(upTime / upThresholdTime) + 1, 4) * defaultThresholdWeight;
        }
    }
}
```

### 服务路由
服务路由就是根据指定的规则，比如指定服务ip，指定服务请求参数等，路由请求到指定的服务提供者，那些不符合规则的服务会被屏蔽。比较典型的使用场景就是灰度发布。
灰度发布是一种平滑的版本发布方式，版本发布时先部署一台新版本服务测试人员对其进行测试，测试结果正常后切入少量的用户流量去做验证。观察服务状态，服务正常运行一段时间后逐个将剩余的服务切换为新版本服务，最终完成版本发布。
[![灰度发布.png](https://s1.ax1x.com/2022/10/11/xNUQt1.png)](https://imgse.com/i/xNUQt1)

伪代码模拟下路由实现
```java
        //注册中心获取所有服务
        List<Service> services =  registry.getService("hello-service");
        //获取参数中的版本号
        String version = param.get("version");
        //路由到指定版本号的服务
        List<Service> routeServices = services.stream().filter(service -> version.equals(service.getVersion)).collect(Collectors.toList());

```

服务路由还可配合负载均衡实现服务预热，流量切分等，根据定制的路由规则可以解决一些特定的场景需求，也是服务路由的功能目标


### 序列化
> 序列化 (Serialization)是将对象的状态信息转换为可以存储或传输的二进制字节数据的过程。
> 反序列化(Deserialization)是将二进制字节数据重新构建转换为对象的过程。

rpc请求是通过网络传输完成的，所以必须将请求数据转换为二进制数据，这就涉及到序列化。而服务端将二进制请求数据转换为请求对象就需要进行反序列化。就像你网购买了一个拼图，店家会把拼图打乱装进包裹，而你收到拼图会把它还原，打乱与还原拼图的过程就类似于序列化反序列化。

[!序列化.png](https://s1.ax1x.com/2022/10/17/xDmuxe.png)](https://imgse.com/i/xDmuxe)

常见的序列化方式有
- jdk序列化:java 默认提供的序列化方式，会将对象类元数据，数据类型序列化，性能较低
- json: json是一种key-value 的数据交换格式，没有数据类型，序列化方式具备优秀的跨语言特性，使用广泛
- hessian:动态类型、二进制、紧凑的，并且可跨语言移植的一种序列化框架。Hessian 协议要比 JDK、JSON 更加紧凑，性能上要比 JDK、JSON 序列化高效很多，而且生成的字节数也更小
- protobuf:  protobuf是谷歌开发的一款无关平台，无关语言，可扩展，轻量级高效的序列化协议，性能较好。

不同的序列化方式其数据转换过程存在差异，所以其内存占用，转换性能各不相同，而且还要其跨平台性、使用复杂度等各不相同，所以选择序列化方式要综合考虑，比如json虽然序列化性能不高，但是具备有优秀的跨平台特性，所以在微服务开发中广泛使用。

#### protobuf 序列化
protobuf 因为其完全基于二进制，序列化后体积小传输效率高，同时序列化性能好，支持java、go、c++、python等语言，在rpc服务通讯中具备更加优异的表现。  
使用protobuf 一般要经过下面三个步骤
- 定义一个`.proto`文件
- 使用`protobuf compiler`编译`.protobuf`文件，生成java文件
- 使用Java protobuf api 序列化数据

1.首先定义一个`user.proto`文件 
```
//指定proto语法 proto2 proto3
syntax = "proto3"; 
//未指定 java_package时 默认的包名
package example; 
//是否生成多个文件
option java_multiple_files = true;
//指定包名
option java_package= "com.example.protos"; 

//消息体
message User{
  //类型 属性名 = 二进制字段标识
  int64 userId = 1;
  string userName = 2;
  int32 age = 3;
  bool female = 4;
}
```
一个proto文件主要包含配置信息与消息体信息。protobuf针对不同的语言定义了一套属性字段，用于在不同语言的序列化与反序列化。具体对应关系可以查看文档[https://developers.google.com/protocol-buffers/docs/proto3#scalar
](https://developers.google.com/protocol-buffers/docs/proto3#scalar/)

2.使用编译器编译
添加依赖
maven
```xml
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>3.21.7</version>
        </dependency>
```
gradle
```
implementation 'com.google.protobuf:protobuf-java:3.21.7'
```

下载编译器protoc-21.7-win64.zip压缩文件，下载地址 https://github.com/protocolbuffers/protobuf/releases。解压后/bin目录会看到一个protoc.exe文件，将创建的`user.proto`文件放置到同一目录 然后在cmd窗口执行命令，生成java 文件
```cmd
protoc ./user.proto --java_out=./
```

3.使用protobuf进行序列化与反序列化
```
        User user = User.newBuilder().setUserId(1).setAge(17).setFemale(false).setUserName("fxtahe").build();
        byte[] bytes = user.toByteArray();
        User user1 = User.parseFrom(bytes);
        System.out.println(user1);
```


简单了解过protobuf 后会发现，其在java中使用起来还是需要一定的成本，每个序列化类都需要指定IDL文件，还需要通过编译生成java类。基于以上问题 `protostuff`的序列化框架完善了在java中的实现，它封装了`protobuf`并且不需要定义IDL文件，可以直接对java对象进行序列化/反序列化,详细文档查看下[protostuff文档](https://github.com/protostuff/protostuff)




### 协议定制
网络数据传输中需要制定一些数据传输规范，包括数据格式，数据内容传输等。比如一个中国人要与一个德国人沟通，双方都使用自己的语言交流，那肯定是无法完成信息的交换，这也是为什么需要定制网络协议。
常见的网络协议有TCP、HTTP、UDP等，以最常用的HTTP 协议为例介绍下。

#### HTTP协议
http 协议传输是一次请求与响应的交流，协议规定了请求响应的格式。

```
起始行 (start line)
头信息 (headers)

主体(entity body)
```
它由三部分组成起始行、头信息与主体。
起始行区分请求与响应，头信息是key:value的格式，可以存在多个头信息，头信息与数据主体间存在一个空行，数据主体可以为空

**请求**
```
GET /index.html HTTP/1.1
Host: www.example.com
```
起始行包含三个信息，以`空格`分隔
- GET 方法。用于说明想要服务器执行的操作，其他操作符还包括POST\DELETE\PUT等
- /index.html 资源的路径。这里指向服务器上的index.html文件。
- HTTP/1.1 协议的版本。HTTP第一个广泛使用的版本是1.0，当前版本为1.1。

头信息`Host`字段标识请求的服务器地址，请求没有传输数据，仅是获取页面信息，所以请求体为空

**响应**
```
HTTP/1.1 200 OK
Content-type: text/plain
Content-length: 12

Hello World!
```
起始行同样包含三个信息，以`空格`分隔
- HTTP/1.1 协议的版本
- 200 响应状态码。其他常见状态码还包括404，302，500等
- OK 状态描述

头信息`Content-type`标识响应资源的类型，`Content-length`标识响应数据的长度  ,最后就是一段文本式响应主体`Hello World!`

上面仅是简单介绍了下HTTP协议，截取了Vamei大神文章的部分内容，详情看下[协议森林15 先生，要点单吗? (HTTP协议概览)](https://www.cnblogs.com/vamei/archive/2013/05/11/3069788.html)。

#### 制定RPC协议
通过了解http协议，大致了解一个网络协议的简单构成。但是http协议使用了文本表示并且，通过了换行与空行作为协议内容的分隔符，传输数据比较大，传输效率低下，而且Http是一个无状态协议，每次发送请求都需要重新建立连接，这对高性能的rpc框架是不可容忍的，所以可以定制一个更加紧凑的rpc传输协议。
  
[![协议.png](https://s1.ax1x.com/2022/10/18/xrWHiQ.md.png)](https://imgse.com/i/xrWHiQ)

协议定制参考了dubbo协议，协议由协议头和协议体两部分组成，协议头共占用16个字节。
- 魔数：协议魔数，标识验证协议的合法性
- 版本号：协议的版本
- 消息id：消息的唯一id，客户端发送请求时缓存消息id，接收到响应时与对应的请求匹配，从而实现异步发送接受。
- 消息类型：请求 or 响应
- 响应标识: 是否需要接受响应标识，请求消息指定
- 心跳标识：是否心跳消息
- 序列化：序列化方式，客户端或者消费者根据标识进行序列化反序列化
- 状态：消息状态
- 数据长度：消息体所占字节长度



### 网络传输

网络传输模块与注册中心都是相对比较独立的模块。该模块主要包含三个功能 协议定制、序列化与网络通信。而网络通信使用了比较成熟的开源框架-netty完成，我们只需要关注另外两个功能的实现。


> Netty is a NIO client server framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server.

netty官网的介绍中说明了netty是一个简单易用异步nio 网络编程框架，而且netty 支持各种网络协议比如http，ws等，支持自定义协议编解码器，可扩展性强，通过简单的代码就可以创建一个网络服务器。了解使用可以参考[Netty文档](https://netty.io/wiki/user-guide-for-4.x.html)。


介绍下netty的组件

- `Channel`:可用于读写数据的网络socket连接
    - `ChannelGroup`:线程安全的`Channel`集合，可对集合内`Channel`进行批量操作 

- `ChannelFuture`:`Channel`操作的异步返回结果，继承自JUC的Future,扩展异步回调
    - `ChannelFutureListener`:可注册到`ChannelFuture`的异步回调监听

- `ChannelHandler`:主要用于处理channel中的IO读写及channel生命周期操作，根据入站和出站不同实现
    - `ChannelInboundHandler`:用于处理入站操作
    - `ChannelInboundHandler`:用于处理出站操作

- `ChannelPipline`:数据通过channel的`ChannelHandler`拦截器链，消息通过pipiline传递到各个`ChannelHandler`，入站与出站的拦截器执行方向相反，就像手扶电梯的上下方向。

- `ChannelHandlerContext`: `ChannelHandler`与`ChannelPipline`关联的上下文容器，每个`ChannelHandler`对应一个上下文。


图解下`Channel`、`ChannelHandler`、`ChannelPipline`及`ChannelHandlerContext`的关系
[![channel.png](https://s1.ax1x.com/2022/10/27/xfQOQe.md.png)](https://imgse.com/i/xfQOQe)



- `ByteBuf`:数据容器，java的`ByteBuffer`替代品，拥有读写指针

- `BootStrap`:客户端启动引导
- `ServerBootStrap`:服务端启动引导

- `EventLoop`:netty的线程抽象，用于处理连接的生命周期中所发生的事件
    - `EventLoopGroup`:`EventLoop`集合，继承自`ScheduledExecutorService`线程池，并进行了扩展

#### 引导用例
通过一个客户端服务端引导代码解释下各个组件功能

- **添加依赖**
```xml
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.56.Final</version>
        </dependency>
```

- **服务端程序**
```java

public class ExampleServer {


    public static void main(String[] args) {
        // boss线程组负责接收连接
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        // worker 线程组负责处理已接收连接
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            //服务引导类
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    //指定接收客户端连接的 socketChannel类型
                    .channel(NioServerSocketChannel.class)
                    //客户端连接初始化Channel程序 指定流水线及配置handler
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            //为ChannelPipeline配置一个ChannelHandler
                            ch.pipeline().addLast(new SimpleChannelInboundHandler<ByteBuf>() {
                                @Override
                                protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
                                    System.out.println("receive msg :"+byteBuf.toString(CharsetUtil.UTF_8));
                                    channelHandlerContext.writeAndFlush("hello client".getBytes(StandardCharsets.UTF_8));
                                    //关闭客户端连接
                                    channelHandlerContext.close();
                                }
                            });
                        }
                    })
                    //option()方法是对NioServerSocketChannel 的属性配置
                    .option(ChannelOption.SO_BACKLOG, 128)
                    //childOption() 方法是对客户端连接Channel 的属性配置
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            // 同步阻塞绑定本地端口
            ChannelFuture f = b.bind(8080).sync();
            // 阻塞等待server socket 关闭
            f.channel().closeFuture().sync();
        } catch (InterruptedException exception) {
            // ignore
        } finally {
            //优雅关闭
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
}
```

- **客户端引导程序**

```java
public class ExampleClient {

    public static void main(String[] args) {

        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            //指定连接事件线程池
            b.group(workerGroup);
            //Channel 连接类型
            b.channel(NioSocketChannel.class);
            //Channel 配置开启keep-live
            b.option(ChannelOption.SO_KEEPALIVE, true);
            //配置创建Channel handler及pipeline
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new SimpleChannelInboundHandler<ByteBuf>() {
                        @Override
                        protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
                            System.out.println("receive msg :"+byteBuf.toString(CharsetUtil.UTF_8));
                        }
                    });
                }
            });

            // 同步阻塞连接服务器
            ChannelFuture f = b.connect("127.0.0.1", 8080).sync();
            //发送消息
            f.channel().writeAndFlush(Unpooled.copiedBuffer("hello server", CharsetUtil.UTF_8));
            //同步阻塞直到channel被关闭
            f.channel().closeFuture().sync();
        } catch (InterruptedException exception) {
            //ignore
        } finally {
            workerGroup.shutdownGracefully();
        }
    }

}
```

需要注意的是 服务端启动类`ServerBootstrap`指定了两个`EventLoop`线程组boss与worker，同时会区分`option`、`childOption`及`ChildHandler`方法，这是由于Netty是通过Reactor 模式实现。

[![reactor.jpg](https://s1.ax1x.com/2022/10/27/xf4Dg0.md.jpg)](https://imgse.com/i/xf4Dg0)

- mainReactor：负责处理服务端与客户端连接请求事件，对应netty服务启动类中指定boss线程组
- subReactor：负责处理服务端与客户端连接的读写事件，对应netty服务启动类中指定worker线程组

#### 编解码器

数据在网络中都是以字节的方式传输，而当需要转换成有意义的数据，就需要进行数据的编码和解码，常见场景是处理各类网络协议，例如http协议。程序员大部分时间并不需要关心这类场景，因为网络编程框架已经帮我们实现，开箱即用，但是当需要定制rpc协议时就要进行一系列的编码解码操作。
netty默认实现了大部分的协议编码解码器，同样也为自定义编解码器提供了入口。
- `ByteToMessageDecoder`:解码器抽象类，将字节转换为消息，继承自`ChannelInboundHandler`处理入站数据
- `MessageToByteEncoder`:编码器抽象类，将消息转换为字节，继承自`ChannelOutboundHandler`处理出战数据

通过netty 提供的接口实现 定制的Rpc协议编解码
[![协议.png](https://s1.ax1x.com/2022/10/18/xrWHiQ.md.png)](https://imgse.com/i/xrWHiQ)
- 消息对象
```java
public class Message implements Serializable {

    private int version;

    private long id;

    private boolean isRequest;

    private boolean heartBeat;

    private boolean twoWay;

    private Object data;

    private String serialType;
    
    //getter setter
}

```
- 属性枚举
```java
public class MessageEnum {
    /**
     * header length
     */
    static final int HEADER_LENGTH = 16;
    /**
     * magic num 17
     */
    static final byte MAGIC_NUM = 0x11;
    /**
     * rpc version
     */
    static final byte VERSION = 0x1;
    /**
     * request or response flag
     */
    static final byte MESSAGE_FLAG = (byte) 0x80;
    /**
     * two way flag
     */
    static final byte TWO_WAY = (byte) 0x40;
    /**
     * heart beat flag
     */
    static final byte HEART_BEAT = (byte) 0x20;
    /**
     * serialization mask
     */
    static final byte SERIALIZATION_MASK = (byte) 0x1f;
    /**
     * java serialization
     */
    static final byte SERIALIZATION_JAVA = (byte) 1;
    /**
     * protobuf serialization
     */
    static final byte SERIALIZATION_PROTOBUF = (byte) 1;

}

```

- 编码器
```java
public class MessageEncoder extends MessageToByteEncoder<Message> {

    @Override
    protected void encode(ChannelHandlerContext channelHandlerContext, Message message, ByteBuf byteBuf) throws Exception {
        //魔数
        byteBuf.writeByte(MessageEnum.MAGIC_NUM);
        //版本号
        byteBuf.writeByte(MessageEnum.VERSION);
        //消息id
        byteBuf.writeLong(message.getId());
        byte messageAttr = 0;
        if(message.isRequest()){
            messageAttr=(byte) (message.getSerialType()|MessageEnum.MESSAGE_FLAG);
        }
        if(message.isHeartBeat()){
            messageAttr |= MessageEnum.HEART_BEAT;
        }
        if(message.isTwoWay()){
            messageAttr |= MessageEnum.TWO_WAY;
        }
        //消息序列化类型等属性
        byteBuf.writeByte(messageAttr);
        Object data = message.getData();
        //序列化
        //Serialization serialization = SerializationFactory.getSerialization(message.getSerialType());
        //byte[] bytes = serialization.serialize(data);
        byte[] bytes = ((String) data).getBytes(StandardCharsets.UTF_8);
        //数据长度
        byteBuf.writeInt(bytes.length);
        //消息体
        byteBuf.writeBytes(bytes);
    }
}
```
- 解码器
```java
public class MessageDecoder extends ByteToMessageDecoder {

    @Override
    protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf, List<Object> list) throws Exception {
        int i = byteBuf.readableBytes();
        if(i<MessageEnum.HEADER_LENGTH){
            return;
        }
        byteBuf.markReaderIndex();
        byte version;
        if (byteBuf.readByte() != MessageEnum.MAGIC_NUM) {
            return;
        }
        if ((version = byteBuf.readByte()) != MessageEnum.VERSION) {
            return;
        }
        long id = byteBuf.readLong();
        byte messageAttr = byteBuf.readByte();
        boolean isRequest = (messageAttr & MessageEnum.MESSAGE_FLAG) != 0;
        boolean twoWay = (messageAttr & MessageEnum.TWO_WAY) !=0;
        boolean heartBeat = (messageAttr & MessageEnum.HEART_BEAT) !=0;
        byte serialType = (byte) (messageAttr & MessageEnum.SERIALIZATION_MASK);
        int dataLength = byteBuf.readInt();
        if(byteBuf.readableBytes()<dataLength){
            byteBuf.resetReaderIndex();
            return;
        }
        byte[] bytes = new byte[dataLength];
        byteBuf.readBytes(bytes);
        //反序列化
        //Serialization serialization = SerializationFactory.getSerialization(serialType);
        //Object data = serialization.deserialize(bytes);
        Object data = new String(bytes, StandardCharsets.UTF_8);
        Message message = new Message();
        message.setVersion(version);
        message.setId(id);
        message.setRequest(isRequest);
        message.setHeartBeat(heartBeat);
        message.setTwoWay(twoWay);
        message.setSerialType(serialType);
        message.setData(data);
        list.add(message);
    }
}
```

协议涉及参考了bubbo,编解码器中有一个巧妙的设计，消息类型、响应标识与心跳标识和序列化类型仅使用了1个bit，使用1bit即可标识类型。协议的编解码过程中，通过位运算就可以解析协议中这几个标识，
[![协议定制](https://s1.ax1x.com/2022/10/19/xsmyJP.png)](https://imgse.com/i/xsmyJP)
 协议构建时通过 `|` 异或操作，协议解析时在再进行对应类型的`&`计算根据结果判断标识。



- **服务端引导程序调整**
``` java
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    //新增编解码器
                    ch.pipeline().addLast(new MessageDecoder())
                            .addLast(new MessageEncoder())
                            .addLast(new SimpleChannelInboundHandler<Message>() {
                        @Override
                        protected void channelRead0(ChannelHandlerContext channelHandlerContext, Message message) throws Exception {
                            System.out.println("receive msg :"+message.getData());
                            message.setData("hello client");
                            message.setRequest(false);
                            channelHandlerContext.writeAndFlush(message);
                            //关闭客户端连接
                            channelHandlerContext.close();
                        }
                    });
                }
            })
```
- 客户端调整

```java
        b.handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline().addLast(new MessageDecoder())
                        .addLast(new MessageEncoder())
                        .addLast(new SimpleChannelInboundHandler<Message>() {
                            @Override
                            protected void channelRead0(ChannelHandlerContext channelHandlerContext, Message message) throws Exception {
                                System.out.println("receive msg :" + message.getData());
                            }
                        });
            }
        });
        // 同步阻塞连接服务器
        ChannelFuture f = b.connect("127.0.0.1", 8080).sync();
        //发送消息
        Message message = new Message();
        message.setRequest(true);
        message.setHeartBeat(false);
        message.setTwoWay(true);
        message.setId(1);
        message.setVersion(MessageEnum.VERSION);
        message.setSerialType(MessageEnum.SERIALIZATION_JAVA);
        message.setData("hello server");
        f.channel().writeAndFlush(message);
        //同步阻塞直到channel被关闭
        f.channel().closeFuture().sync();
```



#### 连接保活
客户端与服务端通信时建立连接，通信完成后就可以断开连接，这种一般称为短连接，但是当通信比较频繁时，每次都是重新建立短连接，会对服务端造成一定压力，毕竟连接的建立需要一定的成本，这时就可以考虑使用长连接。tcp长连接的建立需要建立保活机制，主要是通过客户端定时发送心跳包维持连接。  
但是长连接也并不是完美的，如果建立长连接后长时间没有通信变为空闲连接，服务端维持大量空闲长连接也是对资源的消耗，所以要对空闲长连接定时关闭
。netty提供了相应的解决方案 `IdleStateHandler`。

- `IdleStateHandler(int readerIdleTimeSeconds,int writerIdleTimeSeconds,int allIdleTimeSeconds)`:当长时间没有消息传输会触发一个`IdleStateEvent`事件，可以通过重写`ChannelInboundHandler`的`userEventTriggered()`方法处理该事件。
    - `readerIdleTimeSeconds`:指定时间内没有读取数据触发读取空闲`READER_IDLE`事件
    - `writerIdleTimeSeconds`:指定时间内没有写数据触发写空闲`WRITER_IDLE`事件
    - `allIdleTimeSeconds`:指定时间内没有读写数据触发读写空闲件`ALL_IDLE`事件

- 客户端引导程序调整
```java
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new MessageDecoder())
                            .addLast(new MessageEncoder())
                            //空闲连接处理
                            .addLast(new IdleStateHandler(6000,0,0, TimeUnit.MILLISECONDS))
                            .addLast(new SimpleChannelInboundHandler<Message>() {
                                @Override
                                protected void channelRead0(ChannelHandlerContext channelHandlerContext, Message message) throws Exception {
                                    System.out.println("receive msg :" + message.getData());
                                }
                                @Override
                                public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
                                    //定时发送心跳包
                                    if(evt instanceof IdleStateEvent){
                                        Message message = new Message();
                                        message.setId(ThreadLocalRandom.current().nextLong());
                                        message.setHeartBeat(true);
                                        message.setTwoWay(true);
                                        message.setData("ping");
                                        ctx.writeAndFlush(message);
                                    }else{
                                        super.userEventTriggered(ctx,evt);
                                    }
                                }
                            });
                }
            });

```

- 服务端引导程序
```java
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    //为ChannelPipeline配置一个ChannelHandler
                    ch.pipeline().addLast(new MessageDecoder())
                            .addLast(new MessageEncoder())
                            //空闲连接处理
                            .addLast(new IdleStateHandler(0, 0, 10000, TimeUnit.MILLISECONDS))
                            .addLast(new SimpleChannelInboundHandler<Message>() {
                                @Override
                                protected void channelRead0(ChannelHandlerContext channelHandlerContext, Message message) throws Exception {
                                    System.out.println("receive msg :" + message.getData());
                                    if(message.isHeartBeat()){
                                        message.setData("pong");
                                    }else{
                                        message.setData("hello client");
                                    }
                                    message.setRequest(false);
                                    channelHandlerContext.writeAndFlush(message);
                                }

                                @Override
                                public void userEventTriggered(ChannelHandlerContext channelHandlerContext, Object evt) throws Exception {
                                    if(evt instanceof IdleStateEvent) {
                                        //超时未收到心跳包关闭客户端连接
                                        channelHandlerContext.close();
                                    }
                                }
                            });
                }
            })
```
### 异步请求
同步请求即在同一时间只能处理一个请求，同步等待响应结果，而异步请求指通过异步响应的方式处理请求，无需同步等待响应结果。
rpc框架为了提高性能与吞吐往往会采用异步调用的方式，常见的方式就是返回一个异步结果对象`Future`，或者添加异步通知`Callback`。

通常情况下rpc框架发送请求时会指定一个请求id`requestId`,请求发送完成后创建一个`Future`与id关联，然后继续处理其他请求，在服务端响应时再根据id找到对应的`Future`填充结果，做到异步响应。

[![异步调用.png](https://s1.ax1x.com/2022/10/28/xh45lt.png)](https://imgse.com/i/xh45lt)
当然如果仅希望使用同步请求模式，可以通过阻塞`Future`的方式同步获取结果。在java8 中提供了一个`CompletableFuture`，在`Future`的基础上扩展了异步任务的编排，支持异步回调功能，所以我们可以通过`CompletableFuture`的添加callback的方式去实现异步回调模式。

```
    CompletableFuture<Object> future = client.sendMessage(Object data);
    future.whenComplete((r,t)->{
       //异步回调
    });
    Object result = future.get();
```

## 结语

关于为什么要自己去实现一套rpc框架，其实程序员界流传一句话，不要重复去造轮子，很多常见的轮子都是经过业务的验证，而自己写的轮子很可能场景考虑不全或者出现性能问题。
但是作为一个优秀的程序员，不能对一个轮子仅是停留在会使用的层面，要学会刨析内部设计与实现，当出现问题时可以第一时间定位，迅速解决，而手写一个轮子不仅可以深入到底层，还可以提升设计思维和架构视野。
最近看到的一篇文章《[如何高效的学习技术](https://www.cnblogs.com/xiaoyangjia/%20p/11535486.html)》也是提到要学会造轮子，个人同样认为程序员是一个需要持续学习持续进步的岗位，让自己保持竞争力。
