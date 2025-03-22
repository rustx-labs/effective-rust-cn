# 第 26 条：警惕 feature 蔓延

通过使用 Cargo 的 *特性（feature）* 机制，Rust 允许同一套代码有不同的配置，其底层基于条件编译机制。但是，关于 Rust 中的 feature，有一些值得关注的点，本章将一一阐述。

## 条件编译

通过使用 [`cfg`] 或者 [`cfg_attr`]，Rust 支持[条件编译]，可以让你决定函数、代码行、代码块等内容是否包含在编译后的文件中（相对而言，C/C++ 是基于代码行的预处理器）。这里的“条件”可以是一个像 `test` 这种单纯的名字，也可以是类似 `panic = "abort"` 这种名值对的形式。

注意，名值对的形式下，一个名字的值可以有多个：

```rust
// 构建时设置环境变量 `RUSTFLAGS` 如下：
//   '--cfg myname="a" --cfg myname="b"'
#[cfg(myname = "a")]
println!("cfg(myname = 'a') is set");
#[cfg(myname = "b")]
println!("cfg(myname = 'b') is set");
```

```shell
cfg(myname = 'a') is set
cfg(myname = 'b') is set
```

除了前述这种显式指定 **feature** 值之外，更常用的配置项是由工具链自动引入的构建时的目标环境，包括：目标操作系统（[`target_os`]）、CPU 架构（[`target_arch`]）、指针位宽（[`target_pointer_width`]）、字节序（[`target_endian`]）等，通过构建时的目标平台启用对应的 feature 来实现代码的可移植性。

另外，标准选项 [`target_has_atomic`] 是支持多值的，如果目标平台同时支持 32 位和 64 位架构，则 `[cfg(target_has_atomic = "32")]` 和 `[cfg(target_has_atomic = "64")]` 同时生效。（关于原子性的更多信息，请参考 O'Reilly 出版的由 Mara Bos 所著的 《[Rust Atomics and Locks]》一书的第二章。）

## Features

通过使用基于 `cfg` 的名值对机制，[Cargo] 包管理器提供了[*特性（features）*][features] 选择能力：在构建 crate 的时候，可以有选择地启用 crate 的功能。Cargo 将确保对于每个 crate，都使用所配置的 **feature** 值来进行编译。

这是特定于 Cargo 的功能：对于 Rust 的编译器而言，**feature** 只是另一个可配置选项。

截至本文成稿时间，检测所启用的 feature 的最可靠方式就是查看 [*Cargo.toml*][Cargo.toml] 文件。举例说明，下面这段内容实际上包含了 *6 个* feature：

```toml
[features]
default = ["featureA"]
featureA = []
featureB = []
# 启用 `featureAB` 表示同时启用了 `featureA` 和 `featureB`。
featureAB = ["featureA", "featureB"]
schema = []

[dependencies]
rand = { version = "^0.8", optional = true }
hex = "^0.4"
```

可是，在上面的例子中，`[features]` 小节只有 5 个 feature 啊！因此这里有一些细节需要注意。

首先，`[features]` 中的 `default` 是一个特殊的 feature 名字，它表示默认启用的 feature。当然了，在构建时可以通过 `--no-default-features` 参数忽略默认 feature，或者在 *Cargo.toml* 中这样写：

```toml
[dependencies]
somecrate = { version = "^0.3", default-features = false }
```

无论如何，`default` 仍然是一个可以在代码中正常使用的 feature 名字：

```rust
#[cfg(feature = "default")]
println!("This crate was built with the \"default\" feature enabled.");
#[cfg(not(feature = "default"))]
println!("This crate was built with the \"default\" feature disabled.");
```

在示例的 *Cargo.toml* 中，另一个不明显的 feature 隐藏在 `[dependencies]` 小节：依赖项 `rand` crate 被标记为 `optional = true`，这就使得 `rand` 成为一个 feature 名字 [^1]。当示例的 crate 使用 `--features rand` 编译的时候，`rand` feature 将被激活：

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

Crate 名字是全局的（通常由 `crates.io` 管理），而 feature 名字是 crate 本地的，但事实上 *crate 和 feature 共享命名空间*。因此，**谨慎选择 feature 名字**，以避免和可能依赖的 crate 名字冲突。虽然 Cargo 支持通过修改 `package` 键[重命名所引入的 crate] 来避免潜在的冲突问题，但是提前避免冲突总比重命名来的好。

所以你除了要检查 *Cargo.toml* 中**依赖的 crate 的 `[features]`**，还要检查 **`[dependencies]` 中标注为 `optional` 的 crate**  来确认当前 crate 的全部 feature 名字。如果要启用依赖项的一个 feature，需要在 `[dependencies]` 小节增加 `features` 属性：

```toml
[dependencies]
somecrate = { version = "^0.3", features = ["featureA", "rand" ] }
```

上述的依赖项设置对于 `somecrate` 启用了 `featureA` 和 `rand` 两个 feature。但是，你在 *Cargo.toml* 中指定的 feature 并不代表针对这个 crate 只启用了这些 feature，因为 Cargo 中有 [*feature 联合*][feature unification]现象：最终构建时实际启用的 feature 是构建图中针对此 crate 所启用的所有 feature 的*并集*。换句话说，在上面的例子中，如果引用的某个依赖项也依赖 `somecrate`，并且启用了 `featureB` feature，那么最终构建的时候，会同时启用 `featureA`、`featureB` 以及 `rand` 三个 feature，以满足各个 crate 的需求 [^2]。这个规则也适用于 `default` feature，如果你的 create 通过 `default-features = false` 禁用了默认 feature，但是构建图中其他依赖项没有显式关闭默认 feature，那么最终构建的时候，`default` feature 仍然是启用的。

Feature 联合的特点意味着多个 feature 之间应该是**可累加的**。因此在 crate 中包含不兼容的 feature 并不是一个好主意，毕竟我们没有任何办法阻止不同的使用者同时启用这些相互不兼容的 feature。

例如，下面的例子中，crate 向外暴露了一个结构体，它的字段是公开访问的。把公开访问的字段设计成 feature 依赖的就会显得很糟糕：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// 结构体的字段是公开访问的，
/// 所以使用者可以通过指定字段初始值来构建结构体的实例。
#[derive(Debug)]
pub struct ExposedStruct {
    pub data: Vec<u8>,

    /// 仅当 `schema` feature 启用的时候，
    /// 才需要的额外数据。
    #[cfg(feature = "schema")]
    pub schema: String,
}
```

那么使用这个 crate 的用户可能会存在一些困惑：当构造结构体的实例时，`schema` 字段是否应该填充对应的值？*为了*解决这个问题，他将不得不在 *Cargo.toml* 中启用对应的 feature：

```toml
[features]
# `use-schema` feature 启用了 `somecrate` 中的 `schema` feature
# （为了清晰起见，这里使用了不同的 feature 名字，
# 实际开发中，大概率是重用原有的 feature 名字。）
use-schema = ["somecrate/schema"]
```

然后，在代码中依赖这个 feature：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
let s = somecrate::ExposedStruct {
    data: vec![0x82, 0x01, 0x01],

    // 仅当启用 `somecrate/schema` feature 时，
    // 才填充此字段的值。
    #[cfg(feature = "use_schema")]
    schema: "[int int]",
};
```

但是，这并不能涵盖所有的情况：当这段代码没有激活 `somecrate/schema` feature，但是所用的其他依赖项启用了这个 feature，就会导致错误。问题的关键在于，只有拥有该 feature 的 crate 能够检测到该 feature；对于 crate 的用户来说，无法确定 Cargo 是否启用了 `somecrate/schema`。因此，你应该**避免在结构体中对公共字段进行 feature 门控（feature-gating）**。

类似的考虑也适用于公共的 trait，尤其是暴露出来给其他代码使用的 trait。假设一个 trait 中包含了 feature 门控的方法：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// 为支持 CBOR 序列化的条目设计的 trait。
pub trait AsCbor: Sized {
    /// 将条目序列化成 CORB 数据。
    fn serialize(&self) -> Result<Vec<u8>, Error>;

    /// 从 CBOR 数据反序列化一个条目。
    fn deserialize(data: &[u8]) -> Result<Self, Error>;

    /// 返回这个条目对应的模式。
    #[cfg(feature = "schema")]
    fn cddl(&self) -> String;
}
```

在项目中使用这个 trait 的用户同样会面临困惑：到底要不要实现 `cddl(&self)` 方法？外部代码根本没有办法得知是否应该实现 trait 中 feature 门控的方法。

所以，结论就是：*避免在公共 trait 中设计 feature 门控的方法*。包含默认实现的 trait 方法是个例外（见[第 13 条]）—— 前提是，外部代码永远不会改写默认实现。

Feature 联合也意味着，如果你的 crate 包含了 *N* 个 feature [^3]，那么可能的 feature 组合是 *2<sup>N</sup>* 种。为了避免不必要的问题，应在你的 CI 系统中（见[第 32 条]）通过完备的测试用例（见[第 30 条]）来涵盖所有的 *2<sup>N</sup>*  种 feature 组合。

然而，当需要控制向展开后的依赖图（见[第 25 条]）暴露的内容时，使用可选的 feature 还是非常有帮助的，尤其是那些可以在 **no_std** 环境（见[第 33 条]）中使用的偏底层的 crate，通常会包含 `std` 或者 `alloc`  feature，来方便你在标准环境中使用它们。

## 需要记住的事情

- Feature 和依赖项共享命名空间。
- 应该慎重考虑 feature 的命名，以避免和依赖项名字冲突。
- Feature 应该是可累加的。
- 避免在公开暴露的结构体属性或者 trait 方法上使用 feature 门控。
- 拥有很多相对独立的 feature 会导致可能的构建配置组合数量过于庞大。

## 注释

[^1]: 这种默认行为可以通过在 `features` 节的其他地方使用 `"dep:<crate>"` 来禁用。详细信息请参考[文档][dep-crate-doc]。

[^2]: `cargo tree --edges features` 命令可以帮助你检测哪个 crate 启用了哪些 feature，以及为什么要启用。

[^3]: 一个 feature 可以强制启用另外的 feature，在最上面的例子中，`featureAB` feature 同时启用了 `featureA` 和 `featureB`。

原文[点这里](https://www.lurklurk.org/effective-rust/features.html)查看

<!-- 参考链接 -->

[第 13 条]: ../chapter_2/item13-use-default-impl.md
[第 25 条]: item25-dep-graph.md
[第 30 条]: ../chapter_5/item30-write-more-than-unit-tests.md
[第 32 条]: ../chapter_5/item32-ci.md
[第 33 条]: ../chapter_6/item33-no-std.md

[`cfg`]: https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg-attribute
[`cfg_attr`]: https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg_attr-attribute
[条件编译]: https://doc.rust-lang.org/reference/conditional-compilation.html
[`target_os`]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_os
[`target_arch`]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_arch
[`target_pointer_width`]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_pointer_width
[`target_endian`]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_endian
[`target_has_atomic`]: https://doc.rust-lang.org/reference/conditional-compilation.html#target_has_atomic
[Rust Atomics and Locks]: https://marabos.nl/atomics/
[Cargo]: https://doc.rust-lang.org/cargo/index.html
[features]: https://doc.rust-lang.org/cargo/reference/features.html
[Cargo.toml]: https://doc.rust-lang.org/cargo/reference/manifest.html
[重命名所引入的 crate]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#renaming-dependencies-in-cargotoml
[feature unification]: https://doc.rust-lang.org/cargo/reference/features.html#feature-unification
[dep-crate-doc]: https://doc.rust-lang.org/cargo/reference/features.html#optional-dependencies
