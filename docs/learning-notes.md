# toyDB 学习笔记与会话记录

> 本文持续记录学习偏好、实际进度、已讲知识和配套资料；不是逐字聊天记录。
> 课程目录与知识图谱见 [learning-map.md](learning-map.md)。完整讲解见 [lectures/](lectures/README.md)。

## 当前进度与下次入口

- 学习目标：以**独立读懂实现**为主，能沿接口、数据结构、调用关系和测试理解 toyDB；原理解释为此服务，不默认安排重写项目。
- 学习起点：Rust 语法概念学过一遍但不熟练；已有索引、事务与多数派复制的初步直觉，随源码校准。
- 教学约定：**自底向上、小步带读、讲解为主、用户按需追问；暂时跳过测验、苏格拉底式反问、强制作业与过关检查。** 不再沿用最初的 grilling 访谈方式。
- 时间安排：无明确截止日期，不要求固定周计划或频率。
- 已完成：项目概览；第一节 **01.1**；分章规划与知识依赖图；**00.1–17.8 全部讲义**（`docs/lectures/00.md`–`17.md`，106 个知识点标题与学习地图一一对应）。
- 进度含义：已讲解不等于已检验掌握。只记录实际讲解与验证，不评定学习者能力。
- 尚未执行：运行集群、`cargo test`、交互式事务和故障实验。这些不是讲义缺口，需要真机时再做。
- 本轮只改学习文档，没有修改实现代码。行为示例来自当前源码和仓库脚本的预期输出。
- **下一步：按需追问任意编号，或从 [lectures/README.md](lectures/README.md) 顺序阅读。**

下次可以直接这样开始：

> 请阅读 docs/lectures/README.md。我要追问 04.3 可见性规则 / 从某一章开始细读。讲解为主，先跳过测验和反问。

## 1. 项目概览

toyDB 是 Erik Grinaker 用 Rust 编写的教学用分布式 SQL 数据库。目标是简单、可理解且功能正确，不追求生产级性能、扩展性或可靠性。

主要能力：

- Raft 共识与状态机复制。
- 基于 MVCC 的 ACID 事务和快照隔离。
- 可插拔 KV 存储：Memory 与 BitCask。
- SQL 解析、规划、启发式优化和迭代器式执行。
- Join、聚合、事务、EXPLAIN 和历史版本查询。

### 目录地图

| 模块 | 位置 | 学习重点 |
| --- | --- | --- |
| 存储接口 | `src/storage/engine.rs` | KV 操作与有序扫描 |
| 内存存储 | `src/storage/memory.rs` | 最简单的存储实现 |
| 磁盘存储 | `src/storage/bitcask.rs` | 追加写、索引、恢复、压缩 |
| MVCC | `src/storage/mvcc.rs` | 版本、可见性、冲突、提交和回滚 |
| 编码 | `src/encoding/` | key 的保序编码与 value 编码 |
| Raft | `src/raft/` | 选举、日志复制、状态机、读一致性 |
| SQL 存储与复制 | `src/sql/engine/` | SQL 数据如何落到 KV，以及如何接入 Raft |
| SQL 解析 | `src/sql/parser/` | Lexer、Parser、AST |
| SQL 规划 | `src/sql/planner/` | 名称解析、执行计划、优化 |
| SQL 执行 | `src/sql/execution/` | 执行器、Join、聚合、会话事务 |
| 网络与客户端 | `src/server.rs`、`src/client.rs` | 请求与节点通信 |
| 程序入口 | `src/bin/` | toydb、toysql、toydump、workload |
| 集成测试 | `tests/` | 脚本测试与测试集群 |
| 架构文档 | `docs/architecture/index.md` | 自底向上的源码导览 |
| 分章讲义 | `docs/lectures/` | 00.1–17.8 的完整讲解 |

### 会话中的纠正与边界

- 隔离级别是 **SI（Snapshot Isolation）**，不是 SSI（Serializable Snapshot Isolation）。SI 不等于串行化，不能据此排除写偏斜。
- SQL 与 Raft 使用独立端口，不是单端口多路复用。README 示例中 SQL 为 9601–9605，Raft 为 9701–9705。
- 初次概览中的“约 9000 行代码”来自不完整的 glob 统计，不能作为项目完整代码量。
- “SQL 命令经 Raft 复制”只是粗略概述；究竟复制什么操作、哪些读需要 Raft 协调，应结合 `src/sql/engine/raft.rs` 和架构文档确认，不能直接理解为复制 SQL 字符串。
- 根据架构概览，每个节点存储完整数据，读写在一个节点上执行；不要假设它具备分片扩展能力。
- 项目能容忍少数节点崩溃，但这不等于生产级高可用：架构文档明确指出对复杂故障、性能、安全和兼容性等方面的限制。

## 2. 学习路线概览

当前主线：**有序 KV → BitCask → 编码 → MVCC → SQL 数据与处理 → Raft → 集成与完整请求链路**。

具体章节、小节编号、源码入口和知识依赖，以 [learning-map.md](learning-map.md) 为准。下面保留早期路线与可选演示素材；不是作业，不要求先运行集群，也不作为进度门槛。

不要一开始从 server 入口追所有调用，也不必先啃 Raft。

### 阶段一：体验数据库

阅读：

- `README.md`
- `docs/examples.md`
- `docs/architecture/overview.md`

启动集群：

```bash
./cluster/run.sh
```

另开终端连接：

```bash
cargo run --release --bin toysql
```

可选演示：

1. 建表、插入和查询。
2. 显式开启事务，修改后回滚。
3. 用两个客户端观察未提交修改是否可见。
4. 对过滤和 JOIN 查询运行 EXPLAIN。

要带走的问题：SQL 怎么变成 KV？回滚怎么实现？连接任意节点后，写入最终在哪里执行？

### 阶段二：存储引擎

阅读顺序：

```text
docs/architecture/storage.md
src/storage/engine.rs
src/storage/memory.rs
src/storage/bitcask.rs
```

重点问题：

- 接口为什么除了 get/set，还需要有序扫描？
- BitCask 怎么从 key 定位磁盘数据？
- 覆盖写、删除与重启恢复怎么处理？
- 旧数据何时清理？

实验：结合现有测试观察“写入 → 覆盖 → 删除 → 重新打开”，区分逻辑数据和磁盘记录。

讲解收束：串起“追加写文件为什么也能实现可更新的 KV 数据库”，不安排过关检查。

### 阶段三：编码与 MVCC

阅读顺序：

```text
docs/architecture/encoding.md
src/encoding/keycode.rs
docs/architecture/mvcc.md
src/storage/mvcc.rs
```

重点问题：

- 为什么 key 的字节排序需要保留业务排序？
- 同一 key 的多个版本怎么存？
- 事务如何确定可见版本？
- 并发写冲突如何检测？
- 提交和回滚分别修改哪些状态？
- 历史版本如何支撑时间旅行查询？

讲解素材：由讲解者展开以下时序，结合现有测试或可选双客户端演示，不要求学习者先预测。

```text
T1 开始
T2 开始
T1 写入 x = 10
T1 提交
T2 读取 x
T3 开始并读取 x
```

讲解收束：版本可见性，以及 SI 与可串行化的区别。

### 阶段四：SQL 存储、规划与执行

先读：

- `docs/architecture/sql-storage.md`
- `src/sql/engine/local.rs`

然后追踪一条简单查询：

```sql
SELECT title FROM movies WHERE id = 1;
```

路径：

```text
lexer.rs → parser.rs / ast.rs
         → planner.rs / plan.rs
         → optimizer.rs
         → executor.rs
```

以上文件分别位于 `src/sql/parser/`、`src/sql/planner/` 和 `src/sql/execution/`。另外阅读 `src/sql/execution/session.rs`，理解自动提交与显式事务。

重点观察：

- SQL 文本、token、AST、执行计划之间的区别。
- 表名和列名如何解析。
- 过滤扫描何时变成主键查找。
- 执行器如何从存储取得行并输出结果。

之后再增加 JOIN、聚合和排序，结合 `EXPLAIN` 理解优化前后的等价性。

讲解收束：一行数据如何从 KV 变成查询结果。

### 阶段五：Raft

阅读顺序：

```text
docs/architecture/raft.md
src/raft/message.rs
src/raft/log.rs
src/raft/state.rs
src/raft/node.rs
docs/architecture/sql-raft.md
src/sql/engine/raft.rs
```

重点问题：

- SQL 层交给 Raft 的具体操作是什么？
- Follower 如何处理客户端请求？
- 日志追加、多数派确认、状态机 apply 有何区别？
- 为什么本地日志落盘不等于操作成功？
- 如何防止旧 Leader 返回过期读取结果？

实验：在本地测试集群停止 Leader，观察选举，再恢复节点观察日志追赶。此实验不代表已经验证生产级容灾。

讲解收束：区分 Raft 的复制一致性与 MVCC 的事务隔离。

### 阶段六：完整请求链路

阅读：

- `src/client.rs`
- `src/server.rs`
- `src/bin/toydb.rs`
- `tests/testcluster.rs`
- `tests/tests.rs`

分别跟踪一次读与一次写，标出客户端、SQL 会话、规划执行、Raft、本地事务和 KV 存储之间的真实调用关系。特别确认哪些操作经过 Raft、在哪个节点执行。

## 3. 学习方法与时间安排

每次只讲一个小主题：

1. 讲解者先核对架构文档、当前实现与相关测试。
2. 解释要解决的问题，以及它在整个数据库中的位置。
3. 带读最小代码片段，随用随补 Rust 语法。
4. 展开行为示例和边界；如执行验证，明确说明实际命令和结果。
5. 记录已讲内容和下一节入口，停下等待用户追问或继续。

不要求先答题、不设置过关条件；跳过的是对学习者的检查，不是对讲解内容正确性的核对。

项目依赖 `goldenscript`，可以结合脚本测试阅读输入与预期输出。

建议测试命令（本次尚未运行）：

```bash
cargo test -- --list
cargo test storage::
cargo test raft::
```

先列出实际测试名；名称过滤并不保证覆盖某模块所有相关测试。

不按周排期，按 `learning-map.md` 中的小节推进。复杂主题可拆成多次，不需要一次讲完一个文件。

Rust 按需补所有权、借用、trait、泛型、迭代器、Result、线程与 channel，不必先学完整门语言。

## 4. 配套材料

选择原则：一门课程建立全局，一本书解释取舍，核心论文对照实现。不要同时开启多个大型数据库项目。

### CMU 15-445：Database Systems

链接：<https://15445.courses.cs.cmu.edu/>

- 存储层对应存储、索引与恢复主题。
- SQL 执行对应查询执行和 Join 算法。
- Planner 对应查询规划与优化。
- MVCC 对应并发控制与隔离级别。

建议按源码进度选讲义与视频。BusTub 主要使用 C++，架构与 toyDB 不同，初期不必同时做完整实验。

### Database Internals — Alex Petrov

链接：<https://www.databass.dev/>

用于理解 B-tree、LSM-tree、日志、恢复、压缩、复制和一致性等设计取舍。先理解 toyDB 的 BitCask，再比较其他存储结构。不把书中所有生产机制当成 toyDB 必须添加的功能。

### BitCask 原始论文

标题：Bitcask: A Log-Structured Hash Table for Fast Key/Value Data

链接：<https://riak.com/assets/bitcask-intro.pdf>

对照 `src/storage/bitcask.rs` 阅读追加写、keydir、tombstone、恢复与 merge/compaction。关注内存索引对 key 数量的限制。

可选讲解图：“内存索引 → 文件偏移 → 数据记录”，对照论文与实现。

### Raft 论文与动画

论文标题：In Search of an Understandable Consensus Algorithm

- 论文入口：<https://raft.github.io/>
- 可视化：<https://thesecretlivesofdata.com/raft/>

顺序：动画建立直觉 → 论文选举/复制/安全性 → Figure 2 对照 `node.rs` → 测试中的异常路径。

动画不是完整规范；提交条件、旧任期消息和日志冲突处理要回到论文与代码。

### Designing Data-Intensive Applications（DDIA）

中文常见书名：《设计数据密集型应用》。作者及章节编排随版本确认；第一版作者为 Martin Kleppmann。

链接：<https://dataintensive.net/>

优先阅读事务、复制、分布式故障、一致性与共识主题。用来区分持久化、多数派确认、快照隔离、线性一致性和串行化。

跨章节讲解主题：为什么使用 Raft，并不代表 SQL 事务自动拥有可串行化隔离？

### Hermitage：隔离级别案例

链接：<https://github.com/ept/hermitage>

把并发异常变成具体事务时序。由讲解者选案例、按支持的 SQL 语法解释或演示；需要验证时再执行，不要求学习者先预测。不要预设案例中的异常一定会在 toyDB 出现。

### Use The Index, Luke

链接：<https://use-the-index-luke.com/>

补充索引查找、扫描、联合索引和执行计划的使用直觉。它面向实际 SQL 系统，不代表 toyDB 支持其中全部机制。

### Rust 配套

- The Rust Book：<https://doc.rust-lang.org/book/>
- Rustlings：<https://github.com/rust-lang/rustlings>

按需查阅，不必先深入 unsafe 或异步运行时。

### 项目自带参考资料

`docs/references.md` 是作者的参考资料入口，适合后续查阅。本次只是推荐了该文件，尚未逐项阅读核验其中内容。

上述外部链接是会话推荐资料；本次未进行联网访问或核验课程当前版本。

## 5. 最小配套组合

| 学习模块 | 配套材料 |
| --- | --- |
| 存储 | BitCask 论文 |
| MVCC | DDIA 事务主题 + Hermitage |
| SQL 执行与优化 | CMU 15-445 对应讲义 |
| Raft | 动画 + 论文 Figure 2 |
| 扩展比较 | Database Internals |

让资料回答源码中遇到的问题，不要把读完所有资料变成开始学习的前置条件。

## 6. 已讲记录：01.1 有序 KV 与 Memory 扫描

### 本节问题与源码

主题：**为什么 KV 存储除了 `get/set`，还需要有序扫描？**

已阅读并讲解：

- `src/storage/engine.rs` 的接口部分。
- `src/storage/memory.rs` 的 Memory、get、scan 和迭代器实现。
- `src/storage/testscripts/engine/scan` 的输入与预期输出。
- 对应背景：`docs/architecture/storage.md`。

### 1. 存储层的职责边界

```rust
pub struct Memory(BTreeMap<Vec<u8>, Vec<u8>>);
```

`Memory` 是 Rust 标准库 `BTreeMap` 的包装。key/value 都是字节数组；这一层不认识表、列、SQL，也不直接实现事务。`BTreeMap` 是 B 树映射，不是 B+ 树。

上层负责把业务数据编码成字节。底层负责保存、查询、删除与按 key 顺序遍历。

### 2. 点查需要知道完整 key

```rust
fn get(&mut self, key: &[u8]) -> Result<Option<Vec<u8>>> {
    Ok(self.0.get(key).cloned())
}
```

- `self.0`：取出 tuple struct 包装的 map。
- `&[u8]`：借用 key 的字节，不转移所有权。
- `.cloned()`：复制查到的 value，让调用者拥有返回数据。
- `Ok(Some(value))` 是找到；`Ok(None)` 是不存在；`Err(error)` 是失败。当前 Memory 的这条路径只产生前两种，但共用接口需要表达磁盘错误。

假设上层 key 是 `users/001`、`users/004`、`users/009`，只用 get 无法列出所有用户，因为调用者未必知道现存 ID。这个字符串布局仅为教学示意，不是 toyDB 真实编码。

### 3. 有序 key 把集合访问变成区间访问

按共同前缀组织的 key，在字节字典序中连续排列。范围扫描由此服务于表行扫描、MVCC 版本扫描、Raft 日志后缀扫描等上层需求。

`Engine::scan_prefix` 不是另一套扫描实现，而是调用：

```rust
self.scan(keycode::prefix_range(prefix))
```

前缀如何变成包含/排除边界，留在 03.5 讲解。

### 4. Memory 直接复用标准库范围迭代器

```rust
fn scan(&mut self, range: impl RangeBounds<Vec<u8>>) -> Self::ScanIterator<'_> {
    ScanIterator(self.0.range(range))
}
```

`BTreeMap` 已经维护顺序，不需要扫描后再排序。包装器每次 `next()` 才取得并复制一对 key/value；不是在调用 scan 时提前收集全部结果。

生命周期、关联类型与动态分派暂未深入，归入 01.3。底层逐项迭代不代表跨 Raft 也能全程流式处理，该边界在 15.7 再连接。

### 5. 仓库测试的行为示例

写入顺序：`a=1`、`b=2`、`ba=21`、`bb=22`、`c=3`、`C=3`。

全量扫描预期 key 顺序：

```text
C, a, b, ba, bb, c
```

这不是插入顺序。ASCII 字符 `C` 的字节值小于 `a`，所以排在前面。

```text
scan b..bb     → b, ba
scan b..=bb    → b, ba, bb
```

因为字节字典序下 `b < ba < bb`；`..` 排除右边界，`..=` 包含右边界。现有脚本也覆盖反向扫描和无界范围。

### 本节总结与验证状态

**get 按完整 key 找一个值；有序 scan 按范围找一组值。上层通过设计 key，把业务集合访问转成范围访问。**

- 内容状态：已讲解，用户反馈“很好”；未进行掌握程度测验。
- 验证状态：已阅读仓库测试输入与预期输出，未实际运行测试。
- 后续全部章节已写入 [lectures/](lectures/README.md)，不再按“下一节 02.1”推进。
