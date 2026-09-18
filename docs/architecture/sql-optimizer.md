# SQL Optimization（SQL 优化）

[Query optimization](https://en.wikipedia.org/wiki/Query_optimization) attempts to improve query
performance and efficiency by altering the execution plan. This is a deep and complex field, and
we can only scratch the surface here.

[查询优化](https://en.wikipedia.org/wiki/Query_optimization)试图通过修改执行计划来提升查询的性能与效率。这是一个深入而复杂的领域，我们在这里只能浅尝辄止。

toyDB's query optimizer is very basic -- it only has a handful of rudimentary heuristic
optimizations to illustrate how the process works. Real-world optimizers use much more sophisticated
methods, including statistical analysis, cost estimation, adaptive execution, etc.

toyDB 的查询优化器非常简单——它只有少量基础性的启发式优化，用来展示这个过程是如何运作的。真实世界的优化器会使用复杂得多的方法，包括统计分析、代价估算、自适应执行等等。

The optimizers are located in the [`sql::planner::optimizer`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs) module.
An optimizer `sql::planner::Optimizer` just takes in a plan node `sql::planner::Node` (the root node
in the plan), and returns an optimized node:

各个优化器位于 [`sql::planner::optimizer`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs) 模块。优化器 `sql::planner::Optimizer` 只接收一个计划节点 `sql::planner::Node`（计划中的根节点），并返回一个优化后的节点：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L20-L25>

Optimizations are always implemented as recursive node transformations. To help with this, `Node`
has the helper methods `Node::transform` and `Node::transform_expressions` which recurse into a node
or expression tree and call a given transformation closure on each node, as either
[pre-order](https://en.wikipedia.org/wiki/Tree_traversal#Pre-order,_NLR) or
[post-order](https://en.wikipedia.org/wiki/Tree_traversal#Post-order,_LRN) transforms:

所有优化都以递归节点变换的方式实现。为此，`Node` 提供了辅助方法 `Node::transform` 和 `Node::transform_expressions`，它们会递归遍历节点树或表达式树，并对每个节点调用给定的变换闭包，可以按[前序](https://en.wikipedia.org/wiki/Tree_traversal#Pre-order,_NLR)（pre-order）或[后序](https://en.wikipedia.org/wiki/Tree_traversal#Post-order,_LRN)（post-order）进行变换：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/plan.rs#L269-L371>

A technique that's often useful during optimization is to convert expressions into
[conjunctive normal form](https://en.wikipedia.org/wiki/Conjunctive_normal_form), i.e. "an AND of
ORs". For example, the two following expressions are equivalent, but the latter is in conjunctive
normal form (it's a chain of ANDs):

优化过程中经常用到的一种技术是把表达式转换成[合取范式](https://en.wikipedia.org/wiki/Conjunctive_normal_form)（conjunctive normal form），即"若干 OR 的 AND"。例如，下面两个表达式是等价的，但后者是合取范式（它是一条 AND 链）：

```
(a AND b) OR (c AND d)  →  (a OR c) AND (a OR d) AND (b OR c) AND (b OR d)
```

This is useful because we can often move each AND operand independently around in the plan tree
and still get the same result -- we'll see this in action later. Expressions are converted into
conjunctive normal form via `Expression::into_cnf`, which is implemented using
[De Morgan's laws](https://en.wikipedia.org/wiki/De_Morgan%27s_laws):

这很有用，因为我们通常可以在计划树中独立地移动每个 AND 操作数，而结果保持不变——稍后我们会看到它的实际应用。表达式通过 `Expression::into_cnf` 转换为合取范式，其实现使用了[德摩根定律](https://en.wikipedia.org/wiki/De_Morgan%27s_laws)：

<https://github.com/erikgrinaker/toydb/blob/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/types/expression.rs#L289-L351>

We'll have a brief look at all of toyDB's optimizers, which are listed here in the order they're
applied:

我们将简要介绍 toyDB 的所有优化器，下面按它们被应用的顺序列出：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L9-L18>

Test scripts for the optimizers are in [`src/sql/testscripts/optimizers`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/testscripts/optimizers),
and show how query plans evolve as each optimizer is applied.

优化器的测试脚本位于 [`src/sql/testscripts/optimizers`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/testscripts/optimizers)，展示了每应用一个优化器后查询计划是如何演变的。

## Constant Folding（常量折叠）

The `ConstantFolding` optimizer performs [constant folding](https://en.wikipedia.org/wiki/Constant_folding).
This pre-evaluates constant expressions in the plan during planning, instead of evaluating them
for every row during execution.

`ConstantFolding` 优化器执行[常量折叠](https://en.wikipedia.org/wiki/Constant_folding)（constant folding）。它在规划阶段预先求值计划中的常量表达式，而不是在执行阶段对每一行都求值一次。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L27-L30>

For example, consider the query `SELECT 1 + 2 * 3 - foo FROM bar`. There is no point in
re-evaluating `1 + 2 * 3` for every row in `bar`, because the result is always the same, so we can
just evaluate this once during planning, transforming the expression into `7 - foo`.

例如，考虑查询 `SELECT 1 + 2 * 3 - foo FROM bar`。对 `bar` 中的每一行都重新求值 `1 + 2 * 3` 毫无意义，因为结果总是相同的，所以我们只需在规划阶段求值一次，把表达式变换成 `7 - foo` 即可。

Concretely, this plan:

具体来说，这个计划：

```
Select
└─ Projection: 1 + 2 * 3 - bar.foo
   └─ Scan: bar
```

Should be transformed into this plan:

应当被变换成这个计划：

```
Select
└─ Projection: 7 - bar.foo
   └─ Scan: bar
```

To do this, `ConstantFolding` simply checks whether an `Expression` tree contains an
`Expression::Column` node -- if it doesn't, then it much be a constant expression (since that's the
only dynamic value in an expression), and we can evaluate it with a `None` input row and replace the
original expression node with an `Expression::Constant` node.

为此，`ConstantFolding` 只是检查 `Expression` 树中是否包含 `Expression::Column` 节点——如果不包含，那它必然是常量表达式（因为列引用是表达式中唯一的动态值），于是我们可以用 `None` 输入行对它求值，并把原来的表达式节点替换为 `Expression::Constant` 节点。

This is done recursively for each plan node, and recursively for each expression node (so it does
this both for `SELECT`, `WHERE`, `ORDER BY`, and all other parts of the query). Notably, it does a
post-order expression transform, so it starts at the expression leaf nodes and attempts to transform
each expression node as it moves back up the tree -- this allows it to iteratively evaluate constant
parts as far as possible for each branch.

这一过程对每个计划节点递归执行，也对每个表达式节点递归执行（因此它对 `SELECT`、`WHERE`、`ORDER BY` 以及查询的所有其他部分都会这样做）。值得注意的是，它采用后序表达式变换：从表达式叶子节点开始，在回到树顶的过程中尝试变换每个表达式节点——这使它能够在每个分支上迭代式地尽可能多地对常量部分求值。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L32-L56>

Additionally, `ConstantFolding` also short-circuits logical expressions. For example, the expression
`foo AND FALSE` will always be `FALSE`, regardless of what `foo` is, so we can replace it with
`FALSE`:

此外，`ConstantFolding` 还会对逻辑表达式做短路处理。例如，表达式 `foo AND FALSE` 无论 `foo` 是什么都恒为 `FALSE`，所以我们可以直接把它替换为 `FALSE`：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L58-L84>

As the code comment mentions though, this doesn't fold optimally: it doesn't attempt to rearrange
expressions, which would require knowledge of precedence rules. For example, `(1 + foo) - 2` could
be folded into `foo - 1` by first rearranging it as `foo + (1 - 2)`, but we don't do this currently.

不过正如代码注释所说，这种折叠并不是最优的：它不会尝试重排表达式，而重排需要了解运算优先级规则。例如，`(1 + foo) - 2` 可以先重排为 `foo + (1 - 2)` 再折叠成 `foo - 1`，但我们目前不做这一步。

## Filter Pushdown（谓词下推）

The `FilterPushdown` optimizer attempts to push filter predicates as far down into the plan as
possible, to reduce the number of rows each node has to process.

`FilterPushdown` 优化器尝试把过滤谓词尽可能向计划树的下方推（下推），以减少每个节点需要处理的行数。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L90-L95>

Recall the `movies` query plan from the planning section:

回顾规划章节中的 `movies` 查询计划：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ Filter: movies.released >= 2000
         └─ NestedLoopJoin: inner on movies.genre_id = genres.id
            ├─ Scan: movies
            └─ Scan: genres
```

Even though we're filtering on `release >= 2000`, the `Scan` node still has to read all of them from
disk and send them via Raft, and the `NestedLoopJoin` node still has to join all of them. It would
be nice if we could push this filtering into the `NestedLoopJoin` and `Scan` nodes and avoid this
extra work, and this is exactly what `FilterPushdown` does.

即使我们在按 `release >= 2000` 过滤，`Scan` 节点仍然必须从磁盘读取全部行并通过 Raft 发送，`NestedLoopJoin` 节点也仍然要把所有行都连接一遍。如果我们能把该过滤下推到 `NestedLoopJoin` 和 `Scan` 节点中，从而避免这些额外工作，那就太好了——这正是 `FilterPushdown` 所做的事。

The only plan nodes that have predicates that can be pushed down are `Filter` nodes and
`NestedLoopJoin` nodes, so we recurse through the plan tree and look for these nodes, attempting
to push down.

唯一拥有可下推谓词的计划节点是 `Filter` 节点和 `NestedLoopJoin` 节点，因此我们递归遍历计划树寻找这些节点，并尝试下推。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L97-L110>

When it encounters the `Filter` node, it will extract the predicate and attempt to push it down
into its `source` node:

当遇到 `Filter` 节点时，它会提取出谓词，并尝试把谓词下推到它的 `source` 节点中：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L139-L153>

If the source node is a `Filter`, `NestedLoopJoin`, or `Scan` node, then we can push the predicate
down into it by `AND`ing it with the existing predicate (if any).

如果 source 节点是 `Filter`、`NestedLoopJoin` 或 `Scan` 节点，那么我们就可以把该谓词与已有谓词（如果有）做 `AND` 后下推进去。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L112-L137>

In our case, we were able to push the `Filter` into the `NestedLoopJoin`, and our plan now looks
like this:

在我们的例子里，我们把 `Filter` 下推进了 `NestedLoopJoin`，现在的计划看起来是这样：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ NestedLoopJoin: inner on movies.genre_id = genres.id AND movies.released >= 2000
         ├─ Scan: movies
         └─ Scan: genres
```

But we're still not done, as we'd like to push `movies.released >= 2000` down into the `Scan` node.
Pushdown for join nodes is a little more tricky, because we can only push down parts of the
expression that reference one of the source nodes.

但还没完，我们还想把 `movies.released >= 2000` 进一步下推到 `Scan` 节点。对连接节点的下推要更麻烦一些，因为我们只能下推表达式中仅引用了某一个 source 节点的那部分。

We first have to convert the expression into conjunctive normal form, i.e. and AND of ORs, as we've
discussed previously. This allows us to examine and push down each AND part in isolation, because it
has the same effect regardless of whether it is evaluated in the `NestedLoopJoin` node or one of
the source nodes. Our expression is already in conjunctive normal form, though.

我们首先要把表达式转换成合取范式（即若干 OR 的 AND），就像前面讨论的那样。这样我们就可以孤立地检查并下推每一个 AND 部分，因为无论它在 `NestedLoopJoin` 节点还是某个 source 节点中求值，效果都相同。不过我们的表达式已经是合取范式了。

We then look at each AND part, and check which side of the join it has column references for.  If it
only references one of the sides, then the expression can be pushed down into it. We also make some
effort here to move primary/foreign key constants across to both sides, but we'll gloss over that.

然后我们查看每个 AND 部分引用了连接哪一侧的列。如果它只引用了其中一侧，那么该表达式就可以下推到那一侧。这里我们还会做一些努力，把主键/外键常量移动到两侧，但这一细节我们略过不谈。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L155-L247>

This allows us to push down the `movies.released >= 2000` predicate into the corresponding `Scan`
node, significantly reducing the amount of data transferred across Raft:

这使我们能把 `movies.released >= 2000` 谓词下推到相应的 `Scan` 节点，从而显著减少通过 Raft 传输的数据量：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ NestedLoopJoin: inner on movies.genre_id = genres.id
         ├─ Scan: movies (released >= 2000)
         └─ Scan: genres
```

## Index Lookups（索引查找）

The `IndexLookup` optimizer uses primary key or secondary index lookups instead of full table
scans where possible.

`IndexLookup` 优化器在可能的情况下使用主键或二级索引查找来替代全表扫描。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L250-L252>

The optimizer itself is fairly straightforward. It assumes that `FilterPushdown` has already pushed
predicates down into `Scan` nodes, so it only needs to examine these. It converts the predicate into
conjunctive normal form, and looks for any parts that are direct column lookups -- i.e.
`column = value` (possibly a long OR chain of these).

这个优化器本身相当直接。它假定 `FilterPushdown` 已经把谓词下推到了 `Scan` 节点中，因此它只需检查这些节点。它把谓词转换成合取范式，并寻找其中直接进行列查找的部分——即 `column = value`（也可能是这些比较构成的一条很长的 OR 链）。

If it finds any, and the column is either a primary key or secondary index column, then we convert
the `Scan` node into either a `KeyLookup` or `IndexLookup` node respectively. If there are any
further AND predicates remaining, we add a parent `Filter` node to keep these predicates.

如果找到了这样的部分，并且该列是主键列或二级索引列，我们就把 `Scan` 节点相应地转换成 `KeyLookup` 或 `IndexLookup` 节点。如果还有剩余的其他 AND 谓词，我们就加一个父节点 `Filter` 来保留这些谓词。

For example, the following plan:

例如，下面这个计划：

```
Select
└─ Scan: movies ((id = 1 OR id = 7 OR id = 3) AND released >= 2000)
```

Will be transformed into one that does individual key lookups rather than a full table scan:

会被变换成一个执行逐个键查找而非全表扫描的计划：

```
Select
└─ Filter: movies.released >= 2000
   └─ KeyLookup: movies (1, 3, 7)
```

The code is as outlined above:

代码正如上面所述：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L254-L303>

Helped by `Expression::is_column_lookup()` and `Expression::into_column_values()`:

并借助 `Expression::is_column_lookup()` 和 `Expression::into_column_values()`：

<https://github.com/erikgrinaker/toydb/blob/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/types/expression.rs#L363-L421>

## Hash Join（哈希连接）

The `HashJoin` optimizer will replace a `NestedLoopJoin` with a `HashJoin` where possible.

`HashJoin` 优化器会在可能的情况下用 `HashJoin` 替换 `NestedLoopJoin`。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L305-L307>

A [nested loop join](https://en.wikipedia.org/wiki/Nested_loop_join) is a very inefficient O(n²)
algorithm, which iterates over all rows in the right source for each row in the left source to see
if they match. However, it is completely general, and can join on arbitraily complex predicates.

[嵌套循环连接](https://en.wikipedia.org/wiki/Nested_loop_join)（nested loop join）是一种效率极低的 O(n²) 算法，它会对左 source 中的每一行都遍历右 source 中的所有行，看它们是否匹配。不过，它完全通用，可以在任意复杂的谓词上进行连接。

In the common case where the join predicate is an equality comparison such as
`movies.genre_id = genres.id` (i.e. an [equijoin](https://en.wikipedia.org/wiki/Relational_algebra#θ-join_and_equijoin)),
then we can instead use a [hash join](https://en.wikipedia.org/wiki/Hash_join). This scans the right
table once, builds an in-memory hash table from it, and for each left row it looks up any right rows
in the hash table. This is a much more efficient O(n) algorithm.

当连接谓词是相等比较时（例如 `movies.genre_id = genres.id`，即[等值连接](https://en.wikipedia.org/wiki/Relational_algebra#θ-join_and_equijoin)（equijoin））这种常见情形，我们可以改用[哈希连接](https://en.wikipedia.org/wiki/Hash_join)（hash join）。它只扫描一次右表，据此在内存中构建一张哈希表，然后对左表的每一行在哈希表中查找匹配的右表行。这是一种效率高得多的 O(n) 算法。

In our previous movie example, we are in fact doing an equijoin:

在前面的电影示例中，我们实际上做的正是等值连接：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ NestedLoopJoin: inner on movies.genre_id = genres.id
         ├─ Scan: movies (released >= 2000)
         └─ Scan: genres
```

And so our `NestedLoopJoin` can be replaced by a `HashJoin`:

因此我们的 `NestedLoopJoin` 可以被替换为 `HashJoin`：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ HashJoin: inner on movies.genre_id = genres.id
         ├─ Scan: movies (released >= 2000)
         └─ Scan: genres
```

The `HashJoin` optimizer is extremely simple: if the join predicate is an equijoin, use a hash join.
This isn't always a good idea (the right source can be huge and we can run out of memory for the
hash table), but we keep it simple.

`HashJoin` 优化器极其简单：如果连接谓词是等值连接，就使用哈希连接。这并不总是个好主意（右 source 可能非常庞大，哈希表可能耗尽内存），但我们从简处理。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L309-L348>

Of course there are many other join algorithms out there, and one of the harder problems in SQL
optimization is how to efficiently perform large N-way multijoins. We don't attempt to tackle these
problems here -- the `HashJoin` optimizer is just a very simple example of such join optimization.

当然，还有许多其他的连接算法，而 SQL 优化中较难的问题之一是如何高效地执行大规模 N 路多表连接。我们不在这里尝试解决这些问题——`HashJoin` 优化器只是这类连接优化的一个非常简单的示例。

## Short Circuiting（短路化简）

The `ShortCircuit` optimizer tries to find nodes that can't possibly do any useful work, and either
removes them from the plan, or replaces them with trivial nodes that don't do anything. It is kind
of similar to the `ConstantFolding` optimizer in spirit, but works on plan nodes rather than
expression nodes.

`ShortCircuit` 优化器尝试找出不可能做任何有用工作的节点，要么把它们从计划中移除，要么用什么都不做的平凡节点替换它们。它在精神上与 `ConstantFolding` 优化器有些类似，但作用于计划节点而不是表达式节点。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L350-L354>

For example, `Filter` nodes with a `TRUE` predicate won't actually filter anything:

例如，谓词为 `TRUE` 的 `Filter` 节点实际上不会过滤任何东西：

```
Select
└─ Filter: true
   └─ Scan: movies
```

So we can just remove them:

所以我们可以直接把它移除：

```
Select
└─ Scan: movies
```

Similarly, `Filter` nodes with a `FALSE` predicate will never emit anything:

类似地，谓词为 `FALSE` 的 `Filter` 节点永远不会输出任何行：

```
Select
└─ Filter: false
   └─ Scan: movies
```

There's no point doing a scan in this case, so we can just replace it with a `Nothing` node that
does no work and doesn't emit anything:

这种情况下做扫描毫无意义，所以我们可以直接用一个不做任何工作、也不输出任何行的 `Nothing` 节点来替换它：

```
Select
└─ Nothing
```

The optimizer tries to find a bunch of such patterns. This can also tidy up query plans a fair bit
by removing unnecessary cruft.

该优化器会尝试寻找一大类这样的模式。这也能通过清除不必要的冗余节点，把查询计划整理得更干净。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/optimizer.rs#L356-L438>

---

<p align="center">
← <a href="sql-planner.md">SQL Planning</a> &nbsp; | &nbsp; <a href="sql-execution.md">SQL Execution</a> →
</p>
