# 第 26 条：警惕特征（`feature`）蔓延

通过使用 Cargo 的 *特征（feature）* 机制，Rust 允许同一套代码有不同的配置，其底层基于条件编译机制。但是，关于 Rust 中的特征，有一些值得关注的点，本章将一一阐述。

## 条件编译

通过使用 [`cfg`][cfg] 或者 [`cfg_attr`][cfg_attr] ，Rust 支持[条件编译][conditional compilation] ，可以让你决定函数、代码行、代码块等内容是否包含在编译后的文件中（相对而言，C/C++ 是基于代码行的预处理器）。这里的“条件”可以是一个像 `test` 这种单纯的名字，也可以是类似 `panic = "abort"` 这种名值对的形式。

注意，名值对的形式下，一个名字的值可以有多个：

```rust
// 构建时设置环境变量 `RUSTFLAGS` 如下：
//   '--cfg myname="a" --cfg myname="b"'
#[cfg(myname = "a")]
println!("cfg(myname = 'a') is set");
#[cfg(myname = "b")]
println!("cfg(myname = 'b') is set");
```

```
cfg(myname = 'a') is set
cfg(myname = 'b') is set
```

除了前述这种显式指定 `feature` 值之外，更常用的配置项是由工具链自动引入的构建时的目标环境，包括；目标操作系统（[`target_os`][target_os]）、 CPU 架构（[`target_arch`][target_arch]）、指针位宽（[`target_pointer_width`][target_pointer_width]）、字节序（[`target_endian`][target_endian]）等，通过构建时的目标平台启用对应的特征来实现代码的可移植性。

另外，标准选项 [`target_has_atomic`][target_has_atomic] 是支持多值的，如果目标平台同时支持 32 位和 64 位架构，则 `[cfg(target_has_atomic = "32")]` 和 `[cfg(target_has_atomic = "64")]` 同时生效。（关于原子性的更多信息，请参考 O'Reilly 出版的，Mara Bos 所著的 《[Rust Atomics and Locks][Rust Atomics and Locks]》一书的第二章。）

## 特征

通过使用基于 `cfg` 的名值对机制，[Cargo][Cargo] 包管理器提供了[*特征（features）*][features] 选择能力：在构建 crate 的时候，可以有选择地启用所需的函数或者对应的 crate。Cargo 将确保对于每个 crate，都使用所配置的 `feature` 值来进行编译。

这是 Cargo 的特有功能：对于 Rust 的编译器而言，`feature` 只是另一个可配置选项。

截至本文成稿时间，检测所启用的特征的最可靠方式就是查看 [*Cargo.toml*][Cargo.toml] 文件。举例说明，下面这段内容实际上包含了 *6 个*特征：

```toml
[features]
default = ["featureA"]
featureA = []
featureB = []
# 启用 `featureAB` 表示同时启用了 `featureA` 和 `featureB`.
featureAB = ["featureA", "featureB"]
schema = []

[dependencies]
rand = { version = "^0.8", optional = true }
hex = "^0.4"
```

可是，上面的例子中，`[features]` 小节只有 5 个特征啊！因此这里有一些细节需要注意。

首先，`[features]` 中的 `default` 是一个特殊的特征名字，它表示默认启用的特征。当然了，在构建时可以通过 `--no-default-features` 参数忽略默认特征，或者在 *Cargo.toml* 中这样写：

```toml
[dependencies]
somecrate = { version = "^0.3", default-features = false }
```

无论如何，`default` 仍然是一个可以在代码中正常使用的特征名字：

```rust
#[cfg(feature = "default")]
println!("This crate was built with the \"default\" feature enabled.");
#[cfg(not(feature = "default"))]
println!("This crate was built with the \"default\" feature disabled.");
```

在示例的 *Cargo.toml* 中，另一个不明显的特征隐藏在 `[dependencies]` 小节：依赖项 `rand` crate 被标记为 `optional = true`，这就使得 `rand` 成为一个特征名字 [^1]。当示例的 crate 使用 `--features rand` 编译的时候，`rand` 特征将被激活：

```rust
#[cfg(feature = "rand")]
pub fn pick_a_number() -> u8 {
    rand::random::<u8>()
}

#[cfg(not(feature = "rand"))]
pub fn pick_a_number() -> u8 {
    4 // chosen by fair dice roll.
}
```

虽然 crate 名字是全局的（由 `crates.io` 管理），而特征名字是本地的，但事实上 *crate 和特征共享命名空间* 。因此，**谨慎选择特征名字**，以避免和可能作为特征存在的 crate 名字冲突。虽然 Cargo 支持通过修改 `package` 键[重命名所引入的 crate][rename] 来避免潜在的冲突问题，但是提前避免冲突总比重命名来的好。

所以你要检查 *Cargo.toml* 中**依赖的crate 的 `[features]`**，同时，还要检查 **`[dependencies]` 中标注为 `optional` 的 crate**  来确认当前 crate 的全部特征名字。如果要启用依赖项的一个特征，需要在 `[dependencies]` 小节增加 `features` 属性：

```toml
[dependencies]
somecrate = { version = "^0.3", features = ["featureA", "rand" ] }
```

上述的依赖项设置对于 `somecrate` 启用了 `featureA` 和 `rand` 两个特征。但是，你在 *Cargo.toml* 中指定的特征并不代表针对这个 crate 只启用了这些特征，因为 Cargo 中有[*特征联合*][feature unification]现象：最终构建时实际启用的特征是构建图中针对此 crate 所启用的所有特征的*并集*。换句话说，在上面的例子中，如果引用的某个依赖项也依赖 `somecrate`，并且启用了 `featureB` 特征，那么最终构建的时候，会同时启用 `featureA`，`featureB` 以及 `rand` 三个特征，以满足各个 crate 的需求 [^2]。这个规则也适用于 `default` 特征，如果你的 create 通过 `default-features = false` 禁用了默认特征，但是构建图中其他依赖项没有显式关闭默认特征，那么最终构建的时候，`default` 特征仍然是启用的。

特征联合的特点意味着多个特征之间应该是**可累加的**。因此 crate 中包含不兼容的特征并不是一个好主意，毕竟我们没有任何办法阻止使用者同时启用这些相互不兼容的特征。

例如，下面的例子中，crate 向外暴露了一个结构体，它的字段是公开访问的。把公开访问的字段设计成特征依赖的就会显得很糟糕：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// 结构体的字段是公开访问的，
/// 所以使用者可以通过指定字段初始值来构建结构体的实例。
#[derive(Debug)]
pub struct ExposedStruct {
    pub data: Vec<u8>,

    /// 仅当 `schema` 特征启用的时候，
    /// 才需要的额外数据
    #[cfg(feature = "schema")]
    pub schema: String,
}
```

那么使用这个 crate 的用户可能会存在一些困惑：当构造结构体的实例时，`schema` 字段是否应该填充对应的值？*为了*解决这个问题，他将不得不在 *Cargo.toml* 中启用对应的特征：

```toml
[features]
# `use-schema` 特征启用了 `somecrate` 中的 `schema` 特征
# （为了清晰起见，这里使用了不同的特征名字，
# 实际开发中，大概率是重用原有的特征名字）
use-schema = ["somecrate/schema"]
```

然后，在代码中依赖这个特征：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
let s = somecrate::ExposedStruct {
    data: vec![0x82, 0x01, 0x01],

    // 仅当启用 `somecrate/schema` 特征时，
    // 才填充此字段的值
    #[cfg(feature = "use_schema")]
    schema: "[int int]",
};
```

但是，这并不能涵盖所有的情况：当这段代码没有激活 `somecrate/schema` 特征，但是所用的其他依赖项启用了这个特征，就会导致错误。问题的关键在于，只有拥有该特征的 crate 能够检测到该特征；对于 crate 的用户来说，无法确定 Cargo 是否启用了 `somecrate/schema`。因此，你应该**避免在结构体中对公共字段进行特征隔离（feature-gating）**。

类似的考虑也适用于公开的 trait，尤其是暴露出来给其他代码使用的 trait。假设一个 trait 中包含了特征隔离的方法：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// 为支持 CBOR 序列化的条目设计的 trait
pub trait AsCbor: Sized {
    /// 将条目序列化成 CORB 数据
    fn serialize(&self) -> Result<Vec<u8>, Error>;

    /// 从 CBOR 数据反序列化一个条目
    fn deserialize(data: &[u8]) -> Result<Self, Error>;

    /// 返回这个条目对应的模式
    #[cfg(feature = "schema")]
    fn cddl(&self) -> String;
}
```

在项目中使用这个 trait 的用户同样会面临困惑：到底要不要实现 `cddl(&self)` 方法？外部代码根本没有办法得知是否应该实现 trait 中特征隔离的方法。

所以，结论就是：**避免在公共 trait 中设计特征隔离的方法**。包含默认实现的 trait 方法是个例外（见[第 13 条][Item 13]） —— 前提是，外部代码永远不会改写默认实现。

特征联合也意味着，如果你的 crate 包含了 *N* 个特征 [^3]，那么可能的特征组合是 *2<sup>N</sup>* 种。为了避免不必要的问题，应在你的 CI 系统中（见[第 32 条][Item 32]）通过完备的测试用例（见[第 30 条][Item 30]）来涵盖所有的 *2<sup>N</sup>*  种特征组合。

然而，当需要控制向展开后的依赖图（见[第 25 条][Item 25]）暴露的内容时，使用可选的特征还是非常有帮助的，尤其是那些可以在 `no_std` 环境（见[第 33 条][Item 33]）中使用的偏底层的 crate，通常会包含 `std` 或者 `alloc` 特征，来方便你在标准环境中使用它们。

## 牢记

- 特征和依赖项共享命名空间
- 选择特征名字的时候，应该慎重考虑，以避免和依赖项名字冲突
- 特征应该是可累加的
- 在公开暴露的结构体或者 trait 上，避免使用特征隔离的字段或方法
- 拥有很多相对独立的特征会导致可能的构建配置组合数量过于庞大

原文[点这里][origin]查看

-----

## 注释

[^1]: 这种默认行为可以通过在 `features` 节的其他地方使用 `"dep:<crate>"` 来禁用。详细信息请参考[文档][dep-crate-doc]。
[^2]: `cargo tree --edges features` 命令可以帮助你检测哪个 crate 启用了哪些特征，以及为什么要启用。
[^3]: 一个特征可以强制启用另外的特征，在最上面的例子中，`featureAB` 特征同时启用了 `featureA` 和 `featureB` 。



<!-- 参考链接 -->

[origin]: https://www.lurklurk.org/effective-rust/features.html
[dep-crate-doc]: https://doc.rust-lang.org/cargo/reference/features.html#optional-dependencies
[conditional compilation]: https://doc.rust-lang.org/reference/conditional-compilation.html
[cfg]: https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg-attribute
[cfg_attr]: https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg_attr-attribute
[target_os]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_os
[target_arch]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_arch
[target_pointer_width]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_pointer_width
[target_endian]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_endian
[target_has_atomic]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_has_atomic
[Rust Atomics and Locks]: https://marabos.nl/atomics/
[Cargo]: https://doc.rust-lang.org/cargo/index.html
[features]: https://doc.rust-lang.org/cargo/reference/features.html
[Cargo.toml]: https://doc.rust-lang.org/cargo/reference/manifest.html
[rename]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#renaming-dependencies-in-cargotoml
[feature unification]: https://doc.rust-lang.org/cargo/reference/features.html#feature-unification
[Item 13]: ../chapter_2/item13-use-default-impl.md
[Item 25]: ./item25-dep-graph.md
[Item 30]: ../chapter_5/item30-write-more-than-unit-tests.md
[Item 32]: ../chapter_5/item32-ci.md
[Item 33]: ../chapter_6/item33-no-std.md
