- [Raft优化](#raft优化)
  - [Raft读操作问题](#raft读操作问题)
    - [Raft Log Read](#raft-log-read)
    - [Raft读操作优化](#raft读操作优化)
      - [Read Index](#read-index)
      - [Lease Read](#lease-read)
  - [成员变更](#成员变更)
    - [多步变更](#多步变更)
    - [单步变更](#单步变更)
    - [joint consensus](#joint-consensus)
    - [Follower Catch Up](#follower-catch-up)
  - [Noop日志](#noop日志)

# Raft优化

## Raft读操作问题 

Raft的写操作是一定要写主，那么读操作呢？

1. 写主读从  
当读操作交给 follower 来执行的话，以下情况就可能会发生错误:
   + 仅需超过半数的 follower 接受日志后，leader 就会对其 commit，但读操作的 follower 正好属于小部分。
   + follower 属于大部分接受了该日志，leader 也 commit 了该日志，但是 leader 的心跳包尚未通知到该 follower。
2. 写主读主
如果leader直接进行读操作，那么以下情况就可能会发生错误:
    + leader 仅仅 commit 了写操作日志，还未 apply 就响应了客户端的写操作，此时客户端发起读操作，就会出错。(状态机落后于 committed log 导致脏读)
    + 发生了脑裂，存在两个 leader，写操作的为多数节点的分区的 leader，但读操作的为少数节点分区的 leader。(网络分区导致脏读)

### Raft Log Read

Raft 通过 Raft Log Read 来使得 Raft 的读操作是线性化的，将 读操作 也走一边 Raft 流程，使得其被 apply 时才响应客户端返回读操作的结果。  
这么做的目的是:

1. 当读操作被 commit后 ，那么先前的写操作一定已经 commit 并 apply 完成，所以此时读操作 apply 时进行读取，不会发生脏读。 
2. 读操作日志走一遍 Raft 流程，就会对其余节点发送 appendEntries 请求，这样可以根据反馈得知自己是不是多数节点的分区的 leader，防止脑裂导致脏读。

### Raft读操作优化

Raft Log Read 使得读操作性能不高，可以采用 Read Index 和 Lease Read 对其进行优化。

#### Read Index

Read Index 通过 发送心跳包 与 等待状态机应用到指定index 来对读操作进行优化。

1. Leader 在收到客户端读请求时，记录下当前的 commit index，称之为 read index。
2. Leader 向 followers 发起一次心跳包而不是 appendEntries 请求，这一步是为了确保领导权，避免网络分区时少数派 leader 仍处理请求。
3. 等待状态机至少应用到 read index（即 apply index 大于等于 read index）。
4. 执行读请求，将状态机中的结果返回给客户端。

以此解决 状态机落后于 committed log 导致脏读 与 网络分区导致脏读 这两问题。

#### Lease Read

Lease Read 相对比 Read Index，通过时间租期代替了 Read Index 的心跳包。

- leader 发送 heartbeat 的时候，会首先记录一个时间点 start，当系统大部分节点都回复了 heartbeat response，那么我们就可以认为 leader 的 lease 有效期可以到 start + election timeout这个时间点。  
- 在此时间点之前的读操作，仅需遵守 apply index 大于等于 read index 即可，无需发送心跳包并等待回复。
- 超过此时间点之后，则使用 Read Index 方案。心跳包可以为 leader 继续续约。 

这么做的原理是因为 follower 会在至少 election timeout 的时间之后，才会重新发生选举，所以下一个 leader 选出来的时间一定可以保证大于 start + election timeout。  
虽然采用 lease 的做法很高效，但仍然会面临风险问题，也就是我们有了一个预设的前提，各个服务器的 CPU clock 的时间是准的，即使有误差，也会在一个非常小的 bound 范围里面，如果各个服务器之间 clock 走的频率不一样，有些太快，有些太慢，这套 lease 机制就可能出问题。

## 成员变更

###  多步变更

多步变更可能会导致多个 leader 同时存在的可能性:

- 对于server2来说，它此时还在Cold配置中，可以通过获得1、2的选票成为leader。
- 对于server3来说，它已经处在Cnew配置中，可以获得3、4、5的选票成为leader。

![多部变更多leader](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/多节点变更多leader.jpg)

### 单步变更

单步变更可以解决上面的问题，因为新的group和majority和旧的group的majority一定是overlap的:

![单步变更overlap](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/单步变更overlap.jpg)

但是单步变更会导致两个问题:

- 当节点为偶数个时，可能会一段时间无法选出leader。
- 单步变更在 leader切换和成员变更 同时进行时可能会出现 丢失一个已提交的变更。

[TiDB 在 Raft 成员变更上踩的坑](https://zhuanlan.zhihu.com/p/342319702)

第一个问题

- 原因：quorum的定义过于死板，仅用了majority。
- 解决：重新定义 quorum。

第二个问题

- 原因：c、d、v与u、a、b出现了网络割裂。
- 解决：新leader必须提交一条自己的term的日志(Noop日志), 才允许接变更日志。

> “一条变更在旧的配置中必须通过quorum互斥, 只能有1个变更被认为是committed.”
其实就是多数派写的原理, 2笔写入必须互相有一个节点交集.
而成员变更也是一种多数派写, 所以也要满足这个约束. raft的bug就是没有满足这个约束造成的.

![单步变更bug](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/单步变更bug.jpg)

### joint consensus

[TiDB 在 Raft 成员变更上踩的坑](https://zhuanlan.zhihu.com/p/342319702)

![joint_consensus](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/joint_consensus.jpg)

joint consensus 可以解决以上所有问题：

- C{old,new}日志还没被提交应用时，只有旧配置，只能是C{old}决定选出一个leader，即对应图中的1阶段。
- 在C{old,new}日志被提交，部分节点应用时，这时原先Leader宕机，下一个选举的节点的集群配置要么是C{old}，要么是C{old,new}，不管是哪种情况都会兼容考虑C{old}的节点来产生Leader，没有节点会单纯基于C{new}来产生Leader。
- 在C{old,new}日志大多数节点commit并应用了，C{new}日志还未提交时，那么此时能发起Leader选举成功的都是C{old,new}的配置，只能是C{old,new}决定选出一个leader。
- 在C{old,new}日志大多数节点commit并应用了，C{new}日志被提交，部分节点应用时，这时原先的Leader宕机，下一个选举的节点的集群配置要么是C{old, new}，要么是C{new}，不管是哪种情况都会兼容考虑C{new}的节点来产生Leader，没有节点会单纯基于C{old}来产生Leader。
- 在C{new}日志大多数节点commit并应用了，那么此时能发起Leader选举成功的都是C{new}的配置，只能是C{new}决定选出一个leader。
- 从始至终只有一个leader存在。

在etcd的实现里，其实没有这么复杂，它也是把配置变更分为了两个阶段来变更：

1. EnterJoint阶段：此时会有两个配置，一个是C{new}配置，一个是C{old}配置，也就是对应到上文中的C{old,new}，在进行决策时会需要C{new}和C{old}都会进行决策，比如在决策投票时，C{new}如果失败了，那么也算是失败，只有两个都成功才算是成功；
2. LeaveJoint阶段：此时会把C{old}配置清理掉，只留下C{new}配置，这个应用完后就只有C{new}进行决策了。只有在EnterJoint执行完后才会执行这个阶段（会有参数配置是由raft自动执行这个阶段还是由调用者手动执行这个阶段）。

### Follower Catch Up

当一个成员新加入到Raft Group中，往往是没有日志的。如果允许新成员以空日志的方式加入到组中，那么就更容易出现可用性的问题。

[Follower Catch Up](https://zhuanlan.zhihu.com/p/546360843)

## Noop日志

![Noop](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/Noop.jpg)

当前term不会提交之前的 term 的日志条目，只会提交当前自己所在的 term 下的日志条目:

- (a)时，S1 选举成为 leader，将 term 为2的 log 分发到 S1 和 S2，未提交就宕机。
- (b)时，S5 选举成为 leader，将 term 为的 log 分发到 S5，未提交就宕机。
- (c)时，S1 选举成为 leader，此时其 term 为4：
  - 如果 S1 将 index 为2的 log 继续分发到大多数并提交的话。
    - (c)时之后，S1 宕机，S5 选举成为 leader(S5 可能本身 term 就很大，又因为 S5 的 lastLogTerm 大于其他所有节点，所以成功成为 leader。也有可能 S5 的 term 为3，小于 S2 和 S3 的 term4，于是 S5 的 term 升为4，但是又因为 S5 的 lastLogTerm 为最新的，所以依旧还是 S5 成为 leader，从这里可以看出 peer 的 term 是为 log 服务的，只有 logTerm 才是最终有用的。)。
    - (d1)时，S5 将 term 为3的 log 分发给其他所有节点(S1 并未写入 term 为4 的 log，所以可以成为 S5 的 follower)，此时 index 为2且 term 为3的 log 就会覆盖 S2 和 S3 已经提交了的 index 为2且 term 为2的 log。这是错误的，已经提交了的 log 不能被覆盖。
  - 如果 S1 将 term 为4的 log 继续分发到大多数。
    - 此时就算 S5 当上了 leader，且有更新的 log，并将 term 为2和4的 log 都覆盖了的话，也没事，因为 term 为2和4的 log 都没提交。
    - 此时如果 term 为2和4的 log 都提交了，那么 S5 就无法当上 leader。

可以得出：

- 通过 Noop 日志，使得大多数节点写入最新 term 的日志，一边顺带提交老 term 的日志，一边可以通过最新 term 的日志防止老 term 的日志提交后被覆盖。
- peer 的 term，最终会趋同，log 的 term，决定了 leader 最终的选举。

[ETCD Raft模块](https://www.cnblogs.com/luohaixian/p/16641100.html)