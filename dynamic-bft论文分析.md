## 简述

why n=3f+1,
节点总数是n，其中作恶节点有f，那么剩下的正确节点为n - f，意味着只要收到n - f个消息就能做出决定（所以后面要对f做出限定条件），但是这n - f个消息有可能由f个是由作恶节点（作恶节点也可以什么都不干）冒充的，那么正确的消息就是n - f - f（最恶劣的情况下）个，为了多数一致，正确消息必须占多数，也就是n - f - f > f但是节点必须是整数个，所以n最少是3f+1个


2022 dymic bft简述了动态退出加入流程，并没有关于批量节点的加入退出限制，基本流程如下
1. client obtainconfig， 发送request（含cid）
2. vo c0 leader prepare 阶段收到request，发现是一个membership request, 调用init(),  如果是add，通知新节点起来为learner（让learner同步区块）， remove不变
3. 经过pre-commit,commit
4. leader收到2f+1个投票 确认request， 调用deliver()事件，cid++, Mc = Mc  +/- pj, 新节点可投票， remove节点移除，广播消息，并给新view leader发送viewchange消息
5. replica收到消息后，执行deliver（），更新本地区块，对新view leader发送viewchange消息
  6.新leader收到2f+1 viewchange 开启新view，接受request重复1

libra-bft论文中给出公式满足前后quorum即可批量增删扩缩容
f1 = (N1/3)向上取整 − 1, f2 = （N2|/3）向上取整 − 1 
满足|N1 ∩ N2| > f1 + f2 + max(f1, f2)即可

公式解析：
对于只增操作，

  3f+1 可增加1-2个

  3f+2 可增加1个

  3f+3 可增加3个

对于只减操作，

 3f+1 可减1个

 3f+2 可减2个

 3f+3可减3个






活性，安全性，实现细节见reference

## 结论

扩缩容实现由一个bft对一个配置变更请求达成共识后，在下一配置阶段增加/删除节点，集群满足前后quorum一致的情况下允许批量增删扩缩容，同一变更请求可以包含多个增加和多个删除。

## To solve

假设性能不存在瓶颈，对于raft来说，批量增加节点的限制在于可能形成脑裂，双主。对于bft来说，就算增加100个节点，只会导致容错节点数量增加，不构成影响。不一定遵循该公式。

## Reference

[1] Reconfiguration of the Libra Blockchain by Self-Reflecting Transactions 

[2] Foundations of Dynamic BFT in 2022 IEEE Symposium on Security and Privacy (SP) 
