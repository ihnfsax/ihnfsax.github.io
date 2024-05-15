---
title: "论文阅读：Raft 共识算法"
date: 2024-02-26
categories:
  - Distributed System
tags:
  - Consensus Algorithm
---

> 本文是对 Raft 论文 *In Search of an Understandable Consensus Algorithm (Extended Version)* 的翻译和解释。

### 1 Raft 的出发点与特点

作者解释 Raft 的出发点是提供一个**比 Paxos 更易理解、实现、应用**的共识算法。为此，Raft 做出一些努力，包括分解（将 leader 选举、日志复制和安全机制作为独立的组件），状态空间 (state space) 缩小（与 Paxos 相比，Raft 减少了非确定性和服务器之间可能的不一致性）。

Raft 相比其他共识算法的鲜明特点是：

- **Strong Leader**：Raft 算法采用了比其他共识算法更加强势的 leader 模型。例如，在 Raft 中，日志条目只从领导者流向其他服务器。这种单向流动简化了复制日志的管理，并使 Raft 的逻辑更加直观。
- **Leader election**：Raft 利用随机定时器来选择领导者。这种方法只是在任何共识算法所必需的心跳机制上增加了一小部分复杂性，同时能够简单而迅速地解决冲突。
- **Membership changes**: 对于集群成员的变更，Raft 引入了一种新的 *joint consensus* 机制。在集群配置变更期间，两个不同配置的多数派重叠，这使得集群能够在不中断服务的情况下进行成员变更。这种机制确保了在过渡期间系统的稳定性和一致性。

### 2 复制状态机

复制状态机是分布式系统中一种常见的技术，用于提高系统的容错能力。例如，像 GFS、HDFS 和 RAMCloud 这样的大型分布式系统，通常采用独立的复制状态机来处理 leader 选举，并存储在 leader 发生故障后仍需保持的数据。这些复制状态机可以确保关键配置信息在 leader 选举或故障转移过程中不丢失，从而保持系统的稳定运行。著名的复制状态机实例包括 Chubby 和 ZooKeeper。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-1.png" | relative_url }}" width=400 alt="paper-fig-1" />

关于图 1 有几点要注意：

- 每台机器存储一个包含命令的日志，这些命令按顺序被状态机执行。
- 状态机是确定的，只要每个日志相同，每个状态机就会产生相同的状态和相同的输出序列。
- 使所有日志保持一致是共识模块（算法）的职责。共识模块接收客户端的命令，将它们添加到日志上。它还要和其他服务器上共识模块通信，确保每个日志相同。

### 3 Raft 基础

#### 安全属性

Raft 保证以下安全性要求，即在 Raft 集群内，每时每刻以下属性都成立：

- **Election Safety**：任意一个任期 (term) 内，最多只会有一个 leader 被选出。
- **Leader Append-Only**：一个 leader 从不覆写或删除日志上的条目，它只新增条目。（但是条目可能在 leader 转为 follower 后被删除或覆写）
- **Log Matching**：如果两个日志包含相同索引和任期的条目，那么这两个日志在该索引及其之前的条目都一样。
- **Leader Completeness**：如果一个日志条目在特定的任期中被提交，那么这个条目将会出现在所有更高编号任期的 leader 的日志中。
- **State Machine Safety**：如果一个服务器已经将给定索引处的日志条目应用到其状态机，那么没有其他服务器会应用同一个索引的不同日志条目。

其中 **State Machine Safety** 是 Raft 的核心安全属性（也是 Paxos 共识算法的安全性要求）。

#### 服务器状态

一个 Raft 集群包含多个机器：5 是一个经典数目，可以容忍两台服务器错误。任意时刻，一台服务器处于三种状态 (state) 之一：**leader, follower, candidate**。三种状态的转移条件如图 4 所示。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-4.png" | relative_url }}" width=500 alt="paper-fig-4" />

三种状态的服务器分别遵守如下规则：

- **Leader** 处理所有客户端请求（follower 如果收到客户端请求，重定向到 leader，这种通信不属于 Raft 考虑范畴）。通常 leader 维持身份直到发生故障。
- **Follower** 是被动的，从不发起请求，只响应其他服务器请求。Follower 长时间未与其他服务器通信，就变为 candidate。
- **Candidate** 从大多数服务器收到选票 (vote) 后变为 leader。

#### 任期

Raft 将时间划分为长度不定的**任期 (term)**，每个任期都开始于 **election**，此时一个或多个 candidate 竞选 leader。要么只有一个 candidate 胜出，成为任期剩下时间唯一的 leader，要么没有一个 candidate 胜出 (a split vote)，本次任期直接结束。图 5 展示了上述提到的两种可能性。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-5.png" | relative_url }}" width=500 alt="paper-fig-5" />

我们将选举成功后时间称为 **normal operation**。**在 normal opearation 内，只会有一个 leader，剩余的服务器都是 follower。** 而任期的转换可能在不同的时间被不同的服务器观察到。

每个任期依次被**连续的整数**标记，我们记为**任期号 (term number)**。每个服务器都存储它所认为的**当前任期号 (current term number)**，显而易见的是，当前任期号随时间单调递增。每次服务器之间通信时，都会交换当前任期号，并更新为其中更大的当前任期号。

- 若 leader 和 candidate 发现自己的当前任期号更小，它立即转为 follower。
- 服务器忽略任何收到的任期号较小的，即过时的请求。

#### RPCs

Raft 服务器间通过 RPC 通信。共识算法部分包含两种 RPC 调用：`RequestVote` 和 `AppendEntries`：

- `RequestVote` 由 candidate 在 election 期间调用。
- `AppendEntries` 由 leader 调用，用于复制日志和心跳。

还有一种 RPC 用于传输快照，将在后面介绍。

### 4 选举

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-4.png" | relative_url }}" width=500 alt="paper-fig-4" />

如图 4 所示，当服务器启动时，它的状态是 follower，只要它能不断收到**来自 leader 和 candidate 的 RPC**，它就会保持着 follower 状态。为了维持自己的权威，leader 会定期向 follower 发送**心跳消息**（空的 `AppendEntries` RPC）。注意这里细微的差异：follower 收到来自 candidate 的消息也会保持 follower 状态，但只有 leader 需要发送心跳消息。

当一个 follower 在规定时间内未收到来自其他服务器 (leader/candidate) 任何 RPC 请求，它就发起选举。我们称为这段超时时间为**选举超时时间 (election timeout)**。注意这个名称在其他地方也要用到。

发起选举的 follower 首先转换为 candidate，然后做以下几件事：

- 将当前任期号递增。
- 投选票给自己（不需要网络通信）
- 重置选举计时器（后面会讲）
- 并行地发送 `RequestVote` RPC 给其他所有服务器。

Candidate 将保持它的状态直到下面三种情况之一发生：

- 在选举中胜出。
- 其他服务器成为了 leader。
- 一段时间过去也没有胜者。

当 candidate 收到来自大多数服务的选票，且回复消息中的任期号和自己的任期号一样时，它在选举中胜出，成为 leader。**每个服务器依照先来先服务的规则给至多一个服务器选票**（candidate 投给自己）。具体规则还要考虑各种情况，我们在后续讲图 2 时说明。

等待选票时，candidate 可能收到 `AppendEntries` RPC。如果这个 “leader” 的任期号大于等于自己的任期号，则 candidate 认可这个 leader 的合法性并转换为 follower 状态。如果小于，candidate 拒绝该消息并保持原状。

当 candidate 等待了规定时间后也无法集齐大多数的选票时，它会增大自己的任期号，重新开始一轮选举，发送新的 `RequestVote` RPC。Raft 把 candidate 的超时时间也称为 election timeout。选举失败意味着发生了 split vote，如果没有特殊设置，split vote 可能无限地发生。某种意义上，Raft 也存在活锁。

Raft 使用随机选举超时 (**randomized election timeouts**) 来降低 split vote 出现的概率。Raft 会从一个合理区间内 (e.g. 150-300ms) 随机选择一个超时时间。这种随机化应用在了两种 election timeouts 上来减少 split vote：

1. **Follower 等待随机的无通信时间后，开启选举。** 此举一定程度避免多个 follower 扎堆地超时，从而使超时的那个服务器更容易选举成功。
2. **Candidate 等待随机的选票不足时间后，重启选举，重置随机计时器。** 此举降低新的选举轮次中 split vote 出现的概率。

作者提到，他们尝试过使用 rank 机制来选举 leader（每个服务器有唯一的 rank number，rank number 大的更容易成为 leader），但是发现容易出现可用性问题。类似 rank 机制的缺失，说明 Raft 无法提供一个重要特性：根据某种偏好增大特定服务器成为 leader 的机会。

### 5 日志复制

Leader 将接收到的客户端发来的命令新增为日志上的一条新条目，然后并行地发送 `AppendEntries` RPC 给其他服务器。当条目被安全的复制后（下面描述），leader 将条目应用到它的状态机，并将结果返回给客户端。

只要 follower 因为某种原因无法复制条目，leader 就会一直向它重发 `AppendEntries` RPC 直到所有服务器都复制成功（甚至在 leader 已经应用命令并响应客户端之后）。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-6.png" | relative_url }}" width=500 alt="paper-fig-6" />

如图 6 所示，每个日志条目要包含三个信息：客户端命令、任期号、在日志中的索引号。任期号是 leader 接收到这条命令时的当前任期号，一经确定不会改变。

一旦 leader 判断一个日志条目已经被安全地复制了，它就可以将这条条目应用到状态机，我们称这种条目为 **committed** 条目。什么叫安全地复制呢？就是这条日志已经在集群中大多数服务器上被复制了。

**在 leader 的日志中，如果一个条目是 committed 的，则它之前的所有条目都是 committed 的。**

那么 follower 如何认识到一个条目可以提交呢？

**Leader 会跟踪当前最新 committed 条目的索引，并将它附在所有 `AppendEntries` RPC 中。一旦 follower 认识到一个索引上的条目已经 committed，它也就将该条目应用到状态机上。**

Raft 的日志机制确保以下两个属性始终成立，它们共同组成了 **Log Matching Property**（如果两个日志包含相同索引和任期的条目，那么这两个日志在该索引及其之前的条目都一样）：

1. 如果在两个不同日志上的两个条目索引相同、任期号相同，则它们包含相同的命令。
2. 如果在两个不同日志上的两个条目索引相同、任期号相同，则它们之前的所有日志条目相同。

第一个属性来自于这样一个事实：leader 在给定索引上最多只会创建一个条目，条目一旦创建不能更换位置。

第二个属性来自于 `AppendEntries` RPC 所进行的一致性检查。当发送 `AppendEntries` RPC 时，leader 在其中包含**新条目前一个条目的索引和任期号**。如果 follower 没有找到相同索引和任期的条目，它就拒绝新条目。根据归约法，初始时空日志满足 Log Matching Property，一致性检查确保在日志扩充时的 Log Matching Property。（AppendEntries Consistency Check）

每当 follower 返回成功时，leader 就知道这个 follower 已经成功复制到了最新的条目。

在 normal operation 内，leader 和 followers 的日志保持一致，但是 leader 崩溃可能导致日志间的不一致。如图 7 所示。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-7.png" | relative_url }}" width=500 alt="paper-fig-7" />

> 当最上方的 leader 拥有权威时，(a-f) 都可能发生在 follower 日志上。一个 follower 可能缺少条目 (a-b)，可能有多余的未提交条目 (c-d)，或者都有 (e-f)。可能 f 服务器本来是任期 2 的 leader，在日志上添加了几个条目，但在提交它们前崩溃了，f 服务器迅速重启，变成任期 3 的 leader，添加了几个条目，在提交任何任期 2 和任期 3 条目前，服务器 f 又崩溃了，并持续了几个任期。

这种冲突的 follower 日志条目最终会被 leader 的日志条目覆写。为此，leader 必须找到自己和 follower 日志在哪个位置达成一致，将一致点之后的 follower 日志都删除掉，然后发送给 follower 自己在一致点之后的日志。

Leader 为每个 follower 维护一个 **nextIndex**，表示下一个发送给该 follower 的条目的索引。当 leader 刚成为 leader 时，它将 **nextIndex 初始化当前日志最后一条的后一个**（图 7 的 11）（这里没有说必须是最新 committed 的后一个）。我们前面提到了 AppendEntries Consistency Check，如果该检查失败了，leader 就会将 nextIndex 减少，然后重试 `AppendEntries` RPC。最终，nextIndex 会到达一个 follower 和 leader 日志一致的点，这时 `AppendEntries` RPC 会成功，使得 follower 中任何冲突的日志被删除，且新增来自 leader 的条目。

这个协议可以进一步优化，减少被拒绝的 `AppendEntries` RPC 数量。例如，当拒绝 `AppendEntries` RPC 时，follower 可以在回复中包含**冲突条目的任期号以及那个任期号最早的日志索引**。有了这个信息，leader 可以减少 nextIndex 来跳过所有那个任期的条目。因此，每个包含冲突条目任期只需要一个 `AppendEntries` RPC。

### 6 安全性

#### 选举限制

前面阐述的方案并未解决一个关键问题：可能出现两个状态机执行不同的命令，未满足 State Machine Safety。例如，一个 follower 可能在 leader 提交几个日志条目时处于不可用状态，然后它恢复并成为了 leader，覆写了之前 leader 提交的日志。

为了解决上述问题，Raft 引入了**选举限制 (election restriction)**，直白点说就是不允许上面的 follower 成为 leader。选举限制可以确保每个任期的 leader 能包含所有在之前任期提交的日志条目。其他共识算法可能采用传输丢失的 committed 条目等方法，要复杂很多。

Raft 使用投票流程限制一个不含有所有 committed 条目的 candidate 成为 leader。Candidate 需要和大多数服务器沟通来成为 leader，意味着任一 committed 条目至少在其中一个服务器上存在。**如果这个 candidate 的日志至少和它所沟通的服务器相比是足够新的 (up-to-date)，那么它就拥有所有 committed 条目。**

具体实现在 `RequestVote` RPC 当中。RPC 会包含 candidate 日志的信息，如果投票者发现自己的日志更新（up-to-date），则拒绝该 candidate。那么怎么判断两个日志哪个更新呢？答案是通过比较两个日志最新条目的索引和任期：**如果最新条目的任期号不同，则更大任期号的日志更新。如果最新条目的任期号相同，则日志长度更长的更新。**

#### 提交过去任期的条目

接下来我们看 Raft 如何处理来自之前任期的条目。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-8.png" | relative_url }}" width=500 alt="paper-fig-8" />

如图 8 所示，(d) 和 (e) 两种情况都有可能发生：

- (a) S1 是 leader，部分复制了索引 2 处的条目。
- (b) S1 崩溃；S5 得到 S3, S4, 自己的选票，选举为任期 3 的 leader，在索引 2 接受了一个不同条目。
- (c) S5 崩溃；S1 重启，被选举为 leader，并继续复制。这时来自任期 2 的条目已经复制到了大多数服务器上，但是还没提交。
- (d) S1 崩溃；S5 得到 S2, S3, S4, 自己的选票，选举为 leader，然后用任期 3 的条目覆写了其他服务器上的条目。（因为任期 2 条目并未提交，所以是 OK 的）
- 然而，如果 S1 在崩溃前将当前任期 4 的条目成功复制到大多数机器上，如 (e) 所示，S5 就不会选举成功。

试想如果在 (c) 中 S1 根据任期 2 已被大多数复制的条件去提交了它，情况就大不相同了，已提交的任期 2 条目会在 (d) 中被任期 3 条目覆盖，破坏了安全性。Raft 避免这种事情发生，**不允许 leader 根据一个过往任期条目的复制情况来提交它**。

取而代之的是，leader 只能根据**当前**任期条目的复制情况去提交它，**并在提交它的同时，提交之前所有未提交的条目**（Log Matching Property）。

> Safety Argument 略，见原文。

#### Follower 和 candidate 崩溃

Follower 和 candidate 崩溃的处理相对简单很多。如果它们崩溃了，发往它们的 RPC 只需不断重试。如果它们执行完 RPC 相关处理，但未回复就崩溃了，也影响不大。因为 Raft 的 RPCs 是幂等的 (idempotent)，即处理一次和处理两次效果一样。

#### 时间和可用性

Raft 的可用性免不了依赖于一些时间条件。选举是 Raft 中受时间条件影响最大的，也是可用性的关键。例如，如果信息交互用时比服务器故障间用时还长，那么 candidate 将无法正常工作足够长的时间来赢得选举。

Raft 在以下时间要求 (timing requirement) 下保持稳定的 leader：

broadcastTime ≪ electionTimeout ≪ MTBF

- broadcastTime：一个服务器并行发送 RPCs 给所有服务器并收到回复的平均用时。应该比 electionTimeout 小一个数量级。鉴于接收方需要落盘，broadcastTime 一般在 0.5ms 到 20ms。
- electionTimeout：前面讲到的超时时间，应该比 MTBF 小几个数量级。我们一般设置为 10ms 到 500ms 之间的值。
- MTBF：mean time between failures，一般是以月为单位，完全满足要求。

### 7 集群成员更改

之前的内容里，集群的**配置 (configuration)**（参与共识算法的服务器集合）都是固定的。Raft 提供了了一种机制，可以在保证安全性的前提下，不中断服务地切换两个配置。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-10.png" | relative_url }}" width=500 alt="paper-fig-10" />

为了保证安全性，转换配置任一时间点都不应该出现两个相同任期的 leader。但如果是直接切换就会出现这个问题。如图 10 所示，集群从 3 个服务器增加为 5 个，箭头所指的地方可能出现两个相同任期的 leader，一个选自 $C_{old}$ 的大多数，一个选自 $C_{new}$ 的大多数。

为了保证安全性，必须采用两阶段的方法。在 Raft 中，集群首先切换为一个过渡的配置，被称为 **joint consensus**，一旦 joint consensus 被提交，系统就可以切换为新的配置。Joint consensus 结合了新旧配置：

- 日志条目需要被复制到新旧配置所有的服务器上。
- 来自新旧配置的任意服务器都能参与选举成为 leader。
- 选举和条目提交的协定需要新旧两个配置各自的大多数都同意。

集群配置使用特殊的日志条目进行存储和通信。图 11 展示了配置更改的过程：

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-11.png" | relative_url }}" width=500 alt="paper-fig-11" />

更改过程中，所有服务器遵守一个规则：**一旦添加了一个新配置到它的日志上，它在未来就使用这个配置，换句话说，服务器总是使用日志中最新的配置，不管那个配置的条目是否被提交**。下面我们来看具体过程：

1. 首先是 leader 收到了将配置 $C_{old}$ 更改为配置 $C_{new}$ 的请求，它将 joint consensus 的配置 $C_{old,new}$ 存储为一个日志条目。
2. Leader 开始使用 $C_{old,new}$ 配置。它需要按 joint consensus 的要求提交 $C_{old,new}$ 条目到 $C_{old,new}$（$C_{old}$ 的大多数和 $C_{new}$ 的大多数）
3. 添加了 $C_{old,new}$ 条目的其他服务器也开始使用配置 $C_{old,new}$。
4. 如果 leader 崩溃，新的 leader 可以从 $C_{old}$ 或 $C_{old,new}$ 选出，取决于这个 candidate 有没有收到过 $C_{old,new}$。
5. 一旦 $C_{old,new}$ 被提交，$C_{old}$ 和 $C_{new}$ 都无法不在对方的同意下做决定。Leader Completeness Property 保证只有包含 $C_{old,new}$ 条目的服务器才能被选为 leader。
6. 现在 leader 可以安全地创建 $C_{new}$ 条目并尝试提交到 $C_{new}$。现在已经和 $C_{old}$ 没什么关系了。

还有三个问题未解决。第一个问题是，**新添加的服务器可能并未存储任何日志**，一旦它添加进集群，可能需要很长一段时间来让它 catch-up，在这段时间可能都无法提交新日志。解决办法是在配置切换前添加一个新的阶段：在这个阶段，新服务器添加为 **non-voting 成员**，leader 将日志复制给它们，但它们不被认为是大多数的一员。直到新服务器 catch-up 完成后，再进行配置切换。

第二个问题是，**当前的 leader 可能不是新配置的一员**。这种情况下，leader 一直工作直到提交 $C_{new}$。这意味着有一段时间，leader 在管理一个不包含它的集群，它正常复制日志但不把自己计为大多数的一员。

第三个问题是，**被移除的服务器仍可能影响集群**。这些服务器不会收到 RPC，所以超时、发起选举。它们用新的任期号发送 RPC，导致 leader 变为 follower。为了解决该问题，Raft 规定，如果一个服务器在从 leader 接收到信息加上**最小**选举超时这段时间内收到 `RequestVote` RPC，它不更新自己任期号或投票。对于 leader 来说，意味着它从不因收到其他 `RequestVote` RPC 变回 follower。

> 集群外服务器的 `RequestVote` 对于 follower 来说，算不算通信？

### 8 日志压实

机器存储空间是有限的，我们不能放任 Raft 的日志无限增长，必须对日志做压实 (compaction)。最常见的压实技术是快照 (snapshot)，一整个系统的状态被存储为一个快照。与之相对的，log cleaning 和 LSM-tree 是增量压实技术。Raft 采用了快照技术，但 LSM-tree 也可以用相同的接口实现。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-12.png" | relative_url }}" width=500 alt="paper-fig-12" />

图 12 展示了 Raft 如何做快照。每个服务器独立地生成快照，包含所有 committed 日志条目。快照还要包含元数据：last included index 表示快照所替换的最后一个条目的索引，last included term 是那个条目的任期。这两个元数据用于 AppendEntries consistency check。为了支持配置更改，快照可能需要特别标记出它范围内的最新配置。

有时需要 leader 给 follower 发送快照，因为 follower 太落后，以至于它需要的条目已经被 leader 做出快照了。Leader 使用一个新的 RPC `InstallSnapshot` 来发送快照。在 Follower 收到快照后，一般是快照包含更新的信息，follower 只需抛弃它现有的所有日志；还有一种可能性是快照描述了日志的前缀，则那些被快照覆盖的日志可以被抛弃，剩下的仍然保留。

为什么 Raft 让每个服务器都可以自行生成快照，而不是由 leader 发送给它们？因为 Raft 认为，后者的网络开销和争用比自行生成快照代价更大，并且会使 leader 的实现更加复杂。

还有一些其他因素需要考虑。第一，什么时候做快照最合适？最简单的方法是为日志设置一个大小上限，达到则生成快照。第二，写快照很耗时，如何避免它影响 normal opearation？解决方法之一是 Copy-on-Write。

### 9 客户端交互

客户端将所有请求发送给 leader。当客户端启动时，它随机选一个服务器发送请求。如果客户端第一个请求不是 leader，则服务器会拒绝客户端请求，并提供它所知的最新 leader 的地址（**`AppendEntries` 请求包含 leader 自己的地址**）。如果 leader 崩溃了，客户端请求会超时，然后它再随机选一个服务器请求。

Raft 的目标之一是实现**线性化语义（linearizable semantics）**，即每个操作看起来都能在其被调用和得到响应之间的某个确切时刻瞬间完成，并且只执行一次。然而，现在 Raft 可能执行同一命令多次。例如，leader 在提交条目后回复客户端前崩溃了，客户端会在新 leader 上重试，导致同一命令被执行两次。

解决方法是所有状态机追踪为每一个客户端处理的最新命令的序列号 **(serial number)** 和相应的回复报文。如果它收到了一个命令，其中的序列号和刚刚已经执行的序列号相同，它直接返回报文，不执行任何操作。

只读操作也比较棘手。如果不特殊处理，只读操作可能读到过时的数据，违背了线性读 (Linearizable Read) 的要求。例如，响应客户端的 leader 可能已经被新的 leader 取代，但它未意识到。

为了解决无法线性读的问题，leader 必须拥有哪些日志被提交的最新消息。Leader Completeness Property 保证 leader 拥有所有已提交条目，但它在任期开始可能并不知道有哪些。为了找到它们，**leader 需要在任期刚开始尝试复制一个空 no-op 条目并提交**。此外，leader 需要在处理读请求前确认它是否被罢黜了，它需要在回复客户端请求前与大多数服务器通信。

### 10 论文图 2 和图 13 的翻译

论文图 2 和图 13 包含了对 Raft 共识算法的精炼描述，下面是原图和翻译。

#### 图 2 State

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-2-1.png" | relative_url }}" width=500 alt="paper-fig-2-1" />

所有服务器持久化的状态信息（在回复 RPC 前先在存储上更新）：

- **currentTerm**：服务器见到的最新的任期号（启动时初始化为 0，单调增加）
- **votedFor**：当前任期收到自己投票的 candidateId（如果没有就是 null）
- **log[]**：日志条目；每个条目包含状态机命令，和 leader 收到条目时的任期号（第一个条目的索引是 1）

所有服务器上的易失的状态信息：

- **commitIndex**：最新被提交条目的索引（初始化为 0，单调增加）
- **lastApplied**：最新被应用到状态机的索引（初始化为 0，单调增加）

Leader 上易失的状态信息（选举成功后重置）：

- **nextIndex[]**：对每个服务器，下一个发给那个服务器的条目的索引（初始化为 leader 最新条目的索引 + 1）
- **matchIndex[]**：对每个服务器，已经知道复制到那个服务器的最高条目的索引（初始化为 0，单调增加）

#### 图 2 RequestVote RPC

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-2-2.png" | relative_url }}" width=500 alt="paper-fig-2-2" />

由 candidate 调用，用于收集选票。

参数：

- **term**：candidate 的当前任期号
- **candidateId**：candidate 自己的 ID
- **lastLogIndex**：candidate 最新日志条目的索引
- **lastLogTerm**：candidate 最新日志条目的任期号

结果：

- **term**：接收方的当前任期号，用于 candidata 更新它的任期
- **voteGranted**：为真时说明 candidate 收到了选票

接收方实现：

1. 如果参数里的 term 小于 currentTerm，回复 false (voteGranted)。
2. 如果 voteFor 为空或者为参数里的 candidateId，并且 candidate 的日志至少比接收方的日志更 up-to-date，则投票。

#### 图 2 AppendEntries RPC

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-2-3.png" | relative_url }}" width=500 alt="paper-fig-2-3" />

由 leader 调用来复制日志条目；也用于心跳。

参数：

- **term**：leader 的当前任期号
- **leaderId**：leader 的 ID，方便 follower 为客户端重定向
- **prevLogIndex**：新条目之前那个条目的索引
- **prevLogTerm**：prevLogIndex 条目的任期号
- **entries[]**：想要复制的日志条目（心跳的话就是空；出于效率可能发送多个）
- **leaderCommit**：leader 的 commitIndex

结果：

- **term**：接收方的 currentTerm，用于 leader 更新自己
- **success**：如果 follower 包含匹配 prevLogIndex 和 prevLogTerm 的条目，就为真

接收方实现：

1. 如果参数里的 term 小于 currentTerm，回复 false (success)。
2. 如果日志不包含匹配 prevLogIndex 和 prevLogTerm 的条目，回复 false (success)。
3. 如果当前条目与新条目冲突（相同索引但任期不同），删除冲突的条目以及它之后所有的条目。
4. 将不在日志中的新条目都新增上去。
5. 如果 leaderCommit 大于 commitIndex，设置 commitIndex 为 min(leaderCommit, 最近新条目的索引)

#### 图 2 Rules for Servers

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-2-4.png" | relative_url }}" width=500 alt="paper-fig-2-4" />

服务器规则

所有服务器：

- 如果 commitIndex 大于 lastApplied：增大 lastApplied，应用 log[lastApplied] 到状态机。
- 如果 RPC 请求或回复包含 term T 大于 currentTerm：设置 currentTime 是 T，转为 follower。

Followers：

- 回复来自 leader 和 candidate 的 RPC。
- 如果没有从当前 leader 接收到 AppendEntries RPC 或给 candiate 投票的用时超出了 election timeout：转为 candidate。

Candidates：

- 转换为 candidate 时，发起选举：
  - 递增 currentTerm
  - 给自己投票
  - 重置选举计时器
  - 发送 RequestVote RPC 给其他所有服务器
- 如果从大多数服务器收到了投票：变成 leader。
- 如果收到来自新 leader 的 AppendEntries RPC：转换为 follower。
- 如果选举超时：重新发起选举。

Leaders：

- 在选举之后：向每个服务器发送初始的空 AppendEntries RPC（心跳）；在空闲期间重复此操作以防止选举超时。
- 如果从客户端接收到命令：将条目追加到本地日志中，在条目应用到状态机后响应。
- 如果最新条目索引大于等于一个 follower 的 nextIndex：发送 AppendEntries RPC，包含从 nextIndex 开始的日志。
  - 如果成功，为该 follower 更新 nextIndex 和 matchIndex。
  - 如果因为一致性检查没通过而失败，减小 nextIndex 并重试。
- 如果存在 N，N 大于 commitIndex，matchIndex[i] 的大多数大于 N，log[N].term 等于 currentTerm：将 commitIndex 设为 N。

#### 图 13 InstallSnapshot RPC

快照被分割成多个块 (chunks) 进行传输；这使得每个块都能让 followers 感受到一次生命迹象，因此它可以重置其选举计时器。

<img src="{{ "/assets/images/consensus-algorithm-raft/paper-fig-13.png" | relative_url }}" width=500 alt="paper-fig-13" />

由 leader 调用，用于向一个 follower 发送快照的 chunks。Leaders 总是按顺序发送 chunks。

参数：

- **term**：leader 的当前任期号
- **leaderId**：leader 的 ID，方便 follower 为客户端重定向
- **lastIncludedIndex**：snapshot 中最新的条目的索引
- **lastIncludedTerm**：lastIncludedIndex 条目的任期号
- **offset**：chunk 在快照的开始偏移量
- **data[]**：snapshot chunk 的原始字节，从 offset 开始
- **done**：如果这个最后一个 chunk，为真

结果：

- **term**：接收方的 currentTerm，用于 leader 更新自己

接收方实现：

1. 如果 term 小于 currentTerm，立即回复。
2. 如果是第一个 chunk（offset 为 0），创建快照文件。
3. 将数据写到快照的指定偏移处。
4. 如果 done 为 false，回复 RPC（不做后续操作），并等待更多的 chunk。
5. 否则，保存快照文件，丢弃所有现存的 lastIncludedIndex 更小的快照。
6. 如果存在一个和 lastIncludedIndex 相同索引，和 lastIncludedTerm 相同任期的条目，则保留它之后的所有条目，然后回复（不做后续操作）。
7. 否则，丢弃整个日志。
8. 使用快照内容重置状态机，加载快照中的配置。

### 参考阅读

- D. Ongaro, “Consensus: Bridging Theory and Practice”.
