# Storage Engine（存储引擎）

toyDB uses an embedded [key/value store](https://en.wikipedia.org/wiki/Key–value_database) for data
storage, located in the [`storage`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/storage)
module. This stores arbitrary keys and values as binary byte strings. The storage engine doesn't
know or care what the keys and values contain -- we'll see later how the SQL data model, with tables
and rows, is mapped onto this key/value structure.

toyDB 使用一个嵌入式[键/值存储](https://en.wikipedia.org/wiki/Key–value_database)来做数据存储，位于 [`storage`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/storage) 模块中。它将任意的键和值存储为二进制字节串。存储引擎不知道也不关心键和值的内容是什么——稍后我们会看到，包含表和行的 SQL 数据模型是如何映射到这种键/值结构上的。

The storage engine supports simple set/get/delete operations on individual keys. It does not itself
support transactions -- this is built on top, and we'll get back to it shortly.

存储引擎支持对单个键进行简单的 set/get/delete 操作。它本身不支持事务——事务构建在其之上，我们很快会回到这个话题。

Keys are stored in sorted order. This allows range scans, where we can iterate over all key/value
pairs between two specific keys, or with a specific key prefix. This will be needed by other
components in the system, e.g. to scan all rows in a specific SQL table, to scan all versions of an
MVCC key, to scan the tail of the Raft log, etc.

键按排序顺序存储。这使得范围扫描（range scan）成为可能：我们可以遍历两个特定键之间的所有键/值对，或者遍历具有特定键前缀的所有键/值对。系统中的其他组件会用到这一点，例如扫描某个 SQL 表中的所有行、扫描某个 MVCC 键的所有版本、扫描 Raft 日志的末尾等等。

The storage engine is pluggable: there are multiple implementations, and the user can choose which
one to use in the config file. These implement the `storage::Engine` trait:

存储引擎是可插拔的：它有多个实现，用户可以在配置文件中选择使用哪一个。它们都实现了 `storage::Engine` trait：

<https://github.com/erikgrinaker/toydb/blob/4804df254034c51f367d1380d389d80695cd7054/src/storage/engine.rs#L8-L58>

Let's look at the existing storage engine implementations.

下面我们来看看现有的存储引擎实现。

## `Memory` Storage Engine（`Memory` 存储引擎）

The simplest storage engine is the `storage::Memory` engine. This is a trivial implementation which
stores data in memory using the Rust standard library's
[`BTreeMap`](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html), without persisting
it to disk. It is primarily used for testing.

最简单的存储引擎是 `storage::Memory`。这是一个非常简单的实现，使用 Rust 标准库的 [`BTreeMap`](https://doc.rust-lang.org/std/collections/struct.BTreeMap.html) 将数据存储在内存中，不会持久化到磁盘。它主要用于测试。

Since this is just a wrapper around the `BTreeMap` we can include it in its entirety here:

由于它只是 `BTreeMap` 的一个封装，我们可以在这里完整地展示它：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/memory.rs#L8-L77>

## `BitCask` Storage Engine（`BitCask` 存储引擎）

The main storage engine is `storage::BitCask`. This is a very simple variant of
[BitCask](https://riak.com/assets/bitcask-intro.pdf), used in the [Riak](https://riak.com/)
database. It is kind of like the [LSM-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree)'s
baby cousin.

主要的存储引擎是 `storage::BitCask`。这是 [BitCask](https://riak.com/assets/bitcask-intro.pdf) 的一个非常简单的变体，BitCask 用于 [Riak](https://riak.com/) 数据库。它有点像 [LSM-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) 的远房小表弟。

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L15-L55>

toyDB's BitCask implementation uses a single append-only log file for storage. To write a key/value
pair, we simply append it to the file. To delete a key, we append a special tombstone value. When
reading a key, the last entry for that key in the file is used.

toyDB 的 BitCask 实现使用单个只追加（append-only）日志文件来存储数据。写入一个键/值对时，我们只需把它追加到文件末尾；删除一个键时，我们追加一个特殊的墓碑（tombstone）值；读取一个键时，使用文件中该键的最后一个条目。

The file format for a key/value pair is simply:

一个键/值对的文件格式非常简单：

1. The key length, as a big-endian `u32` (4 bytes).
   键长度，以大端序 `u32` 表示（4 字节）。
2. The value length, as a big-endian `i32` (4 bytes). -1 if tombstone.
   值长度，以大端序 `i32` 表示（4 字节）；如果是墓碑则为 -1。
3. The binary key (n bytes).
   二进制键（n 字节）。
4. The binary value (n bytes).
   二进制值（n 字节）。

For example, the key/value pair `foo=bar` would be written as follows (in hexadecimal):

例如，键/值对 `foo=bar` 会按如下方式写入（十六进制表示）：

```
keylen   valuelen key    value
00000003 00000003 666f6f 626172
```

Because the data file is a simple log, we don't need a separate [write-ahead log](https://en.wikipedia.org/wiki/Write-ahead_logging)
for crash recovery -- the data file _is_ the write-ahead log.

由于数据文件本身就是一条简单的日志，我们不需要单独的[预写日志（write-ahead log）](https://en.wikipedia.org/wiki/Write-ahead_logging)来做崩溃恢复——数据文件_本身_就是预写日志。

To quickly look up key/value pairs when reading, we maintain an in-memory `KeyDir` index which maps
a key to the latest value's position in the file. All keys must therefore fit in memory.

为了在读取时能快速查找键/值对，我们维护一个内存中的 `KeyDir` 索引，它将键映射到最新值在文件中的位置。因此，所有的键都必须能放进内存。

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L57-L65>

We initially generate this index by scanning through the entire file when it is opened:

我们最初在打开文件时通过扫描整个文件来生成这个索引：

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L267-L332>

To write a key, we append it to the file and update the `KeyDir`:

写入一个键时，我们把它追加到文件中并更新 `KeyDir`：

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L155-L159>

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L342-L366>

To delete a key, we append a tombstone value instead:

删除一个键时，我们改为追加一个墓碑值：

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L122-L126>

To read a value for a key, we look up the key's file location in the `KeyDir` index (if the key
exists), and then read it from the file:

读取某个键的值时，我们先在 `KeyDir` 索引中查找该键在文件中的位置（如果键存在），然后从文件中读取它：

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L334-L340>

The `KeyDir` uses an inner stdlib `BTreeMap` to keep track of keys. This allows range scans, where
we iterate over a sorted set of keys between the range bounds, loading each key from the file:

`KeyDir` 内部使用标准库的 `BTreeMap` 来跟踪键。这使得范围扫描成为可能：我们可以遍历范围边界之间的有序键集合，并从文件中加载每个键：

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L144-L146>

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L207-L225>

As keys are updated and deleted, we'll keep accumulating old versions in the log file. To remove
these, the log file is compacted on startup. This writes out the latest value of every live
key/value pair to a new file, and replaces the old file. The keys are written in sorted order, to
make later scans faster.

随着键被更新和删除，日志文件中会不断积累旧版本。为了清除这些旧版本，日志文件会在启动时进行压缩（compaction）：将每个存活键/值对的最新值写入一个新文件，并用它替换旧文件。键按排序顺序写入，以便让后续的扫描更快。

<https://github.com/erikgrinaker/toydb/blob/3e467512dca55843f0b071b3e239f14724f59a41/src/storage/bitcask.rs#L172-L195>

---

<p align="center">
← <a href="overview.md">Overview</a> &nbsp; | &nbsp; <a href="encoding.md">Key/Value Encoding</a> →
</p>
