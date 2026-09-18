# SQL Storage（SQL 存储层）

The SQL storage engine, in the [`sql::engine`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/engine)
module, stores tables and rows. toyDB has two SQL storage implementations:

SQL 存储引擎位于 [`sql::engine`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/sql/engine)
模块中，负责存储表和行。toyDB 有两种 SQL 存储实现：

* `sql::engine::Local`: local storage using a `storage::Engine` key/value store.
* `sql::engine::Raft`: Raft-replicated storage, using `Local` on each node below Raft.

* `sql::engine::Local`：本地存储，使用 `storage::Engine` 键/值存储。
* `sql::engine::Raft`：经 Raft 复制的存储，在每个节点上位于 Raft 之下使用 `Local`。

These implement the `sql::engine::Engine` trait, which specifies the SQL storage API. SQL execution
can use either simple local storage or Raft-replicated storage -- toyDB itself always uses the
Raft-replicated engine, but many tests use a local in-memory engine.

它们实现了 `sql::engine::Engine` trait，该 trait 定义了 SQL 存储 API。SQL 执行既可以采用简单的本地存储，也可以采用经 Raft 复制的存储——toyDB 本身始终使用 Raft 复制引擎，但许多测试使用本地内存引擎。

The `sql::engine::Engine` trait is fully transactional, based on the `storage::MVCC` transaction
engine discussed previously. As such, the trait just has a few methods that begin transactions --
the storage logic itself is implemented in the transaction, which we'll cover in next. The trait
also has a `session()` method to start SQL sessions for query execution, which we'll revisit in the
execution section.

`sql::engine::Engine` trait 是完全事务化的，基于前面讨论过的 `storage::MVCC` 事务引擎。因此，该 trait 只有少数几个用于开启事务的方法——存储逻辑本身实现在事务中，我们接下来会讲到。该 trait 还有一个 `session()` 方法，用于为查询执行启动 SQL 会话，我们将在执行章节中再次讨论它。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/engine/engine.rs#L9-L29>

Here, we'll only look at the `Local` engine, and we'll discuss Raft replication afterwards. `Local`
itself is just a thin wrapper around a `storage::MVCC<storage::Engine>` to create transactions:

本节只讨论 `Local` 引擎，之后会讨论 Raft 复制。`Local` 本身只是一个围绕 `storage::MVCC<storage::Engine>` 的薄封装，用来创建事务：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L50-L97>

## Key/Value Representation（键/值表示）

`Local` uses a `storage::Engine` key/value store to store SQL table schemas, table rows, and
secondary index entries. But how do we represent these as keys and values?

`Local` 使用 `storage::Engine` 键/值存储来保存 SQL 表结构（schema）、表行以及二级索引条目。但我们要如何把它们表示成键和值呢？

The keys are represented by the `sql::engine::Key` enum, and encoded using the Keycode encoding
that we've discussed in the encoding section:

键由 `sql::engine::Key` 枚举表示，并使用编码章节中讨论过的 Keycode 编码进行编码：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L15-L31>

The values are encoded using the Bincode encoding, where the value type is given by the key:

值使用 Bincode 编码，值的类型由键决定：

* `Key::Table` → `sql::types::Table` (table schemas)
* `Key::Index` → `BTreeSet<sql::types::Value>` (indexed primary keys)
* `Key::Row` → `sql::types::Row` (table rows)

* `Key::Table` → `sql::types::Table`（表结构）
* `Key::Index` → `BTreeSet<sql::types::Value>`（被索引的主键）
* `Key::Row` → `sql::types::Row`（表行）

Recall that the Keycode encoding will store keys in sorted order. This means that all `Key::Table`
entries come first, then all `Key::Index`, then all `Key::Row`. These are further grouped and
sorted by their fields.

回想一下，Keycode 编码会按键的排序顺序存储键。这意味着所有 `Key::Table` 条目排在最前，其次是所有 `Key::Index`，然后是所有 `Key::Row`。它们还会进一步按各自的字段分组和排序。

For example, consider these SQL tables containing movies and genres, with a secondary index on
`movies.genre_id` for fast lookups of movies with a given genre:

例如，考虑下面这些包含电影（movies）和类型（genres）的 SQL 表，并在 `movies.genre_id` 上建立了二级索引，以便快速查找某个类型的电影：

```sql
CREATE TABLE genres (
    id INTEGER PRIMARY KEY,
    name STRING NOT NULL
);

CREATE TABLE movies (
    id INTEGER PRIMARY KEY,
    title STRING NOT NULL,
    released INTEGER NOT NULL,
    genre_id INTEGER NOT NULL INDEX REFERENCES genres
);

INSERT INTO genres VALUES (1, 'Drama'), (2, 'Action');

INSERT INTO movies VALUES
    (1, 'Sicario', 2015, 2),
    (2, '21 Grams', 2003, 1),
    (3, 'Heat', 1995, 2);
```

This would result in the following illustrated keys and values, in the given order:

这会产生如下所示的键和值，顺序如下：

```
/Table/genres → Table { name: "genres", primary_key: 0, columns: ... }
/Table/movies → Table { name: "movies", primary_key: 0, columns: ... }
/Index/movies/genre_id/Integer(1) → BTreeSet { Integer(2) }
/Index/movies/genre_id/Integer(2) → BTreeSet { Integer(1), Integer(3) }
/Row/genres/Integer(1) → Row { Integer(1), String("Action") }
/Row/genres/Integer(2) → Row { Integer(2), String("Drama") }
/Row/movies/Integer(1) → Row { Integer(1), String("Sicario"), Integer(2015), Integer(2) }
/Row/movies/Integer(2) → Row { Integer(2), String("21 Grams"), Integer(2003), Integer(1) }
/Row/movies/Integer(3) → Row { Integer(3), String("Heat"), Integer(1995), Integer(2) }
```

Thus, if we want to do a full table scan of the `movies` table, we just do a prefix scan of
`/Row/movies/`. If we want to do a secondary index lookup of all movies with `genre_id = 2`, we
fetch `/Index/movies/genre_id/Integer(2)` and find that movies with `id = {1,3}` have this genre.

因此，如果想对 `movies` 表做全表扫描，只需对 `/Row/movies/` 做前缀扫描即可。如果想通过二级索引查找所有 `genre_id = 2` 的电影，只需取出 `/Index/movies/genre_id/Integer(2)`，即可发现 `id = {1,3}` 的电影属于该类型。

To help with prefix scans, the valid key prefixes are represented as `sql::engine::KeyPrefix`:

为了方便前缀扫描，合法的键前缀由 `sql::engine::KeyPrefix` 表示：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L35-L48>

For a look at the actual on-disk binary storage format, see the test scripts under
[`src/sql/testscripts/writes`](https://github.com/erikgrinaker/toydb/tree/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/testscripts/writes),
which output the logical and raw binary representation of write operations.

如果想了解实际的磁盘二进制存储格式，可以查看 [`src/sql/testscripts/writes`](https://github.com/erikgrinaker/toydb/tree/c2b0f7f1d6cbf6e2cdc09fc0aec7b050e840ec21/src/sql/testscripts/writes) 下的测试脚本，它们会输出写操作的逻辑形式和原始二进制表示。

## Schema Catalog（模式目录）

The `sql::engine::Catalog` trait is used to store table schemas, i.e. `sql::types::Table`. It has a
handful of methods for creating, dropping and fetching tables (recall that toyDB does not support
schema changes). The `Table::name` field is used as a unique table identifier throughout.

`sql::engine::Catalog` trait 用于存储表结构，即 `sql::types::Table`。它提供了少量用于创建、删除和获取表的方法（回想一下，toyDB 不支持修改表结构）。`Table::name` 字段在整个系统中作为表的唯一标识使用。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/engine/engine.rs#L60-L79>

The `Catalog` trait is also fully transactional, as it must be implemented on a transaction via the
`type Transaction: Transaction + Catalog` trait bound on `sql::engine::Engine`.

`Catalog` trait 也是完全事务化的，因为它必须实现到事务上，这是通过 `sql::engine::Engine` 上的 `type Transaction: Transaction + Catalog` trait 约束来保证的。

Creating a table is straightforward: insert a key/value pair with a Keycode-encoded `Key::Table`
for the key and a Bincode-encoded `sql::types::Table` for the value. We first check that the
table doesn't already exist, and validate the table schema using `Table::validate()`.

创建表很简单：插入一个键/值对，键是 Keycode 编码的 `Key::Table`，值是 Bincode 编码的 `sql::types::Table`。我们会先检查该表是否已存在，并使用 `Table::validate()` 校验表结构。

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L340-L347>

Similarly, fetching and listing tables is straightforward: just key/value gets or scans using the
appropriate keys.

类似地，获取和列出表也很直接：只需用相应的键做键/值 get 或扫描即可。

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L390-L399>

Dropping tables is a bit more involved, since we have to perform some validation and also delete the
actual table rows and any secondary index entries, but it's not terribly complicated:

删除表则稍微复杂一些，因为我们需要做一些校验，还要删除实际的表行和所有二级索引条目，不过也不算太复杂：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L349-L388>

## Row Storage and Transactions（行存储与事务）

The workhorse of the SQL storage engine is the `Transaction` trait, which provides
[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations (create, read,
update, delete) on table rows and secondary index entries. For performance (especially with Raft),
it operates on row batches rather than individual rows.

SQL 存储引擎的主力是 `Transaction` trait，它提供对表行和二级索引条目的
[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)（创建、读取、更新、删除）操作。出于性能考虑（尤其是在 Raft 下），它以行批次而非单行的方式进行操作。

<https://github.com/erikgrinaker/toydb/blob/0839215770e31f1e693d5cccf20a68210deaaa3f/src/sql/engine/engine.rs#L31-L58>

The `Local::Transaction` implementation is just a wrapper around an MVCC transaction, and the
commit/rollback methods just call straight through to it:

`Local::Transaction` 的实现只是围绕 MVCC 事务的一层封装，commit/rollback 方法直接透传给它：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L99-L102>

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L182-L192>

To insert new rows into a table, we first have to perform some validation: check that the table
exists and validate the rows against the table schema (including checking for e.g. primary key
conflicts and foreign key references). We then store the rows as a key/value pairs, using a
`Key::Row` with the table name and primary key value. And finally, we update secondary index entries
(if any).

要向表中插入新行，首先要做一些校验：检查表是否存在，并按表结构校验这些行（包括检查主键冲突和外键引用等）。然后把这些行作为键/值对存储，键是携带表名和主键值的 `Key::Row`。最后更新二级索引条目（如果有的话）。

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L252-L268>

Row updates are similar to inserts, but in the case of a primary key change we instead delete the
old row and insert a new one, for simplicity. Secondary index updates also have to update both the
old and new entries.

行更新与插入类似，但如果主键发生了变化，为简单起见，我们直接删除旧行并插入新行。二级索引的更新也必须同时更新旧条目和新条目。

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L296-L337>

Row deletions are also similar: validate that the deletion is safe (e.g. check that there are no
foreign key references to it), then delete the `Key::Row` keys and any secondary index entries:

行删除也类似：先校验删除是安全的（例如检查没有外键引用它），然后删除 `Key::Row` 键以及所有相关的二级索引条目：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L194-L246>

To fetch rows by primary key, we simply call through to key/value gets using the appropriate
`Key::Row`:

要按主键获取行，只需使用相应的 `Key::Row` 直接调用键/值 get：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L248-L250>

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L127-L133>

Similarly, index lookups fetch a `Key::Index` for the indexed value, returning matching primary
keys:

类似地，索引查找会取出被索引值对应的 `Key::Index`，返回匹配的主键：

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L270-L273>

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L115-L125>

Scanning table rows just performs a prefix scan with the appropriate `KeyPrefix::Row`, returning a
row iterator. This can optionally also do row filtering via filter pushdowns, which we'll revisit
when we look at the SQL optimizer.

扫描表行只需使用相应的 `KeyPrefix::Row` 做前缀扫描，返回一个行迭代器。它还可以选择通过谓词下推（filter pushdown）进行行过滤，我们将在讨论 SQL 优化器时再次提到这一点。

<https://github.com/erikgrinaker/toydb/blob/39c6b60afc4c235f19113dc98087176748fa091d/src/sql/engine/local.rs#L275-L294>

And with that, we can now store and retrieve SQL tables and rows on disk. Let's see how to replicate
it across nodes via Raft.

至此，我们已经可以把 SQL 的表和行存储到磁盘并读取出来。接下来看看如何通过 Raft 把它们复制到多个节点上。

---

<p align="center">
← <a href="sql-data.md">SQL Data Model</a> &nbsp; | &nbsp; <a href="sql-raft.md">SQL Raft Replication</a> →
</p>
