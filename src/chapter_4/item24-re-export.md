# 第 24 条：重新导出在 API 中所用的依赖项类型

如果单看本章节的标题，会让人费解。没有关系，下面我们通过一个例子来把事情讲解清楚[^1]。

在[第 25 条]描述了 `cargo` 是如何支持将同一个 crate 的多个不同版本链接到最终输出的二进制文件中的，而且这个过程对用户而言是透明的。假设一个使用 `rand` crate 的二进制项目，更精确一些，使用的是 `0.8` 版本的 `rand`：

```toml
# 顶层二进制 crate 的 Cargo.toml 文件内容

[dependencies]
# 依赖 `rand` crate
rand = "=0.8.5"

# 依赖 `dep-lib` crate 
dep-lib = "0.1.0"
```

```rust
let mut rng = rand::thread_rng(); // rand 0.8
let max: usize = rng.gen_range(5..10);
let choice = dep_lib::pick_number(max);
```

上面的代码中最后一行，使用了一个假想的 crate `dep-lib`，它可能来自 `crates.io`，也可能是通过[路径][path mechanism]引入的本地 crate。

 `dep-lib` 自己使用了 0.7 版本的 `rand` crate：

```toml
# `dep-lib` 的 Cargo.toml 

[dependencies]
# 依赖 `rand` crate
rand = "=0.7.3"
```

```rust
//! The `dep-lib` crate provides number picking functionality.
use rand::Rng;

/// Pick a number between 0 and n (exclusive).
pub fn pick_number(n: usize) -> usize {
    rand::thread_rng().gen_range(0, n)
}
```

细心的读者可能会注意到这两段示例代码之间的区别：

- `dep-lib` 所使用的 0.7 版本的 `rand` 中的 [`rand::gen_range()`][rand-gen-range-0.7] 方法有  `low` 和 `high` 2 个参数。
- 在示例项目中所用的 0.8 版本的 `rand` 中的 [`rand::gen_range()`][rand-gen-range-0.8] 方法只有 `range` 1 个参数。

这种差异属于不兼容的变更，所以根据语义化版本控制的指导原则， `rand` 升级了最左非零版本号（见[第 21 条]）。虽然如此，基于 `cargo` 强大的能力，它仍然能这两个不兼容的版本合并到最终输出的二进制可执行文件中。

如果 `dep-lib` 的公共 API 中暴露了它所使用的依赖项中的类型的话，情况就会变得糟糕了，因为这会导致 `rand` 成为[*公共依赖项*][public dependency]。

举例说明，假设 `dep-lib` crate 暴露出来的函数中使用了来自 `rand` 0.7 版本的 `Rng` trait：

```rust
/// 使用提供的 `Rng` 实例随机生成 0 到 n （不含）
/// 之间的数字
pub fn pick_number_with<R: Rng>(rng: &mut R, n: usize) -> usize {
    rng.gen_range(0, n) // 0.7.x 版本的方法
}
```

顺便说一下，**在你对外暴露的 API 中使用其他 crate 类型之前请谨慎考虑**：这会将你的 crate 与该依赖密切绑定。例如，依赖项的主版本号升级（见[第 21 条]）将会自动要求你的 crate 也进行主版本号升级。

`rand` crate 本身只有少量的依赖项（见[第 25 条]），并且它被大量的其他项目所依赖，已经成为“事实上标准”的 crate 了。在上面的这段代码中，crate 中暴露出来的函数使用 `Rng` 倒也不是不可接受的。

好的，回到我们这个示例中来，假设在顶层的二进制项目中使用 `dep-lib` 的函数：

```rust
let mut rng = rand::thread_rng();
let max: usize = rng.gen_range(5..10);
let choice = dep_lib::pick_number_with(&mut rng, max);
```

将会无法通过编译，并且 Rust 编译器给出的错误消息也[没什么实质的帮助][very helpful]：

```
error[E0277]: the trait bound `ThreadRng: rand_core::RngCore` is not satisfied
  --> src/main.rs:22:44
   |
22 |     let choice = dep_lib::pick_number_with(&mut rng, max);
   |                  ------------------------- ^^^^^^^^ the trait
   |                  |                `rand_core::RngCore` is not
   |                  |                 implemented for `ThreadRng`
   |                  |
   |                  required by a bound introduced by this call
   |
   = help: the following other types implement trait `rand_core::RngCore`:
             &'a mut R
```

上面的错误消息会让人感到困惑：明明 `rand_core::RngCore` *确实*已经实现了 `ThreadRng` trait 了啊！这是因为调用者传入的值实现的是 `RngCore_v0_8_5`，而 `dep-lib` 期望的是 `RngCore_v0_7_3` 的实现。

至此，我们知道了依赖项的版本冲突才是引发上面编译错误的根本原因，那么应该如何解决呢？[^2] 解决的这个问题的关键在于，虽然我们不能在二进制输出文件中*直接*使用同一个 crate 的两个不同版本，但是可以*间接地*做到这一点（就像最前面的例子中所示的那样）。

从二进制 crate 作者的角度来看，可以增加一个中间包装 crate，该 crate 独立于二进制的 crate，可以直接使用来自 `rand` v0.7 的类型，而二进制 crate 仍然使用 `rand` v0.8。虽然可以解决这个问题，但是这种方案实在是不太方便。

更优的一个解决方案是，让库作者显式地[重新导出][re-exporting]下列内容：

- 库中 API 所用的来自其他依赖项的类型
- 或者，完整的依赖项

这种方案显然是更优的，从库中重新导出 0.7 版本的 `Rng` 和 `RngCore` 供调用者使用，还可以提供用于构造类型实例的方法（例如 `thread_rng()`）：

```rust
// 重新导出本 crate 所用版本的 `rand`
pub use rand;
```

调用者使用 0.7 版本的 `rand` 时，需要换一种写法：`dep_lib::rand` ：

```rust
let mut prev_rng = dep_lib::rand::thread_rng(); // v0.7 Rng 实例
let choice = dep_lib::pick_number_with(&mut prev_rng, max);
```

结合这个例子，标题中给出的建议现在应该不那么晦涩难懂了：**重新导出在你 API 中的所用的依赖类型**。这样可以减少用户在版本冲突或依赖管理方面的困扰，并提高库的易用性和兼容性。

## 注释

[^1]: 本示例（包含其中所用的类型），及其解决方法，受 [RustCrypto crates] 启发

[^2]: 还有一些场景也可能引发类似的错误：在项目的依赖图中，针对一个 crate 的*同一个版本*有多个不同的替代项时，以及使用 [path][path] 而不是 `crates.io` 来导入依赖项时

原文[点这里](https://www.lurklurk.org/effective-rust/re-export.html)查看

<!-- 参考链接 -->

[第 21 条]: item21-semver.md
[第 25 条]: item25-dep-graph.md

[RustCrypto crates]: https://docs.rs/signature/1.3.0/signature/index.html#reexports
[path mechanism]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#specifying-path-dependencies
[rand-gen-range-0.7]: https://docs.rs/rand/0.7.3/rand/trait.Rng.html#method.gen_range
[rand-gen-range-0.8]: https://docs.rs/rand/0.8.5/rand/trait.Rng.html#method.gen_range
[public dependency]: https://rust-lang.github.io/api-guidelines/necessities.html#public-dependencies-of-a-stable-crate-are-stable-c-stable
[very helpful]: https://github.com/rust-lang/rust/issues/22750
[path]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#specifying-path-dependencies
[re-exporting]: https://doc.rust-lang.org/reference/items/use-declarations.html#use-visibility
