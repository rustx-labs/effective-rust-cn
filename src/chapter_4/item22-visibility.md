# 第 22 条：最小化可见性

Rust 允许用户控制一段代码中的元素对其他代码而言是隐藏还是可见的。本条目探索可见性机制，以及给出选择合适可见性的一些建议。

## 可见性的语法

模块（module）是 Rust 中控制可见性的基本单元。默认情况下，一个模块中的条目（类型、方法、常量等）是*私有*的，只能被本模块以及其子模块中的代码访问。

使用 `pub` 关键字，可以扩大代码的可见范围。对于大部分 Rust 语法特性来说，将某个条目标记为 `pub` 并不会公开其代码内容，例如：`pub mod` 中的类型和函数不是公开的，`pub struct` 的字段也不是公开的。但是，也有一些例外情况，将可见性应用到类型的内容中也是合理的：

- 一个标记为公开的 `enum`，它所包含的枚举变体以及变体所包含的字段也是公开的。
- 一个标记为公开的 trait，它的所有方法都是公开的。

所以，一个模块中的类型：

```rust
pub mod somemodule {
    // 将 `struct` 标记为公开，不会自动将其字段公开。
    #[derive(Debug, Default)]
    pub struct AStruct {
        // 默认情况下，字段是私有的
        count: i32,
        // 必须显式增加 `pub` 以使得字段对外可见
        pub name: String,
    }

    // 类似的，结构体中的方法也需要 `pub` 标记使其对外可见
    impl AStruct {
        // 默认情况下，方法也是私有的。
        fn canonical_name(&self) -> String {
            self.name.to_lowercase()
        }
        // 必须显式增加`pub` 让其对外可见
        pub fn id(&self) -> String {
            format!("{}-{}", self.canonical_name(), self.count)
        }
    }

    // 标注为公开的 `enum` 它所包含的变体都是对外可见的
    #[derive(Debug)]
    pub enum AnEnum {
        VariantOne,
        // 以及变体中的字段也是对外可见的
        VariantTwo(u32),
        VariantThree { name: String, value: String },
    }

    // 标记为公开的 `trait`，它的方法都是对外可见的
    pub trait DoSomething {
        fn do_something(&self, arg: i32);
    }
}
```

允许外部代码访问模块中标记为 `pub` 的条目，以及，前述提及的例外情况（自动对外可见）：

```rust
use somemodule::*;

let mut s = AStruct::default();
s.name = "Miles".to_string();
println!("s = {:?}, name='{}', id={}", s, s.name, s.id());

let e = AnEnum::VariantTwo(42);
println!("e = {e:?}");

#[derive(Default)]
pub struct DoesSomething;
impl DoSomething for DoesSomething {
    fn do_something(&self, _arg: i32) {}
}

let d = DoesSomething::default();
d.do_something(42);
```

但是，未标记为 `pub` 的，则不可被外部访问：

```rust
let mut s = AStruct::default();
s.name = "Miles".to_string();
println!("(inaccessible) s.count={}", s.count);
println!("(inaccessible) s.canonical_name()={}", s.canonical_name());
```

```plain
error[E0616]: field `count` of struct `somemodule::AStruct` is private
   --> src/main.rs:230:45
    |
230 |     println!("(inaccessible) s.count={}", s.count);
    |                                             ^^^^^ private field
error[E0624]: method `canonical_name` is private
   --> src/main.rs:231:56
    |
86  |         fn canonical_name(&self) -> String {
    |         ---------------------------------- private method defined here
...
231 |     println!("(inaccessible) s.canonical_name()={}", s.canonical_name());
    |                                         private method ^^^^^^^^^^^^^^
Some errors have detailed explanations: E0616, E0624.
For more information about an error, try `rustc --explain E0616`.
```

最常用的控制可见性的标记就是 `pub` 关键字，只要外部可以访问这个模块，那么模块中所有标记为 `pub` 的条目都是对外可见的。这个细节很重要：如果模块（例如：`somecrate::somemodule`）本身对外不可见，那么模块中的条目即使标记为 `pub`，它们也是对外不可见的。

但是，还有一些 `pub` 关键字的变体形式，可以用来约束可见性生效的范围：

- `pub(crate)`：对其所在的 crate 中的其他条目可见。当你需要在 crate 范围提供一个类似助手函数，但是又不希望外部可见是，这个标记就非常实用了。
- `pub(super)`：对当前所在模块的父模块及其子模块可见。这在具有较深层级模块结构的 crate 项目中会很有帮助，它可以让你有选择的扩大可见性。这也是模块级的有效可见性，`mod mymodule` 本身就对其父模块（或 crate）及其子模块可见。
- `pub(in <path>)`：仅对指定 `<path>` 的代码可见。其中，`<path>` 必须是对当前模块的祖先模块的描述。这在组织源代码时偶尔会有用，因为它允许将某些功能子集移动到无需在公共 API 中可见的子模块中。例如，Rust 标准库将所有迭代器的[适配器]合并到一个[内部的 `std::iter::adapters` 子模块]中，并且具有以下内容：
  - 子模块中所需的适配器方法都带有 `pub(in crate::iter)` 可见性标记，例如：[`std::iter::adapters::map::Map::new`]。
  - 在[外部 `std::iter` 模块]中，对 `adapters::` 中的类型使用 `pub use`。
- `pub(self)`：等同于 `pub(in self)`，也即不是 `pub` 的。比较少见，可以用来减少在代码生成宏时特殊场景的数量。

如果你在一个模块定义了一个私有函数，但是并未在任何地方使用它，Rust 编译器会给出警告：

```rust
pub mod anothermodule {
    // 私有函数未被使用
    fn inaccessible_fn(x: i32) -> i32 {
        x + 3
    }
}
```

虽然说警告消息的字面意思说：在本模块“从未使用”，实际上的含义是指由于可见性的限制，你在模块外也*无法*使用此函数：

```rust
warning: function `inaccessible_fn` is never used
  --> src/main.rs:56:8
   |
56 |     fn inaccessible_fn(x: i32) -> i32 {
   |        ^^^^^^^^^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default
```

## 可见性的语义

除了前面提到的*如何*扩大可见范围之外，还有一个问题：*何时*应该扩大可见范围？普遍可接受的答案是：应当*尽可能不扩大可见范围*，特别是对于将来会被使用或者重用的代码。

之所以给出这样的建议，第一个原因是已经扩大的可见范围是很难再收缩回来的。一旦 crate 中的某个条目可以被公开访问，就无法将其再改回私有，否则会破坏使用该 crate 的代码，从而不得不发布一个主版本号升级（见[第 21 条]）。反之则不成立：将一个私有项改为公有通常只需要小版本号的升级，并且不会影响使用 crate 的用户——查阅 [Rust API 兼容性指南]时你会注意到，其中有很多都和公共访问的条目有关系。

还有一个更重要但是不这么明显的原因就是：选择一个尽可能小的可见性范围，可以让你在将来保留更多的选择权。对公共访问暴露的内容越多，将来需要保持不变的东西也越多（除非进行不兼容的更改）。比如说，你对外暴露了数据结构的内部实现细节，那么将来为了使用更高效的算法而进行变更，就会破坏兼容性；如果你暴露了内部辅助函数，那么不可避免地会有一些外部代码依赖于这些函数的具体细节。

当然，这主要是针对可能有多个用户、较长生命周期的库代码的考虑，但是在平时的项目中也养成这种习惯终究是有益无害的。

值得注意的是，限制可见性的建议并不局限于本篇所述，也不局限于 Rust 这一门语言：

- [Rust API 指南]中，包含如下建议：
  - [结构体应该具有私有字段]。
- [Effective Java] 第 3 版（Addison-Wesley Professional 出版）中有以下建议：
  - 第 15 条：最小化类及其成员的可见范围
  - 第 16 条：在公共类中，不要使用公共字段，应使用访问方法
- Scott Meyers 在 [Effective C++] 第 2 版（Addison-Wesley Professional 出版）中建议：
  - 第 18 条：努力让类接口既完整又*精简*（斜体字为原文所加）。
  - 第 20 条：避免在公共接口中使用数据成员。

原文[点这里](https://www.lurklurk.org/effective-rust/visibility.html)查看

<!-- 参考链接 -->

[第 21 条]: ./item21-semver.md

[适配器]: https://doc.rust-lang.org/std/iter/index.html#adapters
[内部的 `std::iter::adapters` 子模块]: https://doc.rust-lang.org/src/core/iter/adapters/mod.rs.html
[`std::iter::adapters::map::Map::new`]: https://doc.rust-lang.org/1.70.0/src/core/iter/adapters/map.rs.html#68
[外部 `std::iter` 模块]: https://doc.rust-lang.org/1.70.0/src/core/iter/mod.rs.html#423-451
[Rust API 兼容性指南]: https://doc.rust-lang.org/cargo/reference/semver.html#api-compatibility
[Rust API 指南]: https://rust-lang.github.io/api-guidelines/future-proofing.html
[结构体应该具有私有字段]: https://rust-lang.github.io/api-guidelines/future-proofing.html#structs-have-private-fields-c-struct-private
[Effective Java]: https://www.oreilly.com/library/view/effective-java/9780134686097/
[Effective C++]: https://en.wikipedia.org/wiki/Special:BookSources?isbn=978-0-201-92488-6
