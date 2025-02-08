# 第 3 条：优先选择Option和Result转换，而非显式match表达式

[第 1 条] 阐述了枚举（`enum`）的优点，并展示了 `match` 表达式如何强制程序员考虑所有可能性；这个方法探讨了在某些情况下，你应尽量避免使用 `match` 表达式 —— 至少是显式地。

[第 1 条] 还介绍了 Rust 标准库提供的两个无处不在的枚举：
- `Option<T>`，表示一个值（类型为 `T`）可能存在也可能不存在。
- `Result<T, E>`，用于当尝试返回一个值（类型为 `T`）的操作可能失败，并可能返回一个错误（类型为 `E`）。

对于这些特定的枚举，显式使用 `match` 通常会导致代码比实际需要的不够紧凑，而且不符合 Rust 的习惯用法。

第一种不需要使用 `match` 的情况是，当只关心值本身，而值的缺失（以及任何相关的错误）可以被忽略时。

```rust
struct S {
    field: Option<i32>,
}

let s = S { field: Some(42) };
match &s.field {
    Some(i) => println!("field is {}", i),
    None => {}
}
```

对于这种情况，使用 `if let` 表达式可以缩短一行代码，而且更重要的是，它的表达更清晰：

```rust
if let Some(i) = &s.field {
    println!("field is {}", i);
}
```

然而，大多数时候，程序员通常的处理方式是提供相应的 `else` 分支：缺少值（`Option::None`），或者返回一个可能会出现的相关错误（`Result::Err(e)`）。设计能够应对失败路径的软件是困难的，而且其中大部分是本质的复杂性，无论多少语法支持都无法帮助——具体来说，就是决定如果一个操作失败了应该发生什么。

在某些情况下，正确的决定是采取“鸵鸟策略”——把我们的头埋进沙子里，明确地不去处理失败。你不能完全忽略错误分支，因为 Rust 要求代码必须处理 `Error` 枚举的两种变体，但你可以选择将失败视为致命的错误。在失败时执行一个 `panic!` 意味着程序会终止，但可以在成功的假设下来编写其余的代码。通过显式 `match` 来执行此操作会不必要地冗长：

```rust
let result = std::fs::File::open("/etc/passwd");
let f = match result {
    Ok(f) => f,
    Err(_e) => panic!("Failed to open /etc/passwd!"),
};
```

`Option` 和 `Result` 都提供了一对方法来提取它们的内部值并在值不存在时执行 `panic!`，它们分别是 [unwrap](https://doc.rust-lang.org/std/result/enum.Result.html#method.unwrap) 和 [expect](https://doc.rust-lang.org/std/result/enum.Result.html#method.expect) 。后者允许个性化失败时的错误消息，并将错误处理委托给 `.unwrap()` 后缀，但无论哪种情况，生成的代码都更短、更简单：

```rust
let f = std::fs::File::open("/etc/passwd").unwrap();
```

但要明确的是：这些辅助函数仍然会引发 `panic!`，所以选择使用它们与选择直接 `panic!`（[第 18 条]）是一样的。

然而，在许多情况下，正确的错误处理是将决策推迟给其他人。这在编写库时尤其如此，因为库的代码可能会在库作者无法预见的各种不同环境中使用。为了使库更易用，优先使用 `Result` 而不是 `Option` 来表示错误，即使这可能涉及不同错误类型之间的转换（[第 4 条]）。

当然，这提出了一个问题：什么算作错误？在此示例中，无法打开文件肯定是一个错误，并且该错误的详细信息（没有此文件？权限被拒绝？）可以帮助用户决定下一步要做什么。另一方面，由于切片为空而未能检索切片的 [first()](https://doc.rust-lang.org/std/primitive.slice.html#method.first) 元素并不是真正的错误，因此它在标准库中表示为 `Option` 返回类型。在两种可能性之间进行选择需要判断，但如果可能通过错误传达任何有用的信息，则倾向于 `Result`。

`Result` 也有一个 `[#must_use]` 属性，用来引导库用户朝着正确的方向前进 —— 如果使用返回的 `Result` 的代码忽略了它，编译器将生成一个警告：

```rust
warning: unused `Result` that must be used
  --> transform/src/main.rs:32:5
   |
32 |     f.set_len(0); // Truncate the file
   |     ^^^^^^^^^^^^^
   |
   = note: `#[warn(unused_must_use)]` on by default
   = note: this `Result` may be an `Err` variant, which should be handled

```

显式使用 `match` 可以让错误传播，但代价是增加了一些可见的样板代码（让人联想到 `Go 语言`）：

```rust
pub fn find_user(username: &str) -> Result<UserId, std::io::Error> {
    let f = match std::fs::File::open("/etc/passwd") {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    // ...
}
```

减少样板代码的关键是 Rust 的问号运算符 `?`。这个语法糖可以处理匹配 `Err` 分支和返回 `Err(...)` 表达式，只用一个字符就完成了：

```rust
pub fn find_user(username: &str) -> Result<UserId, std::io::Error> {
    let f = std::fs::File::open("/etc/passwd")?;
    // ...
}
```

Rust 新手有时会对此感到困惑：问号运算符在一开始很难被注意到，导致人们怀疑这段代码怎么可能正常工作。然而，即使只有一个字符，类型系统仍然在起作用，确保覆盖了相关类型（[第 1 条]）表达的所有可能性——让程序员可以专注于主线代码路径，不受干扰。

更重要的是，这些明显的方法调用通常没有额外的成本：它们都是标记为 `#[inline]` 的泛型函数，所以生成的代码通常会编译成与手动版本相同的机器代码。

这两个因素结合起来意味着你应该优先使用 `Option` 和 `Result` 转换，而不是显式的 `match` 表达式。

在之前的例子中，错误类型是一致的：内部和外部方法都使用 `std::io::Error` 表达错误。然而，情况往往并非如此；一个函数可能从各种不同的子库中累积错误，每个子库都使用不同的错误类型。

关于错误映射的讨论一般见[第 4 条]；现在，只需知道一个手动映射：

```rust
pub fn find_user(username: &str) -> Result<UserId, String> {
    let f = match std::fs::File::open("/etc/passwd") {
        Ok(f) => f,
        Err(e) => {
            return Err(format!("Failed to open password file: {:?}", e))
        }
    };
    // ...
}
```

可以使用更简洁、更符合 Rust 语法的 `.map_err()` 转换来表达：

```rust
pub fn find_user(username: &str) -> Result<UserId, String> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

更好的是，甚至这可能也不必要 —— 如果外部错误类型可以通过实现标准的 `From` `trait`（[第 5 条]）从内部错误类型创建，那么编译器将自动执行转换，无需调用 `.map_err()`。

这类转换具有更广泛的通用性。问号运算符是一个强大的工具；使用 `Option` 和 `Result` 类型上的转换方法将它们调整到可以顺利处理的形态。

标准库提供了各种各样的转换方法来实现这一点，如下面的地图所示。根据[第 18 条]，可能引发 `panic!` 的方法用红色突出显示。

![转换方法](../images/transform.svg)

（此图的[在线版本]可点击：每个框都会链接到相关文档。）

图中未涵盖的一种常见情况是处理引用。例如，考虑一个可能包含一些数据的结构。

```rust
struct InputData {
    payload: Option<Vec<u8>>,
}
```

这个结构上的一个方法尝试将有效载荷传递给一个加密函数，该函数的签名是 `(&[u8]) -> Vec<u8>`，如果简单地尝试获取一个引用，则会失败：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
impl InputData {
    pub fn encrypted(&self) -> Vec<u8> {
        encrypt(&self.payload.unwrap_or(vec![]))
    }
}
```

```rust
error[E0507]: cannot move out of `self.payload` which is behind a shared reference
  --> transform/src/main.rs:62:22
   |
62 |             encrypt(&self.payload.unwrap_or(vec![]))
   |                      ^^^^^^^^^^^^ move occurs because `self.payload` has type `Option<Vec<u8>>`, which does not implement the `Copy` trait
   |
help: consider borrowing the `Option`'s content
   |
62 |             encrypt(&self.payload.as_ref().unwrap_or(vec![]))
   |                                  +++++++++
```

错误消息准确地描述了使代码工作所需的内容，即 `Option` 上的 `as_ref()` 方法[^1]。这个方法将一个对 `Option` 的引用转换为对引用的 `Option`：

```rust
pub fn encrypted(&self) -> Vec<u8> {
    encrypt(self.payload.as_ref().unwrap_or(&vec![]))
}
```

总结一下：

- 习惯使用 `Option` 和 `Result` 的转换，并且优先使用 `Result` 而不是 `Option`。
- 在转换涉及引用时，根据需要使用 `.as_ref()`。
- 在可能的情况下，优先使用它们而不是显式的 `match` 操作。
- 特别是，使用它们将结果类型转换成可以使用 `?` 运算符的形式。

---

#### 注释

[^1]: 注意，这个方法与 `AsRef` `trait` 是独立的，尽管方法名称相同。

原文[点这里](https://www.lurklurk.org/effective-rust/transform.html)查看

<!-- 参考链接 -->

[第 1 条]: item1-use-types.md
[第 4 条]: item4-errors.md
[第 5 条]: item5-casts.md
[第 18 条]: ../chapter_3/item18-panic.md

[在线版本]: https://tinyurl.com/rust-transform
