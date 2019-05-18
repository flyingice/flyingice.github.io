---
title: Basic Paxos
tag: distributed-system
---

Paxos协议的问题背景这里不展开，总之它从理论上解决了分布式系统中多个节点之间达成一致的问题。与普通的2PC不同，Paxos能容错（非拜占庭模型）。这里整理一下看Lamport的论文[Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)之后想到的一些问题。本篇只涉及Basic Paxos，Multi-Paxos留给以后。 

网上有不少讲Paxos的文章，有些开始学习的时候非常不好理解，我觉得是因为里面很多时候把算法的核心设计逻辑和工程上的实现及优化混为一谈。[NEAT ALGORITHMS - PAXOS](http://harry.me/blog/2014/12/27/neat-algorithms-paxos/)和[Basic Paxos](http://www.beyondthelines.net/algorithm/basic-paxos/)这两篇文章作为起点非常不错，不过要理解最原始的Paxos，终究要回到原论文中去。

Paxos有正确性（safety）和可终止（liveness）两方面的要求：safety的核心是选出**唯一**的值；liveness要求**最终**某个值被选择，但是并不保证花多长时间。协议有三个角色：proposer，acceptor和learner。learner跟协议的正确性没什么关系，可以暂时忽略。

Paxos的设计思路可以参考[一步一步理解Paxos算法](https://mp.weixin.qq.com/s?__biz=MjM5MDg2NjIyMA==&mid=203607654&idx=1&sn=bfe71374fbca7ec5adf31bd3500ab95a&key=8ea74966bf01cfb6684dc066454e04bb5194d780db67f87b55480b52800238c2dfae323218ee8645f0c094e607ea7e6f&ascene=1&uin=MjA1MDk3Njk1&devicetype=webwx&version=70000001&pass_ticket=2ivcW%2FcENyzkz%2FGjIaPDdMzzf%2Bberd36%2FR3FYecikmo%3D)，这篇文章基本跟着原论文的推理思路走的。下面是我想到的一些问题。

- 为什么协议设计成多数派批准？

Paxos分两阶段执行，prepare阶段可以看作争取提议权，accept阶段再真正提议，两阶段通过都需要多数票同意。注意到在acceptor总数一定的情况下，任意两个多数派至少会在一个节点上重合。也就是说，至少有一个acceptor同时参与了这两次决策。proposer可以依赖该acceptor的反馈来决定是否坚持自己的提议，保证了协议的正确性。

- 协议分两阶段执行的意义是什么？

第一阶段的prepare有两个意义：一个是去了解一下有没有已经被选出来的值，另一个是让自己迅速抢占位置来阻止竞争者。两个不同的proposer在开始阶段可能对不同的值进行提议，为了保证唯一性，prepare阶段非常关键。

```wiki
proposer P1/P2, acceptor A1/A2/A3
1) P1: prepare(1) -> A1/A2: ACK
2) P1: accept(1,v1) -> A1/A2: ACK
3) P2: prepare(2) -> A2/A3: ACK
4) P2: accept(2,v1) -> A1/A2/A3: ACK
```

P2其实是想提议accept(2, v2)的，但是prepare之后发现A2已经选择了v1，于是发送accept(n,v)的时候把值由v2改成v1。该行为是协议为了保证正确性强制要求的，最终的结果是P1和P2对值v1达成了共识。

另外注意一下，上面例子中的A1并没有收到P2的prepare(2), 但是依然可以接受accept(2, v1)，这是协议允许的。对于acceptor而言，只要不违反第一阶段作出的两个承诺，它就可以接受第二阶段的accept(n,v)，而且是**必须接受**。关于什么是两个承诺，可以参看文章[架构师需要了解的Paxos原理、历程及实战](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=403582309&idx=1&sn=80c006f4e84a8af35dc8e9654f018ace&scene=0&key=710a5d99946419d9c39ba913a16ee674c6016edfedfc691aa0df9db57d008419c1a96168b861e0ef8b01d6ec76c7e693&ascene=7&uin=MTc0MDg1&devicetype=android-19&version=26030931&nettype=WIFI&pass_ticket=J96esr4md7XLhmfoelhpNAXq73CErFPyQ5BlGEWTtHg=)。

- 关于伪代码中minProposal的理解

不少地方在用伪代码描述([一步一步理解Paxos算法](https://mp.weixin.qq.com/s?__biz=MjM5MDg2NjIyMA==&mid=203607654&idx=1&sn=bfe71374fbca7ec5adf31bd3500ab95a&key=8ea74966bf01cfb6684dc066454e04bb5194d780db67f87b55480b52800238c2dfae323218ee8645f0c094e607ea7e6f&ascene=1&uin=MjA1MDk3Njk1&devicetype=webwx&version=70000001&pass_ticket=2ivcW%2FcENyzkz%2FGjIaPDdMzzf%2Bberd36%2FR3FYecikmo%3D))Paxos协议的时候都说acceptor需要记录minProposal，acceptedProposal和acceptedValue这三个变量。minProposal的这个min是针对proposer来说的，即如果proposer想要acceptor接受prepare(n)，n必须大于minProposal。这里尤其需要注意，在Paxos两阶段中，acceptor判断是否接受请求都是以minProposal为标准的。开始我误以为第二阶段中acceptor是否接受accept(n,v)看的是acceptedProposal。为什么第二阶段看acceptedProposal不行？容易构造如下场景：

```wiki
proposer P1/P2, acceptor A1/A2/A3
1) P1: prepare(1) -> A1/A2/A3: ACK
2) P2: prepare(2) -> A1/A2/A3: ACK
3) P1: accept(1,v1) -> A1/A2/A3: ACK
4) P2: accept(2,v2) -> A1/A2/A3: ACK
```

最后P1和P2都认为自己的提议被接受，产生冲突。根源在于步骤3中所有acceptor都还没有接到P2发出的accept(2,v2)，这时候acceptedProposal还为空值，因而才能回应P1的accept(1,v1)。而如果以minProposal为依据，步骤3中P1的请求会被拒绝。简言之，minProposal是一个面向未来的承诺，虽然现在没有看到id=minProposal的提议，但是将来极有可能会有。

- 是否可能在一组acceptor中同时出现不同的值？

完全可能，考虑以下场景：

```wiki
proposer P1/P2, acceptor A1/A2/A3
1） P1: preapre(1) -> A1/A2/A3: ACK
2） A3下线
3） P2: prepare(2) -> A1/A2: ACK
4） P2: accept(2,v2) -> A1/A2: ACK
5） A3上线
6） P1: accept(1,v1) -> A1/A2: NACK, A3: ACK
```

上述时序是协议允许的，这里重点关注A3。A3因为中途意外离线，根本察觉不到P2，但是这并不影响P2提议的v2被多数派选择。A3在第一步对P1作出了承诺，所以接受了P1的accept(1,v1)。这时候在A3眼里，v1才是最后被选择的值。似乎大家意见出现了分歧，A1和A2选择了v2，A3选择了v1，这是不是一种不一致的状态？这里的关键在于系统是否达成一致并不能单看每个acceptor的状态。acceptor都是井底之蛙，永远只知道自己批准了哪个值。诚然，它的选择会对整个系统的最终决议产生影响，但是却无法知道最终决议到底是什么。系统最后选择的值只有proposer才知道，而判断标准就是有没有得到大多数acceptor的响应。

继续上面的例子，步骤6完成后P2已经认为v2是系统认可的值了（更准确的说是步骤4完成后），P1还蒙在鼓里。P1还需要执行一轮完整的Paxos协议:

```wiki
7) P1: prepare(3) -> A1/A2/A3: ACK
8) P1: accept(3, v2) -> A1/A2/A3: ACK
```

注意比较步骤6和步骤8， P1将自己提议的值从v1换成了v2，原因文章前面提到过。最终P1也认可了v2（放弃了自己的提议v1），和P2达成共识。我们不大希望上述情况出现，但是协议的容错能力还是保证了最后能选出唯一值。

- 协议的可终止如何保证？

Paxos的核心算法并不能满足可终止的要求，所以原论文提出需要指定唯一的distinguished proposer来负责提议，以此来防止两个或多个proposer相互抢占的情况：

```wiki
proposer P1/P2，acceptor A1/A2/A3
1) P1: prepare(1) -> A1/A2/A3: ACK
2) P2: prepare(2) -> A1/A2/A3: ACK
3) P1: accept(1,v1) -> A1/A2/A3: NACK
4) P1: prepare(3) -> A1/A2/A3: ACK
5) P2: accept(2,v2) -> A1/A2/A3: NACK
6) P2: prepare(4) -> A1/A2/A3: ACK
...
```

P1和P2交替提议无限循环，阻止了对方的请求。

工程实现中这个问题无法回避。比如开始有n个节点，编号分别为0到n-1，最初可以人为指定编号最大的节点为distinguished proposer。但是，一旦这个唯一的proposer下线或者网络分区，就会出现多个proposer同时抢主而带来潜在的冲突，冲突回过头来还是需要执行Paxos协议解决。实践中可以采用exponential backoff的技巧外加一个随机数来决定重试的时间间隔。

- 如何实现全局唯一的proposalId？

可以借鉴Google Chubby中实现Paxos的方法（参考[Paxos Made Live - An Engineering Perspective](https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf)）。假设共有n个节点，从0到(n-1)依次编号，那么proposalId=m*n + k （m为本地使用过的最大id，k为节点编号）。这样可以保证每个proposer从互不相交的集合中选取数字，而且本地id自增的步长都是n。

另外也可以像[架构师需要了解的Paxos原理、历程及实战](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=403582309&idx=1&sn=80c006f4e84a8af35dc8e9654f018ace&scene=0&key=710a5d99946419d9c39ba913a16ee674c6016edfedfc691aa0df9db57d008419c1a96168b861e0ef8b01d6ec76c7e693&ascene=7&uin=MTc0MDg1&devicetype=android-19&version=26030931&nettype=WIFI&pass_ticket=J96esr4md7XLhmfoelhpNAXq73CErFPyQ5BlGEWTtHg=)介绍的那样，采取高位时间戳低位机器IP的做法。不过文章里说的时间戳应该指的是单调时钟，因为系统的unix时间戳是无法保证单调递增的。实现单调时钟要么直接有标准类库支持（比如C++ 11之后的[steady_clock](https://en.cppreference.com/w/cpp/chrono/steady_clock)）,要么像[Twitter snowflake](https://github.com/twitter-archive/snowflake/releases/tag/snowflake-2010)一样基于普通系统时钟来实现稳定时钟：

```scala
protected def tilNextMillis(lastTimestamp: Long): Long = {
    var timestamp = timeGen()
    while (timestamp <= lastTimestamp) {
      timestamp = timeGen()
    }
    timestamp
  }

protected def timeGen(): Long = System.currentTimeMillis()
```

最后留个问题，在Lamport的[个人页面](http://lamport.azurewebsites.net/pubs/pubs.html#paxos-simple)上对于论文[Paxos Made Simple](http://lamport.azurewebsites.net/pubs/paxos-simple.pdf)有这么一段话：

*In 2015, Michael Dearderuff of Amazon informed me that one sentence in this paper is ambiguous, and interpreting it the wrong way leads to an incorrect algorithm.  Dearderuff found that a number of Paxos implementations on Github implemented this incorrect algorithm.  Apparently, the implementors did not bother to read the precise description of the algorithm in [[122]](http://lamport.azurewebsites.net/pubs/pubs.html#lamport-paxos).  I am not going to remove this ambiguity or reveal where it is.  Prose is not the way to precisely describe algorithms.  Do not try to implement the algorithm from this paper.  Use [122] instead.*

有人在[Github](https://github.com/Tencent/phxpaxos/issues/166)上对腾讯开源出来的在微信里应用的Paxos算法实现提出了一样的问题。这里说的歧义在论文的什么地方呢？