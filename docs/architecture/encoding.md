# Key/Value Encoding（键/值编码）

The key/value store uses binary `Vec<u8>` keys and values, so we need an encoding scheme to
translate between in-memory Rust data structures and the on-disk binary data. This is provided by
the [`encoding`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/encoding)
module, with separate schemes for key and value encoding.

键/值存储使用二进制的 `Vec<u8>` 键和值，因此我们需要一种编码方案，在内存中的 Rust 数据结构与磁盘上的二进制数据之间进行转换。这由 [`encoding`](https://github.com/erikgrinaker/toydb/tree/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/encoding) 模块提供，其中键编码和值编码分别采用不同的方案。

## `Bincode` Value Encoding（`Bincode` 值编码）

Values are encoded using [Bincode](https://github.com/bincode-org/bincode), a third-party binary
encoding scheme for Rust. Bincode is convenient because it can easily encode any arbitrary Rust
data type. But we could also have chosen e.g. [JSON](https://en.wikipedia.org/wiki/JSON),
[Protobuf](https://protobuf.dev), [MessagePack](https://msgpack.org/), or any other encoding.

值使用 [Bincode](https://github.com/bincode-org/bincode) 编码，这是 Rust 的一个第三方二进制编码方案。Bincode 很方便，因为它可以轻松编码任意的 Rust 数据类型。当然，我们也可以选择 [JSON](https://en.wikipedia.org/wiki/JSON)、[Protobuf](https://protobuf.dev)、[MessagePack](https://msgpack.org/) 或任何其他编码。

We won't dwell on the actual binary format here, see the [Bincode specification](https://git.sr.ht/~stygianentity/bincode/tree/trunk/item/docs/spec.md)
for details.

这里不深入探讨实际的二进制格式，详情请参见 [Bincode 规范](https://git.sr.ht/~stygianentity/bincode/tree/trunk/item/docs/spec.md)。

To use a consistent configuration for all encoding and decoding, we provide helper functions in
the [`encoding::bincode`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/encoding/bincode.rs)
module which use `bincode::config::standard()`.

为了让所有编码和解码使用一致的配置，我们在 [`encoding::bincode`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/encoding/bincode.rs) 模块中提供了一组辅助函数，它们都使用 `bincode::config::standard()`。

<https://github.com/erikgrinaker/toydb/blob/0ce1fb34349fda043cb9905135f103bceb4395b4/src/encoding/bincode.rs#L15-L27>

Bincode uses the very common [Serde](https://serde.rs) framework for its API. toyDB also provides an
`encoding::Value` helper trait for value types which adds automatic `encode()` and `decode()`
methods:

Bincode 的 API 基于非常常用的 [Serde](https://serde.rs) 框架。toyDB 还为值类型提供了一个 `encoding::Value` 辅助 trait，可以自动添加 `encode()` 和 `decode()` 方法：

<https://github.com/erikgrinaker/toydb/blob/b57ae6502e93ea06df00d94946a7304b7d60b977/src/encoding/mod.rs#L39-L68>

Here's an example of how this can be used to encode and decode an arbitrary `Dog` data type:

下面这个例子展示了如何用它来编码和解码一个任意的 `Dog` 数据类型：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
struct Dog {
    name: String,
    age: u8,
    good_boy: bool,
}

impl encoding::Value for Dog {}

let pluto = Dog { name: "Pluto".into(), age: 4, good_boy: true };
let bytes = pluto.encode();
println!("{bytes:02x?}");

// Outputs [05, 50, 6c, 75, 74, 6f, 04, 01]:
//
// * Length of string "Pluto": 05.
// * String "Pluto": 50 6c 75 74 6f.
// * Age 4: 04.
// * Good boy: 01 (true).

let pluto = Dog::decode(&bytes)?; // gives us back Pluto
```

## `Keycode` Key Encoding（`Keycode` 键编码）

Unlike values, keys can't just use any binary encoding like Bincode. As mentioned in the storage
section, the storage engine sorts data by key to enable range scans. The key encoding must therefore
preserve the [lexicographical order](https://en.wikipedia.org/wiki/Lexicographic_order) of the
encoded values: the binary byte slices must sort in the same order as the original values.

与值不同，键不能随意使用 Bincode 这样的二进制编码。如存储一节所述，存储引擎按 key 排序数据以支持范围扫描。因此键编码必须保持编码后值的[字典序](https://en.wikipedia.org/wiki/Lexicographic_order)：二进制字节切片的排序必须与原始值的排序一致。

As an example of why we can't just use Bincode, consider the strings "house" and "key". These should
be sorted in alphabetical order: "house" before "key". However, Bincode encodes strings prefixed by
their length, so "key" would be sorted before "house" in binary form:

举个为什么不能直接用 Bincode 的例子，考虑字符串 "house" 和 "key"。它们按字母序应当是 "house" 排在 "key" 之前。但 Bincode 编码字符串时会加上长度前缀，因此二进制形式中 "key" 会排在 "house" 之前：

```
03 6b 65 79        ← 3 bytes: key
05 68 6f 75 73 65  ← 5 bytes: house
```

For similar reasons, we can't just encode numbers in their native binary form: the
[little-endian](https://en.wikipedia.org/wiki/Endianness) representation will order very large
numbers before small numbers, and the [sign bit](https://en.wikipedia.org/wiki/Sign_bit) will order
positive numbers before negative numbers. This would violate the ordering of natural numbers.

出于类似的原因，我们也不能直接用数字的原生二进制形式编码：[小端序](https://en.wikipedia.org/wiki/Endianness)表示会把很大的数排在小数之前，而[符号位](https://en.wikipedia.org/wiki/Sign_bit)会把正数排在负数之前。这都会破坏自然数的大小顺序。

We also have to be careful with value sequences, which should be ordered element-wise. For example,
the pair ("a", "xyz") should be ordered before ("ab", "cd"), so we can't just encode the strings
one after the other like "axyz" and "abcd" since that would sort ("ab", "cd") first.

对于值序列也要小心，它们必须按元素逐个比较排序。例如，("a", "xyz") 应排在 ("ab", "cd") 之前，所以不能把字符串直接连起来编码成 "axyz" 和 "abcd"，那样 ("ab", "cd") 会排到前面。

toyDB provides an order-preserving encoding called "Keycode" in the [`encoding::keycode`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/encoding/keycode.rs)
module. Like Bincode, the Keycode encoding is not self-describing: the binary data does not say what
the data type is, the caller must provide a type to decode into. It only supports a handful of
primitive data types, and only needs to order values of the same type.

toyDB 在 [`encoding::keycode`](https://github.com/erikgrinaker/toydb/blob/213e5c02b09f1a3cac6a8bbd0a81773462f367f5/src/encoding/keycode.rs) 模块中提供了一种保序编码，称为 "Keycode"。与 Bincode 一样，Keycode 编码不是自描述的：二进制数据并不说明数据类型是什么，调用者必须提供要解码成的目标类型。它只支持少数几种基本数据类型，且只需要对相同类型的值正确排序。

Keycode is implemented as a [Serde](https://serde.rs) (de)serializer, which requires a lot of
boilerplate code to satisfy the trait, but we'll just focus on the actual encoding. The encoding
scheme is as follows:

Keycode 实现为 [Serde](https://serde.rs) 序列化器/反序列化器，满足 trait 要求需要大量样板代码，这里我们只关注实际的编码方式。编码方案如下：

* `bool`: `00` for `false` and `01` for `true`.

* `bool`：`false` 编码为 `00`，`true` 编码为 `01`。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L113-L117>

* `u64`: the [big-endian](https://en.wikipedia.org/wiki/Endianness) binary encoding.

* `u64`：[大端序](https://en.wikipedia.org/wiki/Endianness)二进制编码。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L157-L161>

* `i64`: the [big-endian](https://en.wikipedia.org/wiki/Endianness) binary encoding, but with the
   sign bit flipped to order negative numbers before positive ones.

* `i64`：[大端序](https://en.wikipedia.org/wiki/Endianness)二进制编码，但翻转了符号位，使负数排在正数之前。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L131-L143>

* `f64`: the [big-endian IEEE 754](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
  binary encoding, but with the sign bit flipped, and all bits flipped for negative numbers, to
  order negative numbers correctly.

* `f64`：[大端序 IEEE 754](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) 二进制编码，但翻转了符号位，且对负数按位取反，以正确排序负数。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L167-L179>

* `Vec<u8>`: terminated by `00 00`, with `00` escaped as `00 ff` to disambiguate it.

* `Vec<u8>`：以 `00 00` 结尾，内部出现的 `00` 转义为 `00 ff` 以消除歧义。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L190-L205>

* `String`: like `Vec<u8>`.

* `String`：与 `Vec<u8>` 相同。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L185-L188>

* `Vec<T>`, `[T]`, `(T,)`: the concatenation of the inner values.

* `Vec<T>`、`[T]`、`(T,)`：内部各值的编码直接拼接。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L295-L307>

* `enum`: the variant's numerical index as a `u8`, then the inner values (if any).

* `enum`：先写变体的序号（一个 `u8`），再写内部值（如果有）。

    <https://github.com/erikgrinaker/toydb/blob/2027641004989355c2162bbd9eeefcc991d6b29b/src/encoding/keycode.rs#L223-L227>

Like `encoding::Value`, there is also an `encoding::Key` helper trait:

与 `encoding::Value` 类似，还有一个 `encoding::Key` 辅助 trait：

<https://github.com/erikgrinaker/toydb/blob/b57ae6502e93ea06df00d94946a7304b7d60b977/src/encoding/mod.rs#L20-L37>

Different kinds of keys are usually represented as enums. For example, if we wanted to store cars
and video games, we could use:

不同种类的键通常用 enum 表示。例如，如果想存储汽车和电子游戏的数据，可以这样：

```rust
#[derive(serde::Serialize, serde::Deserialize)]
enum Key {
    Car(String, String, u64),    // make, model, year
    Game(String, u64, Platform), // name, year, platform
}

#[derive(serde::Serialize, serde::Deserialize)]
enum Platform {
    PC,
    PS5,
    Switch,
    Xbox,
}

impl encoding::Key for Key {}

let returnal = Key::Game("Returnal".into(), 2021, Platform::PS5);
let bytes = returnal.encode();
println!("{bytes:02x?}");

// Outputs [01, 52, 65, 74, 75, 72, 6e, 61, 6c, 00, 00, 00, 00, 00, 00, 00, 00, 07, e5, 01].
//
// * Key::Game: 01
// * Returnal: 52 65 74 75 72 6e 61 6c 00 00
// * 2021: 00 00 00 00 00 00 07 e5
// * Platform::PS5: 01

let returnal = Key::decode(&bytes)?;
```

Because the keys are sorted in element-wise order, this would allow us to e.g. perform a prefix
scan to fetch all platforms which Returnal (2021) was released on, or perform a range scan to fetch
all models of Nissan Altima released between 2010 and 2015.

由于键是按元素逐个排序的，我们可以执行前缀扫描来获取 Returnal (2021) 发布过的所有平台，或者执行范围扫描来获取 2010 到 2015 年间发布的所有 Nissan Altima 车型。

---

<p align="center">
← <a href="storage.md">Storage Engine</a> &nbsp; | &nbsp; <a href="mvcc.md">MVCC Transactions</a> →
</p>
