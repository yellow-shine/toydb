# SQL Examples（SQL 示例）

The following examples demonstrate some of toyDB's SQL features. For more details, see the
[SQL reference](sql.md).

以下示例演示了 toyDB 的一部分 SQL 功能。更多细节请参阅 [SQL 参考手册](sql.md)。

- [Setup](#setup)
- [Creating Tables and Data](#creating-tables-and-data)
- [Constraints and Referential Integrity](#constraints-and-referential-integrity)
- [Basic SQL Queries](#basic-sql-queries)
- [Expressions](#expressions)
- [Joins](#joins)
- [Explain](#explain)
- [Aggregates](#aggregates)
- [Transactions](#transactions)
- [Time-Travel Queries](#time-travel-queries)

## Setup（环境搭建）

To start a five-node cluster on the local machine (requires a working
[Rust compiler](https://www.rust-lang.org/tools/install)), run:

在本地机器上启动一个五节点集群（需要可用的 [Rust 编译器](https://www.rust-lang.org/tools/install)），运行：

```
$ ./cluster/run.sh
toydb2 19:06:28 [ INFO] Listening on 0.0.0.0:9602 (SQL) and 0.0.0.0:9702 (Raft)
toydb2 19:06:28 [ERROR] Failed connecting to Raft peer 127.0.0.1:9705: Connection refused
toydb5 19:06:28 [ INFO] Listening on 0.0.0.0:9605 (SQL) and 0.0.0.0:9705 (Raft)
[...]
toydb5 19:06:29 [ INFO] Voting for toydb-d in term 1 election
toydb3 19:06:29 [ INFO] Voting for toydb-d in term 1 election
toydb4 19:06:29 [ INFO] Won election for term 1, becoming leader
```

In a separate terminal, start a `toysql` client and check the server status:

在另一个终端中，启动一个 `toysql` 客户端并查看服务器状态：

```
$ cargo run --release --bin toysql
Connected to toyDB node "toydb-a". Enter !help for instructions.
toydb> !status

Server:    5 (leader 4 in term 1 with 5 nodes)
Raft log:  1 committed, 0 applied, 0.000 MB (hybrid storage)
Node logs: 1:1 2:1 3:1 4:1 5:1
SQL txns:  0 active, 0 total (bitcask storage)
```

The cluster is shut down by pressing Ctrl-C. Data is saved under `clusters/toydb-?/data/`,
delete the contents to start over.

按 Ctrl-C 即可关闭集群。数据保存在 `clusters/toydb-?/data/` 目录下，删除其中的内容即可重新开始。

## Creating Tables and Data（创建表和数据）

As a basis for later examples, we'll create a small movie database. The following SQL statements
can be pasted into `toysql`:

作为后续示例的基础，我们先创建一个小型电影数据库。可以把下面的 SQL 语句粘贴到 `toysql` 中执行：

```sql
CREATE TABLE genres (
    id INTEGER PRIMARY KEY,
    name STRING NOT NULL
);
INSERT INTO genres VALUES
    (1, 'Science Fiction'),
    (2, 'Action'),
    (3, 'Drama'),
    (4, 'Comedy');

CREATE TABLE studios (
    id INTEGER PRIMARY KEY,
    name STRING NOT NULL
);
INSERT INTO studios VALUES
    (1, 'Mosfilm'),
    (2, 'Lionsgate'),
    (3, 'StudioCanal'),
    (4, 'Warner Bros'),
    (5, 'Focus Features');

CREATE TABLE movies (
    id INTEGER PRIMARY KEY,
    title STRING NOT NULL,
    studio_id INTEGER NOT NULL INDEX REFERENCES studios,
    genre_id INTEGER NOT NULL INDEX REFERENCES genres,
    released INTEGER NOT NULL,
    rating FLOAT
);
INSERT INTO movies VALUES
    (1,  'Stalker',             1, 1, 1979, 8.2),
    (2,  'Sicario',             2, 2, 2015, 7.6),
    (3,  'Primer',              3, 1, 2004, 6.9),
    (4,  'Heat',                4, 2, 1995, 8.2),
    (5,  'The Fountain',        4, 1, 2006, 7.2),
    (6,  'Solaris',             1, 1, 1972, 8.1),
    (7,  'Gravity',             4, 1, 2013, 7.7),
    (8,  '21 Grams',            5, 3, 2003, 7.7),
    (9,  'Birdman',             4, 4, 2014, 7.7),
    (10, 'Inception',           4, 1, 2010, 8.8),
    (11, 'Lost in Translation', 5, 4, 2003, 7.7),
    (12, 'Eternal Sunshine of the Spotless Mind', 5, 3, 2004, 8.3);
```

toyDB supports some basic datatypes, as well as primary keys, foreign keys, and column indexes.
For more information on these, see the [SQL reference](sql.md). Schema changes such as
`ALTER TABLE` are not supported, only `CREATE TABLE` and `DROP TABLE`.

toyDB 支持一些基本数据类型，以及主键、外键和列索引。更多信息请参阅 [SQL 参考手册](sql.md)。不支持 `ALTER TABLE` 之类的模式变更，只支持 `CREATE TABLE` 和 `DROP TABLE`。

The tables can be inspected via the `!tables` and `!table` commands:

可以通过 `!tables` 和 `!table` 命令查看表：

```sql
toydb> !tables
genres
movies
studios

toydb> !table genres
CREATE TABLE genres (
  id INTEGER PRIMARY KEY,
  name STRING NOT NULL
)
```

## Constraints and Referential Integrity（约束与引用完整性）

Schemas enforce referential integrity and other constraints:

模式（schema）会强制执行引用完整性及其他约束：

```sql
toydb> DROP TABLE studios;
Error: Table studios is referenced by table movies column studio_id

toydb> DELETE FROM studios WHERE id = 1;
Error: Primary key 1 is referenced by table movies column studio_id

toydb> UPDATE movies SET id = 1;
Error: Primary key 1 already exists for table movies

toydb> INSERT INTO movies VALUES (13, 'Nebraska', 6, 3, 2013, 7.7);
Error: Referenced primary key 6 in table studios does not exist

toydb> INSERT INTO movies VALUES (13, 'Nebraska', NULL, 3, 2013, 7.7);
Error: NULL value not allowed for column studio_id

toydb> INSERT INTO movies VALUES (13, 'Nebraska', 'Unknown', 3, 2013, 7.7);
Error: Invalid datatype STRING for INTEGER column studio_id
```

## Basic SQL Queries（基本 SQL 查询）

Most basic SQL query functionality is supported:

支持大部分基本的 SQL 查询功能：

```sql
toydb> SELECT * FROM studios;
1|Mosfilm
2|Lionsgate
3|StudioCanal
4|Warner Bros
5|Focus Features

toydb> SELECT title, rating FROM movies WHERE released >= 2000 ORDER BY rating DESC LIMIT 3;
Inception|8.8
Eternal Sunshine of the Spotless Mind|8.3
Gravity|7.7
```

Column headers can be enabled with `!headers on`:

可以用 `!headers on` 开启列标题显示：

```sql
toydb> !headers on
Headers enabled

toydb> SELECT id, name AS genre FROM genres;
id|genre
1|Science Fiction
2|Action
3|Drama
4|Comedy
```

## Expressions（表达式）

All common mathematical operators are implemented:

所有常见的数学运算符都已实现：

```sql
toydb> SELECT 1 + 2 * 3;
7

toydb> SELECT (1 + 2) * 4 / -3;
-4

SELECT 3! + 7 % 4 - 2 ^ 3;
1
```

64-bit floating point arithmetic is also supported, including infinity and NaN:

也支持 64 位浮点运算，包括无穷大（infinity）和 NaN：

```sql
toydb> SELECT 3.14 * 2.718;
8.53452

toydb> SELECT 1.0 / 0.0;
inf

toydb> SELECT 1e10 ^ 8;
100000000000000000000000000000000000000000000000000000000000000000000000000000000

toydb> SELECT 1e10 ^ 8 / INFINITY, 1e10 ^ 1e10, INFINITY / INFINITY;
0|inf|NaN
```

And of course three-valued logic:

当然还有三值逻辑：

```sql
toydb> SELECT TRUE AND TRUE, TRUE AND FALSE, TRUE AND NULL, FALSE AND NULL;
TRUE|FALSE|NULL|FALSE

toydb> SELECT TRUE OR FALSE, FALSE OR FALSE, TRUE OR NULL, FALSE OR NULL;
TRUE|FALSE|TRUE|NULL

toydb> SELECT NOT TRUE, NOT FALSE, NOT NULL;
FALSE|TRUE|NULL
```

Which would be useless without comparison operators for all types:

如果没有针对所有类型的比较运算符，这些就毫无用处：

```sql
toydb> SELECT 3 > 1, 3 <= 1, 3 = 3.0;
TRUE|FALSE|TRUE

toydb> SELECT 'a' = 'A', 'foo' > 'bar', '👍' != '👎';
FALSE|TRUE|TRUE

toydb> SELECT INFINITY > -INFINITY, NULL = NULL;
TRUE|NULL
```

## Joins（连接）

No SQL database would be complete without joins, and toyDB supports most join types such as
inner joins (both implicit and explicit):

没有连接（join）的 SQL 数据库是不完整的。toyDB 支持大多数连接类型，例如内连接（隐式和显式均可）：

```sql
toydb> SELECT m.id, m.title, g.name FROM movies m JOIN genres g ON m.genre_id = g.id LIMIT 4;
1|Stalker|Science Fiction
2|Sicario|Action
3|Primer|Science Fiction
4|Heat|Action

toydb> SELECT m.id, m.title, g.name FROM movies m, genres g WHERE m.genre_id = g.id LIMIT 4;
1|Stalker|Science Fiction
2|Sicario|Action
3|Primer|Science Fiction
4|Heat|Action
```

Left and right outer joins:

左外连接和右外连接：

```sql
toydb> SELECT s.id, s.name, g.name FROM studios s LEFT JOIN genres g ON s.id = g.id;
1|Mosfilm|Science Fiction
2|Lionsgate|Action
3|StudioCanal|Drama
4|Warner Bros|Comedy
5|Focus Features|NULL

toydb> SELECT g.id, g.name, s.name FROM genres g RIGHT JOIN studios s ON g.id = s.id;
1|Science Fiction|Mosfilm
2|Action|Lionsgate
3|Drama|StudioCanal
4|Comedy|Warner Bros
NULL|NULL|Focus Features
```

And cross joins (both implicit and explicit):

以及交叉连接（隐式和显式均可）：

```sql
toydb> SELECT g.name, s.name FROM genres g, studios s WHERE s.name < 'S';
Science Fiction|Mosfilm
Science Fiction|Lionsgate
Science Fiction|Focus Features
Action|Mosfilm
Action|Lionsgate
Action|Focus Features
Drama|Mosfilm
Drama|Lionsgate
Drama|Focus Features
Comedy|Mosfilm
Comedy|Lionsgate
Comedy|Focus Features
```

We can join on arbitrary predicates, such as joining movies with any genres whose name is
ordered after the movie's title:

我们可以基于任意谓词进行连接，例如将电影与那些名称按字典序排在电影标题之后的类型相连接：

```sql
toydb>  SELECT   m.title, g.name
        FROM     movies m JOIN genres g ON g.name > m.title
        ORDER BY m.title, g.name;

21 Grams|Action
21 Grams|Comedy
21 Grams|Drama
21 Grams|Science Fiction
Birdman|Comedy
Birdman|Drama
Birdman|Science Fiction
Eternal Sunshine of the Spotless Mind|Science Fiction
Gravity|Science Fiction
Heat|Science Fiction
Inception|Science Fiction
Lost in Translation|Science Fiction
Primer|Science Fiction
```

And we can join multiple tables, even using the same table multiple times - like in this example
where we find all science fiction movies released since 2000 by studios that have released any
movie rated 8 or higher:

我们还可以连接多张表，甚至多次使用同一张表——比如这个例子：查找 2000 年以来上映的科幻电影，且其出品公司曾发行过任何评分达到 8 分或以上的电影：

```sql
toydb> SELECT   m.id, m.title, g.name AS genre, m.released, s.name AS studio
       FROM     movies m JOIN genres g ON m.genre_id = g.id,
                studios s JOIN movies good ON good.studio_id = s.id AND good.rating >= 8
       WHERE    m.studio_id = s.id AND m.released >= 2000 AND g.id = 1
       ORDER BY m.title ASC;

7|Gravity|Science Fiction|2013|Warner Bros
10|Inception|Science Fiction|2010|Warner Bros
5|The Fountain|Science Fiction|2006|Warner Bros
```

## Explain（执行计划）

When optimizing complex queries with several joins, it can often be useful to inspect the query
plan via an `EXPLAIN` query:

在优化包含多个连接的复杂查询时，通过 `EXPLAIN` 查询查看查询计划往往很有用：

```sql
toydb> EXPLAIN
       SELECT   m.id, m.title, g.name AS genre, m.released, s.name AS studio
       FROM     movies m JOIN genres g ON m.genre_id = g.id,
                studios s JOIN movies good ON good.studio_id = s.id AND good.rating >= 8
       WHERE    m.studio_id = s.id AND m.released >= 2000 AND g.id = 1
       ORDER BY m.title ASC;

Order: m.title asc
└─ Projection: m.id, m.title, g.name, m.released, s.name
   └─ HashJoin: inner on m.studio_id = s.id
      ├─ HashJoin: inner on m.genre_id = g.id
      │  ├─ Filter: m.released > 2000 OR m.released = 2000
      │  │  └─ IndexLookup: movies as m column genre_id (1)
      │  └─ KeyLookup: genres as g (1)
      └─ HashJoin: inner on s.id = good.studio_id
         ├─ Scan: studios as s
         └─ Scan: movies as good (good.rating > 8 OR good.rating = 8)
```

Here, we can see that the planner does a primary key lookup on `genres` and an index lookup on
`movies.genre_id`, filtering the resulting movies by release year and joining them. It also
does full table scans of `studios` and `movies` (to find the good movies) and joins them, pusing
the `rating >= 8` filter down to the `movies` table scan. The results of these two joins are also
joined to produce the final result, which is then formatted and sorted.

在这里可以看到，规划器（planner）对 `genres` 做了主键查找，对 `movies.genre_id` 做了索引查找，然后按上映年份过滤所得的电影并进行连接。它还对 `studios` 和 `movies`（用于找出好电影）做了全表扫描并将它们连接起来，同时把 `rating >= 8` 过滤条件下推到 `movies` 的表扫描中。这两个连接的结果再进行一次连接以产生最终结果，最后进行格式化和排序。

## Aggregates（聚合）

Most basic aggregate functions are supported:

支持大部分基本的聚合函数：

```sql
toydb> SELECT COUNT(*), MIN(rating), MAX(rating), AVG(rating), SUM(rating) FROM movies;
12|6.9|8.8|7.841666666666668|94.10000000000001
```

We can group by values and filter the aggregate results:

我们可以按值分组并过滤聚合结果：

```sql
toydb> SELECT s.id, s.name, AVG(m.rating) AS average
       FROM movies m JOIN studios s ON m.studio_id = s.id
       GROUP BY s.id, s.name
       HAVING average > 7.8
       ORDER BY average DESC, s.name ASC;
1|Mosfilm|8.149999999999999
4|Warner Bros|7.919999999999999
5|Focus Features|7.900000000000001
```

And we can combine aggregate functions with arbitrary expressions, both inside and outside:

聚合函数还可以与任意表达式组合，无论在函数内部还是外部：

```sql
toydb> SELECT s.id, s.name, ((MAX(rating^2) - MIN(rating^2)) / AVG(rating^2)) ^ (0.5) AS spread
       FROM movies m JOIN studios s ON m.studio_id = s.id
       GROUP BY s.id, s.name
       HAVING MAX(rating) - MIN(rating) > 0.5
       ORDER BY spread DESC;
4|Warner Bros|0.6373540990222496
5|Focus Features|0.39194971607693424
```

## Transactions（事务）

toyDB supports ACID transactions via MVCC-based snapshot isolation. This provides atomic
transactions with good isolation, without taking out locks or blocking reads on writes. As a basic
example, the below transaction is rolled back without taking effect, as opposed to `COMMIT`
which would make it permanent:

toyDB 通过基于 MVCC 的快照隔离（snapshot isolation）支持 ACID 事务。这提供了具有良好隔离性的原子事务，无需加锁，也不会因写入而阻塞读取。举个基本例子：下面的事务被回滚且未生效；与之相对，`COMMIT` 会使其永久生效：

```sql
toydb> BEGIN;
Began transaction 131

toydb:131> INSERT INTO genres VALUES (5, 'Western');
toydb:131> SELECT * FROM genres;
1|Science Fiction
2|Action
3|Drama
4|Comedy
5|Western
toydb:131> ROLLBACK;
Rolled back transaction 131

toydb> SELECT * FROM genres;
1|Science Fiction
2|Action
3|Drama
4|Comedy
```

We'll demonstrate transactions by covering most common transaction anomalies given two
concurrent sessions, and show how toyDB prevents these anomalies in all cases but one. In these
examples, the left half is user A and the right is user B. Time flows downwards such that
commands on the same line happen at the same time.

下面用两个并发会话演示最常见的事务异常，并展示 toyDB 如何在除一种情况外的所有情况下防止这些异常。在这些示例中，左半边是用户 A，右半边是用户 B。时间自上而下流动，同一行上的命令是同时发生的。

**Dirty write:** an uncommitted write by A should not be affected by a concurrent B write.

**脏写（dirty write）：**A 未提交的写入不应受 B 并发写入的影响。

```sql
a> BEGIN;
a> INSERT INTO genres VALUES (5, 'Western');
                                                   b> INSERT INTO genres VALUES (5, 'Romance');
                                                   Error: Serialization failure, retry transaction
a> SELECT * FROM genres WHERE id = 5;
5|Western
```

The serialization failure here occurs because the first write always wins. This may not be an
optimal strategy, but it is correct in terms of preventing serialization anomalies.

这里出现序列化失败是因为第一个写入总是获胜。这未必是最优策略，但就防止序列化异常而言是正确的。

**Dirty read:** an uncommitted write by A should not be visible to B until committed.

**脏读（dirty read）：**A 未提交的写入在提交之前不应对 B 可见。

```sql
a> BEGIN;
a> INSERT INTO genres VALUES (5, 'Western');
                                                  b> SELECT * FROM genres WHERE id = 5;
                                                  No rows returned
a> COMMIT;
                                                  b> SELECT * FROM genres WHERE id = 5;
                                                  5|Western
```

**Lost update:** when A and B both read a value, before updating it in turn, the first write should
not be overwritten by the second.

**丢失更新（lost update）：**当 A 和 B 先后读取同一个值、再依次更新时，第一个写入不应被第二个写入覆盖。

```sql
a> BEGIN;                                         b> BEGIN;
a> SELECT title, rating FROM movies WHERE id = 2; b> SELECT title, rating FROM movies WHERE id = 2;
Sicario|7.6                                       Sicario|7.6
a> UPDATE movies SET rating = 7.8 WHERE id = 2;
                                                  b> UPDATE movies SET rating = 7.7 WHERE id = 2;
                                                  Error: Serialization failure, retry transaction
a> COMMIT;
```

**Fuzzy read:** B should not see a value suddenly change in its transaction, even if A commits a
new value.

**模糊读（fuzzy read）：**即使 A 提交了新值，B 也不应在其事务中看到某个值突然变化。

```sql
a> BEGIN;                                         b> BEGIN;
                                                  b> SELECT * FROM genres WHERE id = 1;
                                                  1|Science Fiction
a> UPDATE genres SET name = 'Scifi' WHERE id = 1;
a> COMMIT;
                                                  b> SELECT * FROM genres WHERE id = 1;
                                                  1|Science Fiction
                                                  b> COMMIT;

                                                  b> SELECT * FROM genres WHERE id = 1;
                                                  1|Scifi
```

**Read skew:** if A reads two values, and B modifies the second value in between the reads, A
should see the old second value.

**读偏斜（read skew）：**如果 A 读取两个值，而 B 在两次读取之间修改了第二个值，A 应当仍看到旧的第二个值。

```sql
a> BEGIN;
a> SELECT * FROM genres WHERE id = 2;
2|Action
                                                  b> BEGIN;
                                                  b> UPDATE genres SET name = 'Drama' WHERE id = 2;
                                                  b> UPDATE genres SET name = 'Action' WHERE id = 3;
                                                  b> COMMIT;
a> SELECT * FROM genres WHERE id = 3;
3|Drama
```

**Phantom read:** when A runs a query with a predicate, and B commits a matching write, A should
not see the write when rerunning it.

**幻读（phantom read）：**当 A 用谓词执行一次查询，而 B 提交了一条匹配该谓词的写入时，A 重新运行查询时不应看到这条写入。

```sql
a> BEGIN;
a> SELECT * FROM genres WHERE id > 2;
3|Drama
4|Comedy
                                                  b> INSERT INTO genres VALUES (5, 'Western');
a> SELECT * FROM genres WHERE id > 2;
3|Drama
4|Comedy
```

**Write skew:** when A reads row X and writes it to row Y, B should not concurrently be able to
read row Y and write it to row X.

**写偏斜（write skew）：**当 A 读取行 X 并将其写入行 Y 时，B 不应能并发地读取行 Y 并将其写入行 X。

```sql
a> BEGIN;                                         b> BEGIN;
a> SELECT * FROM genres WHERE id = 2;
2|Action
                                                  b> SELECT * FROM genres WHERE id = 3;
                                                  3|Drama
                                                  b> UPDATE genres SET name = 'Drama' WHERE id = 2;
a> UPDATE genres SET name = 'Action' WHERE id = 3;
a> COMMIT;                                        b> COMMIT;
```

Here, the writes actually go through. This anomaly is not protected against by snapshot isolation,
and thus not by toyDB either - doing so would require implementing serializable snapshot isolation.
However, this is the only common serialization anomaly not handled by toyDB, and is not among the
most severe.

在这个例子里，两个写入实际上都成功了。快照隔离并不能防范这种异常，toyDB 也一样——要做到这一点需要实现可串行化快照隔离（serializable snapshot isolation）。不过，这是 toyDB 未处理的唯一一种常见序列化异常，而且也不属于最严重的一类。

## Time-Travel Queries（时间旅行查询）

Since toyDB uses MVCC for transactions and keeps all historical versions, the state of the database
can be queried at any arbitrary point in the past. toyDB uses incremental transaction IDs as
logical timestamps:

由于 toyDB 的事务使用 MVCC 并保留所有历史版本，因此可以查询数据库在过去任意时刻的状态。toyDB 使用递增的事务 ID 作为逻辑时间戳：

```sql
toydb> SELECT * FROM genres;
1|Science Fiction
2|Drama
3|Action
4|Comedy

toydb> BEGIN;
Began transaction 173
toydb:173> UPDATE genres SET name = 'Scifi' WHERE id = 1;
toydb:173> INSERT INTO genres VALUES (5, 'Western');
toydb:173> COMMIT;
Committed transaction 173

toydb> SELECT * FROM genres;
1|Scifi
2|Drama
3|Action
4|Comedy
5|Western

toydb> BEGIN READ ONLY AS OF SYSTEM TIME 172;
Began read-only transaction 175 in snapshot at version 172
toydb@172> SELECT * FROM genres;
1|Science Fiction
2|Drama
3|Action
4|Comedy
```
