# TinyKV

* 空日志、幽灵复现、联合共识、单步变更

## 2a

## 2b

## 2c

## 3a

project3a仅仅涉及基础raft算法中的领导者变更和成员变更的实现。

### 领导者变更

着重关注两个命令：MsgTransferLeader &  MsgTimeoutNow

MsgTransferLeader：local message

* 由现任领导者接收并处理，tinykv允许非领导者节点自己发起TransferLeader，所以候选者和跟随者也要处理此消息
* 处理过程
  * 判断转移目标是否存在，检查转移目标是否符合条件
  * 如果不符合，help him，期间不再接收proposal
  * 如果/直至符合，向其发送 MsgTimeoutNow

```
为什么存在自己到自己的trans，没懂
trans到底谁发起的？
```

MsgTimeoutNow：

* 由转移目标接收和实现，可能是候选者或者跟随者
* 处理过程
  * 判断自己是否存在（可能在成员变更时被删除）
  * 如果存在则发起选举

若转移目标一开始不符合条件，那么现任领导者合适或者转移目标变为符合条件：在获得转移目标的日志回复和心跳回复时判断其是否已经满足条件

转移期间收到了新的转移目标的请求：覆盖旧的转移目标；收到了同样的转移请求：忽略

### 成员变更

单步变更，而不是联合共识

这部分仅仅涉及raft基础算法中的成员变更，不考虑成员变更日志的同步/提交和应用（其实这里是应用过程的一部分）

addNode

* 更改本raft节点存储的成员信息
* 如果本raft节点是leader，向新增加的节点发送心跳

removeNode

* 更改本raft节点存储的成员信息
* 如果本raft节点是leader，尝试更新commit（推进commit并发送）

## 3b

project3b主要任务是：完成admin commands的propose和apply（CompactLog在2c已经实现）

### TransferLeader

比较简单。TransferLeader不需要在raft group中进行同步，直接调用RaftGroup.TransferLeader()

### RegionEpoch

It will be used to guarantee the latest region information under network isolation that two leaders in one Region.（没懂）

在应用所有种类的request之前，都要先检查RegionEpoch和key的信息

非admin命令（get put delete snap）

* CheckRegionEpoch
* CheckKeyInRegion

change peer

split



问题：

```go
RaftCmdRequest的region信息是哪里来的呢？
如果是客户端，那么客户端怎么更新region信息？
```

### ChangePeer

即成员变更/Conf Change。

propose:

* 检查PendingConfIndex，不允许在未应用日志中存在多个成员变更日志
* 如果仅剩两个节点，且leader要删除自己，则先进行领导者转移，下次再成员变更

apply

* 首先检查RegionEpoch todo
* 如果是增加节点
  * 判断该节点是否已经存在
  * 增加region.Peers和peerCache信息
* 如果是删除节点
  * 判断该节点是否存在
  * 如果删除的节点正好是自己（没懂）
  * 删除region.Peers和peerCache信息
* 更新RegionEpoch信息
* 更新storeMeta
* 持久化region信息
* 应用到raft层
* 如果本节点是leader，还要通知调度器，更新信息

```
For executing AddNode, the newly added Peer will be created by heartbeat from the leader, check maybeCreatePeer() of storeWorker. At that time, this Peer is uninitialized and any information of its Region is unknown to us, so we use 0 to initialize its Log Term and Index. The leader then will know this Follower has no data (there exists a Log gap from 0 to 5) and it will directly send a snapshot to this Follower.
（没懂）
maybeCreatePeer没懂
联合共识比单步变更的好处
```

### Split

apply

* 检查RegionEpoch todo
* 检查CheckKeyInRegion todo
* 判断peer数量
* 修改旧region信息，建立新的region
* 更新router和storeMeta信息
* 持久化region信息
* 如果本节点是leader，还要通知调度器，更新信息

问题：

3b:5-4-5的讨论

3c:到底是哪里的RegionHeartbeat()

```
Every scheduler should have implemented the Scheduler interface, which you can find in /scheduler/server/schedule/scheduler.go. The Scheduler will use the return value of GetMinInterval as the default interval to run the Schedule method periodically. If it returns null (with several times retry), the Scheduler will use GetNextInterval to increase the interval. By defining GetNextInterval you can define how the interval increases. If it returns an operator, the Scheduler will dispatch these operators as the response of the next heartbeat of the related region.
没懂
```

bug修复：

```
snap返回region的指针，导致出错

问题：为什么拿着过时的region进行scan不会出问题
```

3b两次汇报的笔记本记录（还有一次是纸质记录的）；同学们的ppt

[https://docs.qq.com/pdf/DQVl2ZlhBYllkR2lR?](https://docs.qq.com/pdf/DQVl2ZlhBYllkR2lR?)

[https://docs.qq.com/pdf/DQUZnYnhqZWprUldz?](https://docs.qq.com/pdf/DQUZnYnhqZWprUldz?)
