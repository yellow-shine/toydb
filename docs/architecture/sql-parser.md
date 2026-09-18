# SQL Parsing（SQL 解析）

We finally arrive at SQL. The SQL parser is the first stage in processing SQL queries and
statements, located in the [`sql::parser`](https://github.com/erikgrinaker/toydb/tree/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser)
module.

我们终于来到了 SQL。SQL 解析器是处理 SQL 查询和语句的第一个阶段，位于 [`sql::parser`](https://github.com/erikgrinaker/toydb/tree/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser) 模块中。

The SQL parser's job is to take a raw SQL string and turn it into a structured form that's more
convenient to work with. In doing so, it will validate that the string is in fact valid SQL
_syntax_. However, it doesn't know if the SQL statement actually makes sense -- it has no idea which
tables or columns exist, what their data types are, and so on. That's the job of the planner, which
we'll look at later.

SQL 解析器的任务是接收一段原始 SQL 字符串，并将其转换为更便于处理的结构化形式。在此过程中，它会验证该字符串是否确实是合法的 SQL _语法_。然而，它并不知道这条 SQL 语句是否真的有意义——它不知道存在哪些表或列、它们的数据类型是什么等等。这些是规划器（planner）的工作，我们稍后会介绍。

For example, let's say the parser is given the following SQL query:

举个例子，假设解析器收到以下 SQL 查询：

```sql
SELECT name, price, price * 25 / 100 AS vat
FROM products JOIN categories ON products.category_id = categories.id
WHERE categories.code = 'BLURAY' AND stock > 0
ORDER BY price DESC
LIMIT 10
```

It will generate a structure that looks something like this (in simplified syntax):

它会生成一个大致如下所示的结构（简化语法）：

```rust
// A SELECT statement.
Statement::Select {
    // SELECT name, price, price * 25 / 100 AS vat
    select: [
        (Column("name"), None),
        (Column("price"), None),
        (
            Divide(
                Multiply(Column("price"), Integer(25)),
                Integer(100)
            ),
            Some("vat"),
        ),
    ]

    // FROM products JOIN categories ON products.category_id = categories.id
    from: [
        Join {
            left: Table("products"),
            right: Table("categories"),
            type: Inner,
            predicate: Some(
                Equal(
                    Column("products.category_id)",
                    Column("categories.id"),
                )
            )
        }
    ]

    // WHERE categories.code = 'BLURAY' AND stock > 0
    where: Some(
        And(
            Equal(
                Column("categories.code"),
                String("BLURAY"),
            ),
            GreaterThan(
                Column("stock"),
                Integer(0),
            )
        )
    )

    // ORDER BY price DESC
    order: [
        (Column("price"), Descending),
    ]

    // LIMIT 10
    limit: Some(Integer(10))
}
```

Let's have a look at how this happens.

我们来看看这一切是如何发生的。

## Lexer（词法分析器）

We begin with the `sql::parser::Lexer`, which takes the raw SQL string and performs
[lexical analysis](https://en.wikipedia.org/wiki/Lexical_analysis) to convert it into a sequence of
tokens. These tokens are things like number, string, identifier, SQL keyword, and so on.

我们先从 `sql::parser::Lexer` 开始，它接收原始 SQL 字符串并进行[词法分析](https://en.wikipedia.org/wiki/Lexical_analysis)，将其转换为一系列记号（token）。这些记号包括数字、字符串、标识符、SQL 关键字等等。

This preprocessing is useful to deal with some of the "noise" of SQL text, such as whitespace,
string quotes, identifier normalization, and so on. It also specifies which symbols and keywords are
valid in our SQL queries. This makes the parser's life a lot easier.

这种预处理有助于处理 SQL 文本中的各种“噪声”，例如空白字符、字符串引号、标识符规范化等。它还规定了哪些符号和关键字在我们的 SQL 查询中是合法的。这让解析器的工作轻松了许多。

The lexer doesn't care about SQL structure at all, only that the individual pieces (tokens) of a
string are well-formed. For example, the following input string:

词法分析器完全不关心 SQL 结构，只关心字符串的各个片段（记号）本身是否格式良好。例如，对于以下输入字符串：

```
'foo' ) 3.14 SELECT + x
```

Will result in these tokens:

会产生这些记号：

```
String("foo") CloseParen Number("3.14") Keyword(Select) Plus Ident("x")
```

Tokens and keywords are represented by the `sql::parser::Token` and `sql::parser::Keyword` enums
respectively:

记号和关键字分别由 `sql::parser::Token` 和 `sql::parser::Keyword` 枚举表示：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/lexer.rs#L8-L47>

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/lexer.rs#L86-L155>

The lexer takes an input string and emits tokens as an iterator:

词法分析器接收一个输入字符串，并以迭代器的形式产出记号：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/lexer.rs#L311-L337>

It does this by repeatedly attempting to scan the next token until it reaches the end of the string
(or errors). It can determine the kind of token by looking at the first character:

它通过反复尝试扫描下一个记号来实现，直到到达字符串末尾（或出错）。它可以通过查看第一个字符来确定记号的类型：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/lexer.rs#L358-L373>

And then scan across the following characters as appropriate to generate a valid token. For example,
this is how a quoted string (e.g. `'foo'`) is lexed into a `Token::String` (including handling of
any escaped quotes inside the string):

然后根据需要扫描后续字符，生成合法的记号。例如，下面是带引号的字符串（如 `'foo'`）如何被词法分析为 `Token::String` 的过程（包括处理字符串内部转义的引号）：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/lexer.rs#L435-L451>

These tokens become the input to the parser.

这些记号随后成为解析器的输入。

## Abstract Syntax Tree（抽象语法树）

The end result of the parsing process will be an [abstract syntax tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree)
(AST), which is a structured representation of a SQL statement, located in the
[`sql::parser::ast`](https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/ast.rs) module.

解析过程的最终产物是一棵[抽象语法树](https://en.wikipedia.org/wiki/Abstract_syntax_tree)（AST），它是 SQL 语句的结构化表示，位于 [`sql::parser::ast`](https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/ast.rs) 模块中。

The root of this tree is the `sql::parser::ast::Statement` enum, which represents all the different
kinds of SQL statements that we support, along with their contents:

这棵树的根是 `sql::parser::ast::Statement` 枚举，它表示我们支持的所有不同种类的 SQL 语句及其内容：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/ast.rs#L6-L145>

The nested tree structure is particularly apparent with expressions, which represent values and
operations on them. For example, the expression `2 * 3 - 4 / 2`, which evaluates to the value `4`.

嵌套的树结构在表达式上体现得尤为明显，表达式表示值以及施加在值上的运算。例如，表达式 `2 * 3 - 4 / 2`，其求值结果为 `4`。

We've seen in the data model section how such expressions are represented as
`sql::types::Expression`, but before we get there we have to parse them. The parser has its own
representation `sql::parser::ast::Expression` -- this is necessary e.g. because in the AST, we
represent columns as names rather than numeric indexes (we don't know yet which columns exist or
what their names are, we'll get to that during planning).

在数据模型一节中我们已经看到这类表达式如何表示为 `sql::types::Expression`，但在那之前我们必须先解析它们。解析器有自己的表示形式 `sql::parser::ast::Expression`——这是必要的，例如在 AST 中，我们把列表示为名字而不是数字索引（此时我们还不知道存在哪些列、它们叫什么名字，这要等到规划阶段才能确定）。

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/ast.rs#L147-L170>

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/ast.rs#L204-L234>

For example, `2 * 3 - 4 / 2` is represented as:

例如，`2 * 3 - 4 / 2` 表示为：

```rust
Expression::Operator(Operator::Subtract(
    // The left-hand operand of -
    Expression::Operator(Operator::Multiply(
        // The left-hand operand of *
        Expression::Literal(Literal::Integer(2)),
        // The right-hand operand of *
        Expression::Literal(Literal::Integer(3)),
    )),
    // The right-hand operand of -
    Expression::Operator(Operator::Divide(
        // The left-hand operand of /
        Expression::Literal(Literal::Integer(4)),
        // The right-hand operand of /
        Expression::Literal(Literal::Integer(2)),
    )),
))
```

## Parser（解析器）

The parser, `sql::parser::Parser`, takes lexer tokens as input and builds an `ast::Statement`
from them:

解析器 `sql::parser::Parser` 以词法分析器产出的记号为输入，并据此构建 `ast::Statement`：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L9-L32>

We can determine the kind of statement we're parsing simply by looking at the first keyword:

只需查看第一个关键字，我们就能确定正在解析的语句类型：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L109-L130>

Let's see how a `SELECT` statement is parsed. The different clauses in a `SELECT` (e.g. `FROM`,
`WHERE`, etc.) must always be given in a specific order, and they always begin with the appropriate
keyword, so we can simply try to parse each clause in the expected order:

我们来看看 `SELECT` 语句是如何解析的。`SELECT` 中的各个子句（例如 `FROM`、`WHERE` 等）必须按特定顺序出现，而且总是以相应的关键字开头，所以我们只需按预期的顺序依次尝试解析每个子句即可：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L330-L342>

Parsing each clause is also just a matter of parsing the expected parts in order. For example, the
initial `SELECT` clause is just a comma-separated list of expressions with an optional alias:

解析每个子句也只是按顺序解析各个预期部分而已。例如，开头的 `SELECT` 子句就是一个逗号分隔的表达式列表，每个表达式可以带一个可选的别名：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L344-L365>

The `FROM` clause is a comma-separated list of table name, optionally joined with other tables:

`FROM` 子句是一个逗号分隔的表名列表，表之间可以可选地相互连接（join）：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L367-L427>

And the `WHERE` clause is just a predicate expression to filter by:

而 `WHERE` 子句只是一个用于过滤的谓词表达式：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L429-L435>

Expression parsing is where this gets tricky, because we have to respect the rules of operator
precedence and associativity. For example, according to mathematical order of operations (aka
"PEMDAS") the expression `2 * 3 - 4 / 2` must be parsed as `(2 * 3) - (4 / 2)` which yields 4, not
`2 * (3 - 4) / 2` which yields -1.

表达式解析是其中比较棘手的部分，因为我们必须遵守运算符优先级和结合性的规则。例如，按照数学运算顺序（即“先乘除后加减”的 PEMDAS 规则），表达式 `2 * 3 - 4 / 2` 必须解析为 `(2 * 3) - (4 / 2)`，结果为 4，而不是解析为 `2 * (3 - 4) / 2`，那样结果是 -1。

toyDB does this using the [precedence climbing algorithm](https://en.wikipedia.org/wiki/Operator-precedence_parser#Precedence_climbing_method),
which is a fairly simple and compact algorithm as far as these things go. In a nutshell, it will
greedily and recursively group operators together as long as their precedence is the same or higher
than that of the operators preceding them (hence "precedence climbing"). For example:

toyDB 使用[优先级爬升算法](https://en.wikipedia.org/wiki/Operator-precedence_parser#Precedence_climbing_method)来实现这一点，就这类算法而言，它相当简单紧凑。简而言之，只要当前运算符的优先级不低于其前面的运算符，它就会贪心且递归地把这些运算符组合在一起（因此得名“优先级爬升”）。例如：

```
-----   ----- Precedence 2: * and /
------------- Precedence 1: -
2 * 3 - 4 / 2
```

The algorithm is documented in more detail on `Parser::parse_expression()`:

该算法在 `Parser::parse_expression()` 中有更详细的文档说明：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/parser/parser.rs#L501-L696>

---

<p align="center">
← <a href="sql-raft.md">SQL Raft Replication</a> &nbsp; | &nbsp; <a href="sql-planner.md">SQL Planning</a> →
</p>
