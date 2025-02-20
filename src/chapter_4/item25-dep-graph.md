# 第 25 条：管理依赖项关系图

和大多数现代编程语言一样，Rust 通过 *crate* 的形式，简化了引入外部库的过程。很多相对复杂的 Rust 程序都会使用外部 crate，而这些 crate 本身可能也会有其他依赖，从而为整个程序形成一个*依赖项关系图*（简称依赖图）。

默认情况下，对于 *Cargo.toml* 文件里 `[dependencies]` 节中的 crate，Cargo 会从 [`crates.io`][crates.io] 查找满足需求的版本并下载。

在这个简单的模式之下隐藏着一些微妙之处。首先要注意的是，来自 `crates.io` 的 crate 名称构成了一个扁平的明明空间 —— 并且，全局命名空间也会和 crate 中*特征* （features）的命名产生重叠（见[第 26 条]）[^1]。

关于 crate 的名字，`crates.io` 采用“先到先得”的机制。所以，如果你计划将自己的 crate *发布*到 `crates.io`，那么请先查找一下要使用的名字是否已经被占用。然而，也[不赞成][frowned upon]发布一个空的 crate 来预占一个名字，除非你可以在短时间内发布代码。

需要注意的是，命名空间中的 crate 名称和在代码中使用时的名称略有不同，一个 crate 可以叫做 `some-crate`，但是在代码中使用的时候它的名字是 `some_crate`（中划线变成下划线）。也就是说，如果在代码中看到 `some_crate`，那么对应的 crate 名字可能是 `some-crate` 也可能是 `some_crate`。

还有一点，Cargo 允许在构建中包含同一个 crate 的多个版本，哪怕它们的版本号从语义化版本角度而言是不兼容的。看起来有点儿出乎意料，毕竟在 *Cargo.toml* 中，每个依赖项只能指定一个版本。但是实际上这种情况会经常发生：你的 crate 依赖 `some-crate` 的 3.x 版本，但同时也依赖 `older-crate`，而后者又依赖 `some-crate` 的 1.x 版本。

如果你只是在代码内部使用它们还好说，但是如果需要把依赖项中的类型暴露出来的话，就会导致一些冲突（见[第 24 条]） —— 编译器倒是可以将不同版本视为不同的 crate 来处理，但是编译期间的错误消息可就没有这么清晰了。

如果 crate 中通过 Rust FFI 机制（见[第 34 条]）使用了 C/C++ 的代码，那么在其中引入其他 crate 的多个不同版本可能会导致错误。Rust 的工具链可以在代码中区分多个版本，但是所包含的 C/C++ 代码却必须遵从[单一定义原则][one definition rule]：对于任何函数、常量、全局变量，只能有一个定义。

Cargo 对于同一 crate 多版本支持也是有一定限制的：它*不*允许多个语义化版本号兼容的版本（见[第 21 条]）：

- `some-crate` 1.2 和 `some-crate` 3.1 可以共存
- `some-crate` 1.2 和 `some-crate` 1.3 不可以共存

在语义化版本号的基础上，Cargo 也做了一些扩展。它将最左非零的版本号段当作主版本号来看待：

- `other-crate` 0.1.2 和 `other-crate` 0.2.0 可以共存
- `other-crate` 0.1.2 和 `other-crate` 0.1.4 不可以共存

Cargo 的[版本选择算法][version selection algorithm]决定应该使用的版本。根据语义化版本的规则，*Cargo.toml* 中的每个依赖项都有一系列可接受的版本，当一个 crate 在依赖图中多次出现时，Cargo 会将这个因素考虑进去。如果多个重叠的版本之间是兼容的，默认情况下，Cargo 会选择最新的；如果多个版本是不兼容的，那么 Cargo 会为每个版本单独构建一份拷贝。

Cargo 对所有的依赖项都确定了最终所选的版本之后，选择结果会记录到 *Cargo.lock* 文件中。后续的构建过程就会重用 *Cargo.lock* 中的数据，以确保构建结果的稳定性，并且可以减少不必要的内容下载。

那么，你就面临一个选择：是否应该把 *Cargo.lock* 文件提交到版本控制系统？[Cargo 开发者给出的建议][version selection algorithm]是：

- 构建输出是一个应用，或者二进制文件的，应该把 *Cargo.lock* 提交到版本控制系统，以确保稳定的构建输出。
- 库 crate *不*应该提交 *Cargo.lock*，因为使用这个 crate 的项目会有自己的 *Cargo.lock*；**要知道，库 crate 中的 Cargo.lock 文件会被库的使用者忽略**。

即使对于库 crate，提交 *Cargo.lock* 文件会有助于常规构建以及持续集成（CI，见[第 32 条]）的过程中保持稳定的输出。理论上，语义化版本（见[第 21 条]）会防止由于版本号导致的构建错误，但是实际上这种错误时有发生。仅仅是因为某人变更了一个依赖项的依赖就导致构建失败，实在是令人懊恼。

**如果把 Cargo.lock 提交到版本控制系统，那么需要建立一种升级流程**（例如 Github 的 [Dependabot][dependabot]），否则，你的依赖项就会固定在一个老旧的、过时的、甚至存在安全风险的版本。

通过提交 *Cargo.lock* 文件到版本控制系统从而达到固定依赖项版本的做法并不能解决依赖项升级的问题，但是它可以让你选择合适的时机来进行升级，而不是自动跟随所依赖的 crate 的升级而升级。如果依赖项的更新版本中存在问题，那么通常来说他们会快速修复，手动选择何时升级依赖项版本的做法可以让你跳过存在问题的版本，使用更新后的版本。

第三点，Cargo 的解析过程是特征统一的：在依赖图中不同位置的同一个 crate 的多个版本中的多个特征会形成特征并集，详细信息参见[第 26 条]。

## 版本规范

根据 [Cargo book][cargo book] 中的规则，依赖项的版本号实际上定义了一系列允许的版本：

- *避免指定过于精确的版本号*： 让依赖项锚定到一个精确的版本（`"=1.2.3"`）通常来说并不是一个好主意，你将无法看到新版本（可能包含了安全问题修复），并且大大缩小了依赖图中和其他使用当前依赖的 crate 之间的所允许的版本重叠范围（记住，在语义化版本兼容的范围内，Cargo 仅允许一个版本生效）。你可以借助 *Cargo.lock* 来达到在构建过程中使用一致的依赖集合的目的。
- *避免指定过于宽泛的版本号*：你当然*可以*通过将依赖项的版本号设置为 `"*"` 来允许它的*任意*版本，但这也不是一个好主意。如果一个 crate 对 API 做出了重大变更，发布了一个主版本更新，那么在 `cargo update` 拉取到新版本之后，你的代码大概率上无法正常工作了。

别太精确，也别太宽泛，指定一个语义化版本兼容的（如：`"1"`）版本号，也可以带上次版本和补丁版本（如：`"1.4.23"`），这就是普遍适用的“恰到好处”的规范。

这些版本规范都是 Cargo 的默认行为，它支持与指定的版本语义化兼容的版本，你也可以在版本号前面加入 `^` 让表意更加清晰：

- 版本号 `"1"` 等同于 `"^1"`，允许 1.x 所有的版本，所以它也等同于 `"1.*"`。
- 版本号 `"1.4.23"` 等同于 `"^1.4.23"`，允许高于 1.4.23 版本的任意 1.x 版本。

## 使用工具解决问题

在[第 31 条]中，建议你充分利用 Rust 生态系统提供的优秀工具。本节讲解用来解决依赖图问题的相关工具。

编译器可以很快的告诉你代码中使用了未在 *Cargo.toml* 中定义的依赖项。反之，如果是在 *Cargo.toml* 中定义了，但是并未在代码中使用的依赖，或者说，代码中曾经用过，但是现在不用了，应该如何处理？此时，我们可使用 [`cargo-udeps`][cargo-udeps] 工具，它可以告诉你，*Cargo.toml* 中包含了未曾使用的依赖。

另外一个好用的工具叫做 [`cargo-deny`][cargo-deny]，它可以分析依赖项关系图，然后在整个依赖集合中检测潜在的问题：

- 所用依赖项版本存在安全问题
- 所用依赖项包含不可接受的许可证
- 所用依赖项不可接受
- 所用依赖项在依赖树上有多个不同版本

这些功能中的每一项都可以单独配置，用来处理例外情况。尤其是针对大型项目，通常会有多版本警告。随着依赖关系图的增长，依赖项之间的多版本警告是在所难免的。但是值得我们尝试使用工具尽量减少这种情况，虽然有时候没有办法完全避免，但是至少可以减少二进制构建文件的大小，提升编译速度。

你可以在需要的时候使用这些工具，但是更好的办法是，将这些工具包含到 CI 系统中，以确保它们可以定期可靠的执行（[第 32 条]），这有助于发现新的问题，包括在你代码之外的、上游依赖项中的诸如安全漏洞的相关问题。

如果上述工具报告了一个问题，往往很难精准地定位依赖图中发生问题的地方。此时，可以使用 `cargo` 中的命令 [`cargo tree`][cargo tree] 查看依赖树：

```
dep-graph v0.1.0
├── dep-lib v0.1.0
│   └── rand v0.7.3
│       ├── getrandom v0.1.16
│       │   ├── cfg-if v1.0.0
│       │   └── libc v0.2.94
│       ├── libc v0.2.94
│       ├── rand_chacha v0.2.2
│       │   ├── ppv-lite86 v0.2.10
│       │   └── rand_core v0.5.1
│       │       └── getrandom v0.1.16 (*)
│       └── rand_core v0.5.1 (*)
└── rand v0.8.3
    ├── libc v0.2.94
    ├── rand_chacha v0.3.0
    │   ├── ppv-lite86 v0.2.10
    │   └── rand_core v0.6.2
    │       └── getrandom v0.2.3
    │           ├── cfg-if v1.0.0
    │           └── libc v0.2.94
    └── rand_core v0.6.2 (*)
```

`cargo tree` 命令有一系列的选项可以帮助解决特定问题：

- `--invert`：显示依赖某个特定包的依赖项，可以聚焦特定的有问题的依赖项
- `--edges features`：显示一个依赖项链接激活了哪些特征（feature），可以帮助你搞清楚特征统一时的情况（[第 26 条]）
- `--duplicates`：显示在依赖图中存在多个版本的依赖项

## 依赖什么？

前面的小节讲解了依赖项之间的工作原理，但是还有一个更加哲学性的问题（因此也更加难以回答）：何时使用依赖项？

大部分时候，比较容易做出决定：如果你的 crate 需要一个函数，唯一可选的替代方案就是你自己实现它[^2]。

但是每个依赖都是有相应的成本的，完全由自己实现所需的功能，除了更长的构建时间，以及更大的二进制输出文件，还需要付出极大的努力来修复依赖中发现的问题。

你的项目的依赖图越庞大，就越容易遇到上面提到的问题。和其他包管理生态系统一样，Rust 的 crate 生态系统也会遇到依赖项问题。历史表明，一旦[开发人员移除一个包][remove package]，或者团队[修改了许可证][fix license]，都会带来大范围的连锁反应。

更加令人担忧的供应链攻击。别有用心的攻击者会通过[误植域名][typo]、[劫持维护者账号][hijack]或者其他更加精巧且隐蔽的方式，来破坏被广泛使用的公共依赖项，从而达到攻击的目的。

千万不要以为攻击行为仅在运行代码的时候才会发生，在*编译*期间，依赖项可以通过 [`build.rs`][build.rs] 或者过程宏也可以执行任意代码，这就意味着被破坏的依赖项可以在你的 CI 系统中偷偷运行挖矿程序！

因此，应该慎重考虑是否把那些无关紧要的依赖引入到你的项目中。

当然了，大部分时候的答案肯定的。毕竟，修复依赖项问题所花费的时间远远低于你自己从头开始实现依赖项的功能。

## 牢记

- `crates.io` 上的 crate 名称，形成了一个扁平的命名空间，且与 crate 的特征名称共享。
- Crate 的名字中可以包含中划线（`-`），但是在代码中使用的时候，会转换成下划线（`_`）。
- Cargo 支持同一个依赖项在依赖图中存在多个版本，前提是这些版本在语义化版本的规则下是不兼容的。如果你的代码中包含 FFI 代码，多版本并存的依赖项可能会引发问题。
- 同一依赖项，最好使用在语义化版本层面兼容的版本（`"1"`，或者包含小版本号的版本：`"1.4.23"`）。
- 使用 *Cargo.lock* 来确保构建的可重做性，但是也要知道公开发布的 crate 不会携带 *Cargo.lock*。
- 使用工具协助解决依赖项问题，例如：`cargo truee`，`cargo deny`，`cargo udep` 等。
- 要理解，相比自己开发而言，使用现成的依赖项相可以节约时间，但是也有一定的附加成本

## 注释

[^1]: 除 [crates.io][crates.io] 之外，也可以配置[备用注册中心][alternate registry]（例如公司内部的注册中心）。*Cargo.toml* 中的每个依赖项都可以设置 `registry` 键来表明其来源注册中心。

[^2]: 如果你的目标是 `no_std` 环境，也没有太多其他选择了，因为很多 crate 都未适配 `no_std` 环境，尤其是当 `alloc` 也不可用的时候（见[第 33 条]）。

原文[点这里](https://www.lurklurk.org/effective-rust/dep-graph.html)查看

<!-- 参考链接 -->

[第 21 条]: item21-semver.md
[第 24 条]: item24-re-export.md
[第 26 条]: item26-features.md
[第 31 条]: ../chapter_5/item31-use-tools.md
[第 32 条]: ../chapter_5/item32-ci.md
[第 33 条]: ../chapter_6/item33-no-std.md
[第 34 条]: ../chapter_6/item34-ffi.md

[crates.io]: https://crates.io/
[alternate registry]: https://doc.rust-lang.org/cargo/reference/registries.html
[frowned upon]: https://rust-lang.github.io/rfcs/3463-crates-io-policy-update.html
[one definition rule]: https://en.wikipedia.org/wiki/One_Definition_Rule
[version selection algorithm]: https://doc.rust-lang.org/cargo/reference/resolver.html
[advice from the Cargo developers]: https://doc.rust-lang.org/cargo/faq.html#why-have-cargolock-in-version-control
[dependabot]: https://docs.github.com/en/code-security/dependabot
[cargo book]: https://doc.rust-lang.org/cargo/reference/resolver.html#semver-compatibility
[cargo-udeps]: https://crates.io/crates/cargo-udeps
[cargo-deny]: https://crates.io/crates/cargo-deny
[cargo tree]: https://doc.rust-lang.org/cargo/commands/cargo-tree.html
[remove package]: https://arstechnica.com/information-technology/2016/03/rage-quit-coder-unpublished-17-lines-of-javascript-and-broke-the-internet/
[fix license]: https://www.theregister.com/2021/03/25/ruby_rails_code/
[typo]: https://en.wikipedia.org/wiki/Typosquatting
[hijack]: https://eslint.org/blog/2018/07/postmortem-for-malicious-package-publishes/
[build.rs]: https://doc.rust-lang.org/cargo/reference/build-scripts.html
