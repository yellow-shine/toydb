# SQL Planning（SQL 计划）

The SQL planner in the [`sql::planner`](https://github.com/erikgrinaker/toydb/tree/c64012e29c5712d6fe028d3d5375a98b8faea266/src/sql/planner)
module takes a SQL statement AST from the parser and generates an execution plan for it. We won't
actually execute it just yet though, only figure out how to execute it.

[`sql::planner`](https://github.com/erikgrinaker/toydb/tree/c64012e29c5712d6fe028d3d5375a98b8faea266/src/sql/planner)
模块中的 SQL 计划器（planner）接收解析器输出的 SQL 语句 AST，并为其生成执行计划。不过我们此刻还不会真正执行它，只是弄清楚应该如何执行。

## Execution Plan（执行计划）

A plan is represented by the `sql::planner::Plan` enum. The variant specifies the operation to
execute (e.g. `SELECT`, `INSERT`, `UPDATE`, `DELETE`):

计划由 `sql::planner::Plan` 枚举表示。枚举的变体指定了要执行的操作（例如 `SELECT`、`INSERT`、`UPDATE`、`DELETE`）：

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/plan.rs#L15-L73>

Below the root, the plan is typically made of up of a tree of nested `sql::planner::Node`. Each node
emits a stream of SQL rows as output, and may take streams of input rows from child nodes.

在根节点之下，计划通常由嵌套的 `sql::planner::Node` 构成的树组成。每个节点输出一个 SQL 行流，并可能从子节点接收输入行流。

<https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/planner/plan.rs#L106-L175>

Here is an example, taken from the `Plan` code comment above:

下面是一个示例，取自上面 `Plan` 的代码注释：

```sql
SELECT title, released, genres.name AS genre
FROM movies INNER JOIN genres ON movies.genre_id = genres.id
WHERE released >= 2000
ORDER BY released
```

Which results in this query plan:

它生成的查询计划如下：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ Filter: movies.released >= 2000
         └─ NestedLoopJoin: inner on movies.genre_id = genres.id
            ├─ Scan: movies
            └─ Scan: genres
```

Rows flow from the tree leaves to the root:

行从树的叶节点流向根节点：

1. `Scan` nodes read rows from the tables `movies` and `genres`.

1. `Scan` 节点从 `movies` 和 `genres` 表中读取行。

1. `NestedLoopJoin` joins the rows from `movies` and `genres`.

1. `NestedLoopJoin` 对来自 `movies` 和 `genres` 的行进行连接（join）。

1. `Filter` discards rows with release dates older than 2000.

1. `Filter` 丢弃发布日期早于 2000 年的行。

1. `Projection` picks out the requested column values from the rows.

1. `Projection` 从行中挑出所需的列值。

1. `Order` sorts the rows by release date.

1. `Order` 按发布日期对行排序。

1. `Select` returns the final rows to the client.

1. `Select` 将最终的行返回给客户端。

## Scope and Name Resolution（作用域与名称解析）

One of the main jobs of the planner is to resolve column names to column indexes in the input rows
of each node.

计划器的主要工作之一，是把列名解析为每个节点输入行中的列索引。

In the query example above, the `WHERE released >= 2000` filter may refer to a column `released`
from either the joined `movies` table or the `genres` tables. The planner needs to figure out which
table has a `released` column, and also figure out which column number in the `NestedLoopJoin`
output rows corresponds to the `released` column (for example column number 2).

在上面的查询示例中，`WHERE released >= 2000` 过滤条件里的 `released` 列，可能来自参与连接的 `movies` 表，也可能来自 `genres` 表。计划器需要弄清楚哪张表拥有 `released` 列，还要弄清楚 `NestedLoopJoin` 输出行中的第几列对应 `released` 列（比如第 2 列）。

This job is further complicated by the fact that many nodes can alias, reorder, or drop columns,
and some nodes may also refer to columns that shouldn't be part of the result at all (for example,
it's possible to `ORDER BY` a column that won't be output by a `SELECT` projection at all, but
the `Order` node still needs access to the column data to sort by it).

这项工作还因为另一个事实而更加复杂：许多节点会对列进行别名、重排或丢弃，而且某些节点还会引用根本不应出现在结果中的列（例如，`ORDER BY` 可以使用一个完全不会被 `SELECT` 投影输出的列，但 `Order` 节点仍然需要访问该列的数据才能完成排序）。

The planner uses a `sql::planner::Scope` to keep track of which column names are currently visible,
and which column indexes they refer to. For each node the planner builds, starting from the leaves,
it creates a new `Scope` that contains the currently visible columns, tracking how they are modified
and rearranged by each node.

计划器使用 `sql::planner::Scope` 来跟踪当前哪些列名可见，以及它们对应的列索引。计划器从叶节点开始构建每个节点时，都会创建一个新的 `Scope`，其中包含当前可见的列，并跟踪每个节点如何修改和重排这些列。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L577-L610>

When an AST expression refers to a column name, the planner can use `Scope::lookup_column()` to find
out which column number the expression should take its input value from.

当 AST 表达式引用一个列名时，计划器可以使用 `Scope::lookup_column()` 查出该表达式应当从第几列取得输入值。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L660-L686>

## Planner（计划器）

The planner itself is `sql:planner::Planner`. It uses a `sql::engine::Catalog` to look up
information about tables and columns from storage.

计划器本身是 `sql:planner::Planner`。它使用 `sql::engine::Catalog` 从存储中查询表和列的信息。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L12-L20>

To build an execution plan, the planner first looks at the `ast::Statement` kind to determine
what kind of plan to build:

为了构建执行计划，计划器首先查看 `ast::Statement` 的种类，以决定构建哪种计划：

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L28-L47>

Let's build this `SELECT` plan from above:

我们来构建前面这个 `SELECT` 计划：

```sql
SELECT title, released, genres.name AS genre
FROM movies INNER JOIN genres ON movies.genre_id = genres.id
WHERE released >= 2000
ORDER BY released
```

Which should result in this plan:

它应当生成如下计划：

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ Filter: movies.released >= 2000
         └─ NestedLoopJoin: inner on movies.genre_id = genres.id
            ├─ Scan: movies
            └─ Scan: genres
```

The planner is given the following (simplified) AST from the parser as input:

计划器从解析器那里得到如下（经过简化的）AST 作为输入：

```rust
// A SELECT statement.
Statement::Select {
    // SELECT title, released, genres.name AS genre
    select: [
        (Column("title"), None),
        (Column("released"), None),
        (Column("genres.name"), "genre"),
    ]

    // FROM movies INNER JOIN genres ON movies.genre_id = genres.id
    from: [
        Join {
            left: Table("movies"),
            right: Table("genres"),
            type: Inner,
            predicate: Some(
                Equal(
                    Column("movies.genre_id"),
                    Column("genres.id"),
                )
            )
        }
    ]

    // WHERE released >= 2000
    where: Some(
        GreaterThanOrEqual(
            Column("released"),
            Integer(2000),
        )
    )

    // ORDER BY released
    order: [
        (Column("released"), Ascending),
    ]
}
```

The first thing `Planner::build_select` does is to create an empty scope (which will track column
names and indexes) and build the `FROM` clause which will generate the initial input rows:

`Planner::build_select` 做的第一件事是创建一个空的作用域（用于跟踪列名和列索引），并构建 `FROM` 子句，由它生成最初的输入行：

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L170-L179>

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L283-L289>

`Planner::build_from()` first encounters the `ast::From::Join` item, which joins `movies` and
`genres`. This will build a `Node::NestedLoopJoin` plan node for the join, which is the simplest and
most straightforward join algorithm -- it simply iterates over all rows in the `genres` table for
every row in the `movies` table and emits the joined rows (we'll see how to optimize it with a
better join algorithm later).

`Planner::build_from()` 首先遇到 `ast::From::Join` 项，它对 `movies` 和 `genres` 进行连接。这会为该连接构建一个 `Node::NestedLoopJoin` 计划节点，这是最简单直观的连接算法——对 `movies` 表中的每一行，遍历 `genres` 表中的所有行并输出连接后的行（稍后我们会看到如何用更好的连接算法来优化它）。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L319-L344>

It first recurses into `Planner::build_from()` to build each of the `ast::From::Table` nodes for
each table.  This will look up the table schemas in the catalog, add them to the current scope, and
build a `Node::Scan` node which will emit all rows from each table. The `Node::Scan` nodes are
placed into the `Node::NestedLoopJoin` above.

它首先递归调用 `Planner::build_from()` 为每张表构建 `ast::From::Table` 节点。这会在目录（catalog）中查找表结构（schema），把它们加入当前作用域，并构建一个 `Node::Scan` 节点，由它输出每张表的所有行。这些 `Node::Scan` 节点被放入上面的 `Node::NestedLoopJoin` 之中。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L312-L317>

While building the `Node::NestedLoopJoin`, it also needs to convert the join expression
`movies.genre_id = genres.id` into a proper `sql::types::Expression`. This is done by
`Planner::build_expression()`:

在构建 `Node::NestedLoopJoin` 时，它还需要把连接表达式 `movies.genre_id = genres.id` 转换成真正的 `sql::types::Expression`。这项工作由 `Planner::build_expression()` 完成：

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L493-L568>

Expression building is mostly a direct translation from an `ast::Expression` variant to a
corresponding `sql::types::Expression` variant (for example from
`ast::Expression::Operator(ast::Operator::Equal)` to `sql::types::Expression::Equal`). However, as
mentioned earlier, `ast::Expression` contains column references by name, while
`sql::types::Expression` contains column references as row indexes. This name resolution is done
here, by looking up the column names in the scope:

表达式构建大体上是把 `ast::Expression` 变体直接翻译成对应的 `sql::types::Expression` 变体（例如把 `ast::Expression::Operator(ast::Operator::Equal)` 翻译成 `sql::types::Expression::Equal`）。不过，正如前面提到的，`ast::Expression` 中的列引用是按名称的，而 `sql::types::Expression` 中的列引用是按行索引的。名称解析就在这里完成，即在作用域中查找列名：

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L521-L523>

The expression we're building is the join predicate of `Node::NestedLoopJoin`, so it operates on
joined rows containing all columns of `movies` then all columns of `genres`. It also operates on all
combinations of joined rows (the [Cartesian product](https://en.wikipedia.org/wiki/Cartesian_product)),
and the purpose of the join predicate is to determine which joined rows to actually keep. For
example, the full set of joined rows that are evaluated might be:

我们正在构建的表达式是 `Node::NestedLoopJoin` 的连接谓词，因此它作用于连接后的行，这些行先包含 `movies` 的全部列，再包含 `genres` 的全部列。它还作用于连接行的所有组合（即[笛卡尔积](https://en.wikipedia.org/wiki/Cartesian_product)），连接谓词的作用就是确定哪些连接行应当真正保留。例如，被求值的全部连接行可能是：

| movies.id | movies.title | movies.released | movies.genre_id | genres.id | genres.name |
|-----------|--------------|-----------------|-----------------|-----------|-------------|
| 1         | Sicario      | 2015            | 2               | 1         | Drama       |
| 2         | Sicario      | 2015            | 2               | 2         | Action      |
| 3         | 21 Grams     | 2003            | 1               | 1         | Drama       |
| 4         | 21 Grams     | 2003            | 1               | 2         | Action      |
| 5         | Heat         | 1995            | 2               | 1         | Drama       |
| 6         | Heat         | 1995            | 2               | 2         | Action      |

（上表展示了 `movies` 与 `genres` 的笛卡尔积：每一行都是一部电影与一个类别的组合，连接谓词的作用是从中挑出 `movies.genre_id = genres.id` 的行。）

The join predicate should pick out the rows where `movies.genre_id = genres.id`. The scope will
reflect the column layout in the example above, and can resolve the column names to zero-based row
indexes as `#3 = #4`, which will be the final built `Expression`.

连接谓词应当挑出 `movies.genre_id = genres.id` 的行。作用域会反映上例中的列布局，并把列名解析为零基（从 0 开始）的行索引，得到 `#3 = #4`，这就是最终构建出的 `Expression`。

Now that we've built the `FROM` clause into a `Node::NestedLoopJoin` of two `Node::Scan` nodes, we
move on to the `WHERE` clause. This simply builds the `WHERE` expression `released >= 2000`, like
we've already seen with the join predicate, and creates a `Node::Filter` node which takes its input
rows from the `Node::NestedLoopJoin` and filters them by the given expression. Again, the scope
keeps track of which input columns we're getting from the join node and resolves the `released`
column reference in the expression.

现在我们已经把 `FROM` 子句构建成了由两个 `Node::Scan` 节点组成的 `Node::NestedLoopJoin`，接下来处理 `WHERE` 子句。这一步只是构建 `WHERE` 表达式 `released >= 2000`（与前面构建连接谓词类似），并创建一个 `Node::Filter` 节点，它从 `Node::NestedLoopJoin` 获取输入行，并按给定表达式进行过滤。同样，作用域会跟踪我们从连接节点获得的输入列，并解析表达式中的 `released` 列引用。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L202-L206>

We then build the `SELECT` clause, which emits the `title, released, genres.name AS genre` columns.
This is just a list of expressions that are built in the current scope and placed into a
`Node::Projection` (the expressions could be arbitrarily complex). However, we also have to make
sure to update the scope with the final three columns that are output to subsequent nodes, taking
into account the `genre` alias for the original `genres.name` column (we won't dwell on the "hidden
columns" mentioned there -- they're not relevant for our query).

然后我们构建 `SELECT` 子句，它输出 `title, released, genres.name AS genre` 这些列。这只是一个在当前作用域中构建的表达式列表，被放入 `Node::Projection`（这些表达式可以任意复杂）。不过，我们还必须确保用最终输出给后续节点的那三列来更新作用域，同时考虑原始 `genres.name` 列的 `genre` 别名（代码注释中提到的"隐藏列"（hidden columns）我们不作展开——它们与本查询无关）。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L214-L234>

Finally, we build the `ORDER BY` clause. Again, this just builds a trivial expression for `released`
and places it into an `Node::Order` node which takes input rows from the `Node::Projection` and
sorts them by the order expression.

最后，我们构建 `ORDER BY` 子句。同样，这只是为 `released` 构建一个简单的表达式，并放入 `Node::Order` 节点，该节点从 `Node::Projection` 获取输入行，并按排序表达式对它们排序。

<https://github.com/erikgrinaker/toydb/blob/6f6cec4db10bc015a37ee47ff6c7dae383147dd5/src/sql/planner/planner.rs#L245-L252>

And that's it. The `Node::Order` is placed into the root `Plan::Select`, and we have our final plan.

到此为止。`Node::Order` 被放入根节点 `Plan::Select`，我们就得到了最终的计划。

```
Select
└─ Order: movies.released desc
   └─ Projection: movies.title, movies.released, genres.name as genre
      └─ Filter: movies.released >= 2000
         └─ NestedLoopJoin: inner on movies.genre_id = genres.id
            ├─ Scan: movies
            └─ Scan: genres
```

We'll see how to execute it soon, but first we should optimize it to see if we can make it run
faster -- in particular, to see if we can avoid reading all movies from storage, and if we can do
better than the very slow nested loop join.

我们很快会看到如何执行它，但首先应当对它进行优化，看看能否让它跑得更快——特别是，看看能否避免从存储中读取所有电影，以及能否比非常慢的嵌套循环连接做得更好。

---

<p align="center">
← <a href="sql-parser.md">SQL Parsing</a> &nbsp; | &nbsp; <a href="sql-optimizer.md">SQL Optimization</a> →
</p>
