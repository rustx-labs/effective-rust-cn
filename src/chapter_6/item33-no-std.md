# 第 33 条：考虑使库代码与 no_std 兼容

Rust 附带一个名为 `std` 的标准库，其中包含了从标准数据结构到网络，从多线程支持到文件 I/O 等用于各种常见任务的代码。为了方便使用，`std` 中的一些项目会通过 [prelude] 自动导入到你的程序中，[prelude] 是一组 `use` 语句，可以在不需要指定完整名称的情况下直接使用这些常见类型（例如使用 `Vec` 而不是 `std::vec::Vec`）。

Rust 还支持为无法提供完整标准库的环境构建代码，如引导加载程序、固件或一般的嵌入式平台。通过在 `src/lib.rs` 文件顶部添加 `#![no_std]` 属性，可以指示 crate 在这类受限环境中编译。

本节将探讨在 `no_std` 环境中编译时我们会失去哪些功能，以及仍然可用的库函数。我们会发现，即使在这种受限环境中，依然有相当多的功能可供使用。

不过，这里特别关注的是针对*库*代码的 `no_std` 支持。构建 `no_std` *可执行*文件的复杂性超出了本文的范围[^1]，因此本节重点是对那些不得不在如此极简环境中工作的可怜人来说，如何确保库代码是可用的。

## `core`

即使在为最受限的平台构建时，标准库中的许多基本类型也仍然可用。例如 [`Option`] 和 [`Result`] 以及各种 [`Iterator`]，尽管它们名称不同。

这些基本类型的不同名称以 `core::` 开头，表明它们来自 `core` 库，这是一个即使在大多数 `no_std` 环境中也可用的标准库。这些 `core::` 类型的行为与等效的 `std::` 类型完全相同，因为它们实际上是同个类型 —— 对应的 `std::` 版本都只是底层 `core::` 类型的重新导出。

这意味着有一种快速而肮脏的方法来判断某个 `std::` 项目在 `no_std` 环境中是否可用： 访问 [`doc.rust-lang.org`] 并找到感兴趣的 `std` 项目，点击 “soure”（在右上角）[^2]，如果它跳转到了 `src/core/...` 中，则说明该项可通过 `core::` 在 `no_std` 下使用。

正常情况下 `core` 中的类型可自动用于所有 Rust 程序。但在 `no_std` 环境需要显式 `use`，因为此时没有 `std` [prelude] 帮忙导入。

实际中，对于许多环境（甚至是 `no_std`）来说，纯粹依赖 `core` 会有很大的限制，因为 `core` 的一个核心（双关语）约束是它不会进行堆分配。

虽然 Rust 很擅长将数据放入栈中并安全地跟踪其相应的生命周期（[第 14 条]），但 `core` 的这种限制仍然意味着无法提供标准数据结构（vector, map, set），因为它们都需要分配堆空间来存放内部元素。所以这个限制也大大减少了在此环境中可用的 crate 数量。

## `alloc`

但如果所在的 `no_std` 环境*的确*支持堆分配，则 `std` 中的许多标准数据结构仍然可用。这些数据结构以及其他使用分配功能的部分被分组到 Rust 的 [`alloc`] 库中。

和 `core` 一样，这些 `alloc` 变体实际上在底层是相同的类型。例如 `std::vec::Vec` 的真实名称是 `alloc::vec::Vec`。

如果 `no_std` 的 crate 要使用 `alloc`，需要在 `src/lib.rs` 额外显式声明 `extern crate alloc;`[^3]：

```rust
//! 我的 `no_std` 兼容 crate.
#![no_std]

// 引入 `alloc`.
extern crate alloc;
```

引入 `alloc` 库可以启用许多我们熟悉的朋友，现在他们以真实的名称被称呼：
- [`alloc::boxed::Box<T>`]
- [`alloc::rc::Rc<T>`]
- [`alloc::sync::Arc<T>`]
- [`alloc::vec::Vec<T>`]
- [`alloc::string::String`]
- [`alloc::format!`]
- [`alloc::collections::BTreeMap<K, V>`]
- [`alloc::collections::BTreeSet<T>`]

有了这些朋友，许多 crate 就可以兼容 `no_std` —— 前提是不涉及 I/O 或网络。

但 `alloc` 提供的数据结构显然还缺少了两个集合 —— [`HashMap`] 和 [`HashSet`]，它们特定于 `std` 而不是 `alloc`。这是因为这些基于 hash 的容器依靠随机种子来防止 hash 冲突攻击，但安全的随机数生成需要依赖操作系统的帮助，而 `alloc` 不能假设操作系统存在。

另一个缺失的部分是同步功能，例如 [`std::sync::Mutex`]，这是多线程代码所必需的，但这些类型同样特定于 `std`，它们依赖操作系统的同步原语，如果没有操作系统，这些同步原语也将不可用。在 `no_std` 下编写多线程代码，第三方 crate 可能是唯一选择，例如 [`spin`]。

## 为 `no_std` 编写代码

前文已经指出，对于*某些* crate 库，要使代码兼容 `no_std` 只需要以下修改：
- 用相同的 `core::` 或 `alloc::` crate 替换 `std::` 类型（由于缺少 `std` prelude，还需要 `use` 完整的类型名称）
- 将 `HashMap` / `HashSet` 迁移为 `BTreeMap` / `BTreeSet`

但这些操作只有在依赖的所有 crate（[第 25 条]）也兼容 `no_std` 时才有意义 —— 如果使用您 crate 的用户被迫链接到任何 `std` 中，与 `no_std` 的兼容就会失去意义。

这里还有一个问题：Rust 编译器并不会告诉你一个 `no_std` crate 中是否有引入使用了 `std` 的依赖。这意味着只要添加或更新一个使用了 `std` 的依赖，就能轻松破坏掉 `no_std` crate 的兼容性。

为了避免这个情况，**请为 `no_std` 构建添加 CI 检查**，以便 CI 系统（[第 32 条]）能在这种情况发生时发出警告。Rust 工具链支持开箱即用的交叉编译，因此只需为不支持 `std` 的目标系统（例如 `--target thumbv6m-none-eabi`）执行[交叉编译]，任何无意中依赖 `std` 的代码就会在此时编译失败。

所以，如果依赖项都支持，并且上面的简单修改就够了的话，不妨**考虑让库代码兼容 `no_std`**。这不需要太多额外工作，却能让库有更广泛的适用性。

如果这些转换并*没有*覆盖 crate 中的所有代码，但未覆盖的只是代码中的一小部分或包装良好的部分，那么可以考虑向 crate 添加一个 feature （[第 26 条]）以控制是否启用这部分代码。

这种允许使用 `std` 特定功能的 feature，通常会将其命名为 `std`：

```rust
#![cfg_attr(not(feature = "std"), no_std)]
```

或是将控制是否启用了 `alloc` 派生功能的 feature 命名为 `alloc`：

```rust
#[cfg(feature = "alloc")]
extern crate alloc;
```

注意，设计这类 feature 时有个容易忽视的陷阱：不要设置一个 `no_std` feature 来*禁用*需要 `std` 的功能（或类似的 `no_alloc` feature），正如[第 26 条]中提到的，feature 应当是可累加的，这样做会导致无法将启用了 `no_std` 和没有启用的两个 crate 用户组合在一起 —— 前者会删除后者所依赖的代码。

和所有带有 feautre 门控的代码一样，确保你的 CI 系统（[第 32 条]）构建了所有相关的组合 —— 包括在 `no_std` 平台上关闭 `std` 特性的构建。

## 可错分配（Fallible Allocation）

前面考虑了两种不同的 `no_std` 环境：不允许堆分配的完全嵌入式的环境（`core`）和允许堆分配的更宽松的环境（`core` + `alloc`）。

然而，有一些重要的环境介于两者之间，特别是那些允许堆分配但可能会失败的环境，因为堆空间是有限的。

不幸的是，Rust 的标准 `alloc` 库假设堆分配不会失败，但这个假设并不总是成立。

即使简单地使用 `alloc::vec::Vec`，也可能在每一行代码中都触发分配：

```rust
let mut v = Vec::new();
v.push(1); // 可能会分配
v.push(2); // 可能会分配
v.push(3); // 可能会分配
v.push(4); // 可能会分配
```

这些操作都不会返回 `Result`，当这些分配失败了，会发生什么？

答案取决于工具链、目标平台和[配置]，但很可能会导致 `panic!` 和程序终止。这里无法通过某种方式处理第 3 行的分配失败，并让程序继续执行第 4 行。

这种*可靠分配*（infallible allocation）的假设简化了在“正常”用户空间中运行的代码，因为在这些环境中，内存几乎是无限的 —— 或者内存耗尽意味着计算机本身出现了更严重的问题。

但可靠分配不适用于内存有限且程序必须应对内存不足的环境。这是一个在一些较老、内存安全性较差的语言中反而有更好支持的（罕见）领域：
- C 的级别足够低，分配是手动的，因此可以检查 `malloc` 的返回值是否为 `NULL`。
- C++ 可以使用异常机制，捕获以 [`std::bad_alloc`] 异常形式出现的分配失败[^4]。

历史上，Rust 标准库无法处理分配失败的问题在一些备受瞩目的场合（如 [Linux 内核]、Android 和 [Curl 工具]）中被指出，因此修复这一缺陷的工作正在进行中。

第一个尝试是[添加“可错的集合分配”]，它为许多涉及分配的集合 API 添加了允许可错分配的替代实现，通常是添加一个会返回 `Result<_, AllocError>` 的 `try_<operation>` 操作，例如：
- [`Vec::try_reserve`] 作为 [`Vec::reserve`] 的替代品
- [`Box::try_new`] 作为 [`Box::new`] 的替代品（在 nightly toolchain 中可用）

但这些允许可错分配的 API 功能也仅限于此。例如，（目前为止）还没有与 `Vec::push` 等效的可错分配版本，因此编写组装 vector 的代码时，可能需要进行精确计算，以确保不会发生内存分配错误。

```rust
fn try_build_a_vec() -> Result<Vec<u8>, String> {
    let mut v = Vec::new();

    // 仔细计算一下需要多少空间,这里简化为：
    let required_size = 4;

    v.try_reserve(required_size)
        .map_err(|_e| format!("Failed to allocate {} items!", required_size))?;

    // 我们现在知道这是安全的了
    v.push(1);
    v.push(2);
    v.push(3);
    v.push(4);

    Ok(v)
}
```

除了增加允许可错分配的入口外，还可以通过关闭默认开启的 [`no_global_oom_handling`] 配置来禁用*可靠*分配操作。在堆内存有限的环境（如 Linux 内核）中，可以显式禁用此标志，以确保不会在代码中无意使用了可靠分配。

## 要记住的事
- `std` crate 中的许多项目实际上来自 `core` 或 `alloc`。
- 因此，使库代码兼容 `no_std` 可能比您想象的更简单。
- 通过在 CI 中检查 `no_std` 代码来确认 `no_std` 代码保持 `no_std` 兼容。
- 注意，当前支持在有限堆环境中工作的库十分有限。

## 注释

[^1]: 有关创建 `no_std` 可执行文件所涉及的内容，请参阅 [The Embedonomicon] 或 Philipp Opperamann 的[早期博客文章]。

[^2]: 注意，该方法不一定正确。例如在撰写本文时，`Error` trait 在 `[core::]` 中定义，但被标记为不稳定（unstable），只有 [`std::` 版本]是稳定的（stable）

[^3]: 在 Rust 2018 之前，`extern crate` 声明用来引入依赖项。现在这些依赖完全由 `Cargo.toml` 处理，但仍使用 `extern crate` 机制引入那些 `no_std` 环境中 Rust 标准库的可选部分（即 *[sysroot crates]*）。

[^4]: 还可以为 `new` 调用添加 [`std::nothrow`] 重载，并检查是否返回 `nullptr`。但一些容器方法比如 [`vector<T>::push_back`]，在内部进行分配，因此只能通过抛出异常来表示分配失败。

原文[点这里](https://www.lurklurk.org/effective-rust/no-std.html)查看

<!-- 参考链接 -->

[第 14 条]: ../chapter_3/item14-lifetimes.md
[第 25 条]: ../chapter_4/item25-dep-graph.md
[第 26 条]: ../chapter_4/item26-features.md
[第 32 条]: ../chapter_5/item32-ci.md

[prelude]: https://doc.rust-lang.org/std/prelude/index.html
[`Option`]: https://doc.rust-lang.org/core/option/enum.Option.html
[`Result`]: https://doc.rust-lang.org/core/result/enum.Result.html
[`Iterator`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html
[`doc.rust-lang.org`]: https://doc.rust-lang.org/std/index.html
[The Embedonomicon]: https://docs.rust-embedded.org/embedonomicon/
[早期博客文章]: https://os.phil-opp.com/freestanding-rust-binary/
[core::]: https://doc.rust-lang.org/1.70.0/core/error/trait.Error.html
[`std::` 版本]: https://doc.rust-lang.org/1.70.0/std/error/trait.Error.html
[sysroot crates]: https://doc.rust-lang.org/edition-guide/rust-2018/path-changes.html#an-exception
[`alloc`]: https://doc.rust-lang.org/alloc/
[`alloc::boxed::Box<T>`]: https://doc.rust-lang.org/alloc/boxed/struct.Box.html
[`alloc::rc::Rc<T>`]: https://doc.rust-lang.org/alloc/rc/struct.Rc.html
[`alloc::sync::Arc<T>`]: https://doc.rust-lang.org/alloc/sync/struct.Arc.html
[`alloc::vec::Vec<T>`]: https://doc.rust-lang.org/alloc/vec/struct.Vec.html
[`alloc::string::String`]: https://doc.rust-lang.org/alloc/string/struct.String.html
[`alloc::format!`]: https://doc.rust-lang.org/alloc/macro.format.html
[`alloc::collections::BTreeMap<K, V>`]: https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html
[`alloc::collections::BTreeSet<T>`]: https://doc.rust-lang.org/alloc/collections/btree_set/struct.BTreeSet.html
[`HashMap`]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[`HashSet`]: https://doc.rust-lang.org/std/collections/hash_set/struct.HashSet.html
[`std::sync::Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`spin`]: https://docs.rs/spin/
[交叉编译]: https://www.reddit.com/r/rust/comments/ef8nd9/how_to_avoid_accidentally_breaking_no_std/fbyz6ix/
[配置]: https://doc.rust-lang.org/std/alloc/fn.set_alloc_error_hook.html
[`std::bad_alloc`]: https://en.cppreference.com/w/cpp/memory/new/bad_alloc
[`std::nothrow`]: https://en.cppreference.com/w/cpp/memory/new/nothrow
[`vector<T>::push_back`]: https://en.cppreference.com/w/cpp/container/vector/push_back
[Linux 内核]: https://lkml.org/lkml/2021/4/14/1099
[Curl 工具]: https://github.com/hyperium/hyper/issues/2265#issuecomment-693194229
[添加“可错的集合分配”]: https://github.com/rust-lang/rfcs/pull/2116
[`Vec::try_reserve`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.try_reserve
[`Vec::reserve`]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.reserve
[`Box::try_new`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.try_new
[`Box::new`]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.new
[`no_global_oom_handling`]: https://github.com/rust-lang/rust/pull/84266
