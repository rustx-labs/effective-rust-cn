# 第 27 条：为公共接口撰写文档

如果你的包（crate）会被其他程序员所使用，那么为包中的内容添加文档就是很好的实践，尤其是包中的公共接口。如果你的包不仅仅是随用随丢的代码，那么这个“其他程序员”就包括未来的你，那个已经忘掉了自己代码细节的你。

这个建议并不是 Rust 所独有的，它也并不是一个新的建议 —— 比如，[Effective Java] 第二版（2008年出版）在第 44 条中建议：“为所有导出的 API 元素编写文档注释”。

Rust 文档类型注释的细节 —— 基于 Markdown 格式，以 /// 或者 //! 分割 —— 已经在[Rust book]中介绍了，如下为示例：

```Rust
/// Calculate the [`BoundingBox`] that exactly encompasses a pair
/// of [`BoundingBox`] objects.
pub fn union(a: &BoundingBox, b: &BoundingBox) -> BoundingBox {
    // ...
}
```

然而，关于文档型注释的格式仍有一些值得关注的细节：
* 使用代码格式：对于任何作为源代码的注释，使用反引号来确保在最终的文档中代码会以一种等宽字体来展示，并以此来明确的区分`code`以及一般的文本。
* 添加丰富的引用内容：为任何能够给读者提供上下文信息的内容添加 Markdown 链接。特别地，可以使用比较方便的 [`SomeThing`] 格式的交叉引用标注符语法 —— 括号内的`Something`将会在最终文档中被添加正确的超链接。
* 多添加示例代码：如果接口应该如何使用并非一目了然的，那么添加一个使用该接口的`# Example`段落将会很有用。如在[文档注释]里的示例代码会在你执行`cargo test`（详情查看[第 13 条]）时编译并且运行，这一特性将有助于示例代码和它希望表述的代码保持一致。
* 为`panic`和`unsafe`的代码添加说明文档：如果存在会导致函数`panic`的输入，在文档（`# Panics`段落）里说明规避`panic!`的前置条件。同样地，在文档（`# Safety`段落）里说明`unsafe`代码的使用要求。

Rust 的[标准库]是一个能够实践了上述所有细节的优秀示例。

## 工具

在注释文档中使用 Markdown 格式不仅意味着优美的输出，还意味着需要有一个明确的转换步骤（`cargo doc`）。而转换也就会增加出现问题的可能性。

对于这个问题，最简单的建议是在写完文档后，运行`cargo doc --open`（或者`cargo doc --no-deps --open`，这个指令能够严格约束仅产生当前包中的文档）并来*仔细阅读生成的结果*。

对于所有生成超链接的有效性，你当然可以人工地去校验它们，或者让机器来完成这项工作 —— 通过`broken_intra_dock_links`的包特性[^1]：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
#![deny(broken_intra_doc_links)]

/// The bounding box for a [`Polygone`].
#[derive(Clone, Debug)]
pub struct BoundingBox {
    // ...
}
```

当特性生效的时候，`cargo doc`将会找出无效的链接：
```rust
error: unresolved link to `Polygone`
 --> docs/src/main.rs:4:30
  |
4 | /// The bounding box for a [`Polygone`].
  |                              ^^^^^^^^ no item named `Polygone` in scope
  |
```

你也可以设置要求文档化，通过在包里设置`![warn(missing_docs)]`属性。当设置生效的时候，编译器将会给每个未配置文档的公开条目生成警告信息。然而，这样设置也存在着为了解决编译器报错而提供低质量注释文档的风险 —— 当然设置引入的问题不仅如此。

同样地，为了能够及时发现潜在风险，这些工具应该被纳入到你的持续集成系统（[第 32 条]）。

## 其他的文档位置

`cargo doc`的输出是包中文档所在的主要位置，但并不是唯一的地方 —— 在项目中的其他地方添加注释也可以帮助用户理解如何使用你的代码。

在 Cargo 项目的`examples/`子目录下可以添加一些方便使用包的示例代码。这些代码可以构建并运行，和集成测试（[第 30 条]）的运行方式非常类似，不同的是这些代码提供的是便于理解包中接口使用的代码。

需要说明的是，`tests/`子目录下的集成测试代码也可以给用户提供帮助，虽然它们的主要作用是测试包的对外接口。

## 发布包的文档

如果你的包会发布到`crates.io`，项目的文档就可以在[docs.rs]中查看到。docs.rs 是为发布的包构建并提供文档的官方 Rust 网站。

注意，`crates.io`和`docs.rs`的受众是不同的：`crates.io`旨在为选择包的用户提供服务，而`docs.rs`的受众是那些需要弄明白他们已经引用的包该如何使用的人（很明显的，这两种场景有很大的重叠）。

综上，一个包的主页在不同的地方会展示不同的内容：
* `docs.rs`：展示`cargo doc`产出结果的顶层页面，比如从顶层`src/lib.rs`文件的`//!`生成的文档。
* `crates.io`：展示包含在项目仓库中的任何顶层*README.md* [^2]文件内容。

## 不文档化的内容

当一个项目*要求*公共条目都需要添加注释的时候，很容易就陷入到给无价值的内容也文档化的陷阱中。编译器的缺少注释文档的警告只是提醒你添加真正需要内容 —— 有用的文档 —— 的一种表现，并且仅仅期望程序员添加必要的内容来消除警告。

好的注释文档是一种能够帮助用户了解他们所使用代码的福利；糟糕的注释文档则增加了代码的维护成本并且让用户在它们不再和代码保持一致的时候变得更加困惑。那么好与不好的区别是什么呢？

最重要的建议是*避免重复可以从代码中看出的信息*。[第 1 条]建议你的代码尽量的和 Rust 的类型系统保持一致；一旦你做到了这一点，就通过类型系统来说明这些语意。可以假定使用代码的用户对 Rust 已经熟悉了 —— 可能他们已经读了一些描述了如何高效使用语言的建议 —— 并且不需要重复从代码中的参数类型和函数签名中就能读出来的东西。

回到之前的例子，一个冗余的注释文档可能如下面描述的这样：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// Return a new [`BoundingBox`] object that exactly encompasses a pair
/// of [`BoundingBox`] objects.
///
/// Parameters:
///  - `a`: an immutable reference to a `BoundingBox`
///  - `b`: an immutable reference to a `BoundingBox`
/// Returns: new `BoundingBox` object.
pub fn union(a: &BoundingBox, b: &BoundingBox) -> BoundingBox {
```

这个注释重复了很多从函数签名中就能读到的信息，注释信息毫无益处。

更糟的是，考虑一种代码重构后，将结果存储到其中一个参数（这是一种不兼容的变更；参照[第 21 条]）。没有编译器或者工具能够发现注释没有随之更新，结果就产生了一个未能和代码逻辑保持一致的注释：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// Return a new [`BoundingBox`] object that exactly encompasses a pair
/// of [`BoundingBox`] objects.
///
/// Parameters:
///  - `a`: an immutable reference to a `BoundingBox`
///  - `b`: an immutable reference to a `BoundingBox`
/// Returns: new `BoundingBox` object.
pub fn union(a: &mut BoundingBox, b: &BoundingBox) {
```

相反地，原本恰当的注释在重构中则可以毫发无损地保留下来，因为它的文本描述的是行为，而非语意本身：

```rust
/// Calculate the [`BoundingBox`] that exactly encompasses a pair
/// of [`BoundingBox`] objects.
pub fn union(a: &mut BoundingBox, b: &BoundingBox) {
```

先前的建议也可以帮助提升文档质量：*在文档中包含任何从代码中无法了解的内容*。这包含前置条件、可变性、异常、报错条件以及任何可能会让用户感到意外的事情；如果你的代码不能遵守[最小惊讶原则]，确保这些意外都被记录在文档里，至少你可以说“我已经告诉过你了”。

另一个常见的失败情形是，注释里描述了其他使用这个方法的代码，而非这个方法做了什么：

```rust
/// Return the intersection of two [`BoundingBox`] objects, returning `None`
/// if there is no intersection. The collision detection code in `hits.rs`
/// uses this to do an initial check to see whether two objects might overlap,
/// before performing the more expensive pixel-by-pixel check in
/// `objects_overlap`.
pub fn intersection(
    a: &BoundingBox,
    b: &BoundingBox,
) -> Option<BoundingBox> {
```

像这样的注释几乎不可能和代码保持一致：当使用了这个方法的代码（比如，`hits.rs`）变更的时候，这段描述了调用行为的注释相隔甚远而无法保持一致。

应当将注释重新组织以聚焦在*为什么*这样使用，可以让这段注释更好的适应未来的变更。

```rust
/// Return the intersection of two [`BoundingBox`] objects, returning `None`
/// if there is no intersection.  Note that intersection of bounding boxes
/// is necessary but not sufficient for object collision -- pixel-by-pixel
/// checks are still required on overlap.
pub fn intersection(
    a: &BoundingBox,
    b: &BoundingBox,
) -> Option<BoundingBox> {
```

当编写软件时，“面向未来的编程”[^3]是一种很好的实践：调整代码结构以适应未来的变更。同样的原则也适用于文档：聚焦在语意，为什么这样做以及为什么不这样做，会让文本在未来的运行中始终是有意义的。

## 总结

* 给公共的 API 内容添加注释文档。
* 为那些从代码中无法明确看出的内容添加描述 —— 比如`panics`以及`unsafe`的条件。
* 不要给可以从代码中明确看出的内容重复描述。
* 通过交叉引用及添加标志符来让导航变得明确。

---

### 注释

[^1]: 这个配置也曾称成为`intra_doc_link_resolution_failure`。
[^2]: 包含 *README.md* 的引用动作可以被[Cargo.toml 中的 readme 字段]覆盖。
[^3]: Scott Meyers，More Effective C++ (Addison-Wesley)，第 32 条。


原文[点这里]查看。

<!-- 参考链接 -->

[Effective Java]: https://www.oreilly.com/library/view/effective-java-2nd/9780137150021/
[Rust book]: https://doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html#making-useful-documentation-comments
[文档注释]: https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html
[第 13 条]: ../chapter_2/item13-use-default-impl.md
[标准库]: https://doc.rust-lang.org/std/index.html
[第 32 条]: ./item32-ci.md
[第 30 条]: ./item30-write-more-than-unit-tests.md
[docs.rs]: https://docs.rs/
[第 1 条]: ../chapter_1/item1-use-types.md
[第 21 条]: ../chapter_4/item21-semver.md
[最小惊讶原则]: https://en.wikipedia.org/wiki/Principle_of_least_astonishment
[点这里]: https://www.lurklurk.org/effective-rust/documentation.html
[Cargo.toml 中的 readme 字段]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-readme-field
