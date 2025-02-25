# 第 18 条：不要 panic

> “它看起来非常复杂，这就是为什么它紧凑的塑料盖子上用大大的友好字母写着 DON'T PANIC 的原因之一。”——Douglas Adams

本条款的标题应当更准确的描述为**更应该返回 `Result` 而不是使用 `panic!`**（但是**不要 panic** 更吸引人）。

Rust 的 panic 机制主要是针对程序中不可恢复的错误而设计的，*默认情况*下会终止发出 `panic!` 的线程。然而，除了默认情况还有其他选择。

特别是，来自具有异常系统的语言（例如 Java 或者 C++）的 Rust 新手通常会使用 [`std::panic::catch_unwind`] 作为模拟异常的方法，因为这似乎提供了一种在调用栈上捕获 panic 的机制。

考虑一个因无效输入而 panic 的函数：

```rust
fn divide(a: i64, b: i64) -> i64 {
    if b == 0 {
        panic!("Cowardly refusing to divide by zero!");
    }
    a / b
}
```

尝试用无效输入调用它，会按预期一样报错：

```rust
// 尝试去计算 0/0 是什么...
let result = divide(0, 0);
```

```
thread 'main' panicked at 'Cowardly refusing to divide by zero!', main.rs:11:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

以下包装函数使用 `catch_unwind` 来捕获 panic：

```rust
fn divide_recover(a: i64, b: i64, default: i64) -> i64 {
    let result = std::panic::catch_unwind(|| divide(a, b));
    match result {
        Ok(x) => x,
        Err(_) => default,
    }
}
```

*似乎*可以正常运行并模拟 `catch`：

```rust
let result = divide_recover(0, 0, 42);
println!("result = {result}");
```

```
result = 42
```

然而，外在具有欺骗性。这种方法的第一个问题是，panic 并不总是被回退（unwind）；有一个[编译器选项]（可通过 _Cargo.toml_ [配置文件]配置）可以改变 panic 后的行为，以便立即终止进程：

```
thread 'main' panicked at 'Cowardly refusing to divide by zero!', main.rs:11:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
/bin/sh: line 1: 29100 Abort trap: 6  cargo run --release
```

这使得任何尝试模拟异常的方式完全受到整个项目配置文件的支配。还有一种情况是，无论编译器或项目配置如何，某些目标平台（例如，WebAssembly）*总会*因 panic 而终止。

一个更微妙的问题是，panic 处理引发了[异常安全]：如果 panic 发生在对数据结构进行操作的过程中，则会消除对数据结构已处于自一致状态的任何保证。自 20 世纪 90 年代以来，在存在异常的情况下保持内部不变量已被证明极其困难；[^1]这也是为什么 [Google （众所周知）禁止在其 C++ 代码中使用异常]的主要原因之一。

最后，panic 传播与 FFI（外部函数接口）绑定（[第 34 条]）的[交互也很差](https://doc.rust-lang.org/nomicon/ffi.html#ffi-and-unwinding)； _应使用 `catch_unwind` 来防止 Rust 代码中的 panic_ 跨 FFI 绑定*传播到非 Rust 调用代码*。

那么除了 `panic!` 之外还有什么方法可以处理错误呢？对于库代码，最好的替代方案就是返回具有合适错误类型的 `Result`（[第 4 条]），以此将错误视为[其他人的问题](https://en.wikipedia.org/wiki/Somebody_else%27s_problem)。这允许使用该库的人自行决定下一步该做什么——这可能涉及通过 `?` 运算符将问题传播给队列中的下一个调用者。

错误传播必须在某处停止，如果你可以控制 `main` 的行为，那么根据经验适当调用 `panic!` 是可以的（或者 `unwrap()`，`expect()` 等等）；此时，就没有其他的调用者可以将错误传播给它了。

即使在库代码中，`panic!` 另一个合理的用处是在极少数遇到错误的情况下，并且你不希望用户必须通过 `.unwrap()` 调用来搞乱他们的代码。

如果错误情况*应当*发生只是因为（比如说）内部数据损坏，而不是由于无效输入，那么触发 `panic!` 是合理的。

允许无效输入引发的 panic 有时甚至是有用的，但这种无效输入应该是不常见的。这样的方法在相关的入点成对出现时效果最好：

- “万无一失”函数，其签名意味着它总是会成功运行（如果不能成功就会 panic）
- “可能出错”函数，返回一个 `Result`

对于前者，Rust 的 [API 指南] 建议 `panic!` 应该记录在内联文档的特定部分中（[第 27 条]）。

标准库中的 [`String::from_utf8_unchecked`] 和 [`String::from_utf8`] 的入点是后者的示例（尽管在这种情况下，panic 实际上被推迟到使用无效输入来构造 `String` 的位置）。

假设你正在尝试遵循本条款给出的建议，则需要牢记以下几点。首先，panic 可能以不同形式出现；避免 `panic!` 的同时也要避免以下情况：

- [`unwrap()`] 和 [`unwrap_err()`]
- [`expect()`] 和 [`expect_err()`]
- [`unreachable!()`]

更难发现的情况如下：

- `slice[index]` 索引超出范围
- `x / y` 当 `y` 是零时

关于避免 panic 的第二种观察是，一个依赖于人类持续保持警觉的计划永远不是一个好主意。

然而，让机器去持续保持警觉是另一回儿事：向你的系统持续集成（详见[第 32 条]）检查系统，以发现新的，潜在的 panic 代码要可靠的多。一个简单的版本可以是针对最常见的 panic 入点进行简单地 grep（如前所述）；更彻底的检查可以涉及使用 Rust 生态系统中的其他工具（[第 31 条]），例如设置一个构建变体，并引入 [`no_panic`] crate。

## 注释

[^1]: Tom Cargill 在 1994 年的[文章《C++ Report》](https://ptgmedia.pearsoncmg.com/imprint_downloads/informit/aw/meyerscddemo/DEMO/MAGAZINE/CA_FRAME.HTM)中探讨了保持 C++ 模板代码的异常安全性是多么困难，Herb Sutter 的 [Guru of the Week #8](http://www.gotw.ca/gotw/008.htm) 也有类似的讨论。

原文[点这里](https://www.lurklurk.org/effective-rust/panic.html)查看

<!-- 参考链接 -->

[第 4 条]: ../chapter_1/item4-errors.md
[第 27 条]: ../chapter_5/item27-document-public-interfaces.md
[第 31 条]: ../chapter_5/item31-use-tools.md
[第 32 条]: ../chapter_5/item32-ci.md
[第 34 条]: ../chapter_6/item34-ffi.md
[`std::panic::catch_unwind`]: https://doc.rust-lang.org/std/panic/fn.catch_unwind.html
[编译器选项]: https://doc.rust-lang.org/rustc/codegen-options/index.html#panic
[配置文件]: https://doc.rust-lang.org/cargo/reference/profiles.html#panic
[异常安全]: https://en.wikipedia.org/wiki/Exception_safety
[Google （众所周知）禁止在其 C++ 代码中使用异常]: https://google.github.io/styleguide/cppguide.html#Exceptions
[API 指南]: https://rust-lang.github.io/api-guidelines/documentation.html#function-docs-include-error-panic-and-safety-considerations-c-failure
[`String::from_utf8_unchecked`]: https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8_unchecked
[`String::from_utf8`]: https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8
[`unwrap()`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap
[`unwrap_err()`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap_err
[`expect()`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.expect
[`expect_err()`]: https://doc.rust-lang.org/std/result/enum.Result.html#method.expect_err
[`unreachable!()`]: https://doc.rust-lang.org/std/macro.unreachable.html
[`no_panic`]: https://docs.rs/no-panic
