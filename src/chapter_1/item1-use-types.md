# 第 1 条：使用类型系统表达你的数据结构

> “谁叫他们是程序员，而不是打字员” —— [@thingskatedid](https://twitter.com/thingskatedid/status/1400213496785108997)

本章从编译器提供的基本类型开始，再到各种将数据组合成数据结构的方式，对 Rust 的类型系统做了一个快速导览。

在 Rust 中，枚举（`enum`）类型扮演了重要的角色。虽然基础版本和其他语言相比没有什么区别，但枚举变量与数据字段相结合的能力，提供了更强的灵活性和表达能力。

## 基础类型

对于熟悉其他静态类型编程语言（如 C++、Go 或 Java）的人来说，Rust 类型系统的基本概念应该不太陌生。它包括一系列具有特定大小的整数类型，包括有符号（[`i8`][i8]，[`i16`][i16]，[`i32`][i32] ，[`i64`][i64]， [`i128`][i128]）和无符号（[`u8`][u8] ,  [`u16`][u16] ， [`u32`][u32] ， [`u64`][u64] ， [`u128`][u128]）类型。

还有两种整数类型：有符号（[`isize`][isize]）和无符号（[`usize`][usize]），其大小与目标系统上的指针大小一致。实际上，在 Rust 中，并不会在指针和整数之间进行大量的转换，所以大小是否相等并不重要。但是，标准集合是用 `usize`（来自 `.len()` 方法）类型来返回它们的大小，所以在处理集合索引的时候， `usize` 非常常见 —— 从容量的角度来看，这显然是没问题的，因为内存中集合的项数不可能比系统内存的寻址范围更大。

整数类型让我们第一次意识到 Rust 是一个比 C++ 更严格的世界 —— 在 Rust 中，尝试将一个更大范围的整数类型（`i32`）的值赋给较小范围的整数类型（`i16`）会在编译时产生错误。


```rust
let x: i32 = 42;
let y: i16 = x;
```
```
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

这让人感到安心：当程序员进行有风险的操作时，Rust 不会坐视不理，虽然本例中给出的数值没有超出目标类型 `i16` 能够表达的范围，转换不会有问题，但 Rust 编译器仍然会考虑到转换有*不*成功的可能性：

```rust
let x: i32 = 66_000;
let y: i16 = x; // `y` 的值是什么？
```

从错误提示中，我们也可以获得对 Rust 的初步认识： Rust 虽然拥有严格的规则，但编译失败时也会给出有用的提示信息，指引我们如何遵守这些规则。编译器还给出了解决方案，引导我们如何处理转换过程中可能出现的超出范围的情况。关于错误处理（[第 4 条][第 4 条]）和使用 `panic!`（[第 18 条][第 18 条]）后续我们将会有更多的讨论。

Rust 也不允许一些可能看起来“安全”的操作，哪怕是从更小的整数类型向更大的转换：


```rust
let x = 42i32; // 带有类型后缀的整数字面量
let y: i64 = x;
```
```
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

在这里，转换并不会有引发错误的担忧，但转换仍然需要是显式的。我们将在后面章节更详细地讨论类型转换（[第 5 条][第 5 条]）。

现在继续探讨其他原始类型，Rust 有布尔类型（[`bool`][bool]）、浮点类型（[`f32`][f32]， [`f64`][f64]）和[单元类型][unit type] [`()`][unit]（类似于 `C` 的 `void`）。

更有趣的是 [`char`][char] 字符类型，它持有一个 [`Unicode` 值][Unicode]（类似于 Go 的 [`rune` 类型][rune]）。尽管它在内部以 4 字节存储，但与 32 位整数仍然不支持静默转换。

类型系统的这种精确性迫使你明确地表达你想要表达的内容 —— `u32` 值与 `char` 不同，后者又与序列 UTF-8 字节不同，这又与任意字节序列不同，最终取决于你的准确意图 [^1]。[Joel Spolsky 的著名博客]文章可以帮助理解具体需要哪种数据类型。

当然，有一些辅助方法允许在这不同的类型之间进行转换，但它们的签名迫使你处理（或明确忽略）失败的可能性。例如，一个 `Unicode` 代码点 [^2] 总是可以用 32 位表示，所以 `'a' as u32` 是允许的，但反向转换就比较复杂了（因为 `u32` 值不一定是有效的 Unicode 代码点），例如：

* [char::from_u32][char::from_u32] 返回一个 `Option<char>`，迫使调用者处理失败的情况。
* [char::from_u32_unchecked][char::from_u32_unchecked] 假设有效性，但由于结果是未定义的，因此被标记为 `unsafe`，迫使调用者也使用 `unsafe`（[第 16 条][第 16 条]）。

## 聚合类型

接下来讨论聚合类型，Rust 有：

- 数组（[*Arrays*][Array]），它们持有单个类型的多个实例，实例的数量在编译时已知。例如 `[u32; 4]` 是四个连续的 4 字节整数。
- 元组（[*Tuples*][Tuple]），它们持有多个异构类型的实例，元素的数量和类型在编译时已知，例如 `(WidgetOffset, WidgetSize, WidgetColour)`。如果元组中的类型不够独特 —— 例如 `(i32, i32, &'static str, bool)` —— 最好给每个元素命名并使用，或者使用结构体。
- 结构体（[*Structs*][Struct]），它们也持有编译时已知的异构类型实例，但是允许通过名称来引用整个类型和各个字段。

Rust 也支持元组结构体（*Tuple structs*），它是结构体和元组的杂交体：整个类型有一个名称，但各个字段没有名称 —— 它们通过数字索引来使用：`s.0`, `s.1` 。

```rust
/// 拥有 2 个未命名字段的结构体
struct TextMatch(usize, String);

// 按顺序传入值来构构造实例
let m = TextMatch(12, "needle".to_owned());

// 使用字段下标索引访问其值
assert_eq!(m.0, 12);
```
## 枚举（`enum`）

让我们认识一下 Rust 类型系统中皇冠上的宝石：枚举（`enum`）。

在其基本形式中，很难看出有什么值得兴奋的。与其他语言一样，枚举允许你指定一组互斥的值，这些值还可能附带一个数字或字符串值。


```rust
enum HttpResultCode {
   Ok = 200,
   NotFound = 404,
   Teapot = 418,
}
let code = HttpResultCode::NotFound;
assert_eq!(code as i32, 404);
```

因为每个枚举定义都创建了一个独特的类型，这可以用来提高那些接受诸如布尔参数的函数的可读性和可维护性。例如：

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

在调用的时候是类型安全的，而且易于阅读：

```rust
print_page(Sides::Both, Output::BlackAndWhite);
```

不同于使用布尔值的版本，如果使用该库的用户不小心颠倒了参数的顺序，编译器会立即报错：

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

使用 newtype 模式（[第 6 条][第 6 条]）来包装一个 `bool` 也可以实现类型安全和可维护性；如果语义始终是布尔型的，通常最好使用这种方式，如果将来可能会出现新的选择（例如 `Sides::BothAlternateOrientation`），则应使用枚举类型。

Rust 枚举的类型安全性也延续到 `match` 表达式中。下面这段代码无法编译：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let msg = match code {
   HttpResultCode::Ok => "Ok",
   HttpResultCode::NotFound => "Not found",
   // forgot to deal with the all-important "I'm a teapot" code
};
```

```
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

编译器强制程序员处理枚举类型*所有*的可能性 [^3]，即使只是添加一个默认分支 `_ => {}` 来处理其他情况。（现代 C++ 编译器也能够并且会对枚举缺失的 `switch` 分支发出警告。）

## 带有字段的枚举

Rust 枚举特性的真正强大之处在于每个变体都可以携带数据，使其成为一个[*代数数据类型*][代数数据类型]（ADT）。这对于主流语言的程序员来说不太熟悉。在 C/C++ 的术语中，它类似于枚举与联合（`union`）的组合，并且在 Rust 是类型安全的。

这意味着程序数据结构的不变性可以被编码到 Rust 的类型系统中；不符合不变性状态的代码甚至无法编译。一个设计良好的枚举使得创建者的意图对于人类以及编译器都是清晰的：

```rust
pub enum SchedulerState {
    Inert,
    Pending(HashSet<Job>),
    Running(HashMap<CpuId, Vec<Job>>),
}
```

仅从类型定义来看，可以合理猜测 `Job` 在 `Pending` 状态中排队，直到调度器完全激活，此时它们被分配到某个特定 CPU 的池中。

这突出了本方法的中心主题，即使用 Rust 的类型系统来表达与软件设计相关的概念。

如果代码中需要一些注释来解释什么样的值对这个字段是有效的：

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

这种情况适合用带有数据的枚举来替换：

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

这个简单的例子说明了一个关键的建议：**让你的类型无法表达无效状态**。只支持有效值组合的类型意味着不符合期望的错误会被编译器拒绝，从而使得代码更小、更安全。

## 常用的枚举类型

鉴于枚举的强大功能，有两个概念非常常见，以至于 Rust 标准库内置了枚举类型来表达它们，这些类型在 Rust 代码中随处可见。

### `Option<T>`

第一个是 [`Option`][Option] ：要么存在特定类型的值（`Some(T)`），要么不存在（`None`）。**始终为可能缺失的值使用 `Option`**，永远不要退回到使用哨兵值（`-1`， `nullptr`，……）来表达值的缺失。

然而，有一个微妙的点需要考虑。当您在处理一个包含多个事物的**集合**时，您需要考虑一个情况：集合中没有任何事物（即事物的数量为零）是否等同于没有这个集合。在大多数情况下，这两种情况并无不同，您可以继续使用 `Vec<Thing>`：0 长度的集合表示元素的缺失。

然而，确实存在其他罕见的情况，需要用 `Option<Vec<Thing>>` 来区分这两种情况 —— 例如，加密系统可能需要区分[“负载单独传输”][payload]和“空负载”。（这与 SQL 中 [`NULL` 标记][null marker]列的争论有关。）

同样的，一个常见的边缘情况是 `String` 可能缺失 —— 是用 `""` 还是 `None` 来表示值的缺失更有意义？无论哪种方式都可以，但 `Option<String>` 清楚地传达了可能缺失该值的可能性。

### `Result<T, E>`

第二个常见的概念源于错误处理：如果一个函数执行失败，应该如何报告这个失败？历史上，使用了特殊的哨兵值（例如，Linux 系统调用 的 `-errno` 返回值）或全局变量（POSIX 系统的 `errno`）。最近，一些支持从函数返回多个值或元组值的语言（例如 Go），通常会采用返回一个  `(result, error)` 对的惯例。假设在出现错误时，结果存在某种合适的“零”值。

在 Rust 中，有一个枚举专门用于此目的：**始终将可能失败的操作的 结果编码为 [`Result<T, E>`][Result]**。`T` 保存成功的结果（在 `Ok` 变体中），`E` 在失败时保存错误详情（在 `Err` 变体中）。

使用标准类型使得设计意图更清晰，并且允许使用标准转换（[第 3 条][第 3 条]）和错误处理（[第 4 条][第 4 条]）；它还使得使用 `?` 运算符来简化错误处理成为可能。

---

#### 注释

[^1]: 如果涉及到文件系统，情况会更加复杂，因为在流行各种平台上，文件名介于任意字节和 UTF-8 序列之间：请参阅 [std::ffi::OsString][std::ffi::OsString] 文档。

[^2]: 技术上，是一个 *Unicode 标量值*，而不是代码点。

[^3]: 这也意味着在库中为一个现有枚举添加一个新的变体是一个破坏性的更改（[第 21 条][第 21 条]）：库的用户需要更改他们的代码以适应新的变体。如果一个枚举实际上只是一个旧式的值列表，可以通过将其标记为 [`non_exhaustive`][non_exhaustive] 枚举来避免这种行为；请参阅 [第 21 条][第 21 条]。


原文[点这里](https://www.lurklurk.org/effective-rust/use-types.html)查看

<!-- 参考链接 -->

[第 3 条]: ./item3-transform.md
[第 4 条]: ./item4-errors.md
[第 5 条]: ./item5-casts.md
[第 6 条]: ./item6-newtype.md
[第 7 条]: ./item7-builder.md
[第 16 条]: ../chapter_3/item16-unsafe.md
[第 18 条]: ../chapter_3/item18-panic.md
[第 21 条]: ../chapter_4/item21-semver.md
[i8]: https://doc.rust-lang.org/std/primitive.i8.html
[i16]: https://doc.rust-lang.org/std/primitive.i16.html
[i32]: https://doc.rust-lang.org/std/primitive.i32.html
[i64]: https://doc.rust-lang.org/std/primitive.i64.html
[i128]: https://doc.rust-lang.org/std/primitive.i128.html
[u8]: https://doc.rust-lang.org/std/primitive.u8.html
[u16]: https://doc.rust-lang.org/std/primitive.u16.html
[u32]: https://doc.rust-lang.org/std/primitive.u32.html
[u64]: https://doc.rust-lang.org/std/primitive.u64.html
[u128]: https://doc.rust-lang.org/std/primitive.u128.html
[isize]: https://doc.rust-lang.org/std/primitive.isize.html
[usize]: https://doc.rust-lang.org/std/primitive.usize.html
[bool]: https://doc.rust-lang.org/std/primitive.bool.html
[f32]: https://doc.rust-lang.org/std/primitive.f32.html
[f64]: https://doc.rust-lang.org/std/primitive.f64.html
[unit type]: https://en.wikipedia.org/wiki/Unit_type
[unit]: https://doc.rust-lang.org/std/primitive.unit.html
[char]: https://doc.rust-lang.org/std/primitive.char.html
[char::from_u32]: https://doc.rust-lang.org/std/primitive.char.html#method.from_u32
[char::from_u32_unchecked]: https://doc.rust-lang.org/std/primitive.char.html#method.from_u32_unchecked
[Joel Spolsky 的著名博客]:https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/
[Unicode]: http://www.unicode.org/glossary/#unicode_scalar_value
[rune]: https://golang.org/doc/go1#rune
[Array]: https://doc.rust-lang.org/std/primitive.array.html
[Tuple]: https://doc.rust-lang.org/std/primitive.tuple.html
[Struct]: https://doc.rust-lang.org/std/keyword.struct.html
[代数数据类型]: https://en.wikipedia.org/wiki/Algebraic_data_type
[Option]: https://doc.rust-lang.org/std/option/enum.Option.html
[Result]: https://doc.rust-lang.org/std/result/enum.Result.html
[payload]: https://tools.ietf.org/html/rfc8152#section-4.1
[null marker]: https://en.wikipedia.org/wiki/Null_(SQL)
[std::ffi::OsString]: https://doc.rust-lang.org/std/ffi/struct.OsString.html
[non_exhaustive]: https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute
