# 第 21 条：理解语义化版本

> “如果我们承认语义化版本号（SemVer）是一个有损的评估，并且仅代表可能的变更范围的一个子集，那么就可以将其视为一个有局限的工具。”
>
> —— Titus Winters “[《Google 软件工程》（O'Reilly）]”

Rust 的包管理器 Cargo 允许使用*语义化版本号*（semver）自动选择依赖项的版本（见[第 25 条]）。*Cargo.toml* 中下面的内容：

```toml
[dependencies]
serde = "1.4"
```

表示针对这个依赖项，`cargo` 可以接受的语义化版本。关于可以接受的版本范围的详细描述可以参见[官方文档]，但是最常见的语义化版本如下所示：

- `"1.2.3"`：和 1.2.3 版本兼容的任何版本都可以
- `"^1.2.3"`：同上，语义上更清晰
- `"=1.2.3"`：指定的具体版本，不接受任何其他替代版本
- `"~1.2.3"`：和 1.2.3 版本兼容的任何版本，但是仅允许最后一段不同（也即：1.2.4 可以，但是 1.3.0 就不可以了）
- `"1.2.*"`：任何和通配符匹配的版本的都可以

表 4-1 列出了不同的语义化版本和其可接受版本的对照。

*表 4-1. Cargo 依赖项版本规范*

| 版本       | 1.2.2 | 1.2.3 | 1.2.4 | 1.3.0 | 2.0.0 |
| ---------- | ----- | ----- | ----- | ----- | ----- |
| `"1.2.3"`  | 否    | 是    | 是    | 是    | 否    |
| `"^1.2.3"` | 否    | 是    | 是    | 是    | 否    |
| `"=1.2.3"` | 否    | 是    | 否    | 否    | 否    |
| `"~1.2.3"` | 否    | 是    | 是    | 否    | 否    |
| `"1.2.*"`  | 是    | 是    | 是    | 否    | 否    |
| `"1.*"`    | 是    | 是    | 是    | 是    | 否    |
| `"*"`      | 是    | 是    | 是    | 是    | 是    |

Cargo 在选择依赖项的版本时，会在符合条件的版本中选择最新的。

语义化版本是 `cargo` 依赖处理过程中非常核心的概念，本条款将详细介绍语义化版本。

## 语义化版本基础

关于语义化版本的概述在[官方文档][官方文档_semver_summary]中已经明确了，这里我们将其摘录过来：

> 版本号的格式为：主版本.次版本.补丁版本。增加版本中各段数值的含义如下：
>
> - 主版本：当做了了一些不兼容了的变更时
> - 次版本：增加了一些功能，但是向后兼容
> - 补丁版本：修复了一些问题，同时保持向后兼容

语义化版本号[细则][细则_semver]中，还约定了非常重要的一点：

> 3. 软件包的某个版本一旦发布，**不可以**对其内容进行任何变更。如需变更，**必须**发布新的版本。

换句话说就是：

- 变更*任何*内容，都需要发布一个补丁版本
- 当需要*增加*内容，但是已经使用依赖包的用户可以正常编译并且应用可以正常运行，需要升级次版本号
- 当*移除*或者*改变* API 中的内容时，需要升级主版本号

语义化版本规则[文档][semver_item4]中，还有一条非常重要：

> 4. 初始版本的主版本号应该为 0（0.y.z），表示可能会做出较大的变更，这个版本的 API 应该被视为**不稳定**的版本

针对这个规则，Cargo 的处理方式如下：最左的非零版本号不同的，代表着不兼容的变更。也就是说，0.2.3 和 0.3.0 包含了不兼容的变更，同样，0.0.4 和 0.0.5 也是不兼容的。

## Crate 作者语义化版本控制指南

> “理论上，理论和实际会保持一致。实际上，并非如此”

理论上说，crate 作者遵守语义化版本号第一条规则是非常容易的：但凡变更任何内容，都需要发布一个新版本。Git [标签]可以帮助我们做到这一点：除非你使用 `--force` 选项移动标签，否则一个 Git 标签一定关联到某个确定的提交。发布到 [crates.io] 的 crate 也被强制遵守这条规则：同一个 crate 再次发布已有版本号的时候会被拒绝。唯一的不便就是，你在*刚刚*发布的版本中发现了一个小问题，却不得不抑制住自己想马上修复它的冲动。

语义化版本规范是为了确保 API 的兼容性。如果你只是做了微小的变更，并没有修改 API 的行为，此时仅需发布一个补丁版本就足够了。（但是，如果你的 crate 被广泛依赖，就要留神[海勒姆法则]：无论你对代码做出多么微小的改动，都有人[依赖老版本的行为]，哪怕 API 本身并未改变。）

对于 crate 作者而言，要想完全遵守后面几条规则却非易事，因为需要精准判断所做变更是否向后兼容。有些变更明显不具备兼容性，例如，移除了一些类型，或者修改了方法签名等；另外一些，则明显兼容的，例如：给结构体增加方法，或者增加常量定义等。但是除此之外，还有很多不太容易判断的灰色地带。

为了帮助作者更好的判断变更的兼容性，[Cargo 手册]中有详细的描述。大部分都是意料之中的，但是也有一些值得格外留意的：

- *通常而言*，增加新的条目在兼容性方面不会带来风险。但是可能会存在一些问题：使用这个 crate 的代码正好也增加了一些条目且和 crate 中条目重名了。
  - 如果用户采用[通配符导入]的方式使用 crate 中的条目，通常会带来风险，因为 crate 中的所有条目都会被通配符引入到用户代码的主命名空间。在[第 23 条]中，明确不建议使用通配符导入。
  - 即使没有采用通配符导入的方式，给[已有 trait 增加包含默认实现的方法]（见[第 13 条]），或者[增加内部方法]也可能导致与已经存在的名称冲突。
- 由于 Rust 要求涵盖所有的可能性，所以变更可能性集合也会导致不兼容。
  - 针对 `enum`  做 `match` 要求代码涵盖所有的可能性，所以如果 [crate 中的 `enum` 增加了一个值]，会导致不兼容（除非 `enum` 标记为 [`non_exhaustive`]。但是，增加 `non_exhaustive` 标记本身也是一种不兼容变更）。
  - 显式创建一个 `struct` 实例时，要求提供该 `struct` 所有字段的初始值，所以，[给 `struct` 增加公有初始化字段]的变更，也是不兼容的，私有字段则不存在这个问题。我们可以通过将 `struct` 标记 `non_exhaustive` 来阻止外部用户显式创建实例。
- 将一个 trait 从对象安全的变更为[非*对象安全*]的（见[第 12 条]），属于不兼容变更。那些将这个 trait 当作 trait 对象来使用的代码将无法编译。
- 给 trait 增加泛化实现（blanket implementation）也是不兼容变更。如果用户已经在自己代码中给这个 trait 增加实现了，就会存在两个相互冲突的实现。
- 修改开源 crate 的*许可证*是不兼容变更。其他用户可能由于许可证的变更而无法继续使用。**许可证应作为 API 的一部分来看待**。
- 修改 crate 默认启用的特性（见[第 26 条]）是一种潜在的不兼容变更，移除默认特性也是类似，除非被移除的这个特性已经没有实际作用了。新增默认特性也可能导致兼容性问题。所以，**默认启用的特性集合也应作为 API 的一部分来看待**。
- 修改库代码的时候使用了 Rust 最新特性的，也*可能*导致兼容性问题，因为使用这个 crate 的用户可能还没有更新他们的编译器版本，以支持你所使用的新特性。但是，大部分的 crate 都有对 Rust 最低版本（MSRV）的要求，升级所需的最低版本被视为[兼容变更]。所以，**要考虑 MSRV 是否应该成为 API 的一部分**。

显而易见，crate 中对外公开的条目越少，就越不容易引发不兼容变更（见[第 22 条]）。

但是，不可否认的是，通过将所有的公共 API 条目进行逐一比较来确保两个发布版本之间的兼容性，是一个非常耗时的过程，充其量不过是对变更水平做一个*大致*的评估。鉴于这种对比过程比较机械化，所以希望可以有好用的工具（见[第 31 条]）来简化这个过程。[^1]

如果你确实要发布一个不兼容的主版本变更，最好确保在即使是较大的变更之后，依然可以提供相应的功能。为了方便用户使用，如果可能的话，建议按照下面的顺序进行变更：

1. 发布一个次版本变更，其中包含新的 API 同时将旧的 API 标记为 [`deprecated`]，并且，应包含一份版本升级迁移指导。
2. 然后发布一个移除了旧的 API 的主版本变更。

这里还有一层隐含的意思：**要让不兼容的变更确实是不兼容的**。如果变更对于现有用户是不兼容的，但是又*可以*重用相同的 API，那就不要这样做。强制更改类型（并进行主版本升级）以确保用户不会无意中错误地使用新版本。

对于 API 中不太明确的部分（例如 MSRV 或许可证）可以考虑设置一个持续集成（CI）检查（见[第 32 条]）来检测这些变化，并根据需要使用例如 `cargo-deny` 这样的工具（见[第 25 条]）。

最后，不要因为担心承诺 API 固定下来就害怕发布 1.0.0 版本，许多 crate 由于这个原因陷入了永远停留在 0.x 版本的尴尬境地。因为这会将语义化版本本就有限的表达能力从三个类别（主版本/次版本/补丁）减少到两个（有效主版本/有效次版本）。

## Crate 用户的语义化版本控制指南

对于 crate 用户而言，理论上，一个 crate 新版本的含义如下：

- 一个 crate 的补丁版本升级应该可以正常工作。
- 一个 crate 的次版本升级应该可以正常工作，但是值得去研究 API 中新增加的部分，以发现是否有更好的方式来使用这个 crate。但是，如果你的代码中并未使用 API 中的新增内容，那么可以放心使用新版本，而无需将其退回到老版本。
- 如果一个 crate 的主版本升级了，那么所有的预期都可能失效；你的代码很可能无法正确编译，你需要重写部分代码以匹配新的 API。即使你的代码仍然可以编译，你也应该**检查在主版本更改后对 API 的使用是否仍然有效**，因为库的约束和前提条件可能已经改变。

在实际操作中，根据海勒姆法则，即使是仅仅升级了次版本或者补丁版本，也*可能*给你的项目带来非预期的变更，哪怕你的代码依然可以正确编译。

基于这些原因，为了更好的兼容后续的版本变更，你在使用 crate 的时候，通常应该以 `"1.4.3"` 或者 `"0.7"` 这样的方式来指定其版本。应该*避免*诸如 `"*"` 或者 `"0.*"` 这种完全通配符的版本号写法，因为这意味着对于所用 crate 的*任何*版本以及其提供的*任意* API，都是可以在你的项目中正常使用的，通常情况下，这并不是你真正所希望的。当你把自己开发的 crate 发布到 `crates.io` 的时候，依赖项版本采用如 `"*"` 这种完全通配符格式的，会被[拒绝]。

但是从长远来看，为了保证兼容性而完全忽略所用 crate 的主版本更新，也是不安全的。一旦一个 crate 发布了主版本更新，很可能不会再对先前的版本进行更新了，这包括错误修复以及安全更。例如，一个 crate 发布了 2.x 版本之后，`"1.4"` 版本就会越来越落后于新版本，哪怕是安全问题，也得不到及时解决。

因此，要么将依赖项的版本锚定到旧版本并接受潜在的风险，要么就**跟着依赖项的主版本持续升级**。可以借助 `cargo update` 或者 [Dependabot]（见[第 31 条]）这些工具来帮你检查所用依赖项的更新情况，然后在合适的时机升级。

## 讨论
语义化版本控制是有代价的：对每一个 crate 的更改都必须根据其标准进行评估，以决定适当的版本升级。它只是一个粗略的工具：它最多只能反映 crate 所有者对于当前发布版属于三种类别中的哪一种的猜测。并不是每个人都会正确地处理，也不是所有事情都能明确说明“正确”究竟意味着什么，即便你做对了，也总有可能会受到海勒姆法则的影响。

然而，在不具备像 [Google 高度测试的庞大内部单体库]奢侈工作环境的前提下，语义化版本控制是唯一可用的方法。因此，理解它的概念和局限性对于管理依赖关系是很有必要的。

## 注释

[^1]: 例如：[`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks) 就可以帮你做一些检查工作

原文[点这里](https://www.lurklurk.org/effective-rust/semver.html)查看

<!-- 参考链接 -->

[第 12 条]: ../chapter_2/item12-generics&trait-objects.md
[第 13 条]: ../chapter_2/item13-use-default-impl.md
[第 22 条]: ./item22-visibility.md
[第 23 条]: ./item23-wildcard.md
[第 25 条]: ./item25-dep-graph.md
[第 26 条]: ./item26-features.md
[第 31 条]: ../chapter_5/item31-use-tools.md
[第 32 条]: ../chapter_5/item32-ci.md
[第 35 条]: ../chapter_6/item35-bindgen.md

[《Google 软件工程》（O'Reilly）]: https://abseil.io/resources/swe-book/html/ch21.html#the_limitations_of_semver
[官方文档]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
[官方文档_semver_summary]: https://semver.org/#summary
[细则_semver]: https://semver.org/#spec-item-3
[semver_item4]: https://semver.org/#spec-item-4
[标签]: https://git-scm.com/docs/git-tag
[crates.io]: https://crates.io/
[海勒姆法则]: https://www.hyrumslaw.com/
[依赖老版本的行为]: https://xkcd.com/1172/
[Cargo 手册]: https://doc.rust-lang.org/cargo/reference/semver.html#change-categories
[通配符导入]: https://doc.rust-lang.org/cargo/reference/semver.html#minor-adding-new-public-items
[已有 trait 增加包含默认实现的方法]: https://doc.rust-lang.org/cargo/reference/semver.html#possibly-breaking-adding-a-defaulted-trait-item
[增加内部方法]: https://doc.rust-lang.org/cargo/reference/semver.html#possibly-breaking-change-adding-any-inherent-items
[crate 中的 `enum` 增加了一个值]: https://doc.rust-lang.org/cargo/reference/semver.html#major-adding-new-enum-variants-without-non_exhaustive
[`non_exhaustive`]: https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute
[给 `struct` 增加公有初始化字段的变更]: https://doc.rust-lang.org/cargo/reference/semver.html#major-adding-a-public-field-when-no-private-field-exists
[非*对象安全*]: https://doc.rust-lang.org/cargo/reference/semver.html#trait-object-safety
[兼容变更]: https://github.com/rust-lang/api-guidelines/discussions/231
[`deprecated`]: https://doc.rust-lang.org/reference/attributes/diagnostics.html#the-deprecated-attribute
[拒绝]: https://doc.rust-lang.org/cargo/faq.html#can-libraries-use--as-a-version-for-their-dependencies
[Dependabot]: https://docs.github.com/en/code-security/dependabot
[Google 高度测试的庞大内部单体库]: https://dl.acm.org/doi/pdf/10.1145/2854146
