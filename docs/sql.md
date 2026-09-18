# SQL Reference（SQL 参考）

## Data Types（数据类型）

The following data types are supported:

支持以下数据类型：

* `BOOLEAN` (`BOOL`): logical truth values, i.e. true and false.
* `BOOLEAN`（`BOOL`）：逻辑真值，即 true 和 false。
* `FLOAT` (`DOUBLE`): 64-bit signed floating point numbers, using [IEEE 754 `binary64`](https://en.wikipedia.org/wiki/binary64) encoding. Supports magnitudes of 10⁻³⁰⁷ to 10³⁰⁸ with 53-bit precision (~15 significant figures), as well as the special values infinity and NaN.
* `FLOAT`（`DOUBLE`）：64 位有符号浮点数，采用 [IEEE 754 `binary64`](https://en.wikipedia.org/wiki/binary64) 编码。支持的数值范围为 10⁻³⁰⁷ 到 10³⁰⁸，精度为 53 位（约 15 位有效数字），还支持特殊值 infinity 和 NaN。
* `INTEGER` (`INT`): 64-bit signed integer numbers with a range of ±2⁶³-1.
* `INTEGER`（`INT`）：64 位有符号整数，范围为 ±2⁶³-1。
* `STRING` (`TEXT`, `VARCHAR`): UTF-8 encoded strings.
* `STRING`（`TEXT`、`VARCHAR`）：UTF-8 编码的字符串。

In addition, the special `NULL` value is used for an unknown value, following the rules of [three-valued logic](https://en.wikipedia.org/wiki/Three-valued_logic).

此外，特殊的 `NULL` 值用于表示未知值，遵循[三值逻辑](https://en.wikipedia.org/wiki/Three-valued_logic)的规则。

Numeric types are not interchangable; a float value (even without a fractional part) cannot be stored in an integer column and vice-versa.

数值类型之间不可互换；浮点值（即使没有小数部分）不能存储到整数列中，反之亦然。

## SQL Syntax（SQL 语法）

### Keywords（关键字）

Keywords are reserved words with special meaning in SQL statements. They are case-insensitive, and must be quoted with `"` to be used as identifiers. The complete list is:

关键字是 SQL 语句中具有特殊含义的保留字。它们不区分大小写，如果要作为标识符使用，必须用 `"` 引起来。完整列表如下：

`AS`, `ASC`, `AND`, `BEGIN`, `BOOL`, `BOOLEAN`, `BY`, `COMMIT`, `CREATE`, `CROSS`, `DEFAULT`,`DELETE`, `DESC`, `DOUBLE`, `DROP`, `EXISTS`, `EXPLAIN`, `FALSE`, `FLOAT`, `FROM`, `GROUP`, `HAVING`, `IF`, `INDEX`, `INFINITY`, `INNER`, `INSERT`, `INT`, `INTEGER`, `INTO`, `IS`, `JOIN`, `KEY`, `LEFT`, `LIKE`, `LIMIT`, `NAN`, `NOT`, `NULL`, `OF`, `OFFSET`, `ON`, `ONLY`, `OR`, `ORDER`, `OUTER`, `PRIMARY`, `READ`, `REFERENCES`, `RIGHT`, `ROLLBACK`, `SELECT`, `SET`, `STRING`, `SYSTEM`, `TABLE`, `TEXT`, `TIME`, `TRANSACTION`, `TRUE`, `UNIQUE`, `UPDATE`, `VALUES`, `VARCHAR`, `WHERE`, `WRITE`

### Identifiers（标识符）

Identifiers are names for database objects such as tables and columns. Unless quoted with `"`, they must begin with a Unicode letter followed by any combination of letters, numbers, and `_`, and cannot be reserved keywords. `""` can be used to escape a double quote character. They are always converted to lowercase.

标识符是表、列等数据库对象的名称。除非用 `"` 引起来，否则标识符必须以 Unicode 字母开头，后面可以跟字母、数字和 `_` 的任意组合，并且不能是保留关键字。`""` 可用于转义双引号字符。标识符总是被转换为小写。

### Constants（常量）

#### Named constants（具名常量）

The following keywords evaluate to constants:

以下关键字求值为常量：

* `FALSE`: the boolean false value.
* `FALSE`：布尔值 false。
* `INFINITY`: the floating-point value for infinity.
* `INFINITY`：浮点数中的无穷大值。
* `NAN`: the floating-point value for NaN (not a number).
* `NAN`：浮点数中的 NaN（非数字）值。
* `NULL`: an unknown value.
* `NULL`：未知值。
* `TRUE`: the boolean true value.
* `TRUE`：布尔值 true。

#### String literals（字符串字面量）

String literals are surrounded by single quotes `'`, and can contain any valid UTF-8 character. Single quotes must be escaped by an additional single quote, i.e. `''`, no other escape sequences are supported. For example:

字符串字面量由单引号 `'` 包围，可以包含任何有效的 UTF-8 字符。单引号必须用再加一个单引号的方式转义，即 `''`，不支持其他转义序列。例如：

```
'A string with ''quotes'' and emojis 😀'
```

#### Numeric literals（数字字面量）

Sequences of digits `0-9` are parsed as a 64-bit signed integer. Numbers with decimal points or in scientific notation are parsed as 64-bit floating point numbers. The following pattern is supported:

数字 `0-9` 组成的序列被解析为 64 位有符号整数。带小数点或科学计数法表示的数字被解析为 64 位浮点数。支持以下模式：

```
999[.[999]][e[+-]999]
```

The `-` prefix operator can be used to take negative numbers.

`-` 前缀运算符可用于表示负数。

### Expressions（表达式）

Expressions can be used wherever a value is expected, e.g. as `SELECT` columns nd `INSERT` values. They are made up of constants, a column references, an operator invocations, and a function calls.

凡是需要值的地方都可以使用表达式，例如作为 `SELECT` 列或 `INSERT` 值。表达式由常量、列引用、运算符调用和函数调用组成。

Column references can either be unqualified, e.g. `name`, or prefixed with the relation identifier separated by `.`, e.g. `person.name`. Unqualified identifiers must be unambiguous.

列引用可以是不带限定的，例如 `name`，也可以以关系标识符为前缀并用 `.` 分隔，例如 `person.name`。不带限定的标识符必须无歧义。

## SQL Operators（SQL 运算符）

### Logical operators（逻辑运算符）

Logical operators apply standard logic operations on boolean operands.

逻辑运算符对布尔操作数执行标准的逻辑运算。

* `AND`: the logical conjunction, e.g. `TRUE AND TRUE` yields `TRUE`.
* `AND`：逻辑与，例如 `TRUE AND TRUE` 得到 `TRUE`。
* `OR`: the logical disjunction, e.g. `TRUE OR FALSE` yields `TRUE`.
* `OR`：逻辑或，例如 `TRUE OR FALSE` 得到 `TRUE`。
* `NOT`: the logical negation, e.g. `NOT TRUE` yields `FALSE`.
* `NOT`：逻辑非，例如 `NOT TRUE` 得到 `FALSE`。

The complete truth tables are:

完整的真值表如下：

| `AND`       | `TRUE`  | `FALSE` | `NULL`  |
|-------------|---------|---------|---------|
| **`TRUE`**  | `TRUE`  | `FALSE` | `NULL`  |
| **`FALSE`** | `FALSE` | `FALSE` | `FALSE` |
| **`NULL`**  | `NULL`  | `FALSE` | `NULL`  |

| `OR`        | `TRUE` | `FALSE` | `NULL` |
|-------------|--------|---------|--------|
| **`TRUE`**  | `TRUE` | `TRUE`  | `TRUE` |
| **`FALSE`** | `TRUE` | `FALSE` | `NULL` |
| **`NULL`**  | `TRUE` | `NULL`  | `NULL` |

| `NOT`       |         |
|-------------|---------|
| **`TRUE`**  | `FALSE` |
| **`FALSE`** | `TRUE`  |
| **`NULL`**  | `NULL`  |

上面三张表分别是 `AND`、`OR` 和 `NOT` 运算的完整真值表。

### Comparison operators（比较运算符）

Comparison operators compare values of the same data type, and return `TRUE` if the comparison holds or `FALSE` otherwise. `INTEGER` and `FLOAT` values are interchangeable. `STRING` comparisons use the string's byte values, i.e. case-sensitive with `'B' < 'a'` due to their UTF-8 code points. `FALSE` is considered lesser than `TRUE`. Comparison with `NULL` always yields `NULL` (even `NULL = NULL`).

比较运算符比较相同数据类型的值，如果比较成立则返回 `TRUE`，否则返回 `FALSE`。`INTEGER` 和 `FLOAT` 值可以互换使用。`STRING` 比较使用字符串的字节值，即区分大小写，由于 UTF-8 码点的缘故 `'B' < 'a'`。`FALSE` 被认为小于 `TRUE`。与 `NULL` 的比较总是得到 `NULL`（即使是 `NULL = NULL`）。

Binary operators:

二元运算符：

* `=`: equality, e.g. `1 = 1` yields `TRUE`.
* `=`：相等，例如 `1 = 1` 得到 `TRUE`。
* `!=`: inequality, e.g. `1 != 2` yields `TRUE`.
* `!=`：不相等，例如 `1 != 2` 得到 `TRUE`。
* `>`: greater than, e.g. `2 > 1` yields `TRUE`.
* `>`：大于，例如 `2 > 1` 得到 `TRUE`。
* `>=`: greater than or equal, e.g. `1 >= 1` yields `TRUE`.
* `>=`：大于等于，例如 `1 >= 1` 得到 `TRUE`。
* `<`: lesser than, e.g. `1 < 2` yields `TRUE`.
* `<`：小于，例如 `1 < 2` 得到 `TRUE`。
* `<=`: lesser than or equal, e.g. `1 <= 1` yields `TRUE`.
* `<=`：小于等于，例如 `1 <= 1` 得到 `TRUE`。

Unary operators:

一元运算符：

* `IS NULL`: checks if the value is `NULL`, e.g. `NULL IS NULL` yields `TRUE`.
* `IS NULL`：检查值是否为 `NULL`，例如 `NULL IS NULL` 得到 `TRUE`。
* `IS NOT NULL`: checks if the value is not `NULL`, e.g. `TRUE IS NOT NULL` yields `TRUE`.
* `IS NOT NULL`：检查值是否不为 `NULL`，例如 `TRUE IS NOT NULL` 得到 `TRUE`。
* `IS NAN`: checks if the value is a float `NAN`, e.g. `NAN IS NAN` yields `TRUE`. Errors on
  non-float datatypes, except `NULL` which yields `NULL`.
* `IS NAN`：检查值是否为浮点数 `NAN`，例如 `NAN IS NAN` 得到 `TRUE`。对非浮点数据类型会报错，但 `NULL` 例外（得到 `NULL`）。
* `IS NOT NAN`: checks if the value is not a float `NAN`, e.g. `3.14 IS NOT NAN` yields `TRUE`.
* `IS NOT NAN`：检查值是否不是浮点数 `NAN`，例如 `3.14 IS NOT NAN` 得到 `TRUE`。

### Mathematical operators（数学运算符）

Mathematical operators apply standard math operations on numeric (`INTEGER` or `FLOAT`) operands. If either operand is a `FLOAT`, both operands are converted to `FLOAT` and the result is a `FLOAT`. If either operand is `NULL`, the result is `NULL`. The special values `INFINITY` and `NAN` are handled according to the IEEE 754 spec.

数学运算符对数值（`INTEGER` 或 `FLOAT`）操作数执行标准数学运算。如果任一操作数是 `FLOAT`，则两个操作数都被转换为 `FLOAT`，结果也是 `FLOAT`。如果任一操作数是 `NULL`，结果为 `NULL`。特殊值 `INFINITY` 和 `NAN` 按照 IEEE 754 规范处理。

For `INTEGER` operands, failure conditions such as overflow and division by zero yield an error. For `FLOAT` operands, these return `INFINITY` or `NAN` as appropriate.

对于 `INTEGER` 操作数，溢出和除零等失败情形会产生错误。对于 `FLOAT` 操作数，这些情形会酌情返回 `INFINITY` 或 `NAN`。

Binary operators:

二元运算符：

* `+`: addition, e.g. `1 + 2` yields `3`.
* `+`：加法，例如 `1 + 2` 得到 `3`。
* `-`: subtraction, e.g. `3 - 2` yields `1`.
* `-`：减法，例如 `3 - 2` 得到 `1`。
* `*`: multiplication, e.g. `3 * 2` yields `6`.
* `*`：乘法，例如 `3 * 2` 得到 `6`。
* `/`: division, e.g. `6 / 2` yields `3`.
* `/`：除法，例如 `6 / 2` 得到 `3`。
* `^`: exponentiation, e.g. `2 ^ 4` yields `16`.
* `^`：乘方，例如 `2 ^ 4` 得到 `16`。
* `%`: remainder, e.g. `8 % 3` yields `2`. Unlike modulo, the result has the sign of the dividend.
* `%`：取余，例如 `8 % 3` 得到 `2`。与取模不同，结果的符号与被除数相同。

Unary operators:

一元运算符：

* `+` (prefix): identity, e.g. `+1` yields `1`.
* `+`（前缀）：恒等，例如 `+1` 得到 `1`。
* `-` (prefix): negation, e.g. `- -2` yields `2`.
* `-`（前缀）：取负，例如 `- -2` 得到 `2`。
* `!` (postfix): factorial, e.g. `5!` yields `15`.
* `!`（后缀）：阶乘，例如 `5!` 得到 `15`。

### String operators（字符串运算符）

String operators operate on string operands.

字符串运算符作用于字符串操作数。

* `LIKE`: compares a string with the given pattern, using `%` as multi-character wildcard and `_` as single-character wildcard, returning `TRUE` if the string matches the pattern - e.g. `'abc' LIKE 'a%'` yields `TRUE`.
* `LIKE`：将字符串与给定模式进行比较，`%` 作为多字符通配符，`_` 作为单字符通配符，如果字符串匹配模式则返回 `TRUE` —— 例如 `'abc' LIKE 'a%'` 得到 `TRUE`。

### Operator precedence（运算符优先级）

The operator precedence (order of operations) is as follows:

运算符优先级（运算顺序）如下：

| Precedence | Operator                | Associativity |
|------------|-------------------------|---------------|
| 10         | `+`, `-` (prefix)       | Right         |
| 9          | `!` (postfix)           | Left          |
| 8          | `^`                     | Right         |
| 7          | `*`, `/`, `%`           | Left          |
| 6          | `+`, `-`                | Left          |
| 5          | `>`, `>=`, `<`, `<=`    | Left          |
| 4          | `=`, `!=`, `LIKE`, `IS` | Left          |
| 3          | `NOT`                   | Right         |
| 2          | `AND`                   | Left          |
| 1          | `OR`                    | Left          |

表中列为优先级（数值越大优先级越高）、运算符和结合性（Left 为左结合，Right 为右结合）。

Precedence can be overridden by wrapping an expression in parentheses, e.g. `(1 + 2) * 3`.

可以用括号包裹表达式来覆盖优先级，例如 `(1 + 2) * 3`。

### Functions（函数）

* `sqrt(expr)`: returns the square root of a numerical argument.
* `sqrt(expr)`：返回数值参数的平方根。

### Aggregate functions（聚合函数）

Aggregate function aggregate an expression across all rows, optionally grouped into buckets given by `GROUP BY`, and results can be filtered via `HAVING`.

聚合函数对所有行的表达式进行聚合，可以按 `GROUP BY` 给出的分组进行分组，结果可以通过 `HAVING` 过滤。

* `AVG(expr)`: returns the average of numerical values.
* `AVG(expr)`：返回数值的平均值。

* `COUNT(expr)`: returns the number of rows for which ***`expr`*** evaluates to a non-`NULL` value. `COUNT(*)` can be used to count all rows.
* `COUNT(expr)`：返回 ***`expr`*** 求值为非 `NULL` 值的行数。`COUNT(*)` 可用于统计所有行。

* `MAX(expr)`: returns the maximum value, according to the datatype's ordering.
* `MAX(expr)`：按照数据类型的排序规则返回最大值。

* `MIN(expr)`: returns the minimum value, according to the datatype's ordering.
* `MIN(expr)`：按照数据类型的排序规则返回最小值。

* `SUM(expr)`: returns the sum of numerical values.
* `SUM(expr)`：返回数值之和。

## SQL Statements（SQL 语句）

### `BEGIN`（开始事务）

Starts a new [transaction](#transactions).

开始一个新[事务](#transactions)。

<pre>
BEGIN [ TRANSACTION ] [ READ ONLY | READ WRITE ] [ AS OF SYSTEM TIME <b><i>txn_id</i></b> ]
</pre>

* ***`txn_id`***: A past transaction ID to run a read-only transaction for, for time-travel queries.
* ***`txn_id`***：过去的一个事务 ID，用于在其上运行只读事务，实现时间旅行查询（time-travel queries）。

### `COMMIT`（提交事务）

Commits an active [transaction](#transactions).

提交当前活动[事务](#transactions)。

### `CREATE TABLE`（创建表）

Creates a new table.

创建一个新表。

<pre>
CREATE TABLE <b><i>table_name</i></b> (
    [ <b><i>column_name</i></b> <b><i>data_type</i></b> [ <b><i>column_constraint</i></b> [ ... ] ]  [ INDEX ] [, ... ] ]
)

where <b><i>column_constraint</i></b> is:

{ NOT NULL | NULL | PRIMARY KEY | DEFAULT <b><i>expr</i></b> | REFERENCES <b><i>ref_table</i></b> | UNIQUE }
</pre>

* ***`table_name`***: The name of the table. Must be a [valid identifier](#identifiers). Errors if a table with this name already exists.
* ***`table_name`***：表的名称。必须是[有效标识符](#identifiers)。如果同名表已存在则报错。

* ***`column_name`***: The name of the column. Must be a [valid identifier](#identifiers), and unique within the table.
* ***`column_name`***：列的名称。必须是[有效标识符](#identifiers)，且在表内唯一。

* ***`data_type`***: The data type of the column, see [data types](#data-types) for valid types.
* ***`data_type`***：列的数据类型，有效类型参见[数据类型](#data-types)。

* `NOT NULL`: The column may not contain `NULL` values.
* `NOT NULL`：该列不能包含 `NULL` 值。

* `NULL`: The column may contain `NULL` values. This is the default.
* `NULL`：该列可以包含 `NULL` 值。这是默认行为。

* `PRIMARY KEY`: The column should act as a primary key, i.e. the main row identifier. A table must have exactly one primary key column, and it must be unique and non-nullable.
* `PRIMARY KEY`：该列作为主键，即主要的行标识符。一个表必须恰好有一个主键列，且它必须唯一且不可为 NULL。

* `DEFAULT`***`expr`***: Specifies a default value for the column when `INSERT` statements do not give a value. ***`expr`*** can be any constant expression of an appropriate data type, e.g. `'abc'` or `1 + 2 * 3`. For nullable columns, the default value is `NULL` unless specified otherwise.
* `DEFAULT`***`expr`***：指定当 `INSERT` 语句未给出值时该列的默认值。***`expr`*** 可以是适当数据类型的任何常量表达式，例如 `'abc'` 或 `1 + 2 * 3`。对于可为 NULL 的列，默认值是 `NULL`，除非另有指定。

* `REFERENCES`***`ref_table`***: The column is a foreign key to ***`ref_table`***'s primary key, enforcing referential integrity.
* `REFERENCES`***`ref_table`***：该列是指向 ***`ref_table`*** 主键的外键，用于强制引用完整性。

* `UNIQUE`: The column may only contain unique (distinct) values. `NULL` values are not considered equal, thus a `UNIQUE` column which allows `NULL` may contain multiple `NULL` values. `PRIMARY KEY` columns are implicitly `UNIQUE`.
* `UNIQUE`：该列只能包含唯一（互不相同）的值。`NULL` 值之间不被视为相等，因此允许 `NULL` 的 `UNIQUE` 列可以包含多个 `NULL` 值。`PRIMARY KEY` 列隐含 `UNIQUE`。

* `INDEX`: Create an index for the column.
* `INDEX`：为该列创建索引。

#### Example（示例）

```sql
CREATE TABLE movie (
    id INTEGER PRIMARY KEY,
    title STRING NOT NULL,
    release_year INTEGER INDEX,
    imdb_id STRING INDEX UNIQUE,
    bluray BOOLEAN NOT NULL DEFAULT TRUE
)
```

### `DELETE`（删除行）

Deletes rows in a table.

删除表中的行。

<pre>
DELETE FROM <b><i>table_name</i></b>
    [ WHERE <b><i>predicate</i></b> ]
</pre>

Deletes rows where ***`predicate`*** evaluates to `TRUE`, or all rows if no `WHERE` clause is given.

删除 ***`predicate`*** 求值为 `TRUE` 的行；如果没有给出 `WHERE` 子句，则删除所有行。

* ***`table_name`***: the table to delete from. Errors if it does not exist.
* ***`table_name`***：要从中删除的表。如果不存在则报错。

* ***`predicate`***: an expression which determines which rows to delete by evaluting to `TRUE`. Must evaluate to a `BOOLEAN` or `NULL`, otherwise an error is returned.
* ***`predicate`***：一个谓词表达式，通过求值为 `TRUE` 来决定删除哪些行。其求值结果必须是 `BOOLEAN` 或 `NULL`，否则返回错误。

#### Example（示例）

```sql
DELETE FROM movie
WHERE release_year < 2000 AND bluray = FALSE
```

### `DROP TABLE`（删除表）

Deletes a table and all contained data. Errors if the table does not
exist, unless `IF EXISTS` is given.

删除一个表及其包含的所有数据。如果表不存在则报错，除非给出了 `IF EXISTS`。

<pre>
DROP TABLE [ IF EXISTS ] <b><i>table_name</i></b>
</pre>

* ***`table_name`***: the table to delete.
* ***`table_name`***：要删除的表。

### `EXPLAIN`（查看执行计划）

Outputs the execution plan for the given statement.

输出给定语句的执行计划。

<pre>
EXPLAIN [ <b><i>statement</i></b> ]
</pre>

### `INSERT`（插入行）

Inserts rows into a table.

向表中插入行。

<pre>
INSERT INTO <b><i>table_name</i></b>
    [ ( <b><i>column_name</i></b> [, ... ] ) ]
    VALUES ( <b><i>expression</i></b> [, ... ] ) [, ... ]
</pre>

If column names are given, an identical number of values must be given. If no column names are given, values must be given in the table's column order. Omitted columns will get a default value if specified, otherwise an error will be returned.

如果给出了列名，则必须给出数量相同的值。如果没有给出列名，则必须按表的列顺序给出值。被省略的列如果指定了默认值则取默认值，否则返回错误。

* ***`table_name`***: the table to insert into. Errors if it does not exist.
* ***`table_name`***：要插入的表。如果不存在则报错。

* ***`column_name`***: a column to insert into in the given table. Errors if it does not exist.
* ***`column_name`***：给定表中要插入的列。如果不存在则报错。

* ***`expression`***: an expression to insert into the corresponding column. Must be a constant expression, i.e. it cannot refer to table columns.
* ***`expression`***：要插入到对应列的表达式。必须是常量表达式，即不能引用表列。

#### Example（示例）

```sql
INSERT INTO movie
    (id, title, release_year)
VALUES
    (1, 'Sicario', 2015),
    (2, 'Stalker', 1979),
    (3, 'Her', 2013)
```

### `ROLLBACK`（回滚事务）

Rolls back an active [transaction](#transactions).

回滚当前活动[事务](#transactions)。

### `SELECT`（查询）

Selects rows from a table.

从表中查询行。

<pre>
SELECT [ * | <b><i>expression</i></b> [ [ AS ] <b><i>output_name</i></b> [, ...] ] ]
    [ FROM <b><i>from_item</i></b> [, ...] ]
    [ WHERE <b><i>predicate</i></b> ]
    [ GROUP BY <b><i>group_expr</i></b> [, ...] ]
    [ HAVING <b><i>having_expr</i></b> ]
    [ ORDER BY <b><i>order_expr</i></b> [ ASC | DESC ] [, ...] ]
    [ LIMIT <b><i>count</i></b> ]
    [ OFFSET <b><i>start</i></b> ]

where <b><i>from_item</i></b> is one of:

<b><i>table_name</i></b> [ [ AS ] <b><i>alias</i></b> ]
<b><i>from_item</i></b> <b><i>join_type</i></b> <b><i>from_item</i></b> [ ON <b><i>join_predicate</i></b> ]

where <b><i>join_type</i></b> is one of:

CROSS JOIN
[ INNER ] JOIN
LEFT [ OUTER ] JOIN
RIGHT [ OUTER ] JOIN

</pre>

Fetches rows or expressions, either from table ***`table_name`*** (if given) or generated.

获取行或表达式，可以来自表 ***`table_name`***（如果给出），也可以是生成的。

* ***`expression`***: [expression](#expressions) to fetch (can be a simple column name).
* ***`expression`***：要获取的[表达式](#expressions)（可以是简单的列名）。

* ***`output_name`***: output column [identifier](#identifier), defaults to column name (if single column) otherwise nothing (displayed as `?`).
* ***`output_name`***：输出列的[标识符](#identifier)，单列时默认为列名，否则为空（显示为 `?`）。

* ***`table_name`***: table to fetch rows from.
* ***`table_name`***：要从中获取行的表。

* ***`alias`***: table alias.
* ***`alias`***：表别名。

* ***`predicate`***: only return rows for which this [expression](#expressions) evaluates to `TRUE`.
* ***`predicate`***：只返回该[表达式](#expressions)求值为 `TRUE` 的行。

* ***`group_expr`***: an expression to group aggregates by. Non-aggregate `SELECT` expressions must either reference a column given in `group_expr`, be idential with a `group_expr`, or have an `output_name` that is referenced by a `group_expr` column.
* ***`group_expr`***：用于聚合分组的表达式。非聚合的 `SELECT` 表达式必须引用 `group_expr` 中给出的列、与某个 `group_expr` 完全相同，或者其 `output_name` 被 `group_expr` 中的列所引用。

* ***`having_expr`***: only return aggregate results for which this [expression](#expressions) evaluates to `TRUE`.
* ***`having_expr`***：只返回该[表达式](#expressions)求值为 `TRUE` 的聚合结果。

* ***`order_expr`***: order rows by this expression (can be a simple column name).
* ***`order_expr`***：按此表达式对行排序（可以是简单的列名）。

* ***`count`***: maximum number of rows to return. Must be a constant integer expression.
* ***`count`***：返回的最大行数。必须是常量整数表达式。

* ***`start`***: number of rows to skip. Must be a constant integer expression.
* ***`start`***：要跳过的行数。必须是常量整数表达式。

* ***`join_predicate`***: only return rows for which this [expression](#expressions) evaluates to `TRUE`.
* ***`join_predicate`***：只返回该[表达式](#expressions)求值为 `TRUE` 的行。

Join types:

连接类型：

* `CROSS JOIN`: returns the Carthesian product of the joined tables. Does not accept a join predicate (`ON` clause).
* `CROSS JOIN`：返回被连接表的笛卡尔积。不接受连接谓词（`ON` 子句）。

* `INNER JOIN`: returns the rows of the tables' Carthesian product for which  ***`join_predicate`*** evaluates to `TRUE`.
* `INNER JOIN`：返回表的笛卡尔积中 ***`join_predicate`*** 求值为 `TRUE` 的行。

* `LEFT OUTER JOIN`: returns the rows joined on the ***`join_predicate`***, or for any rows in the left table that does not have a match in the right table a single row is returned with the right table's columns set to `NULL`.
* `LEFT OUTER JOIN`：返回按 ***`join_predicate`*** 连接的行；对于左表中在右表没有匹配的行，返回一行，其中右表的列全部置为 `NULL`。

* `RIGHT OUTER JOIN`: the same as a `LEFT OUTER JOIN` but with the left and right tables switched.
* `RIGHT OUTER JOIN`：与 `LEFT OUTER JOIN` 相同，只是左右表互换。

#### Example（示例）

```sql
SELECT id, title, 2020 - released AS age
FROM movies
WHERE released >= 2000 AND ultrahd
ORDER BY released DESC, title ASC
LIMIT 10
OFFSET 10
```

### `UPDATE`（更新行）

Updates rows in a table.

更新表中的行。

<pre>
UPDATE <b><i>table_name</i></b>
    SET <b><i>column_name</i></b> = <b><i>expression</i></b> | DEFAULT [, ... ]
    [ WHERE <b><i>predicate</i></b> ]
</pre>

Updates columns given by ***`column_name`*** to the corresponding ***`expression`*** for all rows where ***`predicate`*** evaluates to `TRUE`. If no `WHERE` clause is given, all rows are updated.

对于 ***`predicate`*** 求值为 `TRUE` 的所有行，将 ***`column_name`*** 给出的列更新为对应的 ***`expression`***。如果没有给出 `WHERE` 子句，则更新所有行。

* ***`table_name`***: the table to update. Errors if it does not exist.
* ***`table_name`***：要更新的表。如果不存在则报错。

* ***`column_name`***: a column to update. Errors if it does not exist.
* ***`column_name`***：要更新的列。如果不存在则报错。

* ***`expression`***: an expression whose evaluated value will be set for the corresponding column and row. Expressions can refer to column values, and must evaluate to the same datatype as the updated column. Using `DEFAULT` will set the column's default value, if any.
* ***`expression`***：一个表达式，其求值结果将被设置为对应列和行的值。表达式可以引用列值，且必须求值为与被更新列相同的数据类型。使用 `DEFAULT` 将设置该列的默认值（如果有）。

* ***`predicate`***: an expression which determines which rows to update by evaluting to `TRUE`. Must evaluate to a `BOOLEAN` or `NULL`, otherwise an error is returned.
* ***`predicate`***：一个谓词表达式，通过求值为 `TRUE` 来决定更新哪些行。其求值结果必须是 `BOOLEAN` 或 `NULL`，否则返回错误。

#### Example（示例）

```sql
UPDATE movie
SET bluray = TRUE
WHERE release_year >= 2000 AND bluray = FALSE
```

## Transactions（事务）

toyDB supports ACID transactions using MVCC-based snapshot isolation, protecting from the following anomalies: dirty writes, dirty reads, lost updates, fuzzy reads, read skew, and phantom reads. However, write skew anomalies are possible since serializable snapshot isolation is not implemented.

toyDB 使用基于 MVCC 的快照隔离（snapshot isolation）来支持 ACID 事务，可以防止以下异常：脏写（dirty write）、脏读（dirty read）、丢失更新（lost update）、模糊读（fuzzy read）、读偏斜（read skew）和幻读（phantom read）。但是，由于没有实现可串行化快照隔离，写偏斜（write skew）异常仍可能出现。

A new transaction is started with `BEGIN`, and ended with either `COMMIT` (atomically writing all changes) or `ROLLBACK` (discarding all changes). If any conflicts occur between concurrent transactions, the lowest transaction ID wins and the others will fail with a serialization error and must retry.

新事务用 `BEGIN` 开始，用 `COMMIT`（原子地写入所有更改）或 `ROLLBACK`（丢弃所有更改）结束。如果并发事务之间发生冲突，事务 ID 最小的事务获胜，其他事务将以序列化冲突错误失败并必须重试。

All past data is versioned and retained, and can be queried as of a given transaction ID via `BEGIN TRANSACTION READ ONLY AS OF SYSTEM TIME <txn_id>`.

所有历史数据都带版本并保留，可以通过 `BEGIN TRANSACTION READ ONLY AS OF SYSTEM TIME <txn_id>` 按给定的事务 ID 查询过去的数据。

A transaction is still valid for use if a contained statement returns an error. It is up to the client to take appropriate action.

即使事务中的某条语句返回错误，事务仍然有效可用。由客户端自行决定采取何种后续操作。
