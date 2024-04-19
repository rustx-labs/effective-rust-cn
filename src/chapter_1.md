# 类型

这本书的第一部分涵盖了关于 Rust 类型系统的建议。Rust 的类型系统比其他主流语言的表达能力更强；它与“学术性”语言如 [OCaml] 或 [Haskell] 有更多共同点。

其中核心的一部分是 Rust 的枚举类型（`enum`），它比其他语言中的枚举类型具有更强的表达能力，并且允许使用[代数数据类型]。

Rust 类型系统的另一个核心支柱是特征（`trait`）类型。特征大致等同于其他语言中的接口类型，但它们也与 Rust 的 _泛型_（[方法 12]）相关联，允许在不产生运行时开销的情况下重用接口。

[OCaml]: https://ocaml.org/
[Haskell]: https://www.haskell.org/
[代数数据类型]: https://en.wikipedia.org/wiki/Algebraic_data_type
[方法 12]: https://www.lurklurk.org/effective-rust/generics.html
