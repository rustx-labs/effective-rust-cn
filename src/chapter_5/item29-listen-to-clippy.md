# 第 29 条：遵循 Clippy 的建议

> “看起来你在写信。需要什么帮助么？” —— [Microsoft Clippit]

[第 31 条]描述了 Rust 工具箱中很有用的一些工具。但是其中一个特别有用且重要的工具值得拿出来进行进行单独的介绍：[Clippy]。

Clippy 是 Cargo 的一个附加模块（通过`cargo clippy`的方式调用）。它可以生成涵盖多种类别的`warining`信息：
* 正确性：提示常见的编程错误。
* 风格：提示不完全符合 Rust 标准风格的代码结构。
* 简洁性：指出能让代码更加简洁的可行变更。
* 性能：提示能避免无效处理或者内存分配的可选项。
* 可读性：给出能让代码更易读或者更易懂的建议。

比如，如下这段代码编译是正常的：

```rust
pub fn circle_area(radius: f64) -> f64 {
    let pi = 3.14;
    pi * radius * radius
}
```

但是 Clippy 会指出本地的对 π 的近似赋值是没必要且不准确的：

```rust
error: approximate value of `f{32, 64}::consts::PI` found
 --> src/main.rs:5:18
  |
5 |         let pi = 3.14;
  |                  ^^^^
  |
  = help: consider using the constant directly
  = help: for further information visit
    https://rust-lang.github.io/rust-clippy/master/index.html#approx_constant
  = note: `#[deny(clippy::approx_constant)]` on by default
```

链接中的文档解释了问题并且给出了优化代码的方式：

```rust
pub fn circle_area(radius: f64) -> f64 {
    std::f64::consts::PI * radius * radius
}
```

正如示例中所展示的，每个 Clippy 警告都会伴随着一个网页的链接来描述问题。链接中会说明为什么目标代码会被认为是不恰当的。这些说明很重要。它们的存在使得你可以自行判断采纳这些建议或者由于特殊的原因而忽略它们。有的时候，说明文本中还会描述一些校验器的已知问题，这些内容会解释一些令人困惑的误报。

如果你认定一些警告信息和自己的代码没有关系，你可以通过添加(#[allow(clippy::some\_line)])来忽略关联代码的报错，或者在包的顶层（top level）添加(#![allow(clipy::some\_lint)])来忽略整个包中的警告信息。通常情况下，建议调整目标代码而非花费很多时间来确认警告关联的代码是否是一个高明的误报。

无论你选择了修复或者忽略掉这些警告信息，请*确保你的代码中没有 Clippy-warning 的信息*。

这样，当新的警告信息出现时 —— 无论是由于代码发生了调整还是 Clippy 升级后包含了新的校验信息 —— 我们就能够及时的关注到。Clippy 也应当被纳入你的持续集成系统中（[第 32 条]）。

Clippy 的警告信息在你学习 Rust 时特别重要，因为它们可以揭示那些被你忽略的细节，并帮助你熟悉 Rust 的风格。

当我们使用 Clippy 来检查本书中出现的代码时，就会发现一些条目的代码也存在 Clippy 警告信息：

* [第 1 条]建议使用更具表现力的类型，而非一般的`bool`类型。Clippy 同时指出了在[函数参数]以及[结构体]中使用多个`bool`类型的问题。
* 

上述的信息无疑说明了*阅读 Clippy 的警告信息*同样是一中有意义的学习方式 —— 包括那些默认被关掉校验的原因，是由于它们太严苛了还是由于它们会产生虚警？尽管你可能并不希望代码中出现这么多的警告信息，理解这些校验规则出现的原因将会提升你对 Rust 及其风格的理解。

### 注释

原文[点这里]查看

<!-- 参考链接 -->

[Microsoft Clippy]: https://en.wikipedia.org/wiki/Office_Assistant
[第 31 条]:
[Clippy]:
[第 32 条]:

[点这里]: https://www.lurklurk.org/effective-rust/clippy.html
