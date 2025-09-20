---
title: 分布式一致性协议
date: 2019-07-19 20:06:20
tags:
- 分布式
categories:
- 分布式
---


## Paxos算法


Paxos算法核心是一个一致性算法。
在该一致性算法中，有三种参与角色，Proposer，Acceptor和Learner。


### 选定提案算法流程：

![](https://i.imgur.com/ILh54Uy.png)
	
阶段一

1. Proposer选择一个提案编号K，然后想Acceptor的某个超过半数的子集成员发送编号为K的Prepare请求。
2. 如果一个Acceptor收到一个编号为K的Prepare请求，如果K大于该Acceptor已经响应的所有Prepare请求的编号，那么它就会将已经批准过的最大编号提案作为响应反馈给Proposer；如果该Acceptor之前未批准过提案，那么直接返回空响应。**该Acceptor承诺不会再批准小于K的提案，设置接收的提案值为K。**

阶段二：

1. 如果Proposer收到半数以上的Acceptor对于其发出的编号为K的Prepare请求响应，那么它就会发送一个针对[K,V]提案的Accept请求给Acceptor。(V为返回的所有响应中编号最大提案的值)
2. 如果Acceptor收到这个[K,V]提案的Accept请求，只要该Acceptor尚未对编号大于K的Prepare请求作出响应，它就可以通过提案。


### 提案的获取

方案一

一旦Acceptor批准了一个提案，就将该提案发送给所有Learner。
需要让每个Acceptor与所有Learner逐个进行通信，通信次数至少为二者乘积。

方案二

所有的Acceptor将提案批准情况统一发送给一个特定的Learner，它来负责通知其他的Learner。
问题：主Learner随时可能出现故障

方案三

Acceptor将批准的天发送给特定的Learner集合


### 实例

#### Prepare 阶段
下图演示了两个 Proposer 和三个 Acceptor 的系统中运行该算法的初始过程，每个 Proposer 都会向所有 Acceptor 发送 Prepare 请求。

![](https://i.imgur.com/lvlzG5p.png)

当 Acceptor 接收到一个 Prepare 请求，包含的提议为 [n1, v1]，并且之前还未接收过 Prepare 请求，那么发送一个 Prepare 响应，设置当前接收到的提议为 [n1, v1]，并且保证以后不会再接受序号小于 n1 的提议。

如下图，Acceptor X 在收到 [n=2, v=8] 的 Prepare 请求时，由于之前没有接收过提议，因此就发送一个 [no previous] 的 Prepare 响应，设置当前接收到的提议为 [n=2, v=8]，并且保证以后不会再接受序号小于 2 的提议。其它的 Acceptor 类似。

![](https://i.imgur.com/euXdPJZ.jpg)

如果 Acceptor 接收到一个 Prepare 请求，包含的提议为 [n2, v2]，并且之前已经接收过提议 [n1, v1]。如果 n1 > n2，那么就丢弃该提议请求；否则，发送 Prepare 响应，该 Prepare 响应包含之前已经接收过的提议 [n1, v1]，设置当前接收到的提议为 [n2, v2]，并且保证以后不会再接受序号小于 n2 的提议。

如下图，Acceptor Z 收到 Proposer A 发来的 [n=2, v=8] 的 Prepare 请求，由于之前已经接收过 [n=4, v=5] 的提议，并且 n > 2，因此就抛弃该提议请求；Acceptor X 收到 Proposer B 发来的 [n=4, v=5] 的 Prepare 请求，因为之前接收到的提议为 [n=2, v=8]，并且 2 <= 4，因此就发送 [n=2, v=8] 的 Prepare 响应，设置当前接收到的提议为 [n=4, v=5]，并且保证以后不会再接受序号小于 4 的提议。Acceptor Y 类似。
![](https://i.imgur.com/90o9cLJ.jpg)


#### Accept 阶段
当一个 Proposer 接收到超过一半 Acceptor 的 Prepare 响应时，就可以发送 Accept 请求。

Proposer A 接收到两个 Prepare 响应之后，就发送 [n=2, v=8] Accept 请求。该 Accept 请求会被所有 Acceptor 丢弃，因为此时所有 Acceptor 都保证不接受序号小于 4 的提议。

Proposer B 过后也收到了两个 Prepare 响应，因此也开始发送 Accept 请求。需要注意的是，Accept 请求的 v 需要取它收到的最大提议编号对应的 v 值，也就是 8。因此它发送 [n=4, v=8] 的 Accept 请求。

![](https://i.imgur.com/DhgPWlP.png)



#### Learn 阶段
Acceptor 接收到 Accept 请求时，如果序号大于等于该 Acceptor 承诺的最小序号，那么就发送 Learn 提议给所有的 Learner。当 Learner 发现有大多数的 Acceptor 接收了某个提议，那么该提议的提议值就被 Paxos 选择出来。

![](https://i.imgur.com/vROZJFn.jpg)



## Raft算法



> https://github.com/CyC2018/CS-Notes/blob/master/notes/%E5%88%86%E5%B8%83%E5%BC%8F.md#%E4%BA%94paxos

	

