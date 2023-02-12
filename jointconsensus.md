--raft 联合共识



why use joint

![1675995844429](C:\Users\s00598795\AppData\Local\Temp\1675995844429.png)



![1674962392018](C:\Users\s00598795\AppData\Local\Temp\1675842869664.png)

采用joint consensus中间配置过渡，算法保证Cold和Cnew不能单独作用，避免双主



![1674980282631](C:\Users\s00598795\AppData\Local\Temp\1674980282631.png)

客户端发起joint consensus变更，通过propose将confchange传入raft leader, leader将信息封装在log entry里执行log replication，在leader commit了这个entry之后，生效该配置调raft apply接口，当follower和learner收到该entry的commit心跳之后，生效变更。leader在提交Cold&&Cnew之后，提交Cnew，复用log replication。



--两阶段提交

在ColdCnew被提交后，应用applyConfChhange

在Cnew被提交后，结束joint consensus  // null entry， 是否当做交易





具体流程： // restore 流程？

prop用两阶段提交？

**a.** leader收到msg.prop, append到raftlog unstable（raft会开启协程将unstable复制到wal进行持久化，follower同理）, 然后发送msg.append消息广播给 r.progress（tracker）里面节点（尚未更新，Cold节点）

在`stepLeader`方法处理`MsgProp`时，如果发现`ConfChange`消息或`ConfChangeV2`消息，会反序列化消息数据并对其进行一些预处理。

1. `alreadyPending`：上一次合法的`ConfChange`还没被应用时为真。
2. `alreadyJoint`：当前配置正处于*joint configuration*时为真。
3. `wantsLeaveJoint`：如果消息（旧格式的消息会转为`V2`处理）的`Changes`字段为空时，说明该消息为用于退出*joint configuration*而转到CnewC_{new}Cnew的消息。

根据以上3个条件，拒绝该`ConfChange`的情况有3种：

1. `alreadyPending`：Raft同一时间只能有一个未被提交的`ConfChange`，因此拒绝新提议。
2. `alreadyJoint`为真但`wantsLeaveJoint`为假：处于*joint configuration*的集群必须先退出*joint configuration*并转为$C_{new}，才能开始新的`ConfChange`，因此拒绝提议。
3. `alreadyJoint`为假单`wantsLeaveJoint`为真，未处于*joint configuration*，忽略不做任何变化空`ConfChange`消息。

只需要将该日志条目替换为没有任何意义的普通空日志条目`pb.Entry{Type: pb.EntryNormal}`即可。// 拒绝，不处理



而对于合法的`ConfChange`，除了将其追加到日志中外，还需要修改`raft`结构体的`pendingConfIndex`字段，将其置为[上一条ConfChange.Index,当前ConfChange.Index)[上一条ConfChange.Index, 当前ConfChange.Index)[上一条ConfChange.Index,当前ConfChange.Index)的值（这里置为了处理该`MsgProp`之前的最后一条日志的index），以供之后遇到`ConfChange`消息时判断当前`ConfChange`是否已经被应用。



**b.**  follower根据index，term判断是否accpet，并回复

**c.**   leader收到follower回复的MsgAppResp以后，首先判断如果follower reject了日志，就把Progress的Next减回到Match+1，从已经确定同步的日志开始从新发送日志。如果没有reject日志，就用刚刚已经发送的日志index更新Progess的Match和Next，下一次发送日志就可以从新的Next开始了。然后调用maybeCommit把多数节点同步的日志设置为commited。


**d.**得知该entry commited， Cold new commited ?? how //  leader收到follwer appendresp 计算是否大多数，如果是，生成ready结构体，存进commited字段放到channel里通知应用层，应用层通过commited字段知道是entry成功提交的信息



applytoconfig（该entry entry有confchangev2信息），进入enterjoint，根据配置变更消息更新tracker.config

config

voters[2]

learner



Cnew=incoming=voters[0] ,Cold=outgoing=voters[1]

该流程在`Changer::EnterJoint`中实现： 

- 拷贝当前`ProgressTracker`结构体当前的进度（`Progress`）和配置数据（`Config`）。

- 如果当前有在提交的配置，就返回退出，因为同一时间只能有一个未提交的配置变更。如何判断当前是否有未提交的配置？看`Config`中的`outgoing`（即`voters[1]`）是否为空。我们下面再详细解释。
- 下面，以第一步拷贝的配置数据，生成新的配置数据：

​           将`Config`中的`incoming`数据拷贝到`outgoing`中，即先保存当前的配置到`outgoing`。

​           遍历需要修改的配置，根据不同的操作类型做操作，生成新的配置： 

​             1.如果要删除某节点，调用`Changer::remove`函数： 

​		    incoming中删除该节点。

​	            learner和learnernext集合中删除该节点。

  	 2. 如果增加voter，调用Changer::makeVoter函数： 

- 该节点的进度数据中，`IsLearner`变为`false`。

- 从`Learner`以及`LearnerNext`集合中删除该节点。

- 将节点ID加入`incoming`集合中。

   3.如果增加`learner`，调用`Changer::makeLearner`函数 

  ​	调用Changer::remove函数先删除该节点 

  ​	判断是否在outgoing配置中有该节点，表示该节点是降级节点 

  - 有：表示在新配置下变成了`learner`，但是此时并不能直接变成`learner`，所以这种情况下该节点加入到了配置的`LearnersNext`。
  - 否则，说明是新增节点，直接加入到`Learner`集合中。

​       新数据返回给r.prs，节点更新，leader配置进入ColdCnew阶段       //  通知新raft节点，initraft？

**e.** 等到全部commit Cold和Cnew时 (?是否)， 生成空change的msg.prop， 退出joint consensus    //  how , to be checked， 对比raft 

**f.**  待该entry commited，leave joint 清空progress outgoing集合           

用该entry，是个空的entry confchange v2

调applyconfchange 更新到Cnew

走leavejoint



leavejoint分为两种

auto leave时机可为Cold&Cnew这个entry被commited且被持久化// Coldnew Commited之后，accept ready  发空entry即发cnew



第一种是自动leave，当ColdCnew entry commited， 生成ready通知应用层，此时apply进入joint，switch配置，通知新节点init



当收到pb.MsgStorageApplyResp 调raft appliedto（）， 此时判断是否autoleave，如果是step一个空的confchange通知leave



等到这个entry commited，change为空，用该entry，是个空的entry confchange v2， 调applyconfchange 更新到Cnew



第二种根据应用层手动发空  // dont need 



tips：

//raft调 advance，rn.stepsOnadvance存储的pb，step（）处理，里面应该有空change confchange的pb

//如果是autoleave，会在空entrycommit时候，会往advance里面塞空entry，需要调advance，来发 



结构体：

message 包 entry 包 confchangev2



leader，follower收到commit entry之后都会调applyconfchange生效Cold&new / Cnew



tikv:

1. RawNode 是 raft-rs 库与应用交互的主要界面。要在自己的应用中使用 raft-rs，首先就需要持有一个 RawNode 实例，正如 Node 结构体所做的那样。
2. RawNode 的范型参数是一个满足 Storage 约束的类型，可以认为是一个存储了 Raft Log 的存储引擎，示例中使用的是 MemStorage。
3. 在收到 Raft 消息之后，调用 `RawNode::step`方法来处理这条消息。
4. 每隔一段时间（称为一个 tick），调用 `RawNode::tick`方法使 Raft 的逻辑时钟前进一步。
5. 使用 `RawNode::ready`接口从 Raft 中获取收到的最新日志（`Ready::entries`），已经提交的日志（`Ready::committed_entries`），以及需要发送给其他节点的消息等内容。
6. 在确保一个 Ready 中的所有进度被正确处理完成之后，调用 `RawNode::advance`接口。





进入 Joint 状态本身就是一条 log entry；解除 Joint 状态也是一条 log entry

进入 Joint 状态需要满足前状态 {a,b,c} 的 majority

在 Joint 状态中，选举和 log entry commit 都需满足前状态 {a,b,c} 和后状态 {a,b,d} 的 majorities

如果在 Joint 状态中丢失了 Leader 但仍能满足前后状态的 majorities，就可以选出新的 Leader

// process tracker 给出 投票结果

解除 Joint 状态需要满足前状态 {a,b,c} 和后状态 {a,b,d} 的 majorities



#### 新增pb configchange

原来（sdk proto已经更新）

```
message ConfChange {
    BlockMetadata block = 1;
    ConfOp        op    = 2;
    Consenter[]     node  = 3;
}
```

```
enum ConfOp {
    AddVoter        = 0;
    AddLearner      = 1;
    Remove          = 2;
    UpdateToLearner = 3;
    UpdateToVoter   = 4;
}
```

entry type?

message Entry {
    uint64 term  = 1;
    uint64 index = 2;
    oneof  value {
        BlockMetadata block      = 3;
        ConfChange    confChange = 4;
    }
    bytes signature  = 5;
    bytes blockBytes = 6;
}

```
enum EntryType {
   EntryNormal       = 0;
   EntryConfChange   = 1; // corresponds to pb.ConfChange
   EntryConfChangeV2 = 2; // corresponds to pb.ConfChangeV2
}
```



新增

```
message ConChangeV2 {
   repeated ConfChange change = 1;
   bytes context = 2;
}

//enum ConfChangeTransition {
  //  ConfChangeTransitionAuto         = 0;
 //  ConfChangeTransitionJointImplicit = 1;       // 
 //  ConfChangeTransitionJointExplicit = 2;
}
```



// 不需要，默认自动leave就行

`ConfChange`仅支持“one node a time”的简单算法，不再赘述，仅介绍`ConfChangeV2`中支持的配置变更类型。

`ConfChangeV2`的`Transition`字段表示切换配置的行为，其支持的行为有3种：   //  单节点 和 多节点 流程区别

etcd/raft  apply - enterjoint - 

1. `ConfChangeTransitionAuto`（默认）：自动选择行为，当配置变更可以通过简单算法完成时，直接使用简单算法；否则使用`ConfChangeTransitionJointImplicit`行为。
2. `ConfChangeTransitionJointImplicit`：强制使用*joint consensus*，并在适当时间通过`Changes`为空的`ConfChangeV2`消息自动退出*joint consensus*。该方法适用于希望减少*joint consensus*时间且不需要在状态机中保存*joint configuraiont*的程序。
3. `ConfChangeTransitionJointExplicit`：强制使用*joint consensus*，但不会自动退出*joint consensus*，而是需要etcd/raft模块的使用者通过提交`Changes`为空的`ConfChangeV2`消息退出*joint consensus*。该方法适用于希望显式控制配置变更的程序，如自定义了`Context`字段内容的程序。  

`ConfChangeV2`的`Changes`字段表示一系列的节点操作，其支持的操作有：

1. `ConfChangeAddNode`：添加新节点。
2. `ConfChangeRemoveNode`：移除节点。
3. `ConfChangeAddLearnerNode`：添加learner节点。

// 是否需要updatelearner updatevoter?





#### MajorityConfig

中间态Cold,Cnew 需要 各自的majority

配置中voter的集合（voter即有投票权的角色，包括candidate、follower、leader，而不包括learner），是通过`MajorityConfig`表示的，`MajorityConfig`还包括了一些统计majority信息的方法



```
pub struct MajorityConfig {
    voters: HashSet<u64>,
}
```

主要实现两个方法

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| pub fn committed_index(&self, l: &impl AckedIndexer) -> u64  | 根据给定的`AckedIndexer`计算被大多数节点接受了的*commit index* 。从tracker获得follower的match index， 通过n - (n/2+1)获得commited index |
| pub fn vote_result(&self, check: impl Fn(u64) -> Option<bool>) -> quorum::VoteResult | 根据给定的投票统计计算投票结果。根据tracker voter bool hash表获得投票结果 |



#### JointConfig

incoming表示Cnew，outgoing表示Cold，非joint consensus 为空？？

```
pub struct JointConfig {
    pub(crate) incoming: MajorityConfig,
    pub(crate) outgoing: MajorityConfig,
}
```



| 方法                                           | 描述                                       |
| ---------------------------------------------- | ------------------------------------------ |
| `CommittedIndex(l AckedIndexer) Index`         | 返回Cold，Cnew最小值                       |
| `VoteResult(votes map[uint64]bool) VoteResult` | 返回投票结果，VoteWin,VoteLost,Votepending |



Config

![1675933956283](C:\Users\s00598795\AppData\Local\Temp\1675933956283.png)





使用

raft joint扩缩容支持节点的批量扩容或缩容，支持批量操作中既有增加又有删除，对应操作命令如下： 