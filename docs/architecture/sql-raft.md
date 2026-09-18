# SQL Raft Replication（SQL Raft 复制）

toyDB uses Raft to replicate SQL storage across a cluster of nodes (see the Raft section for
details). All nodes will store a full copy of the SQL database, and the Raft leader will replicate
writes across nodes and execute reads.

toyDB 使用 Raft 在集群的多个节点之间复制 SQL 存储（详见 Raft 一节）。所有节点都会保存 SQL 数据库的完整副本，Raft leader 负责在各节点之间复制写操作并执行读操作。

Recall the Raft state machine interface `raft::State`:

回顾一下 Raft 状态机接口 `raft::State`：

<https://github.com/erikgrinaker/toydb/blob/8782c2b05f11333c1586ef248f1a13dc1c8dec4a/src/raft/state.rs#L4-L51>

In toyDB, the state machine is just a `sql::engine::Local` storage engine with a thin wrapper:

在 toyDB 中，状态机就是一个 `sql::engine::Local` 存储引擎再加上一层薄封装：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L278-L291>

Raft will submit read and write commands to this state machine as binary `Vec<u8>` data, so we have
to represent the methods of `sql::engine::Engine` as binary Raft commands. We do this as two
enums, `sql::engine::raft::Read` and `sql::engine::raft::Write`, which we'll Bincode-encode:

Raft 会以二进制 `Vec<u8>` 数据的形式向这个状态机提交读写命令，因此我们必须把 `sql::engine::Engine` 的方法表示为二进制的 Raft 命令。我们用两个枚举 `sql::engine::raft::Read` 和 `sql::engine::raft::Write` 来实现，并对它们进行 Bincode 编码：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L16-L71>

Notice that almost all requests include a `mvcc::TransactionState`. Most of the useful methods of
`sql::engine::Engine` are on the `sql::engine::Transaction`, but unlike the `Local` engine, below
Raft we can't hold on to a `Transaction` object in memory between each command -- nodes may restart
and leadership may move, and we want client transactions to keep working despite this. Instead, we
will use the client-supplied `mvcc::TransactionState` to reconstruct a `Transaction` for every
command via `mvcc::Transaction::resume()` and call methods on it.

注意，几乎所有请求都包含一个 `mvcc::TransactionState`。`sql::engine::Engine` 的大部分有用方法都定义在 `sql::engine::Transaction` 上，但与 `Local` 引擎不同，在 Raft 之下我们无法在命令之间把 `Transaction` 对象保存在内存中——节点可能重启，leader 也可能易主，而我们希望客户端事务在这些情况下仍能继续工作。因此，我们改用客户端提供的 `mvcc::TransactionState`，通过 `mvcc::Transaction::resume()` 为每条命令重建一个 `Transaction`，再在它上面调用方法。

When the state machine receives a write command, it decodes it as a `Write` and calls the
appropriate `Local` method. The result is Bincode-encoded and returned to the caller, who knows what
return type to expect for a given command. The state machine also keeps track of the Raft applied
index of each command as a separate key in the key/value store.

状态机收到写命令时，会把它解码为 `Write` 并调用相应的 `Local` 方法。结果经 Bincode 编码后返回给调用方，调用方知道给定命令应期待什么返回类型。状态机还会把每条命令的 Raft applied index 作为一个单独的键记录在键/值存储中。

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L346-L367>

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L306-L338>

Similarly, read commands are decoded as a `Read` and the appropriate `Local` method is called:

类似地，读命令会被解码为 `Read` 并调用相应的 `Local` 方法：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L369-L404>

That's the state machine running below Raft. But how do we actually send these commands to Raft and
receive results? That's handled by the `sql::engine::Raft` implementation, which uses a channel to
send requests to the local Raft node (we'll see how this plumbing works in the server section):

以上就是运行在 Raft 之下的状态机。但我们实际要如何把这些命令发送给 Raft 并接收结果呢？这由 `sql::engine::Raft` 实现负责，它使用一个通道向本地 Raft 节点发送请求（这些管道机制的工作方式将在 server 一节中介绍）：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L80-L95>

The channel takes a `raft::Request` containing binary Raft client requests and a return channel
where the Raft node can send back a `raft::Response`. The Raft engine has a few convenience methods
to send requests and receive responses, for both read and write requests:

该通道接收包含二进制 Raft 客户端请求的 `raft::Request`，以及一个返回通道，Raft 节点可以通过它送回 `raft::Response`。Raft 引擎为读请求和写请求都提供了几个便捷方法，用于发送请求和接收响应：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L114-L135>

And the implementation of the `sql::engine::Engine` and `sql::engine::Transaction` traits simply
send these requests via Raft:

而 `sql::engine::Engine` 和 `sql::engine::Transaction` 这两个 trait 的实现只是简单地通过 Raft 发送这些请求：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/engine/raft.rs#L194-L276>

One thing to note here is that we don't support streaming data via Raft, so e.g. the
`Transaction::scan` method will buffer the entire result in a `Vec`. With a full table scan, this
will load the entire table into memory -- that's unfortunate, but we keep it simple.

这里需要注意的一点是，我们不支持通过 Raft 进行流式数据传输，因此例如 `Transaction::scan` 方法会把整个结果缓冲在一个 `Vec` 中。对于全表扫描，这会把整张表加载进内存——虽然不太理想，但我们选择保持简单。

To summarize, this is what happens when `Transaction::insert()` is called to insert a row via Raft:

总结一下，当调用 `Transaction::insert()` 通过 Raft 插入一行时，会发生以下事情：

1. `sql::engine::raft::Transaction::insert()`: called to insert a row.
1. `sql::engine::raft::Transaction::insert()`：调用以插入一行。
1. `sql::engine::raft::Write::Insert`: enum representation of the insert command.
1. `sql::engine::raft::Write::Insert`：插入命令的枚举表示。
1. `raft::Request::Write`: raft request containing the Bincode-encoded `Write::Insert` command.
1. `raft::Request::Write`：包含 Bincode 编码的 `Write::Insert` 命令的 Raft 请求。
1. `sql::engine::raft::Engine::tx`: sends the `Request::Write` and response channel to Raft.
1. `sql::engine::raft::Engine::tx`：将 `Request::Write` 和响应通道发送给 Raft。
1. `raft::Node::step()`: the `Request::Write` is given to Raft in a `Message::ClientRequest`.
1. `raft::Node::step()`：`Request::Write` 以 `Message::ClientRequest` 的形式交给 Raft 处理。
1. Raft does its replication thing, and commits the command's log entry.
1. Raft 执行复制流程，并提交该命令的日志条目。
1. `raft::State::apply()`: the Bincode-encoded `Write::Insert` is passed to the state machine.
1. `raft::State::apply()`：Bincode 编码的 `Write::Insert` 被传给状态机。
1. `sql::engine::raft::State::apply()`: decodes the command to a `Write::Insert`.
1. `sql::engine::raft::State::apply()`：将命令解码为 `Write::Insert`。
1. `sql::engine::raft::State::local`: contains the `Local` engine on each node.
1. `sql::engine::raft::State::local`：在每个节点上包含 `Local` 引擎。
1. `sql::engine::local::Engine::resume()`: called to obtain the SQL/MVCC transaction.
1. `sql::engine::local::Engine::resume()`：调用以获取 SQL/MVCC 事务。
1. `sql::engine::local::Transaction::insert()`: the row is inserted to the local engine.
1. `sql::engine::local::Transaction::insert()`：该行被插入到本地引擎中。
1. `raft::RawNode::tx`: the `Ok(())` result is sent as a Bincode-encoded `Message::ClientResponse`.
1. `raft::RawNode::tx`：`Ok(())` 结果以 Bincode 编码的 `Message::ClientResponse` 发送出去。
1. `sql::engine::raft::Transaction::insert()`: receives the result and returns it to the caller.
1. `sql::engine::raft::Transaction::insert()`：接收结果并将其返回给调用方。

The plumbing here will be covered in more details in the server section.

这些管道机制将在 server 一节中更详细地介绍。

---

<p align="center">
← <a href="sql-storage.md">SQL Storage</a> &nbsp; | &nbsp; <a href="sql-parser.md">SQL Parsing</a> →
</p>
