# Server（服务器）

Now that we've gone over the individual components, we'll tie them all together in the toyDB
server `toydb::Server`, located in the [`server`](https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs) module.

在介绍完各个组件之后，现在我们把它们组合在一起，这就是 toyDB 服务器 `toydb::Server`，位于 [`server`](https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs) 模块。

The server wraps an inner Raft node `raft::Node`, which manages the SQL state machine, and is
responsible for routing network traffic between the Raft node, its Raft peers, and SQL clients.

服务器包装了一个内部的 Raft 节点 `raft::Node`（它管理着 SQL 状态机），并负责在 Raft 节点、它的 Raft 对等节点以及 SQL 客户端之间路由网络流量。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L27-L44>

For network protocol, the server uses the Bincode encoding that we've discussed in the encoding
section, sent over a TCP connection. There's no need for any further framing, since Bincode knows
how many bytes to expect for each message depending on the type it's decoding into.

在网络协议方面，服务器使用我们在编码一节中讨论过的 Bincode 编码，并通过 TCP 连接发送。由于 Bincode 根据要解码的目标类型知道每条消息应该有多少字节，因此不需要任何额外的帧定界。

The server does not use [async Rust](https://rust-lang.github.io/async-book/) and e.g.
[Tokio](https://tokio.rs), instead opting for regular OS threads. Async Rust can significantly
complicate the code, which would obscure the main concepts, and any efficiency gains would be
entirely irrelevant for toyDB.

服务器不使用 [async Rust](https://rust-lang.github.io/async-book/) 和 [Tokio](https://tokio.rs) 之类的工具，而是使用普通的操作系统线程。异步 Rust 会显著增加代码复杂度，从而掩盖主要概念，而它带来的任何效率提升对 toyDB 来说都完全无关紧要。

Internally in the server, messages are passed around between threads using
[Crossbeam channels](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html).

在服务器内部，消息通过 [Crossbeam 通道](https://docs.rs/crossbeam/latest/crossbeam/channel/index.html)在线程之间传递。

The main server loop `Server::serve()` listens for inbound TCP connections on port 9705 for Raft
peers and 9605 for SQL clients, and spawns threads to process them. We'll look at Raft and SQL
services separately.

主服务器循环 `Server::serve()` 在 9705 端口监听 Raft 对等节点的入站 TCP 连接，在 9605 端口监听 SQL 客户端的连接，并生成线程来处理它们。我们将分别介绍 Raft 和 SQL 服务。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L66-L110>

## Raft Routing（Raft 路由）

The heart of the server is the Raft processing thread `Server::raft_route()`. This is responsible
for periodically ticking the Raft node via `raft::Node::tick()`, stepping inbound messages from
Raft peers into the node via `raft::Node::step()`, and sending outbound messages to peers.

服务器的核心是 Raft 处理线程 `Server::raft_route()`。它负责通过 `raft::Node::tick()` 周期性地驱动 Raft 节点，通过 `raft::Node::step()` 把来自 Raft 对等节点的入站消息送入节点，并向对等节点发送出站消息。

It also takes inbound Raft client requests from the `sql::engine::Raft` SQL engine, steps them
into the Raft node via `raft::Node::step()`, and passes responses back to the appropriate client
as the node emits them.

它还接收来自 `sql::engine::Raft` SQL 引擎的入站 Raft 客户端请求，通过 `raft::Node::step()` 将其送入 Raft 节点，并在节点发出响应时把响应传回相应的客户端。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L169-L249>

When the node starts up, it spawns a `Server::raft_send_peer()` thread for each Raft peer to send
outbound messages to them.

节点启动时，会为每个 Raft 对等节点生成一个 `Server::raft_send_peer()` 线程，用来向它们发送出站消息。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L84-L91>

These threads continually attempt to connect to the peer via TCP, and then read any outbound
`raft::Envelope(raft::Message)` messages from `Server::raft_route()` via a channel and writes the
messages into the TCP connection using Bincode:

这些线程不断尝试通过 TCP 连接到对等节点，然后通过通道从 `Server::raft_route()` 读取出站的 `raft::Envelope(raft::Message)` 消息，并用 Bincode 把消息写入 TCP 连接：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L146-L167>

The server also continually listens for inbound Raft TCP connections from peers in
`Server::raft_accept()`:

服务器还在 `Server::raft_accept()` 中持续监听来自对等节点的入站 Raft TCP 连接：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L112-L134>

When an inbound connection is accepted, a `Server::raft_receive_peer()` thread is spawned that reads
Bincode-encoded `raft::Envelope(raft::Message)` messages from the TCP connection and sends them to
`Server::raft_route()` via a channel.

当接受一个入站连接后，会生成一个 `Server::raft_receive_peer()` 线程，它从 TCP 连接中读取 Bincode 编码的 `raft::Envelope(raft::Message)` 消息，并通过通道发送给 `Server::raft_route()`。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L136-L144>

The Raft cluster is now fully connected, and the nodes can all talk to each other.

至此，Raft 集群已完全连通，所有节点之间都可以相互通信。

## SQL Service（SQL 服务）

Next, let's serve some SQL clients. The SQL service uses the enums `toydb::Request` and
`toydb::Response` as a client protocol, again Bincode-encoded over TCP.

接下来，让我们为一些 SQL 客户端提供服务。SQL 服务使用枚举 `toydb::Request` 和 `toydb::Response` 作为客户端协议，同样是通过 TCP 进行 Bincode 编码。

The primary request type is `Request::Execute` which executes a SQL statement against a
`sql::execution::Session` and returns a `sql::execution::StatementResult`, as we've seen previously.

主要的请求类型是 `Request::Execute`，它针对 `sql::execution::Session` 执行一条 SQL 语句，并返回一个 `sql::execution::StatementResult`，如前文所述。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L312-L337>

The server sets up a `sql::engine::Raft` SQL engine, with a Crossbeam channel that's used to send
`raft::Request` Raft client requests to `Server::raft_route()` and onwards to the local
`raft::Node`.  It then spawns a `Server::sql_accept()` thread to listen for inbound SQL client
connections:

服务器会建立一个 `sql::engine::Raft` SQL 引擎，并使用一个 Crossbeam 通道把 `raft::Request` Raft 客户端请求发送到 `Server::raft_route()`，再转发给本地的 `raft::Node`。随后它生成一个 `Server::sql_accept()` 线程来监听入站的 SQL 客户端连接：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L104-L106>

When a SQL client connection is accepted, a new client session `sql::execution::Session` is set up
for the client, and we spawn a `Server::sql_session()` thread to serve the connection:

当接受一个 SQL 客户端连接时，会为该客户端建立一个新的客户端会话 `sql::execution::Session`，并生成一个 `Server::sql_session()` 线程来为该连接服务：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L251-L272>

These session threads continually read `Request` messages from the client, execute them against the
SQL session (and ultimately the Raft node), before sending a `Response` back to the client.

这些会话线程不断从客户端读取 `Request` 消息，在 SQL 会话（最终是 Raft 节点）上执行它们，然后把 `Response` 发回给客户端。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/server.rs#L274-L309>

## `toydb` Binary（`toydb` 可执行程序）

The `toydb` binary in `src/bin/toydb.rs` launches the server, and is a thin wrapper around
`toydb::Server`. It is a tiny [`clap`](https://docs.rs/clap/latest/clap/) command:

`src/bin/toydb.rs` 中的 `toydb` 可执行程序负责启动服务器，它只是 `toydb::Server` 的一层薄封装。它是一个很小的 [`clap`](https://docs.rs/clap/latest/clap/) 命令：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/bin/toydb.rs#L82-L89>

It first parses a server configuration from the `toydb.yaml` file:

它首先从 `toydb.yaml` 文件解析服务器配置：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/bin/toydb.rs#L30-L59>

Then it initializes the Raft log storage and SQL state machine:

然后初始化 Raft 日志存储和 SQL 状态机：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/bin/toydb.rs#L105-L133>

And finally it launches the `toydb::Server`:

最后启动 `toydb::Server`：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/bin/toydb.rs#L135-L137>

toyDB is now up and running!

至此，toyDB 已经启动并运行起来了！

---

<p align="center">
← <a href="sql-execution.md">SQL Execution</a> &nbsp; | &nbsp; <a href="client.md">Client</a> →
</p>
