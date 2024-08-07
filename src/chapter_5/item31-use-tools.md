# 第 31 条：使用生态系统工具

Rust 生态系统拥有丰富的附加工具集，它们提供了超出将 Rust 转换为机器代码基本任务的功能。

在设置Rust开发环境时，您可能希望拥有以下基本工具的大多数：[^1]

* [cargo] 工具，用于组织依赖项（[第 25 条]）并驱动编译器
* [rustup] 工具，用于管理安装的 Rust 工具链
* 支持 Rust 的 IDE，或像 [rust-analyzer] 这样的 IDE/编辑器插件，它允许您快速在 Rust 代码库中导航，并为编写 Rust 代码提供自动补全支持
* [Rust playground]，用于独立探索 Rust 的语法，并与同事分享结果
* [Rust 标准库]的文档

除了这些基本工具，Rust 还包括许多帮助维护代码库和提高代码库质量的工具。官方 Cargo 工具链中包含的工具涵盖了除 `cargo build`、`cargo test` 和 `cargo run` 基本操作之外的各种基本任务，例如：
* `cargo fmt`：根据标准约定重新格式化 Rust 代码。
* `cargo check`：执行编译检查而不生成机器代码，这有助于快速进行语法检查。
* `cargo clippy`：执行代码嗅探检查，检测低效或不地道的代码（[第 29 条]）。
* `cargo doc`：生成文档（[第 27 条]）。
* `cargo bench`：运行基准测试（[第 30 条]）。
* `cargo update`：升级依赖项到最新版本，默认选择符合语义版本控制（[第 21 条]）的版本。
* `cargo tree`：显示依赖关系图（[第 25 条]）。
* `cargo metadata`：输出工作空间中存在的包及其依赖项的元数据。

最后提到的这个工具特别有用，尽管它是间接的：因为有一个工具以明确定义的格式输出关于 `crates` 的信息，人们更容易生产出其他使用这些信息的工具（通常通过 `cargo_metadata crate`，它提供了一套 Rust 类型来持有元数据信息）。

[第 25 条]描述了一些由这种元数据可用性启用的工具，例如 `cargo-udeps`（允许检测未使用的依赖项）或 `cargo-deny`（允许检查许多事情，包括重复的依赖项、允许的许可证和安全建议）。

Rust 工具链的可扩展性不仅限于包元数据；编译器的抽象语法树也可以构建，通常通过 `syn crate`。这些信息使得过程宏（[第 28 条]）变得强大，但也为各种其他工具提供了支持：
* `cargo-expand`：显示宏扩展产生的完整源代码，这对于调试复杂的宏定义至关重要。
* `cargo-tarpaulin`：支持生成和跟踪代码覆盖率信息。

任何特定工具的列表总是主观的、过时的和不完整的；更一般的观点是探索可用的工具。

例如，搜索 `cargo-<something>` 工具会得到数十个结果；其中一些可能不合适，一些可能已被放弃，但有些可能正好符合您的需求。

还有各种努力将形式验证应用于 Rust 代码，如果您的代码需要对其正确性有更高层次的保证，这可能会有帮助。

最后，提醒一下：如果一个工具不仅仅是一次性使用，您应该将该工具集成到您的 CI 系统（如[第 32 条]所述）。如果该工具快速且没有误报，将其集成到您的编辑器或 IDE 中也是合理的；`Rust Tools` 页面提供了相关文档的链接。

### 需要记住的工具

除了应该定期和自动运行在您的代码库上的工具（[第 32 条]），书中其他地方还提到了各种其他工具。为了参考，这里将它们汇集在一起——但请记住，还有更多的工具存在：
* [第 16 条]建议在编写微妙的 `unsafe` 代码时使用 `Miri`。
* [第 21 条]和[第 25 条]提到了 `Dependabot`，用于管理依赖项更新。
* [第 21 条]还提到了 `cargo-semver-checks` 作为检查语义版本控制是否正确完成的可能选项。
* [第 28 条]解释了 `cargo-expand` 在调试宏问题时可以提供帮助。
* [第 29 条]完全致力于使用 `Clippy`。
* `Godbolt` 编译器探索器允许您探索与您的源代码对应的机器代码，如[第 30 条]所述。
* [第 30 条]还提到了其他测试工具，例如用于模糊测试的 `cargo-fuzz` 和用于基准测试的 `criterion`。
* [第 35 条]涵盖了使用 `bindgen` 从 `C` 代码自动生成 `Rust FFI` 包装器的使用。

#### 注释

[^1]: 在某些环境中，这个列表可能会减少。例如，在 `Android` 上进行 `Rust` 开发有一个集中控制的工具链（所以没有 `rustup`），并与 `Android` 的 `Soong` 构建系统集成（所以没有 `cargo`）。

<!-- 参考链接 -->
[第 16 条]: ../chapter_3/item16-unsafe.md
[第 21 条]: https://www.lurklurk.org/effective-rust/semver.html
[第 25 条]: https://www.lurklurk.org/effective-rust/dep-graph.html
[第 27 条]: ./item27-document-public-interfaces.md
[第 28 条]: ./item28-use-macros-judiciously.md
[第 29 条]: ./item29-listen-to-clippy.md
[第 30 条]: https://www.lurklurk.org/effective-rust/testing.html
[第 32 条]: https://www.lurklurk.org/effective-rust/ci.html
[第 35 条]: ../chapter_5/item31-use-tools.md

[cargo]: https://doc.rust-lang.org/cargo/
[rustup]: https://github.com/rust-lang/rustup
[rust-analyzer]: https://github.com/rust-lang/rust-analyzer
[Rust playground]: https://play.rust-lang.org/
[Rust 标准库]: https://doc.rust-lang.org/std/
