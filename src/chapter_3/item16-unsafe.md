# 第 16 条：避免写 unsafe 代码

Rust 独特的卖点就是其内存安全保证 —— 而且没有运行时开销 —— 这是其他任何主流语言所没有的特性。但这种保证是有代价的：写 Rust 需要你重新组织你的代码来通过借用检查器（[第 15 条]）并精确指定你使用的引用类型（[第 8 条]）。

Unsafe Rust 是 Rust 语言的超集，削弱了一些限制 —— 以及对应的保证。代码块前面加上 `unsafe` 就可以切换到 unsafe 模式，这会允许通常情况下 Rust 所不支持的操作。特别是，它允许使用*裸指针*，就像旧式 C 指针一样。这些指针不受借用规则的约束，程序员有责任保证他们在解引用时仍指向有效的内存。

因此浅显地说，本条款的建议很简单：如果你只想用 Rust 来编写 C 代码，为什么要用 Rust？然而，在某些情况下，`unsafe` 代码是必要的：对于底层库或当你的 Rust 代码必须与其他语言代码交互时（[第 34 条]）。

本条款的措辞也是相当精确的：*避免**写** `unsafe` 代码*。重点在于“写”，因为大多数时候，你可能需要的 `unsafe` 代码都已经为你编写好了。

Rust 标准库包含大量 `unsafe` 代码；快速查找发现 `alloc` 库中大约有 1000 个`unsafe` 使用，`core` 中有 1,500 个，`std` 中最多，有 2,000 个。这些代码都由专家进行编写，并在数千个 Rust 代码库中久经考验。

其中一些 `unsafe` 代码在我们介绍过的标准库功能的底层：

- [第 8 条]描述的智能指针类型（`Rc`、`RefCell` 和 `Arc` 等）在底层上都使用 `unsafe` 代码（通常是裸指针），以便向用户呈现其特定的语义。
- [第 17 条]中的同步原语（`Mutex`、`RwLock` 和其他相关守卫）在内部使用 `unsafe` 的操作系统特定代码。如果你想了解这些原语中设计的微妙细节，推荐阅读 Mara Bos (O'Reilly) 的[《Rust Atomics and Locks》]。

标准库中还有一些涵盖了更高级特性的功能，其内部也使用 `unsafe` 实现：[^1]

- [`std::pin::Pin`] 强制数据不能在内存中移动（[第 15 条]）。这会允许自引用数据结构，这对于 Rust 新手通常来说是非常令人讨厌的（[bête noire]）。
- [`std::borrow::Cow`] 提供了写时克隆智能指针：同一个指针可用于读取和写入，并且只当写入时才会发生底层数据的克隆。
- [`std::mem`] 中的这些函数（[`take`]、[`swap`] 和 [`replace`]）允许操作内存中的数据，而不会与借用检查器发生冲突。

这些功能可能仍需要小心一些才能正确使用，但 `unsafe` 代码已经通过消除该类问题的方式封装起来了。

除了标准库以外，[`crates.io`] 生态也有很多封装了 `unsafe` 代码来提供常用功能的 crate：

- [`once_cell`]：提供了一种类似全局变量的方法，只初始化一次。
- [`rand`]：提供随机数生成，利用操作系统和 CPU 提供的较低级别的底层功能。
- [`byteorder`]：允许数据在原始字节（raw bytes）与数字（number）之间进行转换。
- [`cxx`]：允许 C++ 代码和 Rust 代码进行互操作（也在[第 35 条]提到）。

还有很多其他例子，但是期望的总体思路是清楚的。如果你想做的事情显然不符合 Rust 的约束（尤其对于[第 14 条]和[第 15 条]），搜索标准库，查看是否已有功能可以满足你的需要。如果你没有找到所需的，也可以尝试在 `crates.io` 中搜索。毕竟，遇到其他人从来没遇到过的独特问题是不寻常的。

当然，总有一些地方强制使用 `unsafe`，比如当你想要通过外部函数接口（FFI）编写的代码进行交互时，正如[第 34 条]中讨论的那样。但当必要时，**考虑编写一个包装层来保存所有的 `unsafe` 代码**，以便其他程序员可以遵循本条款给出的建议。这也有助于定位问题：当出现问题时，`unsafe` 的包装层应是首先被怀疑的对象。

另外，如果你被迫编写 `unsafe` 代码，注意关键字本身蕴含的警告：此处有龙（[Hic sunt dracones]，中世纪航海术语，用来描述该地域很危险）。

- 添加[安全注释]，记录 `unsafe` 代码依赖的前提条件和不变量。Clippy（[第 29 条]）有一个[警告]来提醒您这一点。
- 最小化使用 `unsafe` 代码块，以限制错误影响的潜在范围。考虑启用 [`unsafe_op_in_unsafe_fn` lint]，以便于执行 `unsafe` 操作时需要显式的 `unsafe` 代码块，甚至这些操作本身就是在 `unsafe` 函数中执行的。
- 编写比平时更多的测试（[第 30 条]）。
- 对代码运行附加诊断工具（[第 31 条]）。特别是，**考虑在 `unsafe` 代码上运行 Miri** —— [Miri] 会解释编译器中间层的输出，使其能够检测到 Rust 编译器无法察觉到的某类错误。
- 仔细考虑多线程的使用，特别是有共享状态情况下（[第 17 条]）。

添加 `unsafe` 标记并不意味着不再有任何规则适用 —— 这意味着现在是*你*（程序员）来负责维护 Rust 的安全保证，而不是编译器。

## 注释

[^1]: 实际上，大多数 `std` 功能实际上都由 `core` 提供，因此可用于 `no_std` 代码，如[第 33 条]所述。

原文[点这里](https://www.lurklurk.org/effective-rust/unsafe.html)查看

<!-- 参考链接 -->

[第 8 条]: ../chapter_1/item8-references&pointer.md
[第 14 条]: ../chapter_3/item14-lifetimes.md
[第 15 条]: ../chapter_3/item15-borrows.md
[第 17 条]: ../chapter_3/item17-deadlock.md
[第 29 条]: ../chapter_5/item29-listen-to-clippy.md
[第 30 条]: ../chapter_5/item30-write-more-than-unit-tests.md
[第 31 条]: ../chapter_5/item31-use-tools.md
[第 33 条]: ../chapter_6/item33-no-std.md
[第 34 条]: ../chapter_6/item34-ffi.md
[第 35 条]: ../chapter_6/item35-bindgen.md

[《Rust Atomics and Locks》]: https://marabos.nl/atomics/
[`std::pin::Pin`]: https://doc.rust-lang.org/std/pin/struct.Pin.html
[bête noire]: https://rust-unofficial.github.io/too-many-lists/
[`std::borrow::Cow`]: https://doc.rust-lang.org/std/borrow/enum.Cow.html
[`std::mem`]: https://doc.rust-lang.org/std/mem/index.html
[`take`]: https://doc.rust-lang.org/std/mem/fn.take.html
[`swap`]: https://doc.rust-lang.org/std/mem/fn.swap.html
[`replace`]: https://doc.rust-lang.org/std/mem/fn.replace.html
[`crates.io`]: https://crates.io/
[`once_cell`]: https://docs.rs/once_cell
[`rand`]: https://docs.rs/rand
[`byteorder`]: https://docs.rs/byteorder
[`cxx`]: https://docs.rs/cxx
[Hic sunt dracones]: https://en.wikipedia.org/wiki/Here_be_dragons
[安全注释]: https://std-dev-guide.rust-lang.org/policy/safety-comments.html
[警告]: https://rust-lang.github.io/rust-clippy/master/index.html#/missing_safety_doc
[`unsafe_op_in_unsafe_fn` lint]: https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#unsafe-op-in-unsafe-fn
[Miri]: https://github.com/rust-lang/miri
