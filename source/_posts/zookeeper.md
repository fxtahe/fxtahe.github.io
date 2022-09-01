---
title: zookeeper
date: 2022-08-30 22:15:49
categories:
    - 中间件
tags:
    - zookeeper
---


## 概念

ZooKeeper主要**服务于分布式系统**，可以用ZooKeeper来做：统一配置管理、统一命名服务、分布式锁、集群管理。

## Znode

zookeeper 的数据存储结构类似于unix文件系统，可以看成一棵树，每个节点就是znode。每个节点可以通过路径表达。而数据则存储在各个节点上，通过节点路径获取数据。

### znode类型

znode 存在以下四种类型

- **持久（PERSISTENT）节点**：一旦创建就一直存在，直至zookeeper宕机。可以创建子节点
  
- **临时（EPHEMERAL）节点**：临时节点生命周期与客户端会话（session）绑定，一旦客户端断开链接则自动删除该节点。并且，**临时节点只能做叶子节点** ，不能创建子节点，只能作为叶子节点。
  
- **持久顺序（PERSISTENT_SEQUENTIAL）节点**：与持久节点生命周期一致，不过节点存在顺序，节点名一致时会默认添加序号，/b0000000001,/b0000000002
  
- **临时顺序（EPHEMERAL_SEQUENTIAL）节点**：与临时节点生命周期一致，不允许创建子节点，相同节点名会默认添加顺序序号
  

  <!-- more -->

### znode状态

zookeeper存储了znode的状态信息，通过stat 获取

```shell
cZxid = 0x1f1
ctime = Sat May 07 14:51:17 CST 2022
mZxid = 0x1f1
mtime = Sat May 07 14:51:17 CST 2022
pZxid = 0x1f1
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 0
```

| 状态信息 | 解释  |
| --- | --- |
| cZxid | create ZXID，即该数据节点被创建时的事务 id |
| ctime | create time，即该节点的创建时间 |
| mZxid | modified ZXID，即该节点最终一次更新时的事务 id |
| mtime | modified time，即该节点最后一次的更新时间 |
| pZxid | 该节点的子节点列表最后一次修改时的事务 id，只有子节点列表变更才会更新 pZxid，子节点内容变更不会更新 |
| cversion | 子节点版本号，当前节点的子节点每次变化时值增加 1 |
| dataVersion | 数据节点内容版本号，节点创建时为 0，每更新一次节点内容(不管内容有无变化)该版本号的值增加 1 |
| aclVersion | 节点的 ACL 版本号，表示该节点 ACL 信息变更次数 |
| ephemeralOwner | 创建该临时节点的会话的 sessionId；如果当前节点为持久节点，则 ephemeralOwner=0 |
| dataLength | 数据节点内容长度 |
| numChildren | 当前节点的子节点个数 |

### znode权限控制

ZooKeeper 采用 ACL（AccessControlLists）策略来进行权限控制，类似于 UNIX 文件系统的权限控制。Zookeeper 定义了如下5种权限

- create：创建子节点的权限
  
- delete：删除子节点的权限
  
- read：读取节点的权限
  
- write：写节点的权限
  
- admin：设置节点acl的权限
  

CREATE和DELETE这两种权限都是针对子节点的权限控制，以上五种权限简称cdrwa。

acl控制的znode的权限，节点间的权限不具备继承性，即子节点不继承父节点的权限。acl提供了下面四种方式进行权限控制。

- world:表示任何人都可以访问
  
- auth：只有认证的用户可以访问
  
- digest：用户名密码的验证方式
  
- host/ip:使用客户机的ip地址认证
  

## watcher

znode提供了zookeeper的数据存储机制，而watcher是监听znode变动所设计的核心功能。客户端watcher监听节点的数据状态变化，一旦发生节点发生变化，zookeeper会通知所有监听该节点的客户端，从而做出相应的动作。

zookeeper的大部分使用场景都是基于watcher机制，比如服务发现与注册，当服务节点发生变化，订阅服务的消费者可以拉取最新的服务，实现服务的动态发布更新。

watcher对节点的监听是一次性的，当服务端通知了watcher节点的状态变化，便会删除所以监听该节点的watcher，所以需要在收到通知后再次监听该节点，否则客户端只能接收到一次该节点的变更通知。

## zookeeper集群

为了保证高可用，zookeeper的集群是必须的，避免由于zookeeper的宕机导致整个系统的崩溃。通常 3 台服务器就可以构成一个 ZooKeeper 集群。

### 集群角色

zookeeper集群并不是通常意义上的master\slave模式，它设计了三种角色，分别是Leader、Follower、Observer。Leader可以提供数据读写，而Follower与Observer只能提供数据读权限，而Follower与Observer的区别是，Observer不参与到集群Leader的选举过程。

| 角色  | 功能  |
| --- | --- |
| Leader | 为客户端提供读和写的服务，负责投票的发起和决议，更新系统状态。 |
| Follower | 为客户端提供读服务，如果是写服务则转发给 Leader。在Leader选举过程中参与投票。 |
| Observer | 为客户端提供读服务器，如果是写服务则转发给 Leader。不参与选举过程中的投票。 |

### 选举过程

zookeeper 中的leader是通过集群选举产生，当集群启动或者leader服务器发生宕机网络波动重启等情况都会触发选举过程。

选举过程大致如下：

- 1.每个server会先投票给自己，然后接收集群中的各个服务器的投票数据，投票中包含了SID（服务器的唯一标识）和ZXID（事务ID）
  
- 2.检查投票，优先检查ZXID(事务ID)，ZXID大者胜出；如果ZXID相同，那么就比较SID，SID大者胜出。
  
- 3.确定Leader，投票完成后服务器会统计投票信息，如果一台机器收到了超过半数的投票，那么这个投票对应的SID机器即为Leader。
  

因为投票结果必须保证大于半数，因此zookeeper集群数则必须保证是奇数台。

## zookeeper常用功能

### 分布式锁功能

zookeeper实现分布式锁主要通过znode节点特性与watcher机制，当获取分布式锁时创建临时节点（客户端断开连接时可释放锁，使用临时节点防止锁无法被正常释放），释放锁时删除该节点，当其他应用获取分布式锁失败监听该节点的状态变化，当锁被释放时竞争锁。方案如下：

通过创建临时节点获取当前锁，如果当前临时节点是顺序最小的，则获取锁，否则监听上一个比自己小的节点。当上一个节点被删除则获取到锁，释放锁时删除临时节点，通过临时顺序节点+watcher机制实现分布式锁。

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.data.Stat;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;


public class ZookeeperLock {

    private static CuratorFramework client;

    private static final String BASE_PATH = "/lock";

    private static final String ZK_LOCK_PATH = "/zk_";

    private static final ThreadLocal<String> CURRENT_NODE = new ThreadLocal<>();

    static {
        client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        client.start();
    }

    public static boolean tryLock() {
        try {
            if (CURRENT_NODE.get()==null||CURRENT_NODE.get().trim().length()==0) {
                String currentNode = client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(BASE_PATH + ZK_LOCK_PATH);
                CURRENT_NODE.set(currentNode.substring(currentNode.lastIndexOf("/") + 1));
            }
            List<String> childNodes = client.getChildren().forPath(BASE_PATH);

            if (childNodes == null || childNodes.size() == 0) {
                throw new IllegalStateException("锁未释放");
            }
            Collections.sort(childNodes);
            String minNode = childNodes.get(0);
            String currentNode = CURRENT_NODE.get();
            //当前线程二次获取锁
            if (minNode.equals(currentNode)) {
                System.out.println(CURRENT_NODE.get()+"为锁节点");
                return true;
            }
            //监听上一个临时顺序节点
            String previousNode = childNodes.get(childNodes.indexOf(currentNode) - 1);
            NodeCache nodeCache = new NodeCache(client, BASE_PATH + "/" + previousNode, false);
            nodeCache.start();
            //获取锁阻塞线程
            CountDownLatch countDownLatch = new CountDownLatch(1);
            nodeCache.getListenable().addListener(() -> {
                Stat stat = client.checkExists().forPath(BASE_PATH + "/" + previousNode);
                if (stat == null) {
                    System.out.println(previousNode+"节点被删除");
                    nodeCache.close();
                    countDownLatch.countDown();
                }
            });
            countDownLatch.await();
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        System.out.println(CURRENT_NODE.get()+"为锁节点");
        return true;
    }


    public static void unLock() {
        try {
            if (CURRENT_NODE.get()!=null && CURRENT_NODE.get().trim().length()>0) {
                String currentNode = CURRENT_NODE.get();
                client.delete().forPath(BASE_PATH + "/" + currentNode);
                CURRENT_NODE.remove();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

    }


    public static void main(String[] args) throws Exception {
        ExecutorService executorService = new ThreadPoolExecutor(4, 4, 1000l, TimeUnit.SECONDS, new ArrayBlockingQueue<>(10), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r);
            }
        });
        for (int i = 0; i < 4; i++) {
            executorService.execute(() -> {
                tryLock();
                try {
                    System.out.println(Thread.currentThread().getName()+"获取到分布式锁");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                unLock();
            });
        }
        executorService.shutdown();
        if (executorService.awaitTermination(10000, TimeUnit.MILLISECONDS)) {
            executorService.shutdownNow();
        }

    }

}
```

上述实现demo仅做演示，生产环境下建议使用Curator 封装的分布式锁。

```java
        InterProcessMutex lock = new InterProcessMutex(client,"/zookeeper/lock");
        lock.acquire();
        try{
            // do something
        }finally {
            lock.release();
        }
```

### 注册中心

服务发布者将服务的各类信息发布到服务注册中心，例如ip，端口，参数等，而服务消费者通过订阅服务获取服务信息去调用服务。zookeeper通过其节点特性和watcher机制便能实现注册中心的功能。

服务发布者在发布服务时创建节点，将服务信息存储在节点中。而消费者获取对应znode节点信息，然后调用远程服务，这就是一个简单的注册中心。

```java
public class ZookeeperRegisterAndDiscovery {


    private static CuratorFramework client;

    private static final String BASE_SERVICE = "/service";

    static {
        client = CuratorFrameworkFactory.newClient("127.0.0.1:2181", new ExponentialBackoffRetry(1000, 3));
        client.start();
    }

    /**
     * 注册服务节点
     * |service|helloService
     * |--127.0.0.1:8091
     * |--127.0.0.1:8092
     * |--127.0.0.1:8093
     */
    public static void registerService(ServiceInfo serviceInfo) throws Exception {
        String path = BASE_SERVICE + "/" + serviceInfo.getServiceName() + "/" + serviceInfo.getAddress() + ":" + serviceInfo.getPort();
        if (client.checkExists().forPath(path) == null) {
            client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(
                    BASE_SERVICE + "/" + serviceInfo.getServiceName() + "/" + serviceInfo.getAddress() + ":" + serviceInfo.getPort()
                    , serviceInfo.toString().getBytes(StandardCharsets.UTF_8));
        }

    }

    /**
     * 服务发现
     */
    public static ServiceInfo discoveryService(String serviceName) throws Exception {
        List<String> serviceInfos = client.getChildren().forPath(BASE_SERVICE + "/" + serviceName);
        return loadBalance(serviceInfos, serviceName);
    }

    /**
     * 负载均衡
     */
    public static ServiceInfo loadBalance(List<String> serviceInfos, String serviceName) throws Exception {
        int i = ThreadLocalRandom.current().nextInt(serviceInfos.size());
        String s = serviceInfos.get(i);
        byte[] bytes = client.getData().forPath(BASE_SERVICE + "/" + serviceName + "/" + s);
        return JsonUtils.parse(ServiceInfo.class, new String(bytes, StandardCharsets.UTF_8));
    }


    public static void main(String[] args) throws Exception {

        ServiceInfo serviceInfo1 = new ServiceInfo("127.0.0.1", "8091", "helloService", new Class[]{String.class});
        ZookeeperRegisterAndDiscovery.registerService(serviceInfo1);
        ServiceInfo serviceInfo2 = new ServiceInfo("127.0.0.1", "8092", "helloService", new Class[]{String.class});
        ZookeeperRegisterAndDiscovery.registerService(serviceInfo2);
        ServiceInfo serviceInfo3 = new ServiceInfo("127.0.0.1", "8093", "helloService", new Class[]{String.class});
        ZookeeperRegisterAndDiscovery.registerService(serviceInfo3);

        ServiceInfo discoveryService = ZookeeperRegisterAndDiscovery.discoveryService("helloService");

        // 调用服务 callService(discoveryService);
        System.out.println(discoveryService.toString());
    }


    static class ServiceInfo {
        private String address;

        private String port;

        private String serviceName;

        private Class<?>[] parameterTypes;

        public ServiceInfo() {
        }

        public ServiceInfo(String address, String port, String serviceName, Class<?>[] parameterTypes) {
            this.address = address;
            this.port = port;
            this.serviceName = serviceName;
            this.parameterTypes = parameterTypes;
        }

        public void setAddress(String address) {
            this.address = address;
        }

        public void setPort(String port) {
            this.port = port;
        }

        public void setServiceName(String serviceName) {
            this.serviceName = serviceName;
        }

        public void setParameterTypes(Class<?>[] parameterTypes) {
            this.parameterTypes = parameterTypes;
        }

        public String getAddress() {
            return address;
        }

        public String getPort() {
            return port;
        }

        public String getServiceName() {
            return serviceName;
        }

        public Class<?>[] getParameterTypes() {
            return parameterTypes;
        }

        @Override
        public String toString() {
            return JsonUtils.writeAsString(this);
        }
    }


}
```

上述代码展示了服务发现与注册，完备的注册中心还需要服务上下线，健康检查，服务变更通知等。

