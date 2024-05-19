# 第 1 条：使用类型系统表达你的数据结构

> “谁叫他们是程序员，而不是打字员” —— [@thingskatedid](https://twitter.com/thingskatedid/status/1400213496785108997)

对于来自其他静态类型编程语言（如 C++、Go 或 Java）的人来说，Rust 类型系统的基本概念是非常熟悉的。有一系列具有特定大小的整数类型，包括有符号（i8, i16, i32, i64, i128）和无符号（u8, u16, u32, u64, u128）。

还有两种整数类型，其大小与目标系统上的指针大小匹配：有符号（isize）和无符号（usize）。Rust 并不是那种会在指针和整数之间进行大量转换的语言，所以这种特性并不是特别相关。然而，标准集合返回它们的大小作为一个 usize（来自 .len()），所以集合索引意味着 usize 值非常常见 —— 从容量的角度来看，这是显然没有问题的，因为内存中的集合不可能有比系统上的内存地址更多的项。

整数类型确实让我们第一次意识到 Rust 是一个比 C++ 更严格的世界 —— 尝试将一个 quart（i32）放入 pint pot（i16）会在编译时产生错误。


```rust
let x: i32 = 42;
let y: i16 = x;
```
```rust
error[E0308]: mismatched types
  --> use-types/src/main.rs:14:22
   |
14 |         let y: i16 = x;
   |                ---   ^ expected `i16`, found `i32`
   |                |
   |                expected due to this
   |
help: you can convert an `i32` to an `i16` and panic if the converted value doesn't fit
   |
14 |         let y: i16 = x.try_into().unwrap();
   |                       ++++++++++++++++++++
```

这让人感到安心：当程序员进行有风险的操作时，Rust 不会安静地坐视不管。这也早早地表明，尽管 Rust 有更严格的规则，但它也有助于编译器消息指向如何遵守规则的方法。

建议的解决方案是抛出一个问题，即如何处理转换会改变值的情况，关于`错误处理`（[第4条]）和使用 `panic!`（[第18条]）我们将在后面有更多的讨论。

Rust 也不允许一些可能看起来“安全”的操作：


```rust
let x = 42i32; // Integer literal with type suffix
let y: i64 = x;
```
```rust
error[E0308]: mismatched types
  --> use-types/src/main.rs:23:22
   |
23 |         let y: i64 = x;
   |                ---   ^ expected `i64`, found `i32`
   |                |
   |                expected due to this
   |
help: you can convert an `i32` to an `i64`
   |
23 |         let y: i64 = x.into();
   |                       +++++++
```

在这里，建议的解决方案并没有提出错误处理的方法，但转换仍然需要是显式的。我们将在后面章节更详细地讨论类型转换（[第6条]）。

现在继续探讨不出乎意料的原始类型，Rust 有布尔类型（`bool`）、浮点类型（`f32`, `f64`）和单元类型 `()`（类似于 `C` 的 `void`）。

更有趣的是 `char` 字符类型，它持有一个 [`Unicode` 值]（类似于 Go 的 [`rune 类型`]）。尽管它在内部以 `4 字节`存储，但与 `32 位`整数的转换仍然不会有静默转换。

类型系统中的这种精确性迫使你明确地表达你想要表达的内容 —— u32 值与 char 不同，后者又与序列 UTF-8 字节不同，这又与序列任意字节不同，而且需要你准确地指定你的意思[^1]。[Joel Spolsky 的著名博客]文章可以帮助你理解需要哪种类型。

当然，有一些辅助方法允许你在这不同的类型之间进行转换，但它们的签名迫使你处理（或明确忽略）失败的可能性。例如，一个 `Unicode` 代码点[^2] 总是可以用 `32 位`表示，所以 `'a' as u32` 是允许的，但反向转换就比较复杂了（因为有些 u32 值不是有效的 Unicode 代码点），例如：

* [char::from_u32] 返回一个 `Option<char>`，迫使调用者处理失败的情况
* [char::from_u32_unchecked] 假设有效性，但由于结果是未定义的，因此被标记为`unsafe`，迫使调用者也使用`unsafe`（[第16条]）。

## 聚合类型

继续讨论聚合类型，Rust 有：
- 数组（`Arrays`），它们持有单个类型的多个实例，实例的数量在编译时已知。例如 `[u32; 4]` 是四个连续的 4 字节整数。
- 元组（`Tuples`），它们持有多个异构类型的实例，元素的数量和类型在编译时已知，例如 `(WidgetOffset, WidgetSize, WidgetColour)`。如果元组中的类型不够独特 —— 例如 `(i32, i32, &'static str, bool)` —— 最好给每个元素命名并使用 …
- 结构体（`Structs`），它们也持有编译时已知的异构类型实例，但是允许通过名称来引用整个类型和各个字段。
- 元组结构体（`Tuple structs`）是结构体和元组的杂交体：整个类型有一个名称，但各个字段没有名称 —— 它们通过数字来引用：`s.0`, `s.1` 等。


```rust
struct TextMatch(usize, String);
let m = TextMatch(12, "needle".to_owned());
assert_eq!(m.0, 12);
```

这让我们来到了 Rust 类型系统的皇冠上的宝石：枚举（`enum`）。

在其基本形式中，很难看出有什么值得兴奋的。与其他语言一样，枚举允许你指定一组互斥的值，可能附带一个数字或字符串值。


```rust
enum HttpResultCode {
   Ok = 200,
   NotFound = 404,
   Teapot = 418,
}
let code = HttpResultCode::NotFound;
assert_eq!(code as i32, 404);
```

因为每个枚举定义都创建了一个独特的类型，这可以用来提高那些接受布尔参数的函数的可读性和可维护性。例如：

```rust
print_page(/* both_sides= */ true, /* colour= */ false);
```

可以用 `enum` 替换：

```rust
pub enum Sides {
   Both,
   Single,
}

pub enum Output {
   BlackAndWhite,
   Colour,
}

pub fn print_page(sides: Sides, colour: Output) {
   // ...
}
```

在调用处更加类型安全，而且易于阅读：

```rust
print_page(Sides::Both, Output::BlackAndWhite);
```

不同于布尔版本，如果使用该库的用户不小心颠倒了参数的顺序，编译器会立即报错：

```rust
error[E0308]: mismatched types
  --> use-types/src/main.rs:89:20
   |
89 |         print_page(Output::BlackAndWhite, Sides::Single);
   |                    ^^^^^^^^^^^^^^^^^^^^^ expected enum `enums::Sides`, found enum `enums::Output`
error[E0308]: mismatched types
  --> use-types/src/main.rs:89:43
   |
89 |         print_page(Output::BlackAndWhite, Sides::Single);
   |                                           ^^^^^^^^^^^^^ expected enum `enums::Output`, found enum `enums::Sides`
```

> 使用新类型模式（[第7条]）来包装一个 `bool` 也可以实现类型安全和可维护性；如果语义始终是布尔型的，通常最好使用这种方式，如果将来可能会出现新的选择（例如 `Sides::BothAlternateOrientation`），则应使用`枚举`。

Rust 枚举的类型安全性在 `match` 表达式中继续体现出以下这段代码无法编译：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let msg = match code {
   HttpResultCode::Ok => "Ok",
   HttpResultCode::NotFound => "Not found",
   // forgot to deal with the all-important "I'm a teapot" code
};
```

```rust
error[E0004]: non-exhaustive patterns: `Teapot` not covered
  --> use-types/src/main.rs:65:25
   |
51 | /     enum HttpResultCode {
52 | |         Ok = 200,
53 | |         NotFound = 404,
54 | |         Teapot = 418,
   | |         ------ not covered
55 | |     }
   | |_____- `HttpResultCode` defined here
...
65 |           let msg = match code {
   |                           ^^^^ pattern `Teapot` not covered
   |
   = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
   = note: the matched value is of type `HttpResultCode`
```

编译器强制程序员考虑枚举所表示的所有可能性，即使结果只是添加一个默认分支 `_ => {}`。

> 注意，现代 C++ 编译器能够并且会对枚举缺失的switch分支发出警告。

## 带有字段的`枚举`

Rust枚举特性的真正强大之处在于每个变体都可以携带数据，使其成为一个[代数数据类型]（ADT）。这对于主流语言的程序员来说不太熟悉；在C/C++的术语中，它类似于枚举与联合的组合 —— 只是类型安全的。

这意味着程序数据结构的不变式可以被编码到 Rust 的类型系统中；不符合那些不变式状态的代码甚至无法编译。一个设计良好的枚举使得创建者的意图对于人类以及编译器都是清晰的：

```rust
pub enum SchedulerState {
    Inert,
    Pending(HashSet<Job>),
    Running(HashMap<CpuId, Vec<Job>>),
}
```

仅从类型定义来看，可以合理猜测 Job 在 Pending 状态中排队，直到调度器完全激活，此时它们被分配到某个特定 CPU 的池中。

这突出了本方法的中心主题，即使用 Rust 的类型系统来表达与软件设计相关的概念。

当一个字段或参数何时有效需要通过注释来解释时，这就是一个明显的迹象表明这种情况没有发生：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
struct DisplayProps {
    x: u32,
    y: u32,
    monochrome: bool,
    // `fg_colour` must be (0, 0, 0) if `monochrome` is true.
    fg_colour: RgbColour,
}
```

这是一个非常适合用带有数据的`枚举`来替换的结构体：

```rust
#[derive(Debug)]
enum Colour {
    Monochrome,
    Foreground(RgbColour),
}

struct DisplayProperties {
    x: u32,
    y: u32,
    colour: Colour,
}
```

这个简单的例子说明了一个关键的建议：让你的类型无法表达无效状态。只支持有效值组合的类型意味着整类的错误会被编译器拒绝，从而使得代码更小、更安全。

## 选项与错误

回到枚举的强大功能，有两个概念非常常见，以至于Rust内置了枚举类型来表达它们。

第一个是Option的概念：要么存在特定类型的值（`Some(T)`），要么不存在（`None`）。始终为可能缺失的值使用 `Option`；永远不要退回到使用哨兵值（`-1`, `nullptr`, …）来试图在带内表达相同的概念。

然而，有一个微妙的点需要考虑。如果您处理的是事物的集合，您需要决定集合中没有任何事物是否与没有集合相同。在大多数情况下，这种区别不会出现，您可以继续使用 `Vec<Thing>`：零个事物意味着事物的缺失。

然而，确实存在其他罕见的情况，需要用 `Option<Vec<Thing>>` 来区分这两种情况 —— 例如，加密系统可能需要区分“负载单独传输”和“提供空负载”。（这与 `SQL` 中 `NULL` 标记列的争论有关。）

一个常见的边缘情况是 `String` 可能缺失 —— 是用 `""` 还是 `None` 来表示值的缺失更有意义？无论哪种方式都可以，但 `Option<String>` 清楚地传达了可能缺失该值的可能性。

第二个常见的概念源于`错误处理`：如果一个函数失败，应该如何报告这个失败？历史上，使用了特殊的哨兵值（例如，`Linux 系统调用` 的 `-errno` 返回值）或全局变量（`POSIX 系统`的`errno`）。近年来，支持函数返回多个或元组返回值的语言（如Go）可能有一个约定，即返回一个`(result, error)`对，假设在错误非“零”时，结果存在合适的“零”值。

在Rust中，始终将可能失败的操作的 结果编码为 `Result<T, E>`。`T 类型`保存成功的结果（在`Ok`变体中），`E 类型`在失败时保存错误详情（在`Err`变体中）。使用标准类型使得设计意图清晰，并且允许使用标准转换（[第3条]）和错误处理（[第4条]）；它还使得使用 `?` 运算符来简化错误处理成为可能。

---

#### 注释

[^1]: 如果涉及到文件系统，情况会更加复杂，因为流行平台上的文件名介于任意字节和 UTF-8 序列之间：请参阅 [std::ffi::OsString] 文档。

[^2]: 技术上，是一个 Unicode 标量值，而不是代码点。

[^3]: 这也意味着在库中为一个现有枚举添加一个新的变体是一个破坏性的更改（[第21条]）：库的客户需要更改他们的代码以适应新的变体。如果一个枚举实际上只是一个旧式的值列表，可以通过将其标记为 non_exhaustive 枚举来避免这种行为；请参阅[第21条]。


原文[点这里](https://www.lurklurk.org/effective-rust/use-types.html)查看

<!-- 参考链接 -->

[第3条]: item3-transform.md
[第4条]: item4-errors.md
[第6条]: https://www.lurklurk.org/effective-rust/casts.html
[第7条]: item7-builder.md
[第16条]: https://www.lurklurk.org/effective-rust/unsafe.html
[第18条]: https://www.lurklurk.org/effective-rust/panic.html
[第21条]: https://www.lurklurk.org/effective-rust/semver.html


[char::from_u32]: https://doc.rust-lang.org/std/primitive.char.html#method.from_u32
[char::from_u32_unchecked]: https://doc.rust-lang.org/std/primitive.char.html#method.from_u32_unchecked

[Joel Spolsky 的著名博客]:https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/
[`Unicode` 值]: http://www.unicode.org/glossary/#unicode_scalar_value
[`rune 类型`]: https://golang.org/doc/go1#rune
[代数数据类型]: https://en.wikipedia.org/wiki/Algebraic_data_type

[std::ffi::OsString]: https://doc.rust-lang.org/std/ffi/struct.OsString.html
