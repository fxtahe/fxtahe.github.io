---
title: seata-about
date: 2020-04-25 11:04:54
categories:
- Java
tags: 
- seata
- about
---
### 背景

在分布式微服务架构中，应用部署在不同的服务器环境中，因此需要服务与服务之间远程协作才能完成事务操 作，这种分布式系统环境下由不同的服务之间通过网络远程协作完成事务称之为分布式事务,如何解决分布式事务问题也是在分布式架构系统所必须考虑的。

seata是ali开源的分布式事务解决方案，提供了AT、TCC、及SAGA(长事务)、XA等模式。
<!--more-->
## seat架构

seata 是将各分布式分支事务统一为全局事务，通过全局事务的成功与否判断是否需要回滚提交各分支事务。

<div   align="center" >
<img src="https://imgconvert.csdnimg.cn/aHR0cDovL21lZGlhLnRlYW1zaHViLmNvbS8xMDAwMC90bS8yMDIwLzA0LzE2LzA1MDM1MzNlLTA2ODgtNDYwNS05MmRhLWMwMDVkNDM1ZTE2Ni9jOWFkM2JiNS0zMjViLTQzMjgtYTQ0OC1iODZlYmU2ZjdjMjIucG5n?x-oss-process=image/format,png" />
</div>
seata架构中有三大组件：
- TC（Transaction Coordinator）事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚。
- TM（Transaction Manager）事务管理器 控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议。
- RM（Resouce Manager)资源管理器:控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚。

<div   align="center" >
<img src="https://imgconvert.csdnimg.cn/aHR0cDovL21lZGlhLnRlYW1zaHViLmNvbS8xMDAwMC90bS8yMDIwLzA0LzE2LzkyMWYyNWE0LTBiMWYtNDg5Ny04NGRiLTQ0OGVlNTc4ZjI0MS9mM2EwNTJkMi00MmNkLTQ5OTMtOGUxZi1mZDkyOTQwOTFlMGIucG5n?x-oss-process=image/format,png" />
</div>
分布式事务解决流程：  

- 1.TM向TC注册一个全局事务，TC返回全局事务id，XID  
- 2.XID通过微服务调用链传播  
- 3.RM将本地事务注册为XID到TC的相应全局事务的一个分支。  
- 4.TM通知TC提交或回滚XID所对应的全局事务  
- 5.TC通知XID全局事务下的分支事务提交或回滚  

总的来说主要是通过TC将分支事务统一管理，把分支事务转换为全局事务。分支事务提交则全局提交，某个分支事务异常回滚则全局事务回滚，从而解决了分布式事务问题。

### 事务模式

seata支持多种分布式事务解决模式，包括AT、TCC、SAGA、XA等。（XA模式开发中）

#### AT模式

AT模式是一种无侵入的分布式解决方案。整体机制是两阶段提交协议的演变，seata将业务sql作为一阶段，而二阶段提交和回滚操作由seata生成管理，用户只需要关心业务sql。

<div align="center"><img src="https://imgconvert.csdnimg.cn/aHR0cDovL21lZGlhLnRlYW1zaHViLmNvbS8xMDAwMC90bS8yMDIwLzA0LzE2LzhkYmYyNjhlLTM3ZmUtNDk2YS05ODk5LWNhZGExYzZlZTk1OS8wNTRmOTc1NC1mNTg2LTRmNGQtYmZhMy1lNDQyMzY4NjkxMTgucG5n?x-oss-process=image/format,png" />
</div>

执行流程：  

- 一阶段  
seata 会解析业务SQL，解析找到要更新的业务数据，并保存为前置快照，然后执行业务SQL，并把更新后的数据保存为后置快照。最后生成行锁。以上操作均在同一个数据库事务中保证了原子性。

- 二阶段
  - 提交  
        全局事务成功提交则只需要删除快照数据和行锁，完成数据清理
  - 回滚
        数据校验，避免出现脏写，然后通过前置快照进行数据还原，最后删除快照信息和行锁，完成数据清理

<div align="center"><img src="https://imgconvert.csdnimg.cn/aHR0cDovL21lZGlhLnRlYW1zaHViLmNvbS8xMDAwMC90bS8yMDIwLzA0LzE2LzhkYmYyNjhlLTM3ZmUtNDk2YS05ODk5LWNhZGExYzZlZTk1OS84YzJhMWM3Yi05YzE1LTQxZjAtOGFhNy03Y2UyYzIyNDUxZjkucG5n?x-oss-process=image/format,png" />
</div>

这种模式下，用户无需关心分布式事务的提交与回滚，事务问题交由seata进行管理，实现了无侵入的分布式事务解决方案。

#### TCC模式

TCC模式属于服务化的两阶段提交模式，用户需要根据场景实现一阶段和二阶段的提交回滚方法。对应方法是Try、Confirm及Cancel。

- Try:对操作的资源检测预留
- Confirm:执行业务提交操作
- Cancel:释放预留资源

<div align="center"><img src="https://imgconvert.csdnimg.cn/aHR0cDovL21lZGlhLnRlYW1zaHViLmNvbS8xMDAwMC90bS8yMDIwLzA0LzE2L2Y0NGEyNGFkLTQ2ZmYtNDkwMy1hN2E2LTI0ZDdjYWIxMmE4ZS85MGVjOTFlYy05ZTkyLTRkMzktOTk2NS04YjY2NDJhZWFkMzgucG5n?x-oss-process=image/format,png" />
</div>
这种方式需要用户自己实现接口，对代码的侵入性比较强，同时需要考虑并发控制和异常控制，但是其不依赖底层数据资源的事务支持。

#### SAGA模式

SAGA是一种分布式长事务解决方案，需要用户根据需求实现正向及逆向回滚操作。分布式事务链中全部执行成功则分布式事务提交，如果某个分布式事务执行失败，则会执行前面各分布式事务参与者的逆向回滚操作，实现全局回滚。

<div align="center"><img src="https://imgconvert.csdnimg.cn/aHR0cDovL21lZGlhLnRlYW1zaHViLmNvbS8xMDAwMC90bS8yMDIwLzA0LzE2L2Y0NGEyNGFkLTQ2ZmYtNDkwMy1hN2E2LTI0ZDdjYWIxMmE4ZS9jZjNkY2EzMy1kZDQxLTRjMTYtYWYzMi05YzAzN2Q4NzY2YzQucG5n?x-oss-process=image/format,png"/>
</div>

SAGA模式是基于状态机引擎来实现的

- 1.通过状态图来定义服务调用的流程并生成 json 状态语言定义文件
- 2.状态图中一个节点可以是调用一个服务，节点可以配置它的补偿节点
- 3.状态图 json 由状态机引擎驱动执行，当出现异常时状态引擎反向执行已成功节点对应的补偿节点将事务回滚

SAGA模式适用于业务流程长且需要保证事务最终一致性的业务系统，但是其开发成本较高
