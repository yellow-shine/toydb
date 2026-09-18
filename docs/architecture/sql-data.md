# SQL Data Model（SQL 数据模型）

The SQL data model represents user data in tables and rows. It is made up of data types and schemas,
in the [`sql::types`](https://github.com/erikgrinaker/toydb/tree/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/types)
module.

SQL 数据模型用表和行来表示用户数据。它由数据类型和模式（schema）组成，位于 [`sql::types`](https://github.com/erikgrinaker/toydb/tree/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/types) 模块中。

## Data Types（数据类型）

toyDB supports four basic scalar data types as `sql::types::DataType`: booleans, integers, floats,
and strings.

toyDB 通过 `sql::types::DataType` 支持四种基本的标量数据类型：布尔值、整数、浮点数和字符串。

<https://github.com/erikgrinaker/toydb/blob/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql/types/value.rs#L15-L27>

Specific values are represented as `sql::types::Value`, using the corresponding Rust types. toyDB
also supports SQL `NULL` values, i.e. unknown values, following the rules of
[three-valued logic](https://en.wikipedia.org/wiki/Three-valued_logic).

具体的值用 `sql::types::Value` 表示，使用对应的 Rust 类型。toyDB 还支持 SQL 的 `NULL` 值，即未知值，遵循[三值逻辑](https://en.wikipedia.org/wiki/Three-valued_logic)的规则。

<https://github.com/erikgrinaker/toydb/blob/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql/types/value.rs#L40-L64>

The `Value` type provides basic formatting, conversion, and mathematical operations.

`Value` 类型提供了基本的格式化、转换和数学运算。

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/types/value.rs#L68-L79>

<https://github.com/erikgrinaker/toydb/blob/686d3971a253bfc9facc2ba1b0e716cff5c109fb/src/sql/types/value.rs#L164-L370>

It also specifies comparison and ordering semantics, but these are subtly different from the SQL
semantics. For example, in Rust code `Value::Null == Value::Null` yields `true`, while in SQL
`NULL = NULL` yields `NULL`.  This mismatch is necessary for the Rust code to properly detect and
process `Null` values, and the desired SQL semantics are implemented during expression evaluation
which we'll cover below.

它还定义了比较和排序语义，但这些语义与 SQL 语义有细微差别。例如，在 Rust 代码中 `Value::Null == Value::Null` 的结果是 `true`，而在 SQL 中 `NULL = NULL` 的结果是 `NULL`。这种不一致是必要的，这样 Rust 代码才能正确检测和处理 `Null` 值；而所需的 SQL 语义则在表达式求值时实现，我们稍后会讲到。

<https://github.com/erikgrinaker/toydb/blob/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql/types/value.rs#L91-L162>

During execution, a row of values is represented as `sql::types::Row`, with multiple rows emitted
via `sql::types::Rows` row iterators:

在执行期间，一行值用 `sql::types::Row` 表示，多行数据则通过 `sql::types::Rows` 行迭代器输出：

<https://github.com/erikgrinaker/toydb/blob/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql/types/value.rs#L378-L388>

## Schemas（模式）

toyDB schemas only support tables. There are no named indexes or constraints, and there's only a
single unnamed database.

toyDB 的模式只支持表。没有命名索引或命名约束，并且只有一个未命名的数据库。

Tables are represented by `sql::types::Table`:

表用 `sql::types::Table` 表示：

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/types/schema.rs#L12-L25>

A table is made up of a set of columns, represented by `sql::types::Column`. These support the data
types described above, along with unique constraints, foreign keys, and secondary indexes.

表由一组列组成，用 `sql::types::Column` 表示。列支持上述数据类型，还支持唯一约束、外键和二级索引。

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/types/schema.rs#L29-L53>

The table name serves as a unique identifier, and can't be changed later. In fact, tables schemas
are entirely static: they can only be created or dropped (there are no schema changes).

表名作为唯一标识符，之后不能更改。实际上，表模式是完全静态的：只能创建或删除表（不支持修改模式）。

Table schemas are stored in the catalog, represented by the `sql::engine::Catalog` trait. We'll
revisit the implementation of this trait in the SQL storage section.

表模式存储在目录（catalog）中，由 `sql::engine::Catalog` trait 表示。我们将在 SQL 存储一节中重新讨论这个 trait 的实现。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/engine/engine.rs#L60-L79>

Table schemas are validated when created via `Table::validate()`, which enforces invariants and
internal consistency. It uses the catalog to look up information about other tables, e.g. that
foreign key references point to a valid target column in a different table.

表在创建时会通过 `Table::validate()` 进行校验，以强制保证不变量和内部一致性。它会使用目录来查找其他表的信息，例如外键引用是否指向另一个表中的有效目标列。

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/types/schema.rs#L98-L170>

Table rows are validated via `Table::validate_row()`, which ensures that a `sql::types::Row`
conforms to the schema (e.g. that value types match the column data types). It uses a
`sql::engine::Transaction` to look up other rows in the database, e.g. to check for primary key
conflicts (we'll get back to this later).

表行通过 `Table::validate_row()` 进行校验，以确保 `sql::types::Row` 符合模式（例如值类型与列数据类型匹配）。它会使用 `sql::engine::Transaction` 查找数据库中的其他行，例如检查主键冲突（稍后会详细讨论）。

<https://github.com/erikgrinaker/toydb/blob/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/types/schema.rs#L172-L236>

## Expressions（表达式）

During SQL execution, we also have to model _expressions_, such as `1 + 2 * 3`. These are
represented as values and operations on them, and can be nested as a tree to represent compound
operations.

在 SQL 执行期间，我们还需要对_表达式_建模，例如 `1 + 2 * 3`。表达式由值以及对值的操作表示，并且可以嵌套成树形结构来表示复合运算。

<https://github.com/erikgrinaker/toydb/blob/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/types/expression.rs#L11-L64>

For example, the expression `1 + 2 * 3` (taking [precedence](https://en.wikipedia.org/wiki/Order_of_operations)
into account) is represented as:

例如，表达式 `1 + 2 * 3`（考虑[运算符优先级](https://en.wikipedia.org/wiki/Order_of_operations)）表示为：

```rust
//    +
//   / \
//  1   *
//     /  \
//    2    3
Expression::Add(
    Expression::Constant(Value::Integer(1)),
    Expression::Multiply(
        Expression::Constant(Value::Integer(2)),
        Expression::Constant(Value::Integer(3)),
    ),
)
```

An `Expression` can contain two kinds of values: constant values as
`Expression::Constant(sql::types::Value)`, and dynamic values as `Expression::Column(usize)` column
references. The latter will fetch a `sql::types::Value` from a `sql::types::Row` at the specified
index during evaluation.

`Expression` 可以包含两种值：一种是常量值，即 `Expression::Constant(sql::types::Value)`；另一种是动态值，即 `Expression::Column(usize)` 列引用。后者在求值时会从 `sql::types::Row` 中按指定索引取出一个 `sql::types::Value`。

We'll see later how the SQL parser and planner transforms text expression like `1 + 2 * 3` into an
`Expression`, and how it resolves column names to row indexes like `price * 0.25` to
`row[3] * 0.25`.

我们稍后会看到 SQL 解析器和规划器如何把 `1 + 2 * 3` 这样的文本表达式转换成 `Expression`，以及如何把列名解析为行索引，例如把 `price * 0.25` 解析为 `row[3] * 0.25`。

Expressions are evaluated recursively via `Expression::evalute()`, given a `sql::types::Row` with
input values for column references, and return a final `sql::types::Value` result:

表达式通过 `Expression::evalute()` 递归求值：传入一个 `sql::types::Row`，为列引用提供输入值，最终返回一个 `sql::types::Value` 结果：

<https://github.com/erikgrinaker/toydb/blob/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/types/expression.rs#L73-L208>

Many of the comparison operations like `==` are implemented explicitly here instead of using
`sql::types::Value` comparisons. This is where we implement the SQL semantics of special values like
`NULL`, such that `NULL = NULL` yields `NULL` instead of `TRUE`.

许多比较运算（例如 `==`）在这里是显式实现的，而不是直接使用 `sql::types::Value` 的比较。正是这里实现了 `NULL` 等特殊值的 SQL 语义，使得 `NULL = NULL` 的结果是 `NULL` 而不是 `TRUE`。

For mathematical operations however, we generally dispatch to these methods on `sql::types::Value`:

而对于数学运算，我们通常分派给 `sql::types::Value` 上的这些方法：

<https://github.com/erikgrinaker/toydb/blob/b2fe7b76ee634ca6ad31616becabfddb1c03d34b/src/sql/types/value.rs#L185-L295>

Expression parsing and evaluation is tested via test scripts in
[`sql/testscripts/expression`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/testscripts/expressions).

表达式的解析和求值通过 [`sql/testscripts/expression`](https://github.com/erikgrinaker/toydb/tree/9419bcf6aededf0e20b4e7485e2a5fa3e975d79f/src/sql/testscripts/expressions) 中的测试脚本进行测试。

---

<p align="center">
← <a href="sql.md">SQL Engine</a> &nbsp; | &nbsp; <a href="sql-storage.md">SQL Storage</a> →
</p>
