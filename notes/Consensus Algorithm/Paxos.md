# 分布式一致性算法——Paxos

#### 分布式的必要性

大规模数据读写， 数据密集型、高吞吐量系统

#### 单机系统的性能瓶颈

随着负载的增加，单台机器性能逐渐无法满足业务需求，一般情况下，需要对服务器资源进行扩容，业务上有两类方向。

##### 纵向扩容

指提升单机性能

优势：简单，无需额外操作

问题：单机性能不可能无限增长，存在上限或者说一个最佳平衡点

##### 横向扩容

指增加机器数量

优势：数据备份，负载均衡

问题：理论上可以无限，但是随着机器数量的增加，系统内网络通信会增加，以及带来数据分片管理以及一致性、可用性等问题

#### 与微服务架构的关系

微服务架构中，一类业务内聚而形成服务。一般来说，服务可以通过扩容来提高吞吐量。

微服务架构下，一个任务的流程会跨越多个服务，其中涉及到跨服务、跨模块数据修改，通常情况下服务无状态，且服务之间数据修改往往独立完成，通过消息队列、RPC等方式传递，服务内部彼此透明，且还存在网络异常等潜在问题。

这种情况的解决方式有

1. 共享存储，通过访问统一的数据存储，避免数据分治的出现。涉及到分布式的数据存储系统，就需要狭义上的分布式一致性算法，在存储系统内部维护数据
2. 分布式事务，将任务流程打包成分布式事务，通过分布式事务协议，跨节点地保证事务的ACID特性

> 本文主要描述的是数据的分布式一致性算法，暂未涉及到分布式系统的分布式事务协议。

#### 分布式系统理论

##### CAP理论

分布式系统经典理论，C为一致性(consistency)，A为可用性(availability)，P为分区容错性(partition tolerance)

CAP理论指出，三者不可得兼，只能最多同时满足两条特性，而分布式系统需要的P是必不可少，因此，分布式系统被划分为了CP系统和AP系统。

但是一般实际运行的系统，对于A和C的处理，采取的是折中的方式，在尽可能满足C的情况下，降低对A的影响，也就是常说的BASE理论。

##### BASE理论

BA为基本可用(Basically Available)，S为软状态(Soft state)，E为最终一致性(Eventually consistent)

#### 一致性算法

追求高性能的算法一般都遵循BASE理论，在保证分区容错的情况下，平衡系统可用和数据一致；也有算法保证了严格的数据一致性，一般为类数据库应用，从而提供准确可靠的数据。

##### 分布式网络类型

1. 信任网络：全部由受信任节点组成
2. 非信任网络：分布式公开网络，不排除存在恶意节点。如以太坊网络、比特币网络等。

> 本文主要分析信任网络下的一致性算法，非信任网络的一致性算法，可以参考文章《从远到近认识区块链——共识算法》。



## Paxos

> 论文原文<Paxos Made Simple>  https://lamport.azurewebsites.net/pubs/paxos-simple.pdf

#### 目标

保证向一个系统成功写入value后，该value不会丢失、不被修改、能被正确读取（*写入value，广义上理解为一个事务，可以是任何对系统的持久性操作*）

因此系统能够容错，包括

1. 网络可能中断
2. 节点可能宕机、重启
3. 消息可能延迟、重复、丢失

#### 系统内角色

- Proposer：提议者
- Acceptor：接受者
- Learner：学习者

一个节点可以同时扮演多个角色，在这里不影响整个协议的运行，细节取决于具体实现。

*为了方便理解，类比在工程上，Proposer可以看作系统的对外API，外部系统通过该API影响系统内部（写入value）；Acceptor可以看作系统内的工作节点，负责处理API请求以及数据的保存；Learner可以看作系统中备份节点，某些实现中可以作为只读节点。*

#### 名词解释

- *proposal*：提议，指某次写入value的操作，包含序列号number以及写入值value
- *response*：响应，指acceptor对某次request作出回应
- *accept*：接受，指某个proposal被acceptor采纳
- *chosen*：选中，指某个proposal被集群采纳。定义为集群中大多数（超过一半）的acceptor接受了proposal

#### 基本系统演进

从直观上来说，给定一个目标，系统设计往往从简单开始，发展演进到能够满足需求为止，这也与原文中介绍Paxos算法的思路相同。在以下两个小节中，将从简单到复杂逐步迭代发展算法细节，暂时忽略learner角色。

##### 单acceptor

这是最简单的系统，简单到没有了分布式的语境，因此它不具备容错能力，单点宕机后，系统将处在不可用的状态下。。

##### 多acceptor

这里我们按照原文的推理逻辑，从系统的基本要求，以及产生问题和解决问题的流程，一步一步推进。

-------

> *P1: Acceptor必须接受它收到的第一个prorosal*

这个要求，使得系统能够开始工作，但是问题在于当多个proposer同时提出不同的proposal，各个acceptor可能接受任意proposal，导致可能不会有大多数acceptor接受同一个proposal。接下来讨论对于这个并发问题的解决思路：

1. 常规想法是加一个分布式锁，竞争到锁的proposal能够发起，竞争不到将阻塞；
2. 随之而来的就是分布式锁的设计，如果没有过期时间，假设proposer加锁成功后宕机，系统将被永久阻塞，如果设计过期时间，太长依然会存在系统阻塞问题，太短则会频繁发生锁竞争带来开销；
3. 引入可抢占锁，优先级高的proposal能够抢占锁，简单考虑可以使用自增序列号来为proposal定义优先级，而锁被抽象为acceptor上对于已响应proposal和新发起的proposal的序列号的比较，即last-n和n的比较，last-n小于n时会被接受

有了这个解决方案，P1可以演进到下面的要求：

> *P1^a^: Acceptor接受一个proposal(n, v)，当且仅当它没有响应过任何序列号大于n的prepare(n, v)消息*

----------

不难注意到，既然更大的序列号优先级高，那么新的proposal总是会覆盖老的proposal，当proposal的值v不同时，会与算法的目标（value不被修改）相悖，因此提出新的要求解决这个问题。

> *P2: 如果proposal(n, v)被选中，那么被选中的、具备更高n的proposal(n', v')，v‘的值等于v*

系统选中一个proposal，其基础是acceptor接受，因此将P2的角度换为acceptor，可以进一步描述为：

> *P2^a^: 如果proposal(n, v)被选中，那么被任意一个acceptor接受的、具备更高n的proposal(n', v')，v‘的值等于v*

这个要求内，有一个隐含条件，就是所有的acceptor必须同时收到proposal(n, v)，响应之后，才能进一步响应proposal(n', v')，显然这与系统需要容错矛盾。比如，当某些accptor处于网络不同的状态时，proposal(n, v)发出，随后网络恢复，proposal(n', v')发出，此时由于P1要求，这些accptor会接受proposal(n', v')，尽管其v有可能不同。为了解决该问题，**将对proposal的限制置于流程中更靠前的发出端**，P2^a^进一步演进为：

> *P2^b^: 如果proposal(n, v)被选中，那么任意一个proposer提出的、具备更高n的proposal(n', v')，v‘的值等于v*

要求提出了，但是如何满足呢，论文中使用了归纳法进行了条件说明，将P2^b^进一步改写，这里直接给出结果：

> *P2^c^: 任意n和v，如果proposal(n, v)提出，那么一定存在一个majority集合S，其必然满足以下条件中的一条：*
>
> - *S中不存在接受过小于n的proposal的acceptor*
> - *S中所有接受的小于n的proposal，其n最大的proposal的value值为v*

P2^c^的意义在于，将提出满足P2^b^的proposal的方式明确，即proposer必须感知到**majority中接受的n最大的proposal**（也可能不存在，即没有任何已接受的），且为了简化这个过程，算法中acceptor保证，在收到proposal(n, v)后，不再接受小于n的proposal。

-------

接下来需要设计出满足本节要求的算法，且根据本节P2^c^可以知道，acceptor都必须能够持久存储其接受的proposal中最大的n（准确的说，是其响应的prepare请求中的n，将通过下面一节进行说明），此外proposer也必须记住其发起的proposal中最大的n。

#### 二阶段设计

将算法拆分为两个阶段，使得proposer能够感知majority中接受的n最大的proposal，并通过两阶段能够快速终止无意义（最终肯定无法被选中）的proposal。具体如下：

##### Phase 1 （prepare）

1. proposer选定一个序列号n，并向系统内的acceptor发送准备请求，称为 *prepare(n)*
2. acceptor收到 *prepare(n)* 后，会拒绝低于其响应的最大序列号last-n的请求，接受大于等于last-n的请求；返回acceptor接受的n最大的proposal（如果有），并更新last-n

![paxos-stage-perpare](https://github.com/CanocForYou/canoc.github.io/blob/main/images/paxos-stage-perpare.png)

##### Phase 2 （accept）

1. 当proposer接收到了大多数acceptor对 *prepare(n)* 的响应，那么其会向系统内acceptor发送写请求，称为 *accpet(n, v)*，n是其序列号，v是其接收到的响应中（如果有），n最大的proposal的值，或者自己定义的v（如果没有）；否则终止。当大多数 *accpet(n, v)* 被接收后，写入成功
2. acceptor收到 *accpet(n, v)* 后，会拒绝低于其响应的最大序列号last-n；接受大于等于last-n的请求，并更新last-n

![paxos-stage-accept](https://github.com/CanocForYou/canoc.github.io/blob/main/images/paxos-stage-accept.png)

##### proposer问题

两阶段流程中，均存在判断n并拒绝请求的逻辑，在此基础上可以发现存在两个proposer会彼此打架干扰的情况，导致算法活锁问题的出现。这里一般通过选举和保活等机制让某个proposer代表提出proposal来处理。

#### Learner的工作流程

算法的主流程已经清晰，现在回过头来解决之前忽略的问题，learner是如何"learn"的。由于learner不直接参与value的写入，所以只能依赖acceptor通知，当大多数acceptor接受了某个proposal后，learner将会学习到该proposal，而根据通知的时机和方式不同，给出3种典型方案。

##### 方案一

每当一个acceptor接受proposal时，将会向全部learner广播这一proposal，这种情况下，learner能够最快学习到新的proposal。很显然，这会引起m x n次网络传输（m为大多数acceptor，n为learner数量）

##### 方案二

每当一个acceptor接受proposal时，将会向选定leaner代表（单台）传递这一proposal。网络传输次数下降到了m+n，但是会带来单点故障问题，增加了一轮传输，降低系统的可靠性。

##### 方案三

每当一个acceptor接受proposal时，将会向选定leaner代表（若干）传递这一proposal。网络传输次数为m x k +k x n，避免了单点故障问题，代价是网络传输的增长。

这三种方案大同小异，核心在于广播的规模，在不同的场景、不同的侧重选取不同的方案为佳。除此之外，鉴于总是广播总是有可能丢失，**learner也可以通过将自己额外增加proposer角色来进行proposal的感知**。

#### 总结

Paxos是在信任网络中、多节点、可容错的分布式一致性算法，通过将算法拆解为两阶段，使得其能够快速收敛，可应用在状态机等应用中，让多节点对外表现为单机一样正常工作。

> Google Chubby的作者Mike Burrows说过:
> **这个世界上只有一种一致性算法，那就是Paxos。**

Paxos是分布式一致性算法的基石，众多一致性算法在其基础上被发明，理解Paxos将对我们理解其他分布式一致性算法提供便利。