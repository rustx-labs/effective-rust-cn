# 第 30 条：不仅仅是单测

> “所有的公司都拥有测试环境。但只有那些幸运的公司拥有独立于测试环境的生产环境。” —— @FearlessSon

和大多数其他的现代编程语言一样，Rust 也包含一些便于[编写单测]的特性。借助这些特性，你可以将单测和代码共存。而通过单测，可以提升代码运行准确性的信心。

这并非是兜售单测重要性的文章。从单测最基本功能来说，如果代码缺少了单测，它很可能并非如我们所希望的那样运行。本条目是在你已经建立了**为代码编写单测**这一信念的基础上展开的。

单元测试以及集成测试是测试领域内的两大重要成员。在接下来的两节内将会介绍。但是，Rust 工具链，也包括它的扩展，允许多种多样的测试形式。本条目将会介绍它们的基本使用流程及应用场景。

## 单元测试

Rust 代码中最常见的测试类型是单测：

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

这段单测示例体现了 Rust 所有单测都会有的要素：
* 使用模块对单测函数进行封装。
* 每个单测都会使用`#[test]`属性来进行标志。
* 用来测试的模块使用`#[cfg(test)]`进行声明，所以单测的代码仅会在测试时生效。

示例也展示了一些仅对特殊测试形式有效的要素：
* 这里的测试代码放置在单独的模块里，一般称为`tests`或者`test`。这个模块可以和逻辑代码放置在一起，或者放置在单独的*tests.rs*文件里。放置在单独的文件里可以让人更加便捷的区分代码是用来测试的，还是用于一般的功能逻辑。
* 测试的模块可以使用一个通配符`use super::*`来将父模块的所有依赖都声明到测试模块里。这种操作可以让编写测试代码更加方便（同时依据[第 23 条]来说，通配符的引入是一种要规避的操作）。
* 在模块的一般可达性规则下，单测可以使用所有父模块里包含的内容，无论是否声明为公共的。基于单测对内部功能可访问的特点，可以对代码进行“开箱”测试。
* 测试代码使用`expect()`或者`unwrap()`来标志其希望的结果。显然，[第 18 条]中声明的规则并不适用这些单测的代码。单测中需要使用`panic!`来标注失败的结果。同样的，测试的代码中也会使用`assert_eq!`来校验期待的值，并且会在失败时抛出`panic`。
* 测试代码中使用了一个函数，该函数在一些非法的输入下会造成`panic`。为了校验该函数的这一功能是否生效，单测的函数中使用了`#[should_panic]`特性。这一特性在需要测试一个内部函数且希望保持这个函数的各校验规则不发生改变，或者测试一个公共的函数且由于一些原因需要忽略[第 18 条]中的建议。（这样的函数需要在它的注释文档中有“Panics”小节，就像[第 27 条]中描述的。）

[第 27 条]中建议不要对已经通过类型表述出的内容。同样的，也不需要对已经由类型进行约束的内容进行测试。如果你的`enum`类型派生出了不在声明列表中罗列的变量，你可能遇到了比单测失败更加严重的问题。

然而，如果你的代码依赖了一些依赖库中的独特功能，对这些功能准备基础的单测会很有用。这里的单测目的并非是重复依赖中已经具备的功能测试，而是尽量早地暴露依赖中对这些功能进行了变更的风险 —— 尤其是公共的接口约定发生了变化，通常应当通过版本号来表明（[第 21 条]）。

## 集成测试

Rust 项目中另一种常用到的测试模式是：*集成测试*，测试通常被放置在`tests/`目录下。这个目录下的每个文件都会作为一个单独的测试程序运行，每个测试程序都会执行其包含的所有以`#[test]`标志的测试函数。

集成测试没有访问包内部内容的权限，因此集成测试仅能覆盖包的公共 API。

## 文档测试

[第 27 条]描述了在注释中包含一小段代码的示例，通常是为了说明特定的公共 API 的使用方式。每段这样的代码都包含在一个隐式的`fn main() { ... }`函数中，并且可以在`cargo test`时被执行。这是一种高效的代码添加测试用例的方法，一般被称为*文档测试*。每个类似的测试都可以通过`cargo test --doc <item-name>`的方式来选择性的执行。

定期的通过 CI 系统（[第 32 条]）来执行这些测试可以确保代码不会离包中期望提供的 API 太远。

## 示例

[第 27 条]也描述了为公共接口提供示例代码的实践。在`examples/`目录下的每个 Rust 文件（或者`examples`目录下每个子目录中的`main.rs`文件）都可以通过`cargo run --example <name>`或者`cargo test --example <name>`的方式来作为独立的可执行文件运行。

这些程序仅能访问包中的公共接口，并且可以说明这些公共接口的使用方式。示例代码并非被设计为测试代码（没有`#[test]`，没有`[cfg(test)]`的标注），而且由于处于一些不起眼的角落，它们并不适合放置代码 —— 尤其是，它们并不在`cargo test`时默认执行。

尽管如此，CI 系统（[第 32 条]）构建并且运行这些示例代码（通过`cargo test --examples`）仍然是一个很好的实践。通过执行这些代码，可以为那些会影响大多数用户的接口提供一个很好的回归校验机制。特别地，如果你的示例揭示了接口使用的一般方式，那么示例运行的失败往往意味着存在如下的错误：

* 如果这是个高超的错误，它可能会影响很多用户 —— 示例中的代码将会被很多用户复制、粘贴或者参照。
* 如果公共接口发生了变更，那么这些示例也需要参照最新的接口定义来实现。接口的改变往往意味着不兼容。所以当包被发布时，版本号需要随着调整以说明这是个不兼容的升级（[第 21 条]）。

用户复制、粘贴测试代码的行为意味着示例代码和测试代码的形式有很大的不同。如[第 18 条]中描述的一样，你可以避免对 Results 进行 unwrap() 使用，从而为用户提供一个很好的参照。同样的，在每个示例代码的`main()`函数中返回类似`Result<(), Box<dyn Error>>`的结果，并且在内部使用`?`符号来组织代码（[第 3 条]）也是一种很好的行为。


## 性能基准

[第 20 条]试图说明极致的代码性能优化并非总是必要的。尽管如此，有时性能肯定时很关键的，并且在这种情况下，衡量以及追踪代码的性能变化是很好的实践。具备定期运行的*基准测试*（比如，作为 CI 系统的一部分，[第 32 条]）允许你发觉代码或者工具链的变更可以如何影响代码的性能。

`[cargo bench]`命令可以运行重复执行特定操作的测试代码，并且计算出这个操作的平均耗时。在撰写本文时，Rust 对基准测试的支持还不太稳定，所以基准测试相关的指令需要通过`cargo +nightly bench`的方式来执行。（Rust 不稳定的特性，包括本文中使用的[test]特性，都描述在 Rust [Unstable Book]中。）

然而，这里存在着编译器给出错误结果的风险，尤其是当你将操作约束在很简单的代码时。考虑如下一个简单的算数函数：

```rust
pub fn factorial(n: u128) -> u128 {
    match n {
        0 => 1,
        n => n * factorial(n - 1),
    }
}
```

这段代码的一个简单的基准测试实现是：

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

输出了一段奇妙的结果：

```rust
test bench_factorial             ... bench:           0 ns/iter (+/- 0)
```

在测试过程中使用固定的输入以及很少的代码指令，编译器可能会对迭代进行优化并且直接输出计算结果。这就将会导致不切实际的测试结论。

使用 [std::hint::black_box] 可以解决这一问题。这是一个标志函数，[编辑器识别后将不]对其进行优化。

上述基准测试可以变更为如下形式：

```rust
#[bench]
fn bench_factorial(b: &mut test::Bencher) {
    b.iter(|| {
        let result = factorial(std::hint::black_box(15));
        assert_eq!(result, 1_307_674_368_000);
    });
}
```

给出了如下更加接近实际的结果：

```rust
test blackboxed::bench_factorial ... bench:          16 ns/iter (+/- 3)
```

[Godbolt 编辑资源管理器]也可以通过展示实际的机器码的方式来辅助测试，这样就能让实际执行的优化后的字节码变得清晰以确认是否过度优化而不能得到实际的结果。

最后，如果你为 Rust 代码准备了基准测试，[criterion]包可能提供了[test::bench::Bencher]的替代品，而且使用起来更加便捷（可以在稳定的 Rust 工具链上运行），功能也更多（支持结果的数据统计及图标）。

## 模糊测试

模糊测试是将代码暴露在随机输入中以期能够发现错误，尤其导致异常的场景，的测试方法。从技术校验的角度来说它已经很重要了，而当你的输入会被其他人填充或者攻击时，它将会显得更加重要 —— **所以如果你的代码输入可能暴露给潜在的攻击者时，你应当使用模糊测试**。

从历史上来看，C/C++ 代码通过模糊测试发现的往往时内存安全问题，通常会通过结合模糊测试与内存访问模式的运行时结合来检测（比如[AddressSanitizer]或者[ThreadSanitizer]）。

Rust 对其中的一些（但并非全部）内存安全问题免疫，尤其是未引入`unsafe`的代码时（[第 16 条]）。然而，Rust 并不能杜绝全部的错误，触发`panic!`（[第 18 条]）的代码仍可能引发导致拒绝服务攻击（DOS）。

模糊测试的最佳实践是以*覆盖率引导*：测试的基础设施监控代码的哪些部分被执行，随机更改输入直至能够触发新的代码路径。“[American fuzzy lop]”（AFL）是其中的佼佼者。但是近些年来，类似的功能已经被引入了 LLVM 的工具链，比如[libFuzzer]。

Rust 编译器是在 LLVM 的基础上构建的，因此[cargo-fuzz]自然地为 Rust 引入了`libFuzzer`（仅在部分平台上可用）。

模糊测试的首要要求是确定代码的入口点，该入口点需要采用（或者可以适应）任意字节的数据作为输入：

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

当目标入口点确定后，Rust [Fuzz Book]给出了如何启动测试的说明。它的核心是一个小型的驱动程序，会将目标入口点连接到模糊测试的基础设施上：

```rust
// fuzz/fuzz_targets/target1.rs file
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let _ = somecrate::is_fuzz(data);
});
```

运行`cargo +nightly fuzz run target1`将会持续使用随机数据来执行模糊测试的目标函数，直至异常出现。上述示例中，错误将被立即发现：

```rust
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

一般来说，模糊测试并不能如此快地发现错误，因此将模糊测试作为 CI 流程中的一部分也是没有意义的。模糊测试的不确定性及随之而来的计算成本意味着你需要考虑如何开展以及何时开展模糊测试 —— 可能仅需要在新的发布版本或者主要变更发生时才运行，或者仅运行确定性的时间。

你也可以通过存储和重用先前模糊测试程序已找到用来触发新的代码路径的语料库的方式来加速模糊测试的执行。这将有助于后续的模糊测试过程将时间放在尝试新的输入上，而非重新测试以前访问过的代码路径。

## 测试的建议

一般性的建议也适用于 Rust 项目中：

* 由于测试是需要持续进行的，**每次变更后都需要在 CI 中执行单测**（除了模糊测试）。
* 当你在修复一个错误时，**在修复前，准备一个能反映错误的测试用例**。这样当你完成错误的修复时，就可以通过测试用例的执行来说名修复效果。并且在未来不会重新引入。
* 如果你的包中包含了某些功能（[第 26 条]），**对所有可能的功能组合都要进行测试**。
* 更一般性的，如果你的包中包含了任何特殊的配置，（比如，`#[cfg(target_os="windows")]），**每种包含了独特配置的平台上的测试都需要运行**。

这些建议包含了很多不同类型的测试，在项目中应当选择那些最有价值的测试。

如果你有很多测试的代码并且会将你的包推送到[crates.io]中，那么就需要考虑下哪些测试项发布后是有意义的。一般地，`cargo`项目中会包含单元测试、集成测试、基准测试以及代码示例（但是并没有包含模糊测试，因为`cargo-fuzz`工具会将模糊测试的内容放置在包的子目录中）等等远超一般用户使用所需要的测试项。如果某些测试项并非是必须的，你可以[移除]一些测试项或者将这些测试项（尤其是行为性的测试）移入单独的测试包中。

## 需要注意的点

* 编写单元测试来达到全面测试的目的，包括仅包含内部代码的测试。通过`cargo test`来运行它们。
* 编写集成测试代码来测试公共的接口。通过`cargo test`来运行它们。
* 编写文档测试来校验公共接口的调用方式。通过`cargo test`来调用它们。
* 编写示例代码来完整的说明如何使用包中的公共 API。通过`cargo test --exmaples`或者`cargo run --example <name>`的方式来运行它们。
* 如果代码对性能有很明确的要求，编写基准测试来确认代码的性能表现。通过`cargo bench`来执行它们。
* 如果代码会暴露在未被信任的输入中，编写模糊测试来确认对输入的参数的约束。通过`cargo fuzz`来（持续地）运行它们。

### 注释


原文[点这里]查看


<!-- 参考链接 -->
[编写单测]: https://doc.rust-lang.org/book/ch11-00-testing.html
[第 23 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_4/item23-wildcard.html
[第 18 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_3/item18-panic.html
[第 28 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_5/item28-use-macros-judiciously.html
[第 27 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_5/item27-document-public-interfaces.html
[第 21 条]: https://www.lurklurk.org/effective-rust/semver.html
[第 32 条]: https://www.lurklurk.org/effective-rust/ci.html
[第 3 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_1/item3-transform.html
[cargo test]: https://doc.rust-lang.org/cargo/commands/cargo-bench.html
[test]: https://doc.rust-lang.org/unstable-book/library-features/test.html
[Unstable Book]: https://doc.rust-lang.org/unstable-book/the-unstable-book.html
[std::hint::black_box]: https://doc.rust-lang.org/std/hint/fn.black_box.html
[编辑器识别后将不]: https://rust-lang.github.io/rfcs/2360-bench-black-box.html
[criterion]: https://crates.io/crates/criterion
[test::bench::Bencher]: https://doc.rust-lang.org/test/bench/struct.Bencher.html
[AddressSanitizer]: https://clang.llvm.org/docs/AddressSanitizer.html
[ThreadSanitizer]: https://clang.llvm.org/docs/ThreadSanitizer.html
[第 16 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_3/item16-unsafe.html
[第 18 条]: https://rustx-labs.github.io/effective-rust-cn/chapter_3/item18-panic.html
[American fuzzy lop]: https://lcamtuf.coredump.cx/afl/
[libFuzzer]: https://llvm.org/docs/LibFuzzer.html
[cargo-fuzz]: https://github.com/rust-fuzz/cargo-fuzz
[Fuzz Book]: https://rust-fuzz.github.io/book/
[crates.io]: https://crates.io/
[移除]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-exclude-and-include-fields
[点这里]: https://www.lurklurk.org/effective-rust/testing.html
