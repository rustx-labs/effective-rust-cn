# 第 30 条：不仅仅是单元测试

> “所有的公司都拥有测试环境。但只有那些幸运的公司拥有独立于测试环境的生产环境。” —— [@FearlessSon]

和大多数其他的现代编程语言一样，Rust 也包含一些便于[编写测试]的特性。借助这些特性，你可以将测试和代码共存。而通过测试，可以提升代码运行正确性的信心。

这并非是兜售测试重要性的文章。从测试最基本功能来说，如果代码缺少了测试，它很可能并非如我们所希望的那样运行。本条款是在你已经建立了**为代码编写测试**这一信念的基础上展开的。

在接下来的两个章节介绍的单元测试（unit tests）以及集成测试（integration tests）是测试领域内的两大重要成员。但是，Rust 工具链以及它的扩展还允许更多类型的测试。本条款将会介绍它们的基本使用流程及应用场景。

## 单元测试

Rust 代码中最常见的测试类型是单元测试：

```rust
// ... (code defining `nat_subtract*` functions for natural
//      number subtraction)

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_nat_subtract() {
        assert_eq!(nat_subtract(4, 3).unwrap(), 1);
        assert_eq!(nat_subtract(4, 5), None);
    }

    #[should_panic]
    #[test]
    fn test_something_that_panics() {
        nat_subtract_unchecked(4, 5);
    }
}
```

这段单元测试示例体现了所有 Rust 单元测试都会有的形式：

* 使用模块对单元测试函数进行封装。
* 每个单元测试都会使用 `#[test]` 属性来进行标志。
* 用来测试的模块使用 `#[cfg(test)]` 进行声明，所以单元测试的代码仅会在测试配置下被构建。

该示例也展示了一些可选的、仅在特定测试中才会出现的形式：

* 这里的测试代码放置在单独的模块里，模块名一般为 `tests` 或者 `test`。这个模块可以和业务代码放置在一起，或者放置在单独的 *tests.rs* 文件里。放置在单独的文件里可以让人更加便捷的区分代码是用来测试的，还是用于真实的业务逻辑。
* 测试的模块可以使用一个通配符 `use super::*` 来将父模块的所有依赖都拉取到测试模块里。这种操作可以让编写测试代码更加方便（这也是针对[第 23 条]给出的应避免通配符导入的通用建议的一个例外）。
* 在常规的模块可见性规则下，单元测试可以使用所有父模块里包含的内容，无论是否声明为公共的。单元测试可以访问到对用户不可见的内部特性，相当于允许对代码进行“开箱”测试。
* 测试代码对预期的结果使用了 `expect()` 或者 `unwrap()`。显然，[第 18 条]中描述的规则并不适用这些单元测试的代码，比如单元测试中会用到 `panic!` 来表示失败的结果。同样的，测试代码中也使用了 `assert_eq!` 来检查预期的结果，并且会在检查失败时抛出 `panic`。
* 测试代码中使用了一个函数，该函数在一些非法的输入下会造成 `panic`。为了校验该函数的这一功能是否生效，单元测试的函数中使用了 `#[should_panic]` 属性。当被测试的函数希望其他的代码遵守其不变量或前置条件时，或者被测试的是一个有特别的原因而需要忽略[第 18 条]的建议的公共函数时，这个属性可能是需要的。（这样的函数需要在它的注释文档中有“Panics”小节，如[第 27 条]中所述。）

[第 27 条]建议不要对已经通过类型系统表达出的内容进行文档化。同样的，也不需要对已经由类型系统保证的内容进行测试。如果你的 `enum` 类型保存了不在允许的变体列表里的值，你可能遇到了比单元测试失败更加严重的问题。

然而，如果你的代码依赖了一些依赖库中的独特功能，对这些功能准备基础的单元测试会很有用。这里的单元测试目的并非是重复依赖库自身已经具备的功能测试，而是在你所需要的依赖包里提供的功能发生变更时能尽早地暴露 —— 这里不包括公共的 API 接口签名发生了变化的情况，这种情况应当通过语义化版本号来表明（[第 21 条]）。

## 集成测试

Rust 项目中另一种常用到的测试模式是*集成测试*，通常被放置在 `tests/` 目录下。这个目录下的每个文件都会作为一个单独的测试程序运行，每个测试程序都会执行其包含的所有以 `#[test]` 标记的函数。

集成测试无法访问包的内部内容，因此集成测试仅能基于包的公共 API 进行行为测试。

## 文档测试

[第 27 条]描述了可以在文档注释中包含一小段代码示例，来演示一个特定的公共 API 的使用方式。每段这样的代码都包含在一个隐式的 `fn main() { ... }` 函数中，并且可以在 `cargo test` 时被执行。这是一种有效的为代码添加测试用例的方法，一般被称为*文档测试*（doc tests）。每个这样的测试都可以通过 `cargo test --doc <item-name>` 的方式来选择性的执行。

定期的通过 CI 系统（[第 32 条]）来执行这些测试可以确保这些代码示例不会离你期望提供的 API 太远。

## 示例代码

[第 27 条]也描述了为公共接口提供示例代码的实践。在 `examples/` 目录下的每个 Rust 文件（或者 `examples` 目录下每个包含 `main.rs` 文件的子目录）都可以通过 `cargo run --example <name>` 或者 `cargo test --example <name>` 来作为独立的可执行文件运行。

这些程序仅能访问包中的公共接口，用来演示这些公共接口的使用方式。示例代码并非被设计为测试代码（没有 `#[test]`，也没有 `[cfg(test)]`），而且由于处于一些不起眼的角落，它们并不太适合放置代码 —— 尤其是，它们并不在 `cargo test` 时默认执行。

尽管如此，确保在你的 CI 系统（[第 32 条]）中构建并且运行这些示例代码（通过 `cargo test --examples`）仍然是一个很好的实践。通过执行这些代码，可以为那些会影响大多数用户的接口提供一个很好的回归校验机制。特别地，如果你的示例代码演示了接口使用的一般方式，那么示例代码的运行失败往往意味着有重要的东西出问题了：

* 如果这是个真实的错误，它可能会影响很多用户 —— 示例代码的本质就意味着它们会被很多用户复制、粘贴和采用。
* 如果有意对公共接口进行了变更，那么这些示例也需要参照最新的接口定义来更新。接口的改变往往意味着向后不兼容。所以当包被发布时，版本号需要随着调整以说明这是个不兼容的升级（[第 21 条]）。

用户复制、粘贴示例代码的行为意味着示例代码和测试代码的形式有很大的不同。如[第 18 条]中描述的一样，你应该避免对 `Results` 进行 `unwrap()`，从而为用户提供一个很好的参照。相反，应该在每个示例代码的 `main()` 函数中返回类似 `Result<(), Box<dyn Error>>` 的结果，并且在内部使用 `?` 符号来组织代码（[第 3 条]）。

## 基准测试

[第 20 条]试图说明极致的代码性能优化并非总是必要的。尽管如此，性能有时肯定是很关键的，在这种情况下，测量以及追踪代码的性能变化是很好的实践。具备定期运行的*基准测试*（benchmarks）（比如，作为 CI 系统的一部分，[第 32 条]）允许你在代码或者工具链的变更影响到代码的性能时能及时察觉。

`[cargo bench]` 命令可以重复运行执行某个操作的特殊测试用例，并且计算出这个操作的平均耗时。在撰写本文时，Rust 对基准测试的支持尚未稳定，所以基准测试相关的指令需要通过 `cargo +nightly bench` 的方式来执行。（关于 Rust 中的不稳定的特性，包括这里使用到的 [test] 特性，都描述在 Rust 官方的 [The Unstable Book] 中。）

然而，这里存在着因为编译器的优化导致结果异常的风险，尤其是当你将操作约束在真实代码的一个小的子集中时。考虑如下一个简单的算数函数：

```rust
pub fn factorial(n: u128) -> u128 {
    match n {
        0 => 1,
        n => n * factorial(n - 1),
    }
}
```

针对这段代码的一个简单的基准测试：

```rust
#![feature(test)]
extern crate test;

#[bench]
fn bench_factorial(b: &mut test::Bencher) {
    b.iter(|| {
        let result = factorial(15);
        assert_eq!(result, 1_307_674_368_000);
    });
}
```

输出了有点难以置信的好结果：

```shell
test bench_factorial             ... bench:           0 ns/iter (+/- 0)
```

由于在测试过程中使用了固定的输入以及较少的代码指令，编译器能够对迭代进行优化并且直接输出计算结果。这就导致了不符合实际情况的测试结果。

使用 [std::hint::black_box] 可以帮助解决这个问题。这是一个恒等函数（identity function），[编译器将尽量]不对其进行优化。

将上述基准测试代码改为如下形式：

```rust
#[bench]
fn bench_factorial(b: &mut test::Bencher) {
    b.iter(|| {
        let result = factorial(std::hint::black_box(15));
        assert_eq!(result, 1_307_674_368_000);
    });
}
```

测试结果就实际多了：

```shell
test blackboxed::bench_factorial ... bench:          16 ns/iter (+/- 3)
```

[Godbolt 编译器管理器]也可以通过展示编译器实际输出的机器码的方式来帮助解决这个问题，这样就更容易看出来编译器是否进行了一些在真实场景下无法进行的优化。

最后，如果你为 Rust 代码准备了基准测试，[criterion] 包可能提供了一个比标准库里的 [test::bench::Bencher] 使用起来更便捷（可以在稳定的 Rust 工具链上运行）、特性也更多（支持结果的数据统计及图表）的替代品。

## 模糊测试

模糊测试（fuzzy testing）是将代码暴露在随机输入中以期能够发现代码缺陷（尤其是会导致代码异常出错的那些输入）的测试方法。这个方法从一般意义来说已经是有用的技术，而当你的代码输入是由不可控的其他人填充的甚至可能是来自有意的恶意攻击时，它将会显得更加重要 —— **所以如果你的代码有可能暴露给潜在的攻击者时，你应当使用模糊测试**。

从历史上来看，C/C++ 代码通过模糊测试发现的往往是内存安全问题，通常是通过结合模糊测试与针对内存访问模式的运行时插桩（比如 [AddressSanitizer] 或者 [ThreadSanitizer]）来检测到的。

Rust 对其中的一些（但并非全部）内存安全问题免疫，尤其是未引入 `unsafe` 的代码时（[第 16 条]）。然而，Rust 并不能杜绝任意的代码缺陷，比如会触发 `panic!`（[第 18 条]）的代码就有可能引发拒绝服务（denial-of-service，DOS）攻击。

最有效的模糊测试的形式是*以覆盖率来引导*：测试基础设施监控代码的哪些部分被执行，并更多针对新的代码路径进行随机的输入更改。“[American fuzzy lop]”（AFL）是其中的佼佼者。但是近些年来，类似的功能已经被引入了 LLVM 的工具链，比如 [libFuzzer]。

Rust 编译器是在 LLVM 的基础上构建的，因此 [cargo-fuzz] 自然地为 Rust 引入了 `libFuzzer`（虽然仅在部分平台上可用）。

模糊测试的首要要求是确定代码的入口点，该入口点可以（或者被适配成可以）将任意字节的数据作为输入：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
/// Determine if the input starts with "FUZZ".
pub fn is_fuzz(data: &[u8]) -> bool {
    if data.len() >= 3 /* oops */
    && data[0] == b'F'
    && data[1] == b'U'
    && data[2] == b'Z'
    && data[3] == b'Z'
    {
        true
    } else {
        false
    }
}
```

当目标入口点确定后，[Rust Fuzz Book] 给出了如何启动测试的指导。它的核心是一个小的驱动程序，会将目标入口点连接到模糊测试的基础设施上：

```rust
// fuzz/fuzz_targets/target1.rs file
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let _ = somecrate::is_fuzz(data);
});
```

运行 `cargo +nightly fuzz run target1` 将会持续使用随机数据来执行模糊测试的目标函数，直至异常出现。上述示例中，错误将被立即发现：

```shell
INFO: Running with entropic power schedule (0xFF, 100).
INFO: Seed: 1607525774
INFO: Loaded 1 modules: 1624 [0x108219fa0, 0x10821a5f8),
INFO: Loaded 1 PC tables (1624 PCs): 1624 [0x10821a5f8,0x108220b78),
INFO:        9 files found in fuzz/corpus/target1
INFO: seed corpus: files: 9 min: 1b max: 8b total: 46b rss: 38Mb
#10	INITED cov: 26 ft: 26 corp: 6/22b exec/s: 0 rss: 39Mb
thread panicked at 'index out of bounds: the len is 3 but the index is 3',
     testing/src/lib.rs:77:12
stack backtrace:
   0: rust_begin_unwind
             at /rustc/f77bfb7336f2/library/std/src/panicking.rs:579:5
   1: core::panicking::panic_fmt
             at /rustc/f77bfb7336f2/library/core/src/panicking.rs:64:14
   2: core::panicking::panic_bounds_check
             at /rustc/f77bfb7336f2/library/core/src/panicking.rs:159:5
   3: somecrate::is_fuzz
   4: _rust_fuzzer_test_input
   5: ___rust_try
   6: _LLVMFuzzerTestOneInput
   7: __ZN6fuzzer6Fuzzer15ExecuteCallbackEPKhm
   8: __ZN6fuzzer6Fuzzer6RunOneEPKhmbPNS_9InputInfoEbPb
   9: __ZN6fuzzer6Fuzzer16MutateAndTestOneEv
  10: __ZN6fuzzer6Fuzzer4LoopERNSt3__16vectorINS_9SizedFileENS_
      16fuzzer_allocatorIS3_EEEE
  11: __ZN6fuzzer12FuzzerDriverEPiPPPcPFiPKhmE
  12: _main
```

导致上述错误的测试数据也给出了。

一般来说，模糊测试并不能如此快地发现错误，因此将模糊测试作为 CI 流程中的一部分也是没有意义的。模糊测试的不确定性及随之而来的计算成本意味着你需要考虑如何开展以及何时开展模糊测试 —— 可能仅需要在新的发布版本或者主要变更发生时才运行，或者仅在一个有限的时间周期内运行[^1]。

你也可以通过存储和重用先前模糊测试程序已找到用来触发新的代码路径的语料库的方式来加速模糊测试的执行。这将有助于后续的模糊测试过程将时间放在尝试新的代码路径上，而非重新测试以前访问过的代码路径。

## 测试建议

一个关于测试的条款如果不重复一些常见的建议（大部分都不是 Rust 特有的），那么它将是不完整的。

* 正如条款中反复所述的，**每次变更后都需要在 CI 中执行所有的测试**（除了模糊测试）。
* 当你在修复一个代码缺陷时，**在修复前先写一个能复现该代码缺陷的测试用例**，这样你才能确保该代码缺陷确实是被修复了并且不会在未来重新被引入。
* 如果你的包中包含了一些 feature（[第 26 条]）时，**对所有可能的 feature 组合都要进行测试**。
* 更一般性的，如果你的包中包含了任何配置特定的代码，（比如 `#[cfg(target_os = "windows")]`），**在每个有差异化处理的平台上都需要进行测试**。

本条款覆盖了很多不同类型的测试，你需要确定在你的项目里哪些测试是相关的和有价值的。

如果你有很多的测试代码并且准备将你的包推送到 [crates.io] 中，那么就需要考虑下哪些测试项是需要包含在发布包里的。默认情况下，`cargo` 会包含单元测试、集成测试、基准测试以及示例代码（但是并没有包含模糊测试，因为 `cargo-fuzz` 工具会将模糊测试的内容放置在子目录下的单独包中），这些可能超出了最终用户最需要的。如果某些测试类型并非是必须的，你可以[排除]一些测试项或者将这些测试项（尤其是行为性的测试）移入单独的测试包中。

## 需要记住的事情

* 编写单元测试来达到全面测试的目的，包括仅针对内部代码的测试。通过 `cargo test` 来运行它们。
* 编写集成测试代码来测试公共的接口。通过 `cargo test` 来运行它们。
* 编写文档测试来校验公共接口的调用方式。通过 `cargo test` 来运行它们。
* 编写示例代码程序来完整的说明如何使用包中的公共 API。通过 `cargo test --exmaples` 或者 `cargo run --example <name>` 来运行它们。
* 如果代码对性能有很明确的要求，编写基准测试来确认代码的性能表现。通过 `cargo bench` 来运行它们。
* 如果代码会暴露在不可信的输入中，编写模糊测试来确认对输入参数的约束。通过 `cargo fuzz` 来（持续地）运行它们。

## 注释

[^1]: 如果你的代码是一个被广泛运用的开源包，[Google OSS-Fuzz program] 可能会愿意为你的项目进行模糊测试。

原文[点这里](https://www.lurklurk.org/effective-rust/testing.html)查看

<!-- 参考链接 -->

[第 3 条]: ../chapter_1/item3-transform.md
[第 16 条]: ../chapter_3/item16-unsafe.md
[第 18 条]: ../chapter_3/item18-panic.md
[第 20 条]: ../chapter_3/item20-optimize.md
[第 21 条]: ../chapter_4/item21-semver.md
[第 23 条]: ../chapter_4/item23-wildcard.md
[第 28 条]: item28-use-macros-judiciously.md
[第 27 条]: item27-document-public-interfaces.md
[第 32 条]: item32-ci.md

[@FearlessSon]: https://twitter.com/FearlessSon/status/1405742580952834051
[编写测试]: https://doc.rust-lang.org/book/ch11-00-testing.html
[cargo test]: https://doc.rust-lang.org/cargo/commands/cargo-bench.html
[test]: https://doc.rust-lang.org/unstable-book/library-features/test.html
[The Unstable Book]: https://doc.rust-lang.org/unstable-book/the-unstable-book.html
[std::hint::black_box]: https://doc.rust-lang.org/std/hint/fn.black_box.html
[Godbolt 编译器管理器]: https://rust.godbolt.org/
[编译器将尽量]: https://rust-lang.github.io/rfcs/2360-bench-black-box.html
[criterion]: https://crates.io/crates/criterion
[test::bench::Bencher]: https://doc.rust-lang.org/test/bench/struct.Bencher.html
[AddressSanitizer]: https://clang.llvm.org/docs/AddressSanitizer.html
[ThreadSanitizer]: https://clang.llvm.org/docs/ThreadSanitizer.html
[American fuzzy lop]: https://lcamtuf.coredump.cx/afl/
[libFuzzer]: https://llvm.org/docs/LibFuzzer.html
[cargo-fuzz]: https://github.com/rust-fuzz/cargo-fuzz
[Rust Fuzz Book]: https://rust-fuzz.github.io/book/
[crates.io]: https://crates.io/
[排除]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-exclude-and-include-fields
[Google OSS-Fuzz program]: https://google.github.io/oss-fuzz/getting-started/accepting-new-projects/
