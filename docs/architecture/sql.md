# SQL Engine（SQL 引擎）

The SQL engine provides support for the SQL query language, and is the main database interface. It
uses a key/value store for data storage, MVCC for transactions, and Raft for replication. The SQL
engine itself consists of several distinct components that form a pipeline:

SQL 引擎提供了对 SQL 查询语言的支持，是数据库的主要接口。它使用键/值存储来保存数据，使用 MVCC 处理事务，使用 Raft 做复制。SQL 引擎本身由几个不同的组件组成，构成一条流水线：

> Client → Session → Lexer → Parser → Planner → Optimizer → Executor → Storage

The SQL engine is located in the [`sql`](https://github.com/erikgrinaker/toydb/tree/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql)
module. We'll discuss each of the components in a bottom-up manner.

SQL 引擎位于 [`sql`](https://github.com/erikgrinaker/toydb/tree/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql) 模块中。我们将自底向上逐一讨论各个组件。

The SQL engine is tested as a whole by test scripts under
[`src/sql/testscripts`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/testscripts).
These typically take a raw SQL string as input, execute them against an in-memory storage engine,
and output the result along with intermediate state such as the query plan, storage operations,
and binary key/value data.

SQL 引擎整体由 [`src/sql/testscripts`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/testscripts) 下的测试脚本来测试。这些脚本通常以原始 SQL 字符串作为输入，在内存存储引擎上执行它们，并输出结果以及中间状态，例如查询计划、存储操作和二进制键/值数据。

---

<p align="center">
← <a href="raft.md">Raft Consensus</a> &nbsp; | &nbsp; <a href="sql-data.md">SQL Data Model</a> →
</p>
