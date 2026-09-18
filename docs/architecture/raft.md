# Raft Consensus（Raft 共识）

[Raft](https://raft.github.io) is a distributed consensus protocol which replicates data across a
cluster of nodes in a consistent and durable manner. It is described in the very readable
[Raft paper](https://raft.github.io/raft.pdf), and in the more comprehensive
[Raft thesis](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf).

[Raft](https://raft.github.io) 是一种分布式共识协议，以一致且持久的方式在集群节点之间复制数据。它在非常易读的 [Raft 论文](https://raft.github.io/raft.pdf) 中被描述，更全面的版本见 [Raft 博士论文](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)。

The toyDB Raft implementation is in the [`raft`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/raft)
module, and is described in the module documentation:

toyDB 的 Raft 实现位于 [`raft`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/raft) 模块中，并在模块文档中加以描述：

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/mod.rs#L1-L240>

Raft is fundamentally the same protocol as [Paxos](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
and [Viewstamped Replication](https://pmg.csail.mit.edu/papers/vr-revisited.pdf), but an
opinionated variant designed to be simple, understandable, and practical. It is widely used in the
industry: [CockroachDB](https://www.cockroachlabs.com), [TiDB](https://www.pingcap.com),
[etcd](https://etcd.io), [Consul](https://developer.hashicorp.com/consul), and many others.

Raft 在本质上与 [Paxos](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) 和 [Viewstamped Replication](https://pmg.csail.mit.edu/papers/vr-revisited.pdf) 是同一类协议，但它是为简单、易懂和实用而设计的一个有明确取舍的变体。它在工业界被广泛使用：[CockroachDB](https://www.cockroachlabs.com)、[TiDB](https://www.pingcap.com)、[etcd](https://etcd.io)、[Consul](https://developer.hashicorp.com/consul) 等等。

Briefly, Raft elects a leader node which coordinates writes and replicates them to followers. Once a
majority (>50%) of nodes have acknowledged a write, it is considered durably committed. It is common
for the leader to also serve reads, since it always has the most recent data and is thus strongly
consistent.

简单来说，Raft 会选出一个 leader 节点来协调写入，并将其复制到 follower。一旦多数（>50%）节点确认了某次写入，该写入即被视为持久地提交。leader 通常也负责处理读取，因为它总是拥有最新的数据，因此具备强一致性。

A cluster must have a majority of nodes (known as a [quorum](https://en.wikipedia.org/wiki/Quorum_(distributed_computing)))
live and connected to remain available, otherwise it will not commit writes in order to guarantee
data consistency and durability. Since there can only be one majority in the cluster, this prevents
a [split brain](https://en.wikipedia.org/wiki/Split-brain_(computing)) scenario where two active
leaders can exist concurrently (e.g. during a [network partition](https://en.wikipedia.org/wiki/Network_partition))
and store conflicting values.

集群必须有过半数节点（称为[法定人数(quorum)](https://en.wikipedia.org/wiki/Quorum_(distributed_computing))）存活并保持连接才能继续提供服务，否则它不会提交写入，以保证数据的一致性和持久性。由于集群中只能存在一个多数派，这可以防止[脑裂](https://en.wikipedia.org/wiki/Split-brain_(computing))场景——即两个活跃的 leader 并存（例如在[网络分区](https://en.wikipedia.org/wiki/Network_partition)期间）并存储相互冲突的值。

The Raft leader appends writes to an ordered command log, which is then replicated to followers.
Once a majority has replicated the log up to a given entry, that log prefix is committed and then
applied to a state machine. This ensures that all nodes will apply the same commands in the same
order and eventually reach the same state (assuming the commands are deterministic). Raft itself
doesn't care what the state machine and commands are, but in toyDB's case it's SQL tables and rows
stored in an MVCC key/value store.

Raft leader 将写入追加到一条有序的命令日志中，再复制给 follower。一旦多数节点把日志复制到某个条目，该日志前缀即被提交，随后被应用到状态机。这保证了所有节点以相同的顺序应用相同的命令，并最终达到相同的状态（假设命令是确定性的）。Raft 本身不关心状态机和命令是什么，而在 toyDB 中，状态机是存储在 MVCC 键/值存储中的 SQL 表和行。

This diagram from the Raft paper illustrates how a Raft node receives a command from a client (1),
adds it to its log and reaches consensus with other nodes (2), then applies it to its state machine
(3) before returning a result to the client (4):

下面这张来自 Raft 论文的图展示了一个 Raft 节点如何接收来自客户端的命令 (1)，将其加入自己的日志并与其他节点达成共识 (2)，然后将其应用到状态机 (3)，最后向客户端返回结果 (4)：

<img src="./images/raft.svg" alt="Raft node" width="400" style="display: block; margin: 30px auto;">

You may notice that Raft is not very scalable, since all reads and writes go via the leader node,
and every node must store the entire dataset. Raft solves replication and availability, but not
scalability. Real-world systems typically provide horizontal scalability by splitting a large
dataset across many separate Raft clusters (i.e. sharding), but this is out of scope for toyDB.

你可能注意到 Raft 的可扩展性并不好，因为所有读写都要经过 leader 节点，且每个节点都必须存储完整的数据集。Raft 解决的是复制和可用性问题，而不是可扩展性。现实中的系统通常通过把大数据集拆分到多个独立的 Raft 集群（即分片/sharding）来实现水平扩展，但这超出了 toyDB 的范围。

For simplicitly, toyDB implements the bare minimum of Raft, and omits optimizations described in
the paper such as state snapshots, log truncation, leader leases, and more. The implementation is
in the [`raft`](https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/mod.rs)
module, and we'll walk through the main components next.

为简单起见，toyDB 只实现了 Raft 的最基本部分，省略了论文中描述的诸如状态快照、日志截断、leader 租约等优化。实现在 [`raft`](https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/mod.rs) 模块中，接下来我们将逐一介绍其主要组件。

There is a comprehensive set of Raft test scripts in [`src/raft/testscripts/node`](https://github.com/erikgrinaker/toydb/blob/386153f5c00cb1a88b1ac8489ae132674d96f68a/src/raft/testscripts/node),
which illustrate the protocol in a wide variety of scenarios.

在 [`src/raft/testscripts/node`](https://github.com/erikgrinaker/toydb/blob/386153f5c00cb1a88b1ac8489ae132674d96f68a/src/raft/testscripts/node) 中有一套非常全面的 Raft 测试脚本，用各种各样的场景展示该协议的行为。

## Log Storage（日志存储）

Raft replicates an ordered command log consisting of `raft::Entry`:

Raft 复制的是一条由 `raft::Entry` 组成的有序命令日志：

<https://github.com/erikgrinaker/toydb/blob/90a6cae47ac20481ac4eb2f20eea50f02e6c2b33/src/raft/log.rs#L10-L28>

`index` specifies the position in the log, and `command` contains the binary command to apply to the
state machine. The `term` identifies the leadership term in which the command was proposed: a new
term begins when a new leader election is held (we'll get back to this later).

`index` 指定条目在日志中的位置，`command` 包含要应用到状态机的二进制命令。`term` 标识该命令提出时所处的领导任期：每当举行新的 leader 选举时，就会开始一个新任期（稍后详述）。

Entries are appended to the log by the leader and replicated to followers. Once acknowledged by a
quorum, the log up to that index is committed and will never change. Entries that are not yet
committed may be replaced or removed if the leader changes.

条目由 leader 追加到日志中并复制给 follower。一旦获得法定人数(quorum)的确认，该索引之前的日志即被提交且永不改变。尚未提交的条目在 leader 更换时可能被替换或移除。

The Raft log enforces the following invariants:

Raft 日志强制维护以下不变式：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L80-L91>

`raft::Log` implements a Raft log, and stores log entries in a `storage::Engine` key/value store:

`raft::Log` 实现了 Raft 日志，并将日志条目存储在 `storage::Engine` 键/值存储中：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L43-L116>

It also stores some additional metadata that we'll need later: the current term, vote, and commit
index. These are stored as separate keys:

它还存储了一些之后会用到的额外元数据：当前任期、投票和提交索引。它们以独立的键存储：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L30-L39>

Individual entries are appended to the log via `Log::append`, typically when the leader wants to
replicate a new write:

单个条目通过 `Log::append` 追加到日志中，通常发生在 leader 想要复制一次新写入时：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L190-L203>

Entries can also be appended in bulk via `Log::splice`, typically when entries are replicated to
followers. This also allows replacing existing uncommitted entries, e.g. after a leader change:

条目也可以通过 `Log::splice` 批量追加，通常发生在向 follower 复制条目时。它还允许替换已有的未提交条目，例如在 leader 更换之后：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L269-L343>

Committed entries are marked by `Log::commit`, making them immutable and eligible for state machine
application:

已提交的条目由 `Log::commit` 标记，使其不可变，并可以被应用到状态机：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L205-L222>

The log also has methods to read entries from the log, either individually as `Log::get` or by
iterating over a range with `Log::scan`:

日志还提供了读取条目的方法：可以用 `Log::get` 单个读取，或用 `Log::scan` 按范围迭代：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/log.rs#L224-L267>

## State Machine Interface（状态机接口）

Raft doesn't know or care what the log commands are, nor what the state machine does with them. It
simply takes `raft::Entry` from the log and gives them to the state machine.

Raft 既不知道也不关心日志中的命令是什么，也不关心状态机如何处理它们。它只是从日志中取出 `raft::Entry`，然后交给状态机。

The Raft state machine is represented by the `raft::State` trait. Raft will ask about the last
applied entry via `State::get_applied_index`, and feed it newly committed entries via
`State::apply`. It also allows reads via `State::read`, but we'll get back to that later.

Raft 状态机由 `raft::State` trait 表示。Raft 通过 `State::get_applied_index` 询问最后应用的条目，并通过 `State::apply` 把新提交的条目喂给它。它还允许通过 `State::read` 进行读取，稍后再讨论。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/state.rs#L4-L51>

The state machine does not have to flush its state to durable storage after each transition; on node
crashes, the state machine is allowed to regress, and will be caught up by replaying the unapplied
log entries. It is also possible to implement a purely in-memory state machine (and in fact, toyDB
allows running the state machine with a `Memory` storage engine).

状态机不必在每次状态转换后都把状态刷入持久存储；节点崩溃时，状态机允许回退，之后通过重放未应用的日志条目追平。也完全可以实现纯内存的状态机（事实上，toyDB 允许使用 `Memory` 存储引擎来运行状态机）。

The state machine must take care to be deterministic: the same commands applied in the same order
must result in the same state across all nodes. This means that a command can't e.g. read the
current time or generate a random number -- these values must be included in the command. It also
means that non-deterministic errors, such as an IO error, must halt command application (in toyDB's
case, we just panic and crash the node).

状态机必须保证确定性：相同的命令以相同的顺序应用，必须在所有节点上产生相同的状态。这意味着命令不能读取当前时间或生成随机数——这些值必须包含在命令本身中。这也意味着非确定性错误（例如 IO 错误）必须中止命令的应用（在 toyDB 中，我们直接 panic 并让节点崩溃）。

In toyDB's, the state machine is an MVCC key/value store that stores SQL tables and rows, as we'll
see in the SQL Raft replication section.

在 toyDB 中，状态机是一个存储 SQL 表和行的 MVCC 键/值存储，我们将在 SQL Raft 复制一节中看到。

## Node Roles（节点角色）

In Raft, a node can have one out of three roles:

在 Raft 中，节点可以处于以下三种角色之一：

* **Leader:** replicates writes to followers and serves client requests.
* **Follower:** replicates writes from a leader.
* **Candidate:** campaigns for leadership.

* **Leader（领导者）：** 将写入复制到 follower，并处理客户端请求。
* **Follower（跟随者）：** 从 leader 复制写入。
* **Candidate（候选者）：** 竞选领导地位。

The Raft paper summarizes these roles and transitions in the following diagram (we'll discuss
leader election in detail below):

Raft 论文用下面这张图总结了这些角色及其转换（我们将在下文详细讨论 leader 选举）：

<img src="./images/raft-states.svg" alt="Raft states" width="400" style="display: block; margin: 30px auto;">

In toyDB, a node is represented by the `raft::Node` enum, with variants for each state:

在 toyDB 中，节点由 `raft::Node` 枚举表示，每种状态对应一个变体：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L47-L66>

This wraps the `raft::RawNode<Role>` type which contains the inner node state. It is generic over
the role, and uses the [typestate pattern](http://cliffle.com/blog/rust-typestate/) to provide
methods and transitions depending on the node's current role. This enforces state transitions and
invariants at compile time via Rust's type system -- for example, only `RawNode<Candidate>` has an
`into_leader()` method, since only candidates can transition to leaders (when they win an election).

它包装了包含节点内部状态的 `raft::RawNode<Role>` 类型。该类型对角色是泛型的，并使用 [typestate 模式](http://cliffle.com/blog/rust-typestate/)根据节点当前的角色提供相应的方法和状态转换。这通过 Rust 的类型系统在编译期强制保证状态转换和不变式——例如，只有 `RawNode<Candidate>` 才有 `into_leader()` 方法，因为只有候选者才能转变为 leader（当它赢得选举时）。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L156-L177>

The `RawNode::role` field contains role-specific state as structs implementing the `Role` marker
trait:

`RawNode::role` 字段以实现了 `Role` 标记 trait 的结构体来保存角色特有的状态：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L661-L680>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L242-L255>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L523-L531>

We'll see what the various fields are used for in the following sections.

我们将在接下来的章节中看到这些字段的用途。

## Node Interface and Communication（节点接口与通信）

The `raft::Node` enum has two main methods that drive the node: `tick()` and `step()`. These consume
the current node and return a new node, possibly with a different role.

`raft::Node` 枚举有两个驱动节点的主要方法：`tick()` 和 `step()`。它们消耗当前节点并返回一个新节点，其角色可能已经不同。

`tick()` advances time by a logical tick. This is used to measure the passage of time, e.g. to
trigger election timeouts or periodic leader heartbeats. toyDB uses a tick interval of 100
milliseconds (see `raft::TICK_INTERVAL`), and will call `tick()` on the node at this rate.

`tick()` 推进一个逻辑时钟刻度，用来度量时间的流逝，例如触发选举超时或周期性的 leader heartbeat。toyDB 的 tick 间隔为 100 毫秒（见 `raft::TICK_INTERVAL`），并按此频率对节点调用 `tick()`。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L125-L132>

`step()` processes an inbound message from a different node or client:

`step()` 处理来自其他节点或客户端的入站消息：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L107-L123>

Outbound messages to other nodes are sent via the `RawNode::tx` channel:

发往其他节点的出站消息通过 `RawNode::tx` 通道发送：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L171-L172>

Nodes are identified by a unique node ID, which is given at node startup:

节点由唯一的节点 ID 标识，该 ID 在节点启动时给定：

<https://github.com/erikgrinaker/toydb/blob/90a6cae47ac20481ac4eb2f20eea50f02e6c2b33/src/raft/node.rs#L17-L18>

Messages are wrapped in a `raft::Envelope` specifying the sender and recipient:

消息被封装在指定了发送者和接收者的 `raft::Envelope` 中：

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/message.rs#L10-L21>

The envelope contains a `raft::Message`, an enum which encodes the Raft message protocol. We won't
dwell on the specific message types here, but discuss them invididually in the following sections.
Raft does not require reliable message delivery, so messages may be dropped or reordered at any
time, although toyDB's use of TCP provides stronger delivery guarantees.

信封里包含一个 `raft::Message`，这是一个编码了 Raft 消息协议的枚举。这里不展开具体的消息类型，后面的章节会逐一讨论。Raft 不要求可靠的消息投递，消息随时可能被丢弃或乱序，不过 toyDB 使用 TCP 提供了更强的投递保证。

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/message.rs#L25-L152>

This is an entirely synchronous and deterministic model -- the same sequence of calls on a given
node in a given initial state will always produce the same result. This is very convenient for
testing and understandability. We will see in the server section how toyDB drives the node on a
separate thread, provides a network transport for messages, and ticks it at regular intervals.

这是一个完全同步且确定性的模型——在给定初始状态下，对给定节点执行相同序列的调用总是产生相同的结果。这对测试和可理解性都非常方便。我们将在 server 一节中看到 toyDB 如何在一个单独的线程上驱动节点、为消息提供网络传输，并按固定间隔对节点进行 tick。

## Leader Election and Terms（Leader 选举与任期）

In the steady state, Raft simply has a leader which replicates writes to followers. But to reach
this steady state, we must elect a leader, which is where much of the subtle complexity lies. See
the Raft paper for comprehensive details and safety arguments, we'll summarize it briefly below.

在稳态下，Raft 只是由一个 leader 把写入复制给 follower。但要达到这个稳态，必须先选出 leader，而这里潜藏着许多微妙的复杂性。完整的细节和安全性论证请参阅 Raft 论文，下面只做简要总结。

Raft divides time into _terms_. The term is a monotonically increasing number starting at 1. There
can only be one leader in a term (or none if an election fails), and the term can never regress.
Replicated commands belong to the specific term under which they were proposed.

Raft 把时间划分为 _任期_（term）。任期是一个从 1 开始单调递增的数字。一个任期只能有一个 leader（如果选举失败则没有），任期也永远不会回退。被复制的命令归属于它们提出时所在的特定任期。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L20-L21>

Let's walk through an election, where we bootstrap a brand new, empty toyDB cluster with 3 nodes.

我们来走一遍选举流程：启动一个全新的、空的、包含 3 个节点的 toyDB 集群。

Nodes are initialized by calling `Node::new()`. Since this is a new cluster, they are given an empty
`raft::Log` and `raft::State`, at term 0. Nodes start with role `Follower`, but without a leader.

节点通过调用 `Node::new()` 初始化。由于这是新集群，它们被赋予空的 `raft::Log` 和 `raft::State`，任期为 0。节点以 `Follower` 角色启动，但还没有 leader。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L68-L87>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L266-L290>

Now, nothing really happens for a while, as the nodes are waiting to maybe hear from an existing
leader (there is none). Every 100 ms we call `tick()`, until we reach `election_timeout`:

此时的一段时间内什么都不会发生，因为节点在等待是否会有已有 leader 的消息（并没有）。每 100 毫秒我们调用一次 `tick()`，直到达到 `election_timeout`：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L489-L497>

Notice how `new()` set `election_timeout` to a random value (in the range `ELECTION_TIMEOUT_RANGE`
of 10-20 ticks, i.e. 1-2 seconds). If all nodes had the same timeout, they would likely campaign for
leadership simultaneously, resulting in an election tie -- Raft uses randomized election timeouts to
avoid such ties.

注意 `new()` 把 `election_timeout` 设成了一个随机值（范围为 `ELECTION_TIMEOUT_RANGE` 的 10-20 个 tick，即 1-2 秒）。如果所有节点的超时时间都相同，它们很可能会同时发起竞选，导致选举平票——Raft 使用随机化的选举超时来避免这种情况。

Once a node reaches `election_timeout` it transitions to role `Candidate`:

一旦节点达到 `election_timeout`，它就转换为 `Candidate` 角色：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L292-L312>

When it becomes a candidate it campaigns for leadership by increasing its term to 1, voting for
itself, and sending `Message::Campaign` to all peers asking for their vote:

成为候选者后，它会发起竞选：把任期增加到 1，给自己投票，并向所有对等节点发送 `Message::Campaign` 请求投票：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L647-L658>

In Raft, the term can't regress, and a node can only cast a single vote in each term (even across
restarts), so both of these are persisted to disk via `Log::set_term_vote()`.

在 Raft 中，任期不能回退，且每个节点在每个任期只能投一票（即使跨重启也是如此），因此这两者都通过 `Log::set_term_vote()` 持久化到磁盘。

When the two other nodes (still in state `Follower`) receive the `Message::Campaign` asking for a
vote, they will first increase their term to 1 (since this is a newer term than their local term 0):

当另外两个节点（仍处于 `Follower` 状态）收到请求投票的 `Message::Campaign` 时，它们会先把自己的任期提升到 1（因为这是一个比本地任期 0 更新的任期）：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L347-L351>

They then grant the vote since they haven't yet voted for anyone else in term 1. They persist the
vote to disk via `Log::set_term_vote()` and return a `Message::CampaignResponse { vote: true }` to
the candidate:

由于它们在任期 1 中还没有投给任何其他人，于是同意投票。它们通过 `Log::set_term_vote()` 把投票持久化到磁盘，并向候选者返回 `Message::CampaignResponse { vote: true }`：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L424-L449>

They also check that the candidate's log is at least as long as theirs, which is trivially true in
this case since the log is empty. This is necessary to ensure that a leader has all committed
entries (see section 5.4.1 in the Raft paper).

它们还会检查候选者的日志是否至少和自己的一样长——在本例中由于日志为空，这显然成立。这一检查是必要的，用于确保 leader 拥有所有已提交的条目（见 Raft 论文第 5.4.1 节）。

When the candidate receives the `Message::CampaignResponse` it records the vote from each node. Once
it has a quorum (in this case 2 out of 3 votes including its own vote) it becomes leader in term 1:

候选者收到 `Message::CampaignResponse` 后，会记录来自每个节点的投票。一旦获得法定人数(quorum)（本例中是 3 票中的 2 票，包括自己的一票），它就成为任期 1 的 leader：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L599-L606>

When it becomes leader, it sends a `Message::Heartbeat` to all peers to tell them it is now the
leader in term 1. It also appends an empty entry to its log and replicates it, but we will ignore
this for now (see section 5.4.2 in the Raft paper for why).

成为 leader 后，它会向所有对等节点发送 `Message::Heartbeat`，告知它们自己现在是任期 1 的 leader。它还会向自己的日志追加一个空条目并复制它，不过这里先忽略这一点（原因见 Raft 论文第 5.4.2 节）。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L563-L583>

When the other nodes receive the heartbeat, they become followers of the new leader in its term:

其他节点收到 heartbeat 后，会在该任期内成为新 leader 的 follower：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L359-L384>

From now on, the leader will send periodic `Message::Heartbeat` every 4 ticks (see
`HEARTBEAT_INTERVAL`) to assert its leadership:

从现在起，leader 每 4 个 tick（见 `HEARTBEAT_INTERVAL`）会周期性地发送 `Message::Heartbeat` 来宣告自己的领导地位：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L945-L953>

The followers record when they last received any message from the leader (including heartbeats), and
will hold a new election if they haven't heard from the leader in an election timeout (e.g. due to a
leader crash or network partition):

follower 会记录最后一次收到 leader 任何消息（包括 heartbeat）的时间，如果在选举超时内没有再听到 leader 的消息（例如 leader 崩溃或网络分区），就会发起新的选举：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L353-L356>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L489-L497>

This entire process is illustrated in the test script [`election`](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election),
along with several other test scripts that show e.g. [election ties](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election_tie),
[contested elections](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election_contested),
and other scenarios:

整个过程展示在测试脚本 [`election`](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election) 中，另有若干测试脚本展示了例如[选举平票](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election_tie)、[有竞争的选举](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election_contested)等场景：

<https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/election#L1-L72>

## Client Requests and Forwarding（客户端请求与转发）

Once a leader has been elected, we can submit read and write requests to it. This is done by
stepping a `Message::ClientRequest` into the node using the local node ID, with a unique request ID
(toyDB uses UUIDv4), and waiting for an outbound response message with the same ID:

选出 leader 之后，我们就可以向它提交读写请求。做法是：使用本地节点 ID，把一个带有唯一请求 ID（toyDB 使用 UUIDv4）的 `Message::ClientRequest` step 进节点，然后等待带相同 ID 的出站响应消息：

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/message.rs#L134-L151>

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/message.rs#L164-L188>

The requests and responses themselves are arbitrary binary data which is interpreted by the state
machine. For our purposes here, let's pretend the requests are:

请求和响应本身是由状态机解释的任意二进制数据。为了便于讨论，我们不妨假设请求是：

* `Request::Write("key=value")` → `Response::Write("ok")`（写请求与响应）
* `Request::Read("key")` → `Response::Read("value")`（读请求与响应）

The fundamental difference between read and write requests are that write requests are replicated
through Raft and executed on all nodes, while read requests are only executed on the leader without
being appended to the log. It would be possible to execute reads on followers too, for load
balancing, but these reads would be eventually consistent and thus violate linearizability, so toyDB
only executes reads on the leader.

读请求和写请求的根本区别在于：写请求会通过 Raft 复制并在所有节点上执行，而读请求只在 leader 上执行、不追加到日志中。为了负载均衡，也可以在 follower 上执行读请求，但那样的读是最终一致的，会破坏线性一致性，因此 toyDB 只在 leader 上执行读取。

If a request is submitted to a follower, it will be forwarded to the leader and the response
forwarded back to the client (distinguished by the sender/recipient node ID -- a local client always
uses the local node ID):

如果请求被提交给了 follower，它会被转发给 leader，响应再转发回客户端（通过发送者/接收者节点 ID 区分——本地客户端总是使用本地节点 ID）：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L451-L474>

For simplicity, we cancel the request with `Error::Abort` if a request is submitted to a candidate,
and similarly if a follower changes its role to candidate or discovers a new leader. We could have
held on to these and redirected them to a new leader, but we keep it simple and ask the client to
retry.

为简单起见，如果请求被提交给了 candidate，我们会用 `Error::Abort` 取消该请求；当 follower 把角色变为 candidate 或发现新 leader 时，也做类似处理。我们本可以保留这些请求并把它们重定向到新 leader，但这里保持简单，让客户端重试即可。

We'll look at the actual read and write request processing next.

接下来我们看看实际的读写请求处理。

## Write Replication and Application（写入复制与应用）

When the leader receives a write request, it proposes the command for replication to followers. It
keeps track of the in-flight write and its log entry index in `writes`, such that it can respond to
the client with the command result once the entry has been committed and applied.

leader 收到写请求后，会提出该命令以便复制到 follower。它在 `writes` 中跟踪进行中的写入及其日志条目索引，以便在条目被提交并应用后，用命令结果响应客户端。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L895-L904>

To propose the command, the leader appends it to its log and sends a `Message::Append` to each
follower to replicate it to their logs:

为了提出该命令，leader 会把它追加到自己的日志中，并向每个 follower 发送 `Message::Append`，把条目复制到它们的日志：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L966-L980>

In steady state, `Message::Append` just contains the single log entry we appended above:

在稳态下，`Message::Append` 只包含我们上面追加的那一个日志条目：

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/message.rs#L87-L108>

However, sometimes followers may be lagging behind the leader (e.g. after a crash), or their log may
have diverged from the leader (e.g. unsuccessful proposals from a stale leader after a network
partition). To handle these cases, the leader tracks the replication progress of each follower as
`raft::Progress`:

不过，有时 follower 可能落后于 leader（例如崩溃之后），或者它们的日志可能与 leader 产生分歧（例如网络分区后过期的 leader 提案失败所留下的日志）。为了处理这些情况，leader 用 `raft::Progress` 跟踪每个 follower 的复制进度：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L682-L698>

We'll gloss over these cases here (see the Raft paper and the code in `raft::Progress` and
`maybe_send_append()` for details). In the steady state, where each entry is successfully appended
and replicated one at a time, `maybe_send_append()` will fall through to the bottom and send a
single entry:

这里我们略过这些情况（细节见 Raft 论文以及 `raft::Progress` 和 `maybe_send_append()` 的代码）。在稳态下——每个条目逐个成功追加并复制——`maybe_send_append()` 会一直执行到函数末尾，发送单个条目：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L1068-L1128>

The `Message::Append` contains the index/term of the entry immediately before the new entry as
`base_index` and `base_term`. If the follower's log also contains an entry with this index and term
then its log is guaranteed to match (be equal to) the leader's log up to this entry (see section 5.3
in the Raft paper). The follower can then append the new log entry and return a
`Message::AppendResponse` confirming that the entry was appended and that its log matches the
leader's log up to `match_index`:

`Message::Append` 中包含新条目紧前面那个条目的索引/任期，作为 `base_index` 和 `base_term`。如果 follower 的日志在相同索引处也有相同任期的条目，那么可以保证它的日志在该条目之前与 leader 的日志一致（见 Raft 论文第 5.3 节）。随后 follower 可以追加新的日志条目，并返回 `Message::AppendResponse`，确认条目已追加、且其日志在 `match_index` 之前与 leader 的日志一致：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L386-L410>

When the leader receives the `Message::AppendResponse`, it will update its view of the follower's
`match_index`.

leader 收到 `Message::AppendResponse` 后，会更新它对 follower 的 `match_index` 的记录。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L844-L858>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L701-L710>

Once a quorum of nodes (in our case 2 out of 3 including the leader) have the entry in their log,
the leader can commit the entry and apply it to the state machine. It also looks up the in-flight
write request from `writes` and sends the command result back to the client as
`Message::ClientResponse`:

一旦法定人数(quorum)的节点（本例中是包括 leader 在内的 3 个中的 2 个）的日志中都包含该条目，leader 就可以提交该条目并将其应用到状态机。它还会从 `writes` 中查找进行中的写请求，并通过 `Message::ClientResponse` 把命令结果返回给客户端：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L982-L1032>

The leader will also propagate the new commit index to followers via the next heartbeat, so that
they can also apply any pending log entries to their state machine. This isn't strictly necessary,
since reads are executed on the leader and nodes have to apply pending entries before becoming
leaders, but we do it anyway so that they don't fall too far behind on application.

leader 还会通过下一次 heartbeat 把新的提交索引传播给 follower，使它们也能把待处理的日志条目应用到自己的状态机。这并非严格必要，因为读取都在 leader 上执行、节点在成为 leader 前也必须应用完待处理条目，但我们仍然这样做，以免它们的应用进度落后太多。

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L359-L384>

This process is illustrated in the test scripts [`append`](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/append) and [`heartbeat_commits_follower`](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/heartbeat_commits_follower)
(along with many other scenarios):

这一过程展示在测试脚本 [`append`](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/append) 和 [`heartbeat_commits_follower`](https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/heartbeat_commits_follower) 中（还有许多其他场景）：

<https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/append#L1-L43>

<https://github.com/erikgrinaker/toydb/blob/cb234a0b776484608118fd9382869ee5bc30d4f0/src/raft/testscripts/node/heartbeat_commits_follower#L1-L50>

## Read Processing（读取处理）

For linearizable (aka strongly consistent) reads, we must execute read requests on the leader, as
mentioned above. However, this is not sufficient: under e.g. a network partition, a node may think
it's still the leader while in fact a different leader has been elected elsewhere (in a later term)
and executed writes there.

如前所述，要实现线性一致性（即强一致）的读取，读请求必须在 leader 上执行。但这还不够：例如在网络分区下，一个节点可能以为它仍是 leader，而实际上别处（在更新的任期中）已经选出了另一个 leader 并在那里执行了写入。

To handle this case, the leader must confirm that it is still the leader for each read, by sending a
`Message::Read` to its followers containing a read sequence number. Only if a quorum confirms that
it is still the leader can the read be executed. This incurs an additional network roundtrip, which
is clearly inefficient, so real-world systems often use leader leases instead (see section 6.4.1 of
the Raft _thesis_, not the paper) -- but it's fine for toyDB.

为了处理这种情况，leader 必须对每次读取确认自己仍是 leader：向 follower 发送带有读序列号的 `Message::Read`。只有法定人数(quorum)确认它仍是 leader，读取才能执行。这会带来一次额外的网络往返，显然效率不高，因此现实中的系统通常改用 leader 租约（见 Raft _博士论文_第 6.4.1 节，而非会议论文）——不过对 toyDB 来说这样就够了。

<https://github.com/erikgrinaker/toydb/blob/d96c6dd5ae7c0af55ee609760dcd958c289a44f2/src/raft/message.rs#L125-L132>

When the leader receives the read request, it increments the read sequence number, stores the
pending read request in `reads`, and sends a `Message::Read` to all followers:

leader 收到读请求后，会递增读序列号，把待处理的读请求存入 `reads`，并向所有 follower 发送 `Message::Read`：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L906-L917>

When the followers receive the `Message::Read`, they simply respond with a `Message::ReadResponse`
if it's from their current leader (messages from stale terms are ignored):

follower 收到 `Message::Read` 后，如果消息来自当前 leader，就直接回复 `Message::ReadResponse`（来自过期任期的消息会被忽略）：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L342-L346>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L412-L422>

When the leader receives the `Message::ReadResponse` it records it in the peer's `Progress`, and
executes the read once a quorum have confirmed the sequence number:

leader 收到 `Message::ReadResponse` 后，把它记录在该对等节点的 `Progress` 中，一旦法定人数(quorum)确认了序列号，就执行读取：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L860-L866>

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/node.rs#L1034-L1066>

We now have a Raft-managed state machine with replicated writes and linearizable reads.

至此，我们有了一个由 Raft 管理的状态机，具备复制的写入和线性一致的读取。

---

<p align="center">
← <a href="mvcc.md">MVCC Transactions</a> &nbsp; | &nbsp; <a href="sql.md">SQL Engine</a> →
</p>
