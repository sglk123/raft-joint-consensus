# 联合共识算法扩缩容

## 介绍

联合共识(joint consensus)扩缩容方案基本遵从Raft论文（见reference引用），借鉴部分开源项目的实现，支持Raft共识集群一次变更多个成员，变更操作支持增加，删除. 在一次变更中，支持不同变更类型. 本方案支持在进行成员变更的同时继续响应客户端的请求，保证了共识算法的可用性。



## 算法简析

在raft集群中发生增删节点的核心问题是发生了脑裂，产生了多个leader，导致的数据不一致。所以joint算法的目的是为了保证在配置变更过程中只有一个leader既不能存在同一时刻满足Cold和Cnew的quorum.见下图：

joint提供了一种中间态，这种状态会保存新配置的voter节点和旧配置的voter节点，需要同时满足新旧配置的quorum.

例子：

假设原raft集群有节点a,b,c，现在发送配置变更消息新增4个节点d,e,f,g. 既存在时刻，当a是leader，d/e/f/g可能在收到leader心跳之前满足集群的quorum生成新的leader，既存在两个leader。对于新节点来说只要进入joint需满足新老配置的quorum就可以避免双主。 



下图是raft论文中joint流程

![1675995844429](C:\Users\s00598795\AppData\Local\Temp\1675995844429.pn

![1674962392018](C:\Users\s00598795\AppData\Local\Temp\1675842869664.png)





## 算法层结构体方法说明

#### MajorityConfig

中间态Cold,Cnew 需要 各自的majority

配置中voter的集合（voter即有投票权的角色，不包括learner），是通过`MajorityConfig`表示的，`MajorityConfig`还包括了一些统计majority信息的方法

```
pub struct MajorityConfig {
    voters: HashSet<u64>,
}
```

主要实现两个方法

| 方法                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| pub fn committed_index(&self, l: &impl AckedIndexer) -> u64  | 根据给定的`AckedIndexer`计算被大多数节点接受了的*commit index* 。从tracker获得follower的match index， 通过n - (n/2+1)获得满足quorumm的需要被commited的index |
| pub fn vote_result(&self, check: impl Fn(u64) -> Option<bool>) -> quorum::VoteResult | 根据给定的投票统计计算投票结果。根据tracker的voter map获得投票结果 |

根据quorum返回投票结果与原先一致



#### JointConfig

incoming表示Cnew，outgoing表示Cold，非joint outgoing为空

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

Cold == win && Cnew == win， return win

Cold == lost | Cnew == lost， return lost

others， return pending



#### Configuration

```
pub struct Configuration {
    pub(crate) voters: JointConfig,
    pub(crate) learners: HashSet<u64>,
    pub(crate) learners_next: HashSet<u64>,
}
```

![1675933956283](C:\Users\s00598795\AppData\Local\Temp\1675933956283.png)



#### Tracker

每个节点维护一个Configuration

```
pub struct Tracker {
    pub config: Configuration,

    pub nodes: HashMap<String, NodeStatus>,

    pub votes: HashMap<String, bool>,

    pub voters: Vec<String>,
}
```



//initial state pb

//减少接口，发送空entry，apply，尽量在raft core内部实现

// confstate 放在state store里, 让外部取

#### ApplyConfigChange

```
pub fn apply_conf_change(&mut self, cc: &ConfChangeV2) -> Result<ConfState，String> 
```

通过入参区分enter / leave

1. 如果是空的confchange,走leave joint
2. 不为空的confchange则走enterjoint



## 流程说明

**a.** leader收到提案消息, append到本地, 然后发送msg.append消息广播给 tracker里面节点（尚未更新，Cold节点）

在leader处理`Proposal`时，如果发现`ConfChange`消息，会反序列化消息数据并对其进行一些预处理。

1. `alreadyPending`：上一次合法的`ConfChange`还没被应用时为真。
2. `alreadyJoint`：当前配置正处于*joint configuration*时为真。
3. `wantsLeaveJoint`：如果消息的`Changes`字段为空时，说明该消息为用于退出*joint configuration*而转到CnewC_{new}Cnew的消息。

根据以上3个条件，拒绝该`ConfChange`的情况有3种：

1. `alreadyPending`：Raft同一时间只能有一个未被提交的`ConfChange`，因此拒绝新提议。
2. `alreadyJoint`为真但`wantsLeaveJoint`为假：处于*joint configuration*的集群必须先退出*joint configuration*并转为$C_{new}，才能开始新的`ConfChange`，因此拒绝提议。
3. `alreadyJoint`为假单`wantsLeaveJoint`为真，未处于*joint configuration*，忽略不做任何变化空`ConfChange`消息。

满足拒绝条件的entry不处理，直接丢弃。 



而对于合法的`ConfChange`，除了将其追加到日志中外，还需要修改`raft`结构体的`pendingConfIndex`字段等于当前raftlogindex+1，以供之后遇到`ConfChange`消息时判断当前`ConfChange`是否已经被应用。



**b.**  follower根据index，term判断是否accpet，并回复

**c.**   leader收到follower回复的MsgAppResp以后，首先判断如果follower reject了日志，就把Progress的Next减回到Match+1，从已经确定同步的日志开始从新发送日志。如果没有reject日志，就用刚刚已经发送的日志index更新Progess的Match和Next，下一次发送日志就可以从新的Next开始了。然后调用maybeCommit把多数节点同步的日志设置为commited。


**d.** 通过当前raft算法output接口得知该entry commited，外部应用层调用算法applyconfigchange() 改变raft配置状态，进入joint.



该流程在`ConfChanger::EnterJoint`中实现： 

- 拷贝当前`Tracker`结构体当前的进度（`Process）和配置数据（`Config）。

- 如果当前有在提交的配置，就返回退出，因为同一时间只能有一个未提交的配置变更。
- 下面，以第一步拷贝的配置数据，生成新的配置数据：

​           将`Config`中的`incoming`数据拷贝到`outgoing`中，即先保存当前的配置到`outgoing`。

​           遍历需要修改的配置，根据不同的操作类型做操作，生成新的配置： 

​          1.如果要删除某节点，调用`Changer::remove`函数： 

​		    incoming中删除该节点。

​	            learner和learnernext集合中删除该节点。

  	 2. 如果增加voter，调用Changer::makeVoter函数： 

  		该节点的进度数据中，`IsLearner`变为`false`。

			从`Learner`以及`LearnerNext`集合中删除该节点。
	
			将节点ID加入`incoming`集合中。

​        3.如果增加`learner`，调用`Changer::makeLearner`函数 

​		调用Changer::remove函数先删除该节点 

​		判断是否在outgoing配置中有该节点，表示该节点是降级节点 

- 有：表示在新配置下变成了`learner`，但是此时并不能直接变成`learner`，所以这种情况下该节点加入到了配置的`LearnersNext`。
- 否则，说明是新增节点，直接加入到`Learner`集合中。

  



**e.**  通过算法接口applytoconfig()获得confstate返回值，通知新节点初始化raft服务。

**f.**  发送一个空的change的confchangev2消息给raft leader，待该entry committed之后，调用applyconfigchange(this entry)更新raft配置状态,退出joint。此时remove节点中止raft服务。






## rust侧结构体&&proto说明

#### 新增pb&&更改

原来（sdk更新）

```
message ConfChange {
    BlockMetadata block = 1;
    ConfOp        op    = 2;
    Consenter     node  = 3;                
}
```

```
enum ConfOp {
    AddVoter        = 0;                      
    AddLearner      = 1;
    Remove          = 2;
    UpdateToLearner = 3;              // unused
    UpdateToVoter   = 4;              // unused
}
```



更改

```
message Entry {
    uint64 term  = 1;
    uint64 index = 2;
    oneof  value {
        BlockMetadata block      = 3;
        ConfChangeV2  confChangeV2 = 4;   //原来的保留 新增
    }
    bytes signature  = 5;
    bytes blockBytes = 6;
}
```



新增

用于收到的提案结构体

```
message ConfChangeV2 {                      //JointConfChange
   repeated ConfChange change = 1;
}

```



调用applyconfchange返回给算法外部的结构体

```
message ConfState {
    repeated uint64 voters = 1;
    repeated uint64 learners = 2;
    repeated uint64 voters_outgoing = 3;
    repeated uint64 learners_next = 4;
}
```



新增apply_conf_change接口

```
pub trait CoreInterface {
 ...
 fn apply_conf_change(&mut self, cc: ConChangeV2) -> Result<ConfState, String>;
}
```





### Reference

[1] Ongaro D, Ousterhout J. In search of an understandable consensus algorithm (extended version)[J]. Retrieved July, 2016, 20: 2018.

[2] Ongaro D. Consensus: Bridging theory and practice[D]. Stanford University, 2014.