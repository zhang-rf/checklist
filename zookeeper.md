# 参考
版本：zookeeper-3.6.1

# 角色
* Leader
* Learner
    * Follower
    * Observer（非Quorum，不参与投票）

# Leader选举

```java
class Vote {
    int version = 0x0;
    long id = self; // Leader ID
    long zxid = 0; // ZK事务ID
    long electionEpoch = 1; // 正在选举的epoch
    long peerEpoch = 0; // 当前epoch
}
```

QuorumPeer启动后，通过`FastLeaderElection.lookForLeader()`查找Leader。

首先投票给自己，并向其他QuorumPeer发送Notification，QuorumPeer收到外部选票后，根据`electionEpoch > zxid > id`的优先级，判断是否更新自己的选票，更新投票后将再次向其他QuorumPeer发送Notification。直到超过一半的QuorumPeer都投票给同一Leader后，等待`finalizeWait（200ms）`，如果出现新的选票则继续进行选举，否则选举完成。

选举完成后，Leader进入`Leader.lead()`，Follower进入`Follower.followLeader()`。

Leader向所有Follower发起`NEWLEADER`包，并在`tickTime x initLimit`时间范围内等待超过半数的Follower响应`ACK`包，之后Follower调用`Learner.syncWithLeader()`进行`NONE/DIFF/SNAP/TRUNC`同步，至此集群完成初始化，ZabState进入`BROADCAST`状态。

Leader每`1/2 tickTime`向Follower发送`PING`包，如果在`tickTime x syncLimit`时间范围内，没有超过半数的Follower响应`PING`包，则重新进入选举阶段。Follower在超过`tickTime x syncLimit`时间后没收到`PING`包，也重新进入选举阶段。（electionEpoch++）

# 消息广播

消息广播类似二阶段协议，分为Propose和Commit两个阶段。

Leader首先向所有保持连接的Follower（`forwardingFollowers`）发送`PROPOSAL`包，并在包中携带修改内容，Follower收到`PROPOSAL`包后，将变更记录到日志中，并以`ACK`包回应Leader。

Leader在收到超过半数的`ACK`包后，向所有保持连接的Follower（`forwardingFollowers`）发送`COMMIT`包，至此Leader完成此次消息广播，`COMMIT`包由TCP协议确保送达。Follower在收到`COMMIT`包后，完成此次消息写入。如果Follower长时间没有收到`COMMIT`包，Follower会定时重新向Leader发送本次消息的`REQUEST`包以重启本次消息广播。

以上二阶段必须在`tickTime x syncLimit`时间范围内完成，否则此次消息广播失败，并重新进入选举阶段。

# 一致性

因为消息广播只需超过半数QuorumPeer同意，并且`COMMIT`包的传输也需要时间，再或者是当前QuorumPeer已经出现网络问题，但是还没有达到`tickTime x syncLimit`超时时间。如果客户端连接的不是Leader，那么还是可能读到过期的数据。为此，客户端提供了`ZooKeeper.sync()`方法，可保证读取到调用`sync()`方法之前的所有变更。

客户端的`sync`请求首先将被对应QuorumPeer转发给Leader，Leader判断`outstandingProposals`集合是否为空，如果为空说明所有消息都已提交，Leader直接回复`SYNC`包给对应的QuorumPeer，QuorumPeer再响应给客户端。如果不为空，说明有正在Propose的消息，将SYNC请求以`lastProposed`为Key放入`pendingSyncs`中，等待`lastProposed`为止的所有Proposal都Commit后，回复`SYNC`包给对应的QuorumPeer。

`sync`的原理在于TCP协议保证了顺序性，所以QuorumPeer收到`SYNC`包前，必然已经收到了更早发送的`COMMIT`包。