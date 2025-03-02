# Traits

Rust 类型系统的第二个核心支柱是 trait 的使用，它允许编码在不同类型之间通用的行为。Trait 在其他语言中大致等同于接口类型，但它们也与 Rust 的泛型（[第 12 条]）相关联，允许在不产生运行时开销的情况下重用接口。

本章中的条款描述了 Rust 编译器和 Rust 工具链提供的标准 trait，并提供了关于如何设计和使用 trait 编码行为的建议。

## 注释

原文[点这里](https://www.lurklurk.org/effective-rust/traits.html)查看

<!-- 参考链接 -->

[第 12 条]: ../chapter_2/item12-generics&trait-objects.md
