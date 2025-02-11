# 第 5 条：理解类型转换

Rust 的类型转换分为三个类别：

- 手动：通过实现 `From trait` 和 `Into trait` 来提供的用户自定义类型转换
- 半自动：使用 `as` 关键字在值之间进行的显式类型转换（`Casts`）
- 自动：通过隐式类型转换（`Coercion`）转换为新的类型

本章节的重点主要是第一种，即手动转换类型，因为后两种大多数情况下不适用于用户定义类型的转换。但也有一些例外，所以本章节最后的部分将讨论显式类型转换和隐式类型转换 —— 包括它们如何适用于用户自定义的类型。

请注意，与许多较旧的语言不同，`Rust` 在数值类型之间不会执行自动转换。这甚至适用于整数类型的“安全”转换：

```rust
let x: u32 = 2;
let y: u64 = x;
```

```rust
error[E0308]: mismatched types
  --> src/main.rs:70:18
   |
70 |     let y: u64 = x;
   |            ---   ^ expected `u64`, found `u32`
   |            |
   |            expected due to this
   |
help: you can convert a `u32` to a `u64`
   |
70 |     let y: u64 = x.into();
   |                   +++++++

```

## 用户自定义类型转换

与语言的其他特性（[第 10 条]）一样，在不同用户自定义类型值之间执行转换的能力被封装为标准 `trait` —— 或者更确切地说，是一组相关的泛型 `trait`。

表达类型值转换能力的四个相关 `trait` 如下：
- `From<T>`：这种类型的项可以由类型 `T` 的项构建，并且转换总是成功。
- `TryFrom<T>`：这种类型的项可以由类型 `T` 的项构建，但转换可能不会成功。
- `Into<T>`：这种类型的项可以转换为类型 `T` 的项，并且转换总是成功。
- `TryInto<T>`：这种类型的项可以转换为类型 `T` 的项，但转换可能不会成功。

鉴于[第 1 条]中关于在类型系统中表达事物的讨论，不难发现 `Try...` 变体的区别在于，其唯一的 `trait` 方法所返回的是一个 `Result` 而不是一个确保存在的新项。`Try... trait` 定义还要求一个关联类型，它给出了失败情况下发出的错误 `E` 的类型。

因此，第一条建议是，如果可能转换失败，则 **（仅）实现`Try... trait`**，与[第 4 条]一致。另一种方法是**忽略错误的可能性（例如，使用 `.unwrap()`）**，但这需要是深思熟虑的选择，在大多数情况下，最好将这个选择留给调用者。

类型转换 `trait` 具有明显的对称性：如果类型 `T` 可以转换为类型 `U`（通过 `Into<U>`），难道这不等于可以通过从类型 `T` 的项转换来创建类型 `U` 的项（通过 `From<T>`）吗？

确实如此，这导致了第二条建议：为转换实现 `From trait`。`Rust` 标准库必须在这两个可能性中选择一个，以防止系统在眩晕的圆圈中旋转，[^1] 它选择了自动提供 `From` 实现的 `Into`。

如果你正在使用这两个 `trait` 中的一个来作为你自己的新泛型的 `trait` 约束，那么建议是相反的：**对 `trait` 约束使用 `Into trait`**。这样，约束将同时满足直接实现 `Into` 的内容和仅直接实现 `From` 的内容。

`From` 和 `Into` 的文档强调了这种自动转换，但标准库代码的相关部分也值得一读，这是一个通用的 `trait` 实现：

```rust
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

将 `trait` 规范翻译成文字可以帮助理解更复杂的 `trait` 约束。在这个案例中，它相当简单："只要 `U` 已经实现了 `From<T>`，我就可以为类型 `T` 实现 `Into<U>`"。

标准库还包括了为标准库类型实现这些转换 `trait` 的各种情况。正如你所预期的，对于整数转换，当目标类型包括源类型的所有可能值时（例如，`u64` 的 `From<u32>`），会有 `From` 实现，而当源值可能不适合目标时（例如，`u32` 的 `TryFrom<u64>`），会有 `TryFrom` 实现。

除了前面显示的 `Into` 版本的泛型 `trait` 实现之外，还有各种其他的通用 `trait` 实现，主要用于智能指针类型，允许智能指针从其持有的类型的实例自动构造。这意味着接受智能指针参数的泛型方法也可以用普通的旧项调用；更多内容将在后续介绍和[第 8 条]中展开。

`TryFrom trait` 还有一个通用实现，适用于任何已经以相反方向实现 `Into trait` 的类型 —— 这自动包括了（如先前所示）以相同方向实现 `From` 的任何类型。换句话说，如果你可以无误地将 `T` 转换为 `U`，你也可以尝试从 `T` 获取 `U`；由于这种转换总是成功，所以与之关联的错误类型是 `Infallible`。[^2]

还有一个泛型的 `From` 实现方式值得注意，那就是自反实现：

```rust
impl<T> From<T> for T {
    fn from(t: T) -> T {
        t
    }
}
```

翻译成文字，这只是说“给定一个 `T` ，我可以获得一个 `T` ”。这是一个如此明显的“嗯，当然了”一样的废话，但是值得停下来理解为什么它是有用的。

考虑一个简单的 `newtype` 结构体（[第 6 条]）和一个对其操作的函数（虽然这个函数更应该被表示为一个方法）：

```rust
/// Integer value from an IANA-controlled range.
#[derive(Clone, Copy, Debug)]
pub struct IanaAllocated(pub u64);

/// Indicate whether value is reserved.
pub fn is_iana_reserved(s: IanaAllocated) -> bool {
    s.0 == 0 || s.0 == 65535
}
```

这个函数可以使用结构体的实例来调用：

```rust
let s = IanaAllocated(1);
println!("{:?} reserved? {}", s, is_iana_reserved(s));
// output: "IanaAllocated(1) reserved? false"
```

但是，即使为 newtype 包装实现了 `From<u64>`：

```rust
impl From<u64> for IanaAllocated {
    fn from(v: u64) -> Self {
        Self(v)
    }
}
```

该函数也不能直接以 `u64` 值调用：

```rust
if is_iana_reserved(42) {
    // ...
}
```

```rust
error[E0308]: mismatched types
  --> src/main.rs:77:25
   |
77 |     if is_iana_reserved(42) {
   |        ---------------- ^^ expected `IanaAllocated`, found integer
   |        |
   |        arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:7:8
   |
7  | pub fn is_iana_reserved(s: IanaAllocated) -> bool {
   |        ^^^^^^^^^^^^^^^^ ----------------
help: try wrapping the expression in `IanaAllocated`
   |
77 |     if is_iana_reserved(IanaAllocated(42)) {
   |                         ++++++++++++++  +
```

但是，该函数的能接受（并显式转换）任何满足 `Into<IanaAllocated>` 的内容的泛型版本:

```rust
pub fn is_iana_reserved<T>(s: T) -> bool
where
    T: Into<IanaAllocated>,
{
    let s = s.into();
    s.0 == 0 || s.0 == 65535
}
```

是允许这种使用的：

```rust
if is_iana_reserved(42) {
    // ...
}
```

有了这个 `trait` 约束， `From<T>` 的自反 `trait` 实现就有意义了：这意味着泛型函数可以处理已经是 `IanaAllocated` 实例的项，不需要转换。

这种模式还解释了为什么（以及如何）`Rust` 代码有时好像会在类型之间进行隐式转换：`From<T>` 实现与 `Into<T>` `trait` 约束的组合，导致了在调用点看似神奇、但在幕后仍然是安全和显式转换的代码。当与引用类型及其相关转换 `trait` 结合时，这种模式变得更为强大；更多内容见[第 8 条]。

## 显式类型转换（Casts）

`Rust` 包含 `as` 关键字以在某些类型对之间执行显式转换。

可以通过这种方式转换的类型对构成了一个相当有限的集合，并且它包括的唯一用户自定义类型是“类 `C`”的枚举（那些只有相关联的整数值的枚举）。尽管如此，它还是包括了常规整数转换，为 `into()` 提供了一个替代方案：

```rust
let x: u32 = 9;
let y = x as u64;
let z: u64 = x.into();
```

`as` 版本还允许进行有损转换：[^3]

```rust
let x: u32 = 9;
let y = x as u16;
```

而这将会被 `from/into` 版本拒绝：

```rust
error[E0277]: the trait bound `u16: From<u32>` is not satisfied
   --> src/main.rs:136:20
    |
136 |     let y: u16 = x.into();
    |                    ^^^^ the trait `From<u32>` is not implemented for `u16`
    |
    = help: the following other types implement trait `From<T>`:
              <u16 as From<NonZeroU16>>
              <u16 as From<bool>>
              <u16 as From<u8>>
    = note: required for `u32` to implement `Into<u16>`
```

为了保持一致性和安全性，您应该**优先使用 `from/into` 转换而不是 `as` 显式类型转换**，除非您理解并需要精确的显式类型转换语义（例如，用于 `C` 语言互操作性）。这个建议可以通过 `Clippy`（[第 29 条]）得到加强，`Clippy` 包含了关于 `as` 转换的几个 `lint`；然而，这些 `lint` 默认是禁用的。

## 隐式类型转换（Coercion）

上一节描述的显式 `as` 类型转换是编译器会默默执行的隐式类型转换的超集：任何隐式类型转换都可以用显式的 `as` 来强制执行，但反之则不成立。特别是，上一节执行的整数类型转换并不是隐式类型转换，因此将始终需要 `as`。

大多数隐式类型转换涉及指针和引用类型，这些转换对程序员来说是有意义且方便的，例如转换以下内容：

- 从可变引用到不可变引用（这样您就可以将 `&mut T` 作为接受 `&T` 的函数的参数）
- 从引用到原始指针（这并不 `unsafe` —— 不安全性发生在您愚蠢地去解引用一个原始指针的时候）
- 从恰好没有捕获任何变量的闭包到裸函数指针（[第 2 条]）
- 从数组到切片
- 从具体项到 `trait` 对象，对于该具体项所实现的 `trait`
- 从一个生命周期到“更短”的生命周期（[第 14 条]）[^4]

只有两种隐式类型转换的行为可能受到用户自定义类型的影响。第一种情况是用户自定义的类型实现了 `Deref` 或 `DerefMut` `trait`。这些 `trait` 表明用户定义的类型正在充当某种智能指针（[第 8 条]），在这种情况下，编译器会将智能指针项的引用隐式转换为智能指针包含的类型项的引用（由其 `Target` 指示）。

用户自定义类型的第二种隐式类型转换发生在具体项转换为 `trait` 对象时。这个操作构建了一个指向项的胖指针；这个指针之所以胖，是因为它既包括了指向项在内存中位置的指针，也包括了指向具体类型的 `trait` 实现的 `vtable` 指针 —— 参见[第 8 条]。

#### 注释

[^1]: 更准确地称为 `trait` 一致性规则。

[^2]: 暂时如此 —— 这可能会在未来的 `Rust` 版本中被 `!` "`never`" 类型所取代。

[^3]: 在 Rust 中允许有损转换可能是个错误，已经有过尝试去除这种行为的讨论。

[^4]: `Rust` 将这些转换称为“子类型化”，但它与面向对象语言中“子类型化”的定义大不相同。

原文[点这里](https://www.lurklurk.org/effective-rust/casts.html)查看

<!-- 参考链接 -->
[第 1 条]: item1-use-types.md
[第 2 条]: item2-use-types-2.md
[第 4 条]: item4-errors.md
[第 6 条]: item6-newtype.md
[第 8 条]: item8-references&pointer.md
[第 10 条]: ../chapter_2/item10-std-traits.md
[第 14 条]: ../chapter_3/item14-lifetimes.md
[第 29 条]: ../chapter_5/item29-listen-to-clippy.md
