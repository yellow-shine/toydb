# Overview（总览）

toyDB consists of a cluster of nodes that execute [SQL](https://en.wikipedia.org/wiki/SQL)
transactions against a replicated state machine. Clients can connect to any node in the cluster and
submit SQL statements. The cluster remains available if a minority of nodes crash or disconnect,
but halts if a majority of nodes fail.

toyDB 由一个节点集群组成，针对复制状态机执行 [SQL](https://en.wikipedia.org/wiki/SQL) 事务。客户端可以连接到集群中的任意节点并提交 SQL 语句。当少数节点崩溃或断开连接时，集群仍然可用；但当多数节点失效时，集群会停止服务。

## Properties（特性）

* **Distributed:** runs across a cluster of nodes.
  **分布式：** 运行在一个节点集群之上。
* **Highly available:** tolerates failure of a minority of nodes.
  **高可用：** 能够容忍少数节点的故障。
* **SQL compliant:** correctly supports most common [SQL](https://en.wikipedia.org/wiki/SQL)
  features.
  **符合 SQL 标准：** 正确支持最常见的 [SQL](https://en.wikipedia.org/wiki/SQL) 特性。
* **Strongly consistent:** committed writes are immediately visible to all readers ([linearizability](https://en.wikipedia.org/wiki/Linearizability)).
  **强一致：** 已提交的写入对所有读取者立即可见（[线性一致性](https://en.wikipedia.org/wiki/Linearizability)）。
* **Transactional:** provides [ACID](https://en.wikipedia.org/wiki/ACID) transactions
  **支持事务：** 提供 [ACID](https://en.wikipedia.org/wiki/ACID) 事务
  * **Atomic:** groups of writes are applied as a single, atomic unit.
    **原子性：** 一组写入作为一个单一的原子单元被应用。
  * **Consistent:** database constraints and referential integrity are always enforced.
    **一致性：** 数据库约束和引用完整性始终被强制执行。
  * **Isolated:** concurrent transactions don't affect each other ([snapshot isolation](https://en.wikipedia.org/wiki/Snapshot_isolation)).
    **隔离性：** 并发事务之间互不影响（[快照隔离](https://en.wikipedia.org/wiki/Snapshot_isolation)）。
  * **Durable:** committed writes are never lost.
    **持久性：** 已提交的写入永远不会丢失。

For simplicity, toyDB is:

出于简单性的考虑，toyDB 做出了以下取舍：

* **Not scalable:** every node stores the full dataset, and reads/writes execute on one node.
  **不可扩展：** 每个节点都存储完整数据集，读写都在单个节点上执行。
* **Not reliable:** only handles crash failures, not e.g. partial network partitions or node stalls.
  **不可靠：** 只处理崩溃故障，不处理诸如部分网络分区或节点停顿等情况。
* **Not performant:** data processing is slow, and not optimized at all.
  **性能不佳：** 数据处理速度慢，完全没有做优化。
* **Not efficient:** loads entire tables into memory, no compression or garbage collection, etc.
  **效率不高：** 会把整张表加载进内存，没有压缩、垃圾回收等机制。
* **Not full-featured:** only basic SQL functionality is implemented.
  **功能不全：** 只实现了基础的 SQL 功能。
* **Not backwards compatible:** changes to data formats and protocols will break databases.
  **不向后兼容：** 数据格式和协议的变更会导致已有数据库失效。
* **Not flexible:** nodes can't be added or removed while running, and take a long time to join.
  **不灵活：** 运行期间不能增删节点，且节点加入耗时较长。
* **Not secure:** there is no authentication, authorization, nor encryption.
  **不安全：** 没有认证、授权，也没有加密。

## Components（组件）

Internally, toyDB is made up of a few main components:

在内部，toyDB 由几个主要组件构成：

* **Storage engine:** stores data on disk and manages transactions.
  **存储引擎（storage engine）：** 将数据存储在磁盘上并管理事务。
* **Raft consensus engine:** replicates data and coordinates cluster nodes.
  **Raft 共识引擎：** 复制数据并协调集群节点。
* **SQL engine:** organizes SQL data, manages SQL sessions, and executes SQL statements.
  **SQL 引擎：** 组织 SQL 数据、管理 SQL 会话并执行 SQL 语句。
* **Server:** manages network communication, both with SQL clients and Raft nodes.
  **服务端（server）：** 管理网络通信，包括与 SQL 客户端和 Raft 节点的通信。
* **Client:** provides a SQL user interface and communicates with the server.
  **客户端（client）：** 提供 SQL 用户界面并与服务端通信。

This diagram illustrates the internal structure of a single toyDB node:

下图展示了单个 toyDB 节点的内部结构：

![toyDB architecture](./images/architecture.svg)

We will go through each of these components from the bottom up.

接下来我们将自底向上逐一介绍这些组件。

---

<p align="center">
← <a href="index.md">toyDB Architecture</a> &nbsp; | &nbsp; <a href="storage.md">Storage Engine</a> →
</p>
