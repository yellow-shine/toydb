# SQL Execution（SQL 执行）

Now that the planner and optimizer have done all the hard work of figuring out how to execute a
query, it's time to actually execute it.

在规划器和优化器完成了“如何执行查询”这一最艰巨的工作之后，接下来就是真正去执行它了。

## Plan Executor（计划执行器）

Plan execution is done by `sql::execution::Executor` in the
[`sql::execution`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/execution)
module, using a `sql::engine::Transaction` to access the SQL storage engine.

计划的执行由 [`sql::execution`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/execution)
模块中的 `sql::execution::Executor` 完成，它使用 `sql::engine::Transaction` 来访问 SQL 存储引擎。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/execution/executor.rs#L14-L49>

The executor takes a `sql::planner::Plan` as input, and will return an `ExecutionResult` depending
on the statement type.

执行器以 `sql::planner::Plan` 作为输入，并根据语句类型返回一个 `ExecutionResult`。

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L331-L339>

When executing the plan, the executor will branch off depending on the statement type:

在执行计划时，执行器会根据语句类型进行分支处理：

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L57-L101>

We'll focus on `SELECT` queries here, which are the most interesting.

我们这里重点关注最有意思的 `SELECT` 查询。

toyDB uses the iterator model (also known as the volcano model) for query execution. In the case of
a `SELECT` query, the result is a row iterator, and pulling from this iterator by calling `next()`
will drive the entire execution pipeline by recursively calling `next()` on the child nodes' row
iterators. This maps very naturally onto Rust's iterators, and we leverage these to construct the
execution pipeline as nested iterators.

toyDB 使用迭代器模型（也称为火山模型）来执行查询。对于 `SELECT` 查询，结果是一个行迭代器，通过调用 `next()` 从该迭代器拉取数据时，会递归地调用子节点行迭代器的 `next()`，从而驱动整个执行流水线。这种模型与 Rust 的迭代器非常自然地对应，我们利用它们把执行流水线构建成嵌套的迭代器。

Execution itself is fairly straightforward, since we're just doing exactly what the planner tells us
to do in the plan. We call `Executor::execute_node` recursively on each `sql::planner:Node`,
starting with the root node. Each node returns a result row iterator that the parent node can pull
its input rows from, process them, and output the resulting rows via its own row iterator (with the
root node's iterator being returned to the caller):

执行本身相当直接，因为我们只是完全按照规划器在计划中指示的那样去做。我们从根节点开始，对每个 `sql::planner:Node` 递归调用 `Executor::execute_node`。每个节点返回一个结果行迭代器，父节点可以从它那里拉取输入行、进行处理，并通过自己的行迭代器输出结果行（根节点的迭代器则返回给调用者）：

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L103-L104>

`Executor::execute_node()` will simply look at the type of `Node`, recursively call
`Executor::execute_node()` on any child nodes, and then process the rows accordingly.

`Executor::execute_node()` 只需查看 `Node` 的类型，对子节点递归调用 `Executor::execute_node()`，然后相应地处理这些行。

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L103-L212>

We won't discuss every plan node in detail, but let's consider the movie plan we've looked at
previously:

我们不会详细讨论每个计划节点，但来看一下之前看过的电影查询计划：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ HashJoin: inner on movies.genre_id = genres.id
         ├─ Scan: movies (released >= 2000)
         └─ Scan: genres
```

We'll recursively call `execute_node()` until we end up in the two `Scan` nodes. These simply
call through to the SQL engine (either using Raft or local disk) via `Transaction::scan()`, passing
in the scan predicate if any, and return the resulting row iterator:

我们将递归调用 `execute_node()`，直到到达两个 `Scan` 节点。它们只需通过 `Transaction::scan()` 调用 SQL 引擎（使用 Raft 或本地磁盘），传入扫描谓词（如果有的话），并返回得到的行迭代器：

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L203-L204>

`HashJoin` will then join the output rows from the `movies` and `genres` iterators by using a
hash join. This builds an in-memory table for `genres` and then iterates over `movies`, joining
the rows:

然后，`HashJoin` 会使用哈希连接（hash join）来连接 `movies` 和 `genres` 迭代器输出的行。它会先为 `genres` 构建一张内存中的哈希表，再遍历 `movies`，对行进行连接：

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L128-L141>

<https://github.com/erikgrinaker/toydb/blob/889aef9f24c0fa4d58e314877fa17559a9f3d5d2/src/sql/execution/join.rs#L103-L183>

The `Projection` node will simply evaluate the (trivial) column expressions using each joined
row as input:

`Projection` 节点只需以每个连接后的行作为输入，对（简单的）列表达式进行求值：

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L179-L186>

And finally the `Order` node will sort the results (which requires buffering them all in memory):

最后，`Order` 节点会对结果进行排序（这需要把它们全部缓冲在内存中）：

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L173-L177>

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/execution/executor.rs#L298-L328>

The output row iterator of `Order` is returned via `ExecutionResult::Select`, and the caller can now
go ahead and pull the resulting rows from it.

`Order` 的输出行迭代器通过 `ExecutionResult::Select` 返回，调用者随后就可以从中拉取结果行。

## Session Management（会话管理）

The entry point to the SQL engine is the `sql::execution::Session`, which represents a single user
session. It is obtained via `sql::engine::Engine::session()`.

SQL 引擎的入口是 `sql::execution::Session`，它表示一个用户会话。会话通过 `sql::engine::Engine::session()` 获得。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/execution/session.rs#L14-L21>

The session takes a series of raw SQL statement strings as input and parses them:

会话以一系列原始 SQL 语句字符串作为输入，并对它们进行解析：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/execution/session.rs#L29-L33>

For each statement, it returns a result depending on the kind of statement:

对于每条语句，它会根据语句的类型返回相应的结果：

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/execution/session.rs#L132-L148>

The session itself performs transaction control. It handles `BEGIN`, `COMMIT`, and `ROLLBACK`
statements, and modifies the transaction accordingly.

会话本身负责事务控制。它处理 `BEGIN`、`COMMIT` 和 `ROLLBACK` 语句，并相应地修改事务。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/execution/session.rs#L34-L70>

Any other statements are processed by the SQL planner, optimizer, and executor as we've seen in
previous sections.

其他语句则由 SQL 规划器、优化器和执行器处理，如前面几节所述。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/execution/session.rs#L77-L83>

These statements are always executed using the session's current transaction. If there is no active
transaction, the session will create a new, implicit transaction for each statement.

这些语句总是使用会话当前的事务来执行。如果没有活动事务，会话会为每条语句创建一个新的隐式事务。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/execution/session.rs#L87-L112>

And with that, we have a fully functional SQL engine!

至此，我们就拥有了一个功能完备的 SQL 引擎！

---

<p align="center">
← <a href="sql-optimizer.md">SQL Optimization</a> &nbsp; | &nbsp; <a href="server.md">Server</a> →
</p>
