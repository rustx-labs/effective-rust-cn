# 第 31 条：使用生态系统工具

Rust 生态系统拥有丰富的附加工具集，它们提供的功能远超出了将 Rust 转换为机器代码的基本任务。

在设置 Rust 开发环境时，你可能希望拥有以下基本工具中的大多数：[^1]

* [cargo] 工具，用于组织依赖项（[第 25 条]）并驱动编译器
* [rustup] 工具，用于管理安装的 Rust 工具链
* 支持 Rust 的 IDE，或像 [rust-analyzer] 这样的 IDE/编辑器插件，它允许你快速在 Rust 代码库中导航，并为编写 Rust 代码提供自动补全支持
* [Rust playground]，用于独立探索 Rust 的语法，并与同事分享结果
* [Rust 标准库]的文档

除了这些基本工具，Rust 还包括许多帮助维护代码库和提高代码库质量的工具。官方 Cargo 工具链中[包含的工具]涵盖了除 `cargo build`、`cargo test` 和 `cargo run` 基本操作之外的各种基本任务，例如：
* `cargo fmt`：根据标准约定重新格式化 Rust 代码。
* `cargo check`：执行编译检查而不生成机器代码，这有助于快速进行语法检查。
* `cargo clippy`：执行静态代码检查，检测低效或不地道的代码（[第 29 条]）。
* `cargo doc`：生成文档（[第 27 条]）。
* `cargo bench`：运行基准测试（[第 30 条]）。
* `cargo update`：升级依赖项到最新版本，默认选择符合语义版本控制（[第 21 条]）的版本。
* `cargo tree`：显示依赖关系图（[第 25 条]）。
* `cargo metadata`：输出工作空间中存在的包及其依赖项的元数据。

最后提到的这个工具特别有用，尽管作用是间接的：只要有一个工具可以用良好定义的格式输出关于 crates 的信息，人们要制作其他能使用这些信息的工具就容易多了（通常通过 `cargo_metadata` crate，它提供了一套 Rust 类型来持有元数据信息）。

[第 25 条]描述了一些由于这种元数据可获得性而产生的工具，例如 `cargo-udeps`（支持检测未使用的依赖项）或 `cargo-deny`（支持检测许多事情，包括重复的依赖项、允许的许可证和安全建议）。

Rust 工具链的可扩展性不仅限于包的元数据，编译器的抽象语法树也可以构建，通常通过 `syn` crate。这些信息使得过程宏（[第 28 条]）变得强大，也为其他各种工具提供了支持：
* [`cargo-expand`]：支持显示宏展开产生的完整源代码，这对于调试复杂的宏定义至关重要。
* [`cargo-tarpaulin`]：支持生成和跟踪代码覆盖率信息。

任何特定工具的列表总是主观的、过时的和不完整的；更一般的观点是探索可用的工具。

例如，[搜索 `cargo-<something>` 工具]会得到数十个结果；其中一些可能不合适，一些可能已被放弃，但有些可能正好符合你的需求。

还有各种努力[将形式验证应用于 Rust 代码]，如果你的代码需要对其正确性有更高层次的保证，这可能会有帮助。

最后，提醒一下：如果一个工具不仅仅是一次性使用，你应该将该工具集成到你的 CI 系统（如[第 32 条]所述）。如果该工具运行快速且没有误报，将其集成到你的编辑器或 IDE 中也是合理的；[Rust Tools] 页面提供了相关文档的链接。

## 需要记住的工具

除了应该定期和自动运行在你的代码库上的工具（[第 32 条]），书中其他地方还提到了各种其他工具。为了参考，这里将它们汇集在一起 —— 但请记住，还有更多的工具存在：
* [第 16 条]建议在编写微妙的 `unsafe` 代码时使用 [Miri]。
* [第 21 条]和[第 25 条]提到了用于管理依赖项更新 [Dependabot]。
* [第 21 条]还提到了将 [`cargo-semver-checks`] 作为检查语义版本控制是否正确完成的可能选项。
* [第 28 条]解释了 [`cargo-expand`] 在调试宏问题时可以提供帮助。
* [第 29 条]完全致力于使用 Clippy。
* [Godbolt 编译器管理器]允许你探索与你的源代码对应的机器代码，如[第 30 条]所述。
* [第 30 条]还提到了其他测试工具，例如用于模糊测试的 [`cargo-fuzz`] 和用于基准测试的 [`criterion`]。
* [第 35 条]涵盖了使用 [`bindgen`] 从 C 代码自动生成 Rust FFI 的包装器。

## 注释

[^1]: 在某些环境中，这个列表可能会减少。例如，[在 Android 上进行 Rust 开发](https://source.android.com/docs/setup/build/rust/building-rust-modules/overview)有一个集中控制的工具链（所以没有 `rustup`），并与 Android 的 Soong 构建系统集成（所以没有 `cargo`）。

原文[点这里](https://www.lurklurk.org/effective-rust/use-tools.html)查看

<!-- 参考链接 -->

[第 16 条]: ../chapter_3/item16-unsafe.md
[第 21 条]: ../chapter_4/item21-semver.html
[第 25 条]: ../chapter_4/item25-dep-graph.md
[第 27 条]: item27-document-public-interfaces.md
[第 28 条]: item28-use-macros-judiciously.md
[第 29 条]: item29-listen-to-clippy.md
[第 30 条]: item30-write-more-than-unit-tests.md
[第 32 条]: item32-ci.md
[第 35 条]: ../chapter_6/item35-bindgen.md

[cargo]: https://doc.rust-lang.org/cargo/
[rustup]: https://github.com/rust-lang/rustup
[rust-analyzer]: https://github.com/rust-lang/rust-analyzer
[Rust playground]: https://play.rust-lang.org/
[Rust 标准库]: https://doc.rust-lang.org/std/
[包含的工具]: https://doc.rust-lang.org/cargo/commands/index.html
[`cargo-expand`]: https://github.com/dtolnay/cargo-expand
[`cargo-tarpaulin`]: https://docs.rs/cargo-tarpaulin
[搜索 `cargo-<something>` 工具]: https://docs.rs/releases/search?query=cargo-
[将形式验证应用于 Rust 代码]: https://alastairreid.github.io/automatic-rust-verification-tools-2021
[Rust Tools]: https://rust-lang.org/tools
[Miri]: https://github.com/rust-lang/miri
[Dependabot]: https://docs.github.com/en/code-security/dependabot
[`cargo-semver-checks`]: https://github.com/obi1kenobi/cargo-semver-checks
[`cargo-expand`]: https://github.com/dtolnay/cargo-expand
[Godbolt 编译器管理器]: https://rust.godbolt.org
[`cargo-fuzz`]: https://github.com/rust-fuzz/cargo-fuzz
[`criterion`]: https://crates.io/crates/criterion
[`bindgen`]: https://rust-lang.github.io/rust-bindgen/
