# toyDB

toyDB is a distributed SQL database in Rust, built from scratch as an educational project. Main
features:

toyDB 是一个用 Rust 从零构建的分布式 SQL 数据库，作为一个教学项目。主要特性：

* Raft distributed consensus for linearizable state machine replication.

* 基于 Raft 分布式共识、可线性化的状态机复制。

* ACID transactions with MVCC-based snapshot isolation.

* 基于 MVCC 快照隔离的 ACID 事务。

* Pluggable storage engine with BitCask and in-memory backends.

* 可插拔存储引擎，提供 BitCask 与内存后端。

* Iterator-based query engine with heuristic optimization and time-travel  support.

* 基于迭代器的查询引擎，支持启发式优化与时间旅行查询。

* SQL interface including joins, aggregates, and transactions.

* SQL 接口，包括连接、聚合与事务。

toyDB is not distributed as a crate, see <https://github.com/erikgrinaker/toydb> for more.

toyDB 并未作为 crate 发布，更多信息见 <https://github.com/erikgrinaker/toydb>。

This crate used to contain the [joydb](https://crates.io/crates/joydb) database. Thanks to Serhii
Potapov for donating the crate name.

本 crate 此前曾属于 [joydb](https://crates.io/crates/joydb) 数据库。感谢 Serhii Potapov 捐赠该 crate 名称。
