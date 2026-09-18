# MVCC Transactions（MVCC 事务）

Transactions are groups of reads and writes (e.g. to different keys) that are submitted together as
a single unit. For example, a bank transaction that transfers $100 from account A to account B might
consist of this group of reads and writes:

事务是把一组读取和写入（例如针对不同的 key）作为一个单独的单元一起提交。例如，一笔把 100 美元从账户 A 转到账户 B 的银行转账，可能由下面这组读取和写入组成：

```
a = get(A)
b = get(B)
if a < 100:
    error("insufficient balance")
set(A, a - 100)
set(B, b + 100)
```

toyDB provides [ACID](https://en.wikipedia.org/wiki/ACID) transactions, a set of very strong
guarantees:

toyDB 提供了 [ACID](https://en.wikipedia.org/wiki/ACID) 事务，即一组非常强的保证：

* **Atomicity:** all of the writes take effect as an single, atomic unit, at the same instant, when
  they are _committed_. Other users will never see some of the writes without the others.

* **原子性（Atomicity）：**所有写入在事务被_提交_的那一刻，作为一个单独的原子单元一起生效。其他用户永远不会只看到一部分写入而看不到其余的写入。

* **Consistency:** database constraints are never violated (e.g. referential integrity or uniqueness
  contraints). We'll see how this is implemented later in the SQL execution layer.

* **一致性（Consistency）：**数据库约束（例如引用完整性或唯一性约束）永远不会被违反。我们稍后会在 SQL 执行层看到这是如何实现的。

* **Isolation:** users should appear to have the entire database to themselves, unaffected by other
  simultaneous users. Two transactions may conflict, in which case one has to retry, but if a
  transaction succeeds then the user knows with certainty that the operations were executed without
  interference by anyone else. This eliminates the risk of [race conditions](https://en.wikipedia.org/wiki/Race_condition).
  
* **隔离性（Isolation）：**用户应当看起来独占整个数据库，不受其他同时操作的用户影响。两个事务可能发生冲突，此时其中一个必须重试；但如果一个事务成功了，用户就可以确定这些操作在执行时没有受到任何其他人的干扰。这消除了[竞态条件（race condition）](https://en.wikipedia.org/wiki/Race_condition)的风险。
  
* **Durability:** committed writes are never lost (even if the system crashes).

* **持久性（Durability）：**已提交的写入永远不会丢失（即使系统崩溃）。

To illustrate how transactions work, here's an example MVCC test script where two concurrent users
modify a set of bank accounts (there's many [other test scripts](https://github.com/erikgrinaker/toydb/tree/aa14deb71f650249ce1cab8828ed7bcae2c9206e/src/storage/testscripts/mvcc)
there too):

为了说明事务是如何工作的，下面是一个 MVCC 测试脚本示例，其中两个并发用户修改一组银行账户（那里还有许多[其他测试脚本](https://github.com/erikgrinaker/toydb/tree/aa14deb71f650249ce1cab8828ed7bcae2c9206e/src/storage/testscripts/mvcc)）：

<https://github.com/erikgrinaker/toydb/blob/a73e24b7e77671b9f466e0146323cd69c3e27bdf/src/storage/testscripts/mvcc/bank#L1-L69>

To provide these guarantees, toyDB uses a common technique called
[Multi-Version Concurrency Control](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)
(MVCC). It is implemented at the key/value storage level, in the [`storage::mvcc`](https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs)
module. It uses a `storage::Engine` for actual data storage.

为了提供这些保证，toyDB 使用了一种称为[多版本并发控制（Multi-Version Concurrency Control）](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)（MVCC）的常见技术。它在 key/value 存储层面实现，位于 [`storage::mvcc`](https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs) 模块中。它使用 `storage::Engine` 来进行实际的数据存储。

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L220-L231>

MVCC provides an [isolation level](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels)
called [snapshot isolation](https://en.wikipedia.org/wiki/Snapshot_isolation): a transaction sees a
snapshot of the database as it was when the transaction began. Any later changes are invisible to
it.

MVCC 提供了一种称为[快照隔离（snapshot isolation）](https://en.wikipedia.org/wiki/Snapshot_isolation)的[隔离级别](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels)：事务看到的是事务开始时数据库的一个快照。之后发生的任何变更对它都不可见。

It does this by storing historical versions of key/value pairs. The version number is simply a
number that's incremented for every new transaction:

它通过存储 key/value 对的历史版本来实现这一点。版本号只是一个随着每个新事务递增的数字：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L155-L158>

Each transaction has its own unique version number. When it writes a key/value pair it appends its
version number to the key as `Key::Version(&[u8], Version)` (using the Keycode encoding we've seen
previously). If an old version of the key already exists, it will have a different version number
suffix and therefore be stored as a separate key in the storage engine. Deleted keys are versions
with a special tombstone value.

每个事务都有自己唯一的版本号。当它写入一个 key/value 对时，会把它的版本号追加到 key 上，形成 `Key::Version(&[u8], Version)`（使用我们之前介绍过的 Keycode 编码）。如果该 key 的旧版本已经存在，它会有一个不同的版本号后缀，因此会作为存储引擎中的一个单独的 key 被存储。被删除的 key 则是带有特殊墓碑（tombstone）值的版本。

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L183-L189>

Here's a simple diagram of what a history of versions 1 to 5 of keys `a` to `d` might look like:

下面是一个简单的示意图，展示 key `a` 到 `d` 的版本 1 到 5 的历史可能是什么样子：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L11-L26>

Additionally, we need to keep track of the currently ongoing (uncommitted) transaction versions,
known as the "active set".

此外，我们还需要跟踪当前正在进行中的（未提交的）事务版本，即所谓的"活跃集（active set）"。

With versioning and the active set, we can summarize the MVCC protocol with a few simple rules:

有了版本化和活跃集，我们可以用几条简单的规则来概括 MVCC 协议：

1. When a new transaction begins, it:
    * Obtains the next available version number.
    * Takes a snapshot of the active set (other uncommitted transactions).
    * Adds its version number to the active set.

1. 当一个新事务开始时，它会：
    * 获取下一个可用的版本号。
    * 对活跃集（其他未提交的事务）做一个快照。
    * 把自己的版本号加入活跃集。

1. When the transaction reads a key, it:
    * Returns the latest version of the key at or below its own version.
    * Ignores versions above its own version.
    * Ignores versions in its active set snapshot.

1. 当事务读取一个 key 时，它会：
    * 返回该 key 在不高于自身版本号范围内的最新版本。
    * 忽略高于自身版本的版本。
    * 忽略其活跃集快照中的版本。

1. When the transaction writes a key, it:
    * Looks for a key version above its own version; errors if found.
    * Looks for a key version in its active set snapshot; errors if found.
    * Writes a key/value pair with its own version.

1. 当事务写入一个 key 时，它会：
    * 查找该 key 是否存在高于自身版本的版本；如果找到则报错。
    * 查找该 key 是否存在于其活跃集快照中；如果找到则报错。
    * 用自己的版本号写入一个 key/value 对。

1. When the transaction commits, it:
    * Flushes all writes to disk.
    * Removes itself from the active set.

1. 当事务提交时，它会：
    * 将所有写入刷写到磁盘。
    * 把自己从活跃集中移除。

The magic happens when the transaction removes itself from the active set. This is a single, atomic
operation, and when it completes all of its writes immediately become visible to _new_ transactions.
However, ongoing transactions still won't see these writes, because the version is still in their
active set snapshot or at a later version (hence they are isolated from this transaction).

神奇之处在于事务把自己从活跃集中移除的那一刻。这是一个单独的原子操作，当它完成时，该事务的所有写入会立即对_新_事务变得可见。不过，正在进行中的事务仍然看不到这些写入，因为该版本仍然在它们的活跃集快照中，或者版本号更晚（因此它们与这个事务是隔离的）。

Furthermore, the transaction could see its own uncommitted writes even though noone else could, and
if any writes conflicted with another transaction it would error out and have to retry.

此外，事务能够看到自己的未提交写入，即使其他人都看不到；而且如果任何写入与另一个事务发生了冲突，它会报错并必须重试。

Not only that, this also allows us to do time-travel queries, where we can query the database as it
was at any time in the past: we simply pick a version number to read at.

不仅如此，这也让我们能够执行时间旅行查询（time-travel queries），即查询数据库在过去任意时刻的状态：我们只需选择一个版本号来读取即可。

There are a few more details that we've left out here: transaction rollbacks need to keep track of
the writes and undo them, and read-only queries can avoid allocating new version numbers. We also
don't garbage collect old version, for simplicity. See the module documentation for more details:

这里还有一些省略的细节：事务回滚需要跟踪写入并撤销它们，只读查询可以避免分配新的版本号。为了简单起见，我们也不对旧版本做垃圾回收。更多细节请参见模块文档：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L1-L140>

Let's walk through a simple example with code pointers to get a feel for how this is implemented.
Notice how we don't have to deal with any version numbers when we're using the MVCC API -- this is
an internal MVCC implementation detail.

让我们通过一个带代码指引的简单例子来感受一下这是如何实现的。注意，在使用 MVCC API 时我们完全不必处理任何版本号——这是 MVCC 内部实现的细节。

```rust
// Open a BitCask database in the file "toy.db" with MVCC support.
let path = PathBuf::from("toy.db");
let db = MVCC::new(BitCask::new(path)?);

// Begin a new transaction.
let txn = db.begin()?;

// Read the key "foo", and decode the binary value as a u64 with bincode.
let bytes = txn.get(b"foo")?.expect("foo not found");
let mut value: u64 = bincode::deserialize(&bytes)?;

// Delete "foo".
txn.delete(b"foo")?;

// Add 1 to the value, and write it back to the key "bar".
value += 1;
let bytes = bincode::serialize(&value);
txn.set(b"bar", bytes)?;

// Commit the transaction.
txn.commit()?;
```

First, we begin a new transaction with `MVCC::begin()`, which calls through to
`Transaction::begin()`. This obtains a version number stored in `Key::NextVersion` and increments
it, then takes a snapshot of the active set in `Key::ActiveSet` and adds itself to it:

首先，我们用 `MVCC::begin()` 开始一个新事务，它会调用 `Transaction::begin()`。这一步会获取存储在 `Key::NextVersion` 中的版本号并递增它，然后对 `Key::ActiveSet` 中的活跃集做一个快照，并把自己加入其中：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L368-L391>

This returns a `Transaction` object which provides the main key/value API, with get/set/delete
methods. It keeps track of the main state of the transaction: it's version number and active set.

这会返回一个 `Transaction` 对象，它提供主要的 key/value API，包括 get/set/delete 方法。它跟踪事务的主要状态：它的版本号和活跃集。

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L294-L327>

Next, we call `Transaction::get(b"foo")` to read the value of the key `foo`. This finds the latest
version that's visible to us (ignoring future versions and the active set). Recall that we store
multiple version of each key as `Key::Version(key, version)`. The Keycode encoding ensures that all
versions are stored in sorted order, so we can do a reverse range scan from `Key::Version(b"foo",
self.version)` to  `Key::Version(b"foo", 0)` and return the latest version that's visible to us:

接下来，我们调用 `Transaction::get(b"foo")` 来读取 key `foo` 的值。这一步会找到对我们可见的最新版本（忽略未来版本和活跃集）。回忆一下，我们把每个 key 的多个版本存储为 `Key::Version(key, version)`。Keycode 编码保证所有版本按排序顺序存储，因此我们可以从 `Key::Version(b"foo", self.version)` 反向范围扫描到 `Key::Version(b"foo", 0)`，并返回对我们可见的最新版本：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L564-L581>

We then call `Transaction::delete(b"foo")` and `Transaction::set(b"bar", value)`. Both of these just
call through to the same `Transaction::write_version()` method, but use `Some(value)` for a regular
key/value pair and `None` as a deletion tombstone:

然后，我们调用 `Transaction::delete(b"foo")` 和 `Transaction::set(b"bar", value)`。这两者都只是调用同一个 `Transaction::write_version()` 方法，只是对普通的 key/value 对使用 `Some(value)`，而用 `None` 表示删除墓碑（tombstone）：

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L514-L522>

To write a new version of a key, we first have to check for conflicts by seeing if there's a
version of the key that's invisible to us -- if it is, we conflicted with a concurrent transaction.
We use a range scan for this, like we did in `Transaction::get()`.

要写入一个 key 的新版本，我们首先必须检查冲突，看看是否存在一个对我们不可见的该 key 的版本——如果存在，说明我们与一个并发事务发生了冲突。为此我们使用范围扫描，就像在 `Transaction::get()` 中那样。

If there are no conflicts, we go on to write `Key::Version(b"foo", self.version)` and encode the
value as an `Option<value>` to accomodate the `None` tombstone marker. We also write a
`Key::TxnWrite(version, key)` to keep track of the keys we've written in case we have to roll back.

如果没有冲突，我们继续写入 `Key::Version(b"foo", self.version)`，并把值编码为 `Option<value>` 以容纳 `None` 墓碑标记。我们还会写入一个 `Key::TxnWrite(version, key)`，以跟踪我们写过的 key，以便在需要回滚时使用。

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L524-L562>

Finally, `Transaction::commit()` will make our transaction take effect and become visible. It does
this simply by removing itself from the active set in `Key::ActiveSet`, and also cleaning up its
`Key::TxnWrite` write tracking. As the comment says, we don't actually have to flush to durable
storage here, because the Raft log will provide durability for us -- we'll get back to this later.

最后，`Transaction::commit()` 会让我们的事务生效并变得可见。它只是通过把自己从 `Key::ActiveSet` 中的活跃集移除来实现这一点，同时清理它的 `Key::TxnWrite` 写入跟踪。正如注释所说，我们实际上不必在这里刷写到持久化存储，因为 Raft 日志会为我们提供持久性——我们稍后会回到这个话题。

<https://github.com/erikgrinaker/toydb/blob/8f8eae0dcf70b1a0df2e853b1f6600e0c7075340/src/storage/mvcc.rs#L466-L485>

---

<p align="center">
← <a href="encoding.md">Key/Value Encoding</a> &nbsp; | &nbsp; <a href="raft.md">Raft Consensus</a> →
</p>
