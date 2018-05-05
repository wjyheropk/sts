
## 1、简介: ##

STS(SOA Transaction Service)是一款基于**事务补偿**的分布式事务框架，用来保障在分布式环境下事务的最终一致性。STS 从架构上分为 sts-client、 sts-monitor量部分，前者是一个嵌入客户端应用的jar包，主要负责分布式事务控制；后者是一个独立的系统，主要负责异常监控、告警和恢复。

STS比较适用于SOA系统以及微服务架构的系统中，各模块有独立的数据库，用来保障跨服务的数据一致性，且对各种业务场景几乎没有特殊要求，通用性强。

**数据一致性问题的现有解决方案:**

- 方案a，XA协议，作为资源管理器（数据库）与事务管理器的接口标准，各大数据库厂家都提供对XA的支持。XA协议采用两阶段提交方式来管理分布式事务，最主要缺点是性能差，更重要的是，这是一种资源层的分布式事务控制，适合同一个系统/模块中需要分库分表的业务场景，而SOA系统需要的是应用层的分布式事务控制

- 方案b，柔性事务/补偿型事务，常见的是TCC型事务（Try/Confirm/Cancel），还有最大努力送达等方案。最主要缺点是业务侵入性太强，需要大量开发工作进行业务改造，给业务升级、运维都带来困难。

- 其他方案，例如阿里gts等，也有很多公司目前还是由人工处理分布式事务问题。

**STS特性:**

- **接入成本低**：对系统原有代码无侵入，只需新增配置、注解（服务提供者需增加一些回滚代码逻辑）原有业务不需感知分布式事务！

- **最终一致**：事务处理过程中，会有短暂不一致的情况，但通过恢复系统，可以让事务的数据达到最终一致的目标。

- **服务幂等**：STS支持服务幂等性保障

- **与 RPC 协议无关**：在 SOA 架构下，一个或多个 DB 操作往往被包装成一个一个的 Service，Service 与 Service 之间通过 RPC 协议通信。STS 框架构建在 SOA 架构上，与底层协议无关。

- **与本地事务实现无关**： STS 是一个抽象的基于 Service 层的概念，在思路上，与底层本地事务实现无关，暂时只支持MYSQL。

- **一定程度上支持水平分库场景**： 如果不使用分库分表中间件，STS可以支持跨库事务，如果使用了某种中间件，那就需要进行定制。


## 2、STS的设计思路: ##

### 2.1 设计目标

- 最终一致性：分布式系统中不可能达到（或很难达到）强一致性，系统需要容忍可能的短暂的不一致

- 服务幂等性

| 目标       | 方案  |
| ------------- | -------------|
| 最终一致性，通用性强，与业务解耦，与底层RPC框架无关    | 1、不改变业务流程，服务层进行事务补偿，业务无感知。2、在异常情况下，由发起方，发起进行事务回滚，每个参与者回滚之前的操作，最后发起方回滚本地事务 |  
| 服务幂等性       |   通过transId + 事务信息持久化来保障，业务操作可以重复调用，回滚操作可以重复调用。极端情况下，能够保存重要信息，便于事务恢复，并主动报警|事务信息持久化 + 事务状态；整个SOA事务状态——doing/done，每个原子操作状态——try/success/rollback_success   | 
| 使用方便，开发简单|仅需要实现callback接口并增加注解，对原有代码无侵入|


### 2.2 事务恢复

事物恢复是我的核心目标，从目标出发，来考虑一下如何进行恢复。我们先对SOA系统中的一般流程进行抽象

**核心概念:**

- **发起方**： client端，负责启动分布式事务，触发创建相应的主事务记录。是分布式事务的协调者，负责调用参与者的服务，并记录相应的事务日志，感知整个分布式事务状态来决定整个事务是 COMMIT 还是 ROLLBACK。

- **参与者**：server端，参与者是分布式事务中的一个原子单位，为client端提供业务接口（API）和rollback接口，并保证其业务数据的幂等性，也是回滚逻辑的执行者。

- **Transaction**：这里并非本地事务，而是分布式事务，其最核心的数据结构是事务号和事务状态，它是在启动分布式事务的时候，由发起方持久化写入数据库的。

- **Operation**：分布式事务中的一个原子操作，client调用server的一个接口，就是一个原子操作，原子操作也有id和状态。

> <i class="icon-pencil"></i>**Note：**以下分布式事务简称trans，原子操作简称op。

<div align=center><img width="500" src="https://github.com/wjyheropk/ImageStore/blob/master/sts-1.png"/></div>

问题一：谁发起事务恢复？

- 因为整个事物是否成功完成，只有发起方才知道。所以一旦发现失败，发起方就应发起事物恢复

问题二：什么时候发起事务恢复？

- 发起方感知到任何异常时，就应发起事务恢复

问题三：怎么进行事务恢复？

- 发起恢复之后，需要各个参与者的配合，比如说第一个操作，是由参与者1完成的，那么该操作的回滚，也需要他负责，这样就需要各个参与者要提供rollbackApi供发起方调用，发起方本地的回滚就靠本地事务保障

总结起来就是，整个事物由发起方控制，各参与者一起配合完成的过程



### 2.3 事务信息持久化

问题一：需要持久化什么信息？

- 作为发起方，需要知道，整个事物的完成状态、事物里都包含哪些操作，才能发起恢复

- 作为参与者，需要知道，这个操作之前是不是由我做的，有没有做成功，才能回滚操作，所以想恢复，必先持久化事物信息，如下图所示，绿色的框中已经列举了需要持久化的必要信息

- 同时，因为持久化了事务id，原子操作id和状态，所以业务服务、rollbackApi就可以支持幂等了

<div align=center><img width="800" src="https://github.com/wjyheropk/ImageStore/blob/master/sts-2.png"/></div>


问题二：怎么样保证持久化的事务信息可信？

- 因为事物信息的持久化也同样存在事物问题，所以如何确保被持久化的信息可信呢？答案是：利用本地事务。

- 以分布式事务中的某个operation A为例，参与者的服务被调用时，先插入op记录，此时新起本地事务，即使后面业务异常回滚，op信息已经持久化；当服务执行成功时，更新op状态为try—>success，此时嵌入本地事务，因为只要本地事务提交成功，op状态必然 =success。




附：事务状态、原子操作状态的流转

| -        | 状态枚举值   |  保存位置  |
| ------------- |-------------| -----|
| **Trans**     | doing/done |   client端     |
| **Op**        |   try/success/rollback_success   |   Server端   |


<br />

### 2.4 易用性

框架的易用性也是很重要的一点，前面说的事务信息持久化、rollbackApi、回滚流程全部由STS框架封装，业务无感知，源码无侵入

如图所示，以参与者为例，左边业务代码，右边是框架封装代码，业务原有代码不用改，只需额外实现我提供的callback接口即可，callback用于辅助框架工作，比如生成事务信息、具体回滚逻辑等等

框架这侧，核心是事务管理器，在不同的阶段，框架会自动对callback进行调用

![](https://github.com/wjyheropk/ImageStore/blob/master/sts-3.png)


### 2.5 详细工作流程图

略。（有需要可以私信）

注：sts会保障业务操作、回滚操作幂等，且支持并发（例如并发回滚同一个原子操作）

### 2.6 水平分库场景事务支持

很多项目都需要分库分表，可能使用各种各样的中间件来实现，当然也有不使用中间件的情况。

1、不使用中间件的情况：sts解决思路是，选择一个dataSource作为主库，用本地事务控制，其他库看做辅库，类似于分布式事务中的"参与者"，进行处理。不同的是，这些辅库回滚时需要发起方自己处理，而不是调用参与者的api，在使用@SoaAtomic标记原子操作时，设置rollbackType=RollbackType.CLIENT_BASE

2、使用中间件的情况：需要根据具体情况进行定制

## 3、附录: ##

### 3.1 级联事务支持

除了使用transId来标识一个分布式事务之外，STS还使用trainId来串联一系列的级联事务。在参与者进行事务恢复时，会先查找本地事务持久化数据，判断是否存在级联事务，如果存在则该模块先发起下游级联事务的恢复，再恢复上游事务（无论级联事务是否恢复成功）

![](https://github.com/wjyheropk/ImageStore/blob/master/sts-4.png)

### 3.2 监控中心

各模块将事务信息上报至监控中心，用于事务对账、监控报警、数据统计等工作



### 3.3 其他语言模块支持

STS由JAVA语言编写，暂时只支持JAVA模块接入，但整体思路不局限于JAVA，其他语言模块一样可以支持

### 3.4 与TCC思路的本质区别

除了对现有业务改造成本的区别之外，sts与tcc的本质区别在于：100米跑步，如果已经跑过了50米，这个时候你摔倒了，你是要回到起跑线，还是要继续到终点？
TCC思路认为，如果try成功，那么confirm一定成功，所以如果confrim失败，是不会执行cancel的，而是会一直尝试confirm，也就是说，如果你跑过了50米，就应该继续跑向终点，而本框架sts认为，有任何异常发生，都应该回到起点。

所以到底用不用Tcc还是要根据实际的业务场景去综合考量。个人认为tcc适合金融、电商等可以进行资源预留、预分配的业务场景，但不具备通用性。

