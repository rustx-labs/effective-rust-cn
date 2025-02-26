# 第 29 条：遵循 Clippy 的建议

> “看起来你在写信。需要什么帮助么？” —— [Microsoft Clippit]

[第 31 条]会描述了 Rust 工具箱中一些很有用的工具。但是其中一个特别有用且重要的工具值的在这里进行进行单独的介绍：[Clippy]。

Clippy 是 Cargo 的一个附加模块（通过`cargo clippy`的方式调用）。它可以生成涵盖多种类别的`warining`信息：
* 正确性：提示常见的编程错误。
* 风格：提示不完全符合 Rust 标准风格的代码结构。
* 简洁性：指出能让代码更加简洁的可行变更。
* 性能：提示能避免无效处理或者内存分配的可选项。
* 可读性：给出能让代码更易读或者更易懂的建议。

比如，如下这段代码编译是正常的：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
pub fn circle_area(radius: f64) -> f64 {
    let pi = 3.14;
    pi * radius * radius
}
```

但是 Clippy 会指出这里对 π 的近似赋值是没必要且不准确的：

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

链接中的文档解释了问题并且给出了优化代码的方式。上述示例可调整为：

```rust
pub fn circle_area(radius: f64) -> f64 {
    std::f64::consts::PI * radius * radius
}
```

正如示例中所展示的，每个 Clippy 警告都会伴随着一个网页的链接来描述问题。链接的网页中会说明为什么目标代码会被认为是不恰当的。这些说明很重要：它们的存在使得你可以自行判断采纳这些建议或者由于特殊的原因而忽略它们。有的时候，说明文本中还会描述一些校验器的已知问题，这些描述会解释一些令人困惑的误报。

如果你认定一些警告信息和自己的代码没有关系，你可以通过添加(#[allow(clippy::some\_line)])来忽略关联代码的报错，或者在包的顶层（top level）添加(#![allow(clipy::some\_lint)])来忽略整个包中的警告信息。通常情况下，建议调整目标代码而非花费很多时间来确认警告关联的代码是否是一个罕见的误报。

无论你选择了修复或者忽略掉这些警告信息，请**确保你的代码中没有 Clippy-warning 的信息**。

这样，当新的警告信息出现时 —— 无论是由于代码发生了调整还是 Clippy 升级后包含了新的校验信息 —— 我们就能够及时的关注到。Clippy 也应当被纳入你的持续集成系统中（[第 32 条]）。

Clippy 的警告信息在你学习 Rust 时特别重要，因为它们可以揭示那些被你忽略的细节，并帮助你熟悉 Rust 的风格。

本书中提到的很多建议，在 Clippy 中均存在相关的警告信息：

* [第 1 条]建议使用更具表现力的类型，而非一般的`bool`类型。Clippy 也指出了在[函数参数]以及[结构体]中使用多个`bool`类型的问题。
* [第 3 条]包括了一些`Option`及`Result`类型的操作。Clippy 指出了一些可行的精简行为，比如：
  * [Unnecessarily converting `Result` to `Option`]。
  * [Opportunities to use `unwrap_or_default`]。
* [第 3 条]同样建议了应当将错误返回给调用方。Clippy [指出了应当返回的地方]。
* [第 5 条]应当实现 `From` `trait` 而非 `Into` `trait`。
* [第 5 条]还描述了一些强制转换，而 Clippy 给出了如下的警告（对应的检查项默认是关掉的）：
  * [`as` casts that could be from instead]。
  * [`as` casts that might truncate]。
  * [`as` casts that might wrap]。
  * [`as` casts that lose precision]。
  * [`as` casts that might convert signed negative numbers to large positive numbers]。
  * [any use of `as`]。
* [第 8 条]描述了胖指针类型，并且很多 Clippy 的校验器指出了一些非必要的额外的指针间接访问：
  * [Holding a heap-allocated collection in a `Box`]。
  * [Holding a heap-allocated collection of `Box` items]。
  * [Taking a reference to a `Box`]。
* [第 9 条]描述了操作`Iterator`实例的诸多方法。Clippy 包含了诸多的可以简化迭代器方法使用的校验器[^1]。
* [第 10 条]描述了 Rust 的标准特性，并且包含了很多 Clippy 会校验到的实现时的要求：
  * [`Ord` must agree with `PartialOrd`]。
  * [`PartialEq::ne` should not need a nondefault implementation]（参照[第 13 条]）。
  * [`Hash` and `Eq` must be consistent]。
  * [`Clone` for `Copy` types should match]。
* [第 18 条]提供了一些关于减少[`panic!`]或类似[`expect`]的方法，这些建议也会在 Clippy 的校验器中检查到。
* [第 21 条]表述了引入通过通配符限定的包是不明智的。Clippy 同样对此[进行了校验]。
* [第 23 条]及[第 25 条]涉及到一种不同版本的包出现在同一个项目的依赖中。Clippy 可以通过配置，[在问题出现时给出警告信息]。
* [第 26 条]叙述了 Cargo 特性的一些相加性，而 Clippy 会将与此原则相违背特性提示为[“否定”的特性]。
* [第 26 条]同样表述了一个包的可选依赖项同样是其 feature 集的一部分。如果存在[需要明确的 feature 名（如 "use-crate-x" 形式的命名）]时，Clippy 将会提示直接应当使用明确的特性名，而非模糊的。
* [第 27 条]描述了文档注释的约束，Clippy 同时有如下的提示：
  * [Missing descriptions of `panic!`s]。
  * [Missing descriptions] of [`unsafe` concerns]。

上述的信息无疑说明了**阅读 [Clippy 的警告信息]列表**同样是一种有意义的学习方式 —— 包括那些默认被关掉校验的原因，是由于它们太严苛了还是由于它们会产生虚警？尽管你可能并不希望代码中出现这么多的警告信息，领悟这些校验规则出现的原因将会提升你对 Rust 及其风格的理解。

## 注释

[^1]: 部分校验器列举如下。[explicit_counter_loop]，[explicit_iter_loop]，[explicit_into_iter_loop]，[filter_map_identity]，[from_iter_instead_of_collect]，[into_iter_on_ref]，[iter_count]，[iter_next_loop]，[iter_not_returning_iterator]，[manual_filter_map]，[manual_find_map]，[map_clone]，[needless_range_loop]，[search_is_some]，[skip_while_next]，[suspicious_map]，[unnecessary_filter_map]，[unnecessary_fold]。

原文[点这里](https://www.lurklurk.org/effective-rust/clippy.html)查看

<!-- 参考链接 -->

[第 1 条]: ../chapter_1/item1-use-types.md
[第 3 条]: ../chapter_1/item3-transform.md
[第 5 条]: ../chapter_1/item5-casts.md
[第 8 条]: ../chapter_1/item8-references&pointer.md
[第 9 条]: ../chapter_1/item9-iterators.md
[第 10 条]: ../chapter_2/item10-std-traits.md
[第 13 条]: ../chapter_2/item13-use-default-impl.md
[第 18 条]: ../chapter_3/item18-panic.md
[第 21 条]: ../chapter_4/item21-semver.md
[第 23 条]: ../chapter_4/item23-wildcard.md
[第 25 条]: ../chapter_4/item25-dep-graph.md
[第 26 条]: ../chapter_4/item26-features.md
[第 27 条]: item27-document-public-interfaces.md
[第 31 条]: item31-use-tools.md
[第 32 条]: item32-ci.md

[Microsoft Clippy]: https://en.wikipedia.org/wiki/Office_Assistant
[Clippy]: https://github.com/rust-lang/rust-clippy#clippy
[函数参数]: https://rust-lang.github.io/rust-clippy/stable/index.html#/fn_params_excessive_bools
[结构体]: https://rust-lang.github.io/rust-clippy/stable/index.html#/struct_excessive_bools
[Unnecessarily converting `Result` to `Option`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/ok_expect
[Opportunities to use `unwrap_or_default`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/unwrap_or_else_default
[应当返回的地方]: https://rust-lang.github.io/rust-clippy/stable/index.html#/unwrap_in_result
[建议]: https://rust-lang.github.io/rust-clippy/stable/index.html#/from_over_into
[`as` casts that could be from instead]: https://rust-lang.github.io/rust-clippy/stable/index.html#/cast_lossless
[`as` casts that might truncate]: https://rust-lang.github.io/rust-clippy/stable/index.html#/cast_possible_truncation
[`as` casts that might wrap]: https://rust-lang.github.io/rust-clippy/stable/index.html#/cast_possible_wrap
[`as` casts that lose precision]: https://rust-lang.github.io/rust-clippy/stable/index.html#/cast_precision_loss
[`as` casts that might convert signed negative numbers to large positive numbers]: https://rust-lang.github.io/rust-clippy/stable/index.html#/cast_sign_loss
[any use of `as`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/as_conversions
[Holding a heap-allocated collection in a `Box`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/box_collection
[Holding a heap-allocated collection of `Box` items]: https://rust-lang.github.io/rust-clippy/stable/index.html#/vec_box
[Taking a reference to a `Box`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/borrowed_box
[explicit_counter_loop]: https://rust-lang.github.io/rust-clippy/stable/index.html#/explicit_counter_loop
[explicit_iter_loop]: https://rust-lang.github.io/rust-clippy/stable/index.html#/explicit_iter_loop
[explicit_into_iter_loop]: https://rust-lang.github.io/rust-clippy/stable/index.html#/explicit_into_iter_loop
[filter_map_identity]: https://rust-lang.github.io/rust-clippy/stable/index.html#/filter_map_identity
[from_iter_instead_of_collect]: https://rust-lang.github.io/rust-clippy/stable/index.html#/from_iter_instead_of_collect
[into_iter_on_ref]: https://rust-lang.github.io/rust-clippy/stable/index.html#/into_iter_on_ref
[iter_count]: https://rust-lang.github.io/rust-clippy/stable/index.html#/iter_count
[iter_next_loop]: https://rust-lang.github.io/rust-clippy/stable/index.html#/iter_next_loop
[iter_not_returning_iterator]: https://rust-lang.github.io/rust-clippy/stable/index.html#/iter_not_returning_iterator
[manual_filter_map]: https://rust-lang.github.io/rust-clippy/stable/index.html#/manual_filter_map
[manual_find_map]: https://rust-lang.github.io/rust-clippy/stable/index.html#/manual_find_map
[map_clone]: https://rust-lang.github.io/rust-clippy/stable/index.html#/map_clone
[needless_range_loop]: https://rust-lang.github.io/rust-clippy/stable/index.html#/needless_range_loop
[search_is_some]: https://rust-lang.github.io/rust-clippy/stable/index.html#/search_is_some
[skip_while_next]: https://rust-lang.github.io/rust-clippy/stable/index.html#/skip_while_next
[suspicious_map]: https://rust-lang.github.io/rust-clippy/stable/index.html#/suspicious_map
[unnecessary_filter_map]: https://rust-lang.github.io/rust-clippy/stable/index.html#/unnecessary_filter_map
[unnecessary_fold]: https://rust-lang.github.io/rust-clippy/stable/index.html#/unnecessary_fold
[`Ord` must agree with `PartialOrd`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/derive_ord_xor_partial_ord
[`PartialEq::ne` should not need a nondefault implementation]: https://rust-lang.github.io/rust-clippy/stable/index.html#/partialeq_ne_impl
[`Hash` and `Eq` must be consistent]: https://rust-lang.github.io/rust-clippy/stable/index.html#/derived_hash_with_manual_eq
[`Clone` for `Copy` types should match]: https://rust-lang.github.io/rust-clippy/stable/index.html#/expl_impl_clone_on_copy
[`panic!`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/panic
[`expect`]: https://rust-lang.github.io/rust-clippy/stable/index.html#/expect_used
[进行了校验]: https://rust-lang.github.io/rust-clippy/stable/index.html#/wildcard_dependencies
[在问题出现时给出警告信息]: https://rust-lang.github.io/rust-clippy/stable/index.html#/multiple_crate_versions
[“否定”的特性]: https://rust-lang.github.io/rust-clippy/stable/index.html#/negative_feature_names
[需要明确的 feature 名（如 "use-crate-x" 形式的命名）]: https://rust-lang.github.io/rust-clippy/stable/index.html#/redundant_feature_names
[Missing descriptions of `panic!`s]: https://rust-lang.github.io/rust-clippy/stable/index.html#/missing_panics_doc
[Missing descriptions]: https://rust-lang.github.io/rust-clippy/stable/index.html#/missing_safety_doc
[`unsafe` concerns]: https://rust-lang.github.io/rust-clippy/stable/index.html#/undocumented_unsafe_blocks
[Clippy 的警告信息]: https://rust-lang.github.io/rust-clippy/stable/index.html
