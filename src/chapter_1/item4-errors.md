# 第 4 条：优先使用惯用的错误类型

[第 3 条]描述了如何使用标准库为 `Option` 和 `Result` 类型提供的转换，以允许使用 `?` 运算符简洁、惯用地处理结果类型。但它没有讨论如何最好地处理作为 `Result<T, E>` 第二个类型参数出现的各种不同的错误类型 `E`；这就是本章节的内容。

只有当有多种不同的错误类型时，这才有相关性。如果函数遇到的所有不同错误已经是同一类型，它可以只返回该类型。当有不同类型的错误时，需要做出一个决定，即是否保留子错误类型信息。

## Error Trait

了解标准 `trait`（[第 10 条]）总是一个好主意，这里相关的 `trait` 是 `std::error::Error`。`Result` 的 `E` 类型参数不必是实现 `Error` 的类型，但这是一个常见的约定，它允许包装器表达适当的 `trait` 约束 —— 因此，最好为您的错误类型实现 `Error`。

首先要注意的是，对于错误类型，唯一硬性要求是 `trait` 约束：实现 `Error` 的任何类型也必须实现以下 `trait`：

- `Display` `trait`，意味着可以使用 `{}` 进行格式化

- `Debug` `trait`，意味着可以使用 `{:?}` 进行格式化

换句话说，应该能够将错误类型显示给用户和程序员。

`trait` 中唯一的方法是 `source()`，[^1] 它允许错误类型公开一个内部的、嵌套的错误。此方法是可选的 —— 它带有一个返回 `None` 的默认实现（[第 13 条]），表示内部错误信息不可用。

最后要注意的一点是：如果您正在为 `no_std` 环境（[第 33 条]）编写代码，可能无法实现 `Error` —— `Error` `trait` 目前在 `std` 中实现，而不是 `core`，因此不可用。[^2]

## 最小错误（Minimal Errors）

如果不需要嵌套错误信息，那么错误类型的实现不必比 `String` 复杂多少 —— “字符串类型”的变量就能满足要求的情况并不常见。但它需要的确实比 `String` 多一点；虽然可以使用 `String` 作为 `E` 类型参数：

```rust
pub fn find_user(username: &str) -> Result<UserId, String> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

`String` 类型并没有实现 `Error`，虽然我们希望是这样，以便代码的其他部分可以处理 `Errors`。为 `String` 实现 `Error` 是不可能的，因为 `trait` 和类型都不属于我们（所谓的孤儿规则）：

```rust
impl std::error::Error for String {}
```

```rust
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
  --> src/main.rs:18:5
   |
18 |     impl std::error::Error for String {}
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^------
   |     |                          |
   |     |                          `String` is not defined in the current crate
   |     impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

[类型别名]也无济于事，因为它并没有创建一个新的类型，所以也不会改变错误信息：

```rust
pub type MyError = String;

impl std::error::Error for MyError {}
```

```rust
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
  --> src/main.rs:41:5
   |
41 |     impl std::error::Error for MyError {}
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^-------
   |     |                          |
   |     |                          `String` is not defined in the current crate
   |     impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

像往常一样，编译器错误消息为解决问题提供了一个线索。定义一个包装 `String` 类型的元组结构体（"newtype 模式"，[第 6 条]）允许实现 `Error` `trait`，前提是也实现了 `Debug` 和 `Display`：

```rust
#[derive(Debug)]
pub struct MyError(String);

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for MyError {}

pub fn find_user(username: &str) -> Result<UserId, MyError> {
    let f = std::fs::File::open("/etc/passwd").map_err(|e| {
        MyError(format!("Failed to open password file: {:?}", e))
    })?;
    // ...
}
```

为了方便起见，实现 `From<String>` `trait` 可能是有意义的，以便可以轻松地将字符串值转换为 `MyError` 实例（[第 5 条]）：

```rust
impl From<String> for MyError {
    fn from(msg: String) -> Self {
        Self(msg)
    }
}
```

当编译器遇到问号运算符（`?`）时，它会自动应用任何需要的 `From` `trait` 实现，以便达到目标错误返回类型。这样代码就更简洁了：

```rust
pub fn find_user(username: &str) -> Result<UserId, MyError> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

这里的错误路径涵盖了以下步骤：
- `File::open` 返回一个类型为 `std::io::Error` 的错误。
- `format!` 使用 `std::io::Error` 的 `Debug` 实现将其转换为 `String`。
- `?` 使编译器寻找并使用一个 `From` 实现，该实现可以将它从 `String` 转换为 `MyError`。

## 嵌套错误

另一种情况是，嵌套错误的内容重要到足以需要被保留并供调用者使用。

考虑一个库函数，它尝试返回文件的第一行作为字符串，只要这一行不是太长。稍微思考一下就会发现（至少）三种可能发生的不同类型的失败：
- 文件可能不存在或者无法读取。
- 文件可能包含不是有效 `UTF-8` 的数据，因此无法转换为 `String`。
- 文件可能有一个过长的一行。

根据[第 1 条]，您可以使用类型系统来表达并包含所有这些可能性作为一个`枚举`：

```rust
#[derive(Debug)]
pub enum MyError {
    Io(std::io::Error),
    Utf8(std::string::FromUtf8Error),
    General(String),
}
```

这个`枚举`定义包括了 `derive(Debug)`，但为了满足 `Error` `trait`，还需要一个 `Display` 的实现：

```rust
impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            MyError::Io(e) => write!(f, "IO error: {}", e),
            MyError::Utf8(e) => write!(f, "UTF-8 error: {}", e),
            MyError::General(s) => write!(f, "General error: {}", s),
        }
    }
}
```

为了方便访问嵌套错误，覆盖默认的 `source()` 实现也是很有意义的：

```rust
use std::error::Error;

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            MyError::Io(e) => Some(e),
            MyError::Utf8(e) => Some(e),
            MyError::General(_) => None,
        }
    }
}
```

使用`枚举`允许错误处理保持简洁，同时仍然保留不同错误类别的所有类型信息：

```rust
use std::io::BufRead; // for `.read_until()`

/// Maximum supported line length.
const MAX_LEN: usize = 1024;

/// Return the first line of the given file.
pub fn first_line(filename: &str) -> Result<String, MyError> {
    let file = std::fs::File::open(filename).map_err(MyError::Io)?;
    let mut reader = std::io::BufReader::new(file);

    // (A real implementation could just use `reader.read_line()`)
    let mut buf = vec![];
    let len = reader.read_until(b'\n', &mut buf).map_err(MyError::Io)?;
    let result = String::from_utf8(buf).map_err(MyError::Utf8)?;
    if result.len() > MAX_LEN {
        return Err(MyError::General(format!("Line too long: {}", len)));
    }
    Ok(result)
}
```

为所有子错误类型实现 `From` `trait` 也是一个好主意（[第 5 条]）：

```rust
impl From<std::io::Error> for MyError {
    fn from(e: std::io::Error) -> Self {
        Self::Io(e)
    }
}
impl From<std::string::FromUtf8Error> for MyError {
    fn from(e: std::string::FromUtf8Error) -> Self {
        Self::Utf8(e)
    }
}
```

这防止了库用户自己受到孤儿规则的影响：他们不允许在 `MyError` 上实现 `From`，因为 `trait` 和结构体对他们来说是外部的。

更好的是，实现 `From` 能让代码更加简洁，因为[问号运算符]将自动执行任何必要的 `From` 转换，从而消除了 `.map_err()` 的需求：

```rust
use std::io::BufRead; // for `.read_until()`

/// Maximum supported line length.
pub const MAX_LEN: usize = 1024;

/// Return the first line of the given file.
pub fn first_line(filename: &str) -> Result<String, MyError> {
    let file = std::fs::File::open(filename)?; // `From<std::io::Error>`
    let mut reader = std::io::BufReader::new(file);
    let mut buf = vec![];
    let len = reader.read_until(b'\n', &mut buf)?; // `From<std::io::Error>`
    let result = String::from_utf8(buf)?; // `From<string::FromUtf8Error>`
    if result.len() > MAX_LEN {
        return Err(MyError::General(format!("Line too long: {}", len)));
    }
    Ok(result)
}
```

编写一个完整的错误类型可能涉及相当多的样板代码，这使得它成为通过派生宏（[第 28 条]）自动化的好候选。然而，没有必要自己编写这样的宏：**考虑使用 `David Tolnay` 提供的 [thiserror] crate**，它提供了一个高质量、广泛使用的宏实现。`thiserror` 生成的代码也小心翼翼地避免在生成的 `API` 中使任何 `thiserror` 类型可见，这意味着与[第 24 条]相关的问题不适用。

## `trait` 对象（`Trait Objects`）

第一种处理嵌套错误的方法丢弃了所有子错误的细节，只保留了某些字符串输出（`format!("{:?}", err)`）。

第二种方法保留了所有可能子错误的全类型信息，但需要完整枚举所有可能的子错误类型。

这就引出了一个问题，这两种方法之间有没有中间地带，可以在不需要手动包含每个可能的错误类型的情况下保留子错误信息？

将子错误信息编码为 [`trait` 对象][trait object]避免了为每种可能性都定义一个`枚举`变体的需要，但擦除了特定基础错误类型的细节。接收此类对象的调用者将能够访问 `Error` `trait` 及其 `trait` 约束的方法 —— `source()`、`Display::fmt()` 和 `Debug::fmt()`，依次类推 —— 但不会知道子错误原始的静态类型：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
#[derive(Debug)]
pub enum WrappedError {
    Wrapped(Box<dyn Error>),
    General(String),
}

impl std::fmt::Display for WrappedError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Wrapped(e) => write!(f, "Inner error: {}", e),
            Self::General(s) => write!(f, "{}", s),
        }
    }
}
```

结果是这是可能的，但出奇地微妙。部分困难来自于 `trait` 对象的对象安全约束（[第 12 条]），但 `Rust` 的一致性规则也发挥作用，它们（大致）指出一种类型对于一个 `trait` 最多只能有一个实现。

一个假设的 `WrappedError` 类型可能会天真地预期同时实现以下两个 `trait`：
- `Error` `trait`，因为它本身就是一个错误。
- `From<Error>` `trait`，以便子错误可以被轻松包装。

这意味着可以从一个内部的 `WrappedError` 创建一个 `WrappedError`，因为 `WrappedError` 实现了 `Error`，但是这与 `From` 的通用自反实现（Blanket Reflective Implementation）冲突了：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
impl Error for WrappedError {}

impl<E: 'static + Error> From<E> for WrappedError {
    fn from(e: E) -> Self {
        Self::Wrapped(Box::new(e))
    }
}
```

```rust
error[E0119]: conflicting implementations of trait `From<WrappedError>` for
              type `WrappedError`
   --> src/main.rs:279:5
    |
279 |     impl<E: 'static + Error> From<E> for WrappedError {
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
    = note: conflicting implementation in crate `core`:
            - impl<T> From<T> for T;

```

`David Tolnay` 的 [anyhow] 是一个已经解决了这些问题（通过[使用 Box 来添加一个额外的间接层](https://github.com/dtolnay/anyhow/issues/63#issuecomment-582079114)）并增加了其他有用功能（如堆栈跟踪）的 `crate`。因此，它迅速成为了错误处理的推荐标准 —— 在这里也是一样：**考虑在应用程序中使用 `anyhow` crate 进行错误处理。**

## 库与应用程序

上一节的最后建议中包含了这样的限定：“……用于**应用程序**中的错误处理”。这是因为在库中编写的代码和上层应用程序的代码之间通常有一个区别。[^3]

为库编写的代码无法预测代码使用的环境，因此最好发出具体、详细的错误信息，让调用者去弄清楚如何使用这些信息。这倾向于前面描述的枚举风格的嵌套错误（并且在库的公共 `API` 中避免了依赖 `anyhow`，参见[第 24 条]）。

然而，应用程序代码通常需要更多地关注如何向用户呈现错误。它还可能不得不应对其依赖关系图中所有库发出的所有不同错误类型（[第 25 条]）。因此，一个更动态的错误类型（如 `anyhow::Error`）使得错误处理在应用程序中更简单、更一致。

## 需要记住的事情

- 标准 `Error` `trait` 对您的要求很少，因此最好为您的错误类型实现它。
- 在处理异构的基础错误类型时，决定是否需要保留这些类型。
    - 如果不是，考虑在应用程序代码中使用 `anyhow` 来包装子错误。
    - 如果是，将它们编码在一个`枚举`中并提供转换。考虑使用 `thiserror` 来帮助做到这一点。
- 考虑在应用程序代码中使用 `anyhow crate` 进行便捷、惯用的错误处理。
- 决定权在您手中，但无论您决定什么，都要在类型系统中编码它（[第 1 条]）。


#### 注释

[^1]: 或者至少是唯一一个非废弃的、稳定的方法。

[^2]: 在撰写本文时，`Error` 已经[被移动到 `core`](https://github.com/rust-lang/rust/issues/103765)，但在稳定版的 `Rust` 中尚不可用。

[^3]: 本节灵感来源于 `Nick Groenen` 的文章[《Rust: 2020年在错误处理和结构化》](https://nick.groenen.me/posts/rust-error-handling/)。

原文[点这里](https://www.lurklurk.org/effective-rust/errors.html)查看

<!-- 参考链接 -->

[第 1 条]: item1-use-types.md
[第 3 条]: item3-transform.md
[第 5 条]: item5-casts.md
[第 6 条]: item6-newtype.md
[第 10 条]: ../chapter_2/item10-std-traits.md
[第 12 条]: ../chapter_2/item12-generics&trait-objects.md
[第 13 条]: ../chapter_2/item13-use-default-impl.md
[第 24 条]: ../chapter_4/item24-re-export.md
[第 25 条]: ../chapter_4/item25-dep-graph.md
[第 28 条]: ../chapter_5/item28-use-macros-judiciously.md
[第 33 条]: ../chapter_6/item33-no-std.md

[anyhow]: https://docs.rs/anyhow
[thiserror]: https://docs.rs/thiserror
[类型别名]: https://doc.rust-lang.org/reference/items/type-aliases.html
[问号运算符]: https://doc.rust-lang.org/reference/expressions/operator-expr.html#the-question-mark-operator
[trait object]: https://doc.rust-lang.org/reference/types/trait-object.html
