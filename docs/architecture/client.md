# Client（客户端）

The toyDB client is in the [`client`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/client.rs)
module. It uses the same Bincode-based protocol that we saw in the server section, sending
`toydb::Request` and receiving `toydb::Response`.

toyDB 客户端位于 [`client`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/client.rs)
模块。它使用与服务器一节中相同的基于 Bincode 的协议，发送 `toydb::Request` 并接收 `toydb::Response`。

## Client Library（客户端库）

The main client library `toydb::Client` is used to communicate with a toyDB server:

主客户端库 `toydb::Client` 用于与 toyDB 服务器通信：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/client.rs#L15-L24>

When initialized, it connects to a toyDB server over TCP, which establishes a SQL session for it:

初始化时，它会通过 TCP 连接到一台 toyDB 服务器，服务器会为它建立一个 SQL 会话：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/client.rs#L27-L33>

It can then send Bincode-encoded `toydb::Request` to the server, and receive `toydb::Response`
back.

随后它就可以向服务器发送 Bincode 编码的 `toydb::Request`，并接收返回的 `toydb::Response`。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/client.rs#L35-L40>

In particular, `Client::execute` can be used to execute arbitrary SQL statements in the client's
current session:

特别是，`Client::execute` 可用于在客户端当前会话中执行任意 SQL 语句：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/client.rs#L42-L56>

## `toysql` Binary（`toysql` 可执行程序）

However, `toydb::Client` is a programmatic API, and we want a more convenient user interface.
The `toysql` client in [`src/bin/toysql.rs`](https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs)
provides a typical [REPL](https://en.wikipedia.org/wiki/Read–eval–print_loop) (read-evaluate-print loop) where users can enter SQL statements and view the results.

不过，`toydb::Client` 是一个编程 API，我们还需要一个更方便的用户界面。[`src/bin/toysql.rs`](https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs)
中的 `toysql` 客户端提供了一个典型的 [REPL](https://en.wikipedia.org/wiki/Read–eval–print_loop)（读取-求值-打印循环），用户可以在其中输入 SQL 语句并查看结果。

Like `toydb`, `toysql` is a tiny [`clap`](https://docs.rs/clap/latest/clap/) command that takes a
toyDB server address to connect to and starts an interactive shell:

与 `toydb` 一样，`toysql` 也是一个很小的 [`clap`](https://docs.rs/clap/latest/clap/) 命令，它接收要连接的 toyDB 服务器地址，并启动一个交互式 shell：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs#L29-L53>

It first attempts to connect to the toyDB server using the `toydb::Client` client, and then starts
an interactive shell using the [Rustyline](https://docs.rs/rustyline/latest/rustyline/) library.

它首先尝试使用 `toydb::Client` 客户端连接 toyDB 服务器，然后使用 [Rustyline](https://docs.rs/rustyline/latest/rustyline/) 库启动交互式 shell：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs#L55-L81>

The shell is simply a loop that prompts the user to input a SQL statement:

这个 shell 就是一个循环，提示用户输入 SQL 语句：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs#L216-L250>

Each statement is the executed against the server via `toydb::Client::execute`, and the response
is formatted and printed as output:

每条语句通过 `toydb::Client::execute` 在服务器上执行，响应会被格式化并打印为输出：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs#L83-L92>

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/bin/toysql.rs#L175-L204>

And with that, we have a fully functional SQL database system and can run queries to our heart's
content. Have fun!

至此，我们就拥有了一个功能完备的 SQL 数据库系统，可以随心所欲地运行查询了。玩得开心！

---

<p align="center">
← <a href="server.md">Server</a>
</p>
