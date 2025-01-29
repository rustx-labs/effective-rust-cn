# 后记

希望这本书中的建议、建议和信息能帮助你成为一个流利、富有成效的 `Rust` 程序员。正如前言所述，这本书旨在涵盖在此过程第二步，在你从核心 `Rust` 参考书籍中学到基础知识之后。但是，你可以采取更多的步骤和探索方向：
- 这本书没有涉及异步 Rust，但对于高效、并发服务器端应用程序来说，这是可能需要的。[在线文档]提供了异步的介绍，而即将出版的《[异步 Rust]》（由 Maxwell Flitton 和 Caroline Morton 撰写，O'Reilly，2024）可能也会有所帮助。
- 向另一个方向发展，裸机 Rust 可能与你的兴趣和要求相符。这超出了[第 33 条]对 no_std 的介绍，进入一个没有操作系统和分配的世界。综合 [Rust 在线课程]的[裸机 Rust] 部分提供了很好的介绍。
- 无论你的兴趣是低级还是高级，第三方开源包的 [`crates.io`] 生态系统都值得探索和贡献。像 [blessed.rs] 或 [lib.rs] 这样的精选摘要可以帮助导航大量可能的选择。
- Rust 讨论论坛，如 [Rust 语言论坛]或 [Reddit 的 r/rust]，可以提供帮助 —— 包括一个可搜索的索引，列出了之前提出（并回答了）的问题。
- 如果你发现自己依赖一个不是用 Rust 编写的现有库（如[第 34 条]所述），你可以尝试用 `Rust` 重写它（`RiiR`）。但不要[低估重现久经测试]、成熟的代码库所需的努力。
- 随着你在 Rust 方面变得更加熟练，Jon Gjengset 的《Rust for Rustaceans》（No Starch，2022）是高级 Rust 领域的重要参考。

祝你好运！


<!-- 参考链接 -->

[第 33 条]: chapter_6/item33-no-std.md
[第 34 条]: chapter_6/item34-ffi.md

[在线文档]: https://rust-lang.github.io/async-book/
[异步 Rust]: https://learning.oreilly.com/library/view/async-rust/9781098149086/
[裸机 Rust]: https://google.github.io/comprehensive-rust/bare-metal.html
[Rust 在线课程]: https://google.github.io/comprehensive-rust
[Rust 语言论坛]: https://users.rust-lang.org/
[Reddit 的 r/rust]: https://reddit.com/r/rust
[低估重现久经测试]: https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/

[blessed.rs]: https://blessed.rs/
[lib.rs]: https://lib.rs/
[`crates.io`]: https://crates.io/