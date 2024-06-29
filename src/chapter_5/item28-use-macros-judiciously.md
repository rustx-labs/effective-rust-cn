# 第 28 条：在合适的时候使用宏

> “在一些场景下，我们会很容易来决定应该使用宏（macro）而非函数（function），因为只有宏才能满足我们的需求。” - Paul Graham，“[On Lisp (Prentice Hall)]”

Rust 的宏能够让你实现元编程（metaprogramming）：在项目中使用代码来生成代码。这一特性在需要编写很多确定性、重复性都很强的“样板代码”时会很有用，不借助宏的话我们就只能手动维护这些代码了。

程序员接触 Rust 之前可能已经预先了解了 C/C++ 中通过预处理（preprocessor）来实现的宏，这种方式是在预处理阶段通过文本替换来展开宏定义。而 Rust 的宏则有一些不同，它会在词法分析（parsed tokens of the program）或者在抽象语法树（abstract syntax tree, AST）的基础上实现的宏，而非在文本处理阶段。

这就意味着 Rust 的宏是能够理解代码结构并且规避掉一系列的文本替换方式实现的宏所存在的意外情况。比如说，在接下来的内容中，我们可以看到 Rust 所声明的宏是[干净的] —— 在宏里不会意外引用（或者捕获）宏所嵌入代码中的变量信息。

```c
/* 这段内容较为晦涩。引用一段维基百科上的内容来说明文本替换方式实现的宏所带来的问题。*/
#define INCI(i) { int a=0; ++i; }
int main(void)
{
  int a = 4, b = 8;
  INCI(a);
  INCI(b);
  printf("a is now %d, b is now %d\n", a, b);
  return 0;
}

/* 以上代码中的 INCI 宏期望分别对 a, b 进行加一操作。文本替换后的结果如下所示。*/

int main(void)
{
    int a = 4, b = 8;
    { int a = 0; ++a; }; // 注意这里对 a 进行了重新声明，实际上是对声明的这个 a 进行了自增。
    { int a = 0; ++b; }; 
    printf("a is now %d, b is now %d\n", a, b);
    return 0;
}

/* 最终的结果会输出如下。 
 *
 * a is now 4, b is now 9
 *
 * 这显然是不符合预期的。产生这一结果的原因是由于文本替换过于粗暴，而无法进行实际语意上的理解。
 *
 * 本注释内容为译者添加。
 */

```

一种理解宏的方式是将其视为代码的不同抽象方式。函数也是代码的一种简单抽象：它将同一类型的不同值的不同抽象出来，实现了针对这一类型，而非特定的值，会做的操作及方法。而宏中的生成则是另外一个层面的抽象：宏是对符合同一特性的不同类型进行抽象，使用针对这些不同类型所具备的相同特性，而非特性的类型，进行代码的实现。

宏可以对不同程序中扮演相同角色（类型、标记、表达式等）的代码抽象出来，然后这些程序就可以以同一种方式来使用这些角色。

Rust 提供了两种方式来定义宏：
* 声明式的宏，也被成为“示例宏”，允许将任意的 Rust 程序集成到代码中，基于输入到宏中的参数，使用参数在语法分析树中的角色来进行划分。 -- 待调整
* 程序宏，允许将任意的 Rust 程序集成到代码中，基于源码中的解析语素。这种方式往往用在`derive`宏中。`derive`宏可以基于代码的结构定义来展开代码。 -- 待调整。

## 声明式的宏

尽管这篇文章不是为了重新复述下[声明式宏]的内容，仍然有必要来提醒下需要注意的一些细节。

首先，需要注意的是声明式宏的作用域范围和一般理解的是不同的。如果一个声明式宏在源代码中被声明了，就只有*跟着*宏的这些参数能够被使用：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
fn before() {
    println!("[before] square {} is {}", 2, square!(2));
}

/// Macro that squares its argument.
macro_rules! square {
    { $e:expr } => { $e * $e }
}

fn after() {
    println!("[after] square {} is {}", 2, square!(2));
}
```

```rust
error: cannot find macro `square` in this scope
 --> src/main.rs:4:45
  |
4 |     println!("[before] square {} is {}", 2, square!(2));
  |                                             ^^^^^^
  |
  = help: have you added the `#[macro_use]` on the module/import?

```

`#[macro_export]`特性让宏可以访问更多的数据，但是也存在一些奇怪的事情：一个宏出现在了包的顶层，尽管它并没有在模块中定义。

```rust
mod submod {
    #[macro_export]
    macro_rules! cube {
        { $e:expr } => { $e * $e * $e }
    }
}

mod user {
    pub fn use_macro() {
        // Note: *not* `crate::submod::cube!`
        let cubed = crate::cube!(3);
        println!("cube {} is {}", 3, cubed);
    }
}
```

Rust 的声明式宏是干净的：宏内部展开的代码无法使用所在作用域的局部变量。比如，一个宏其使用时，存在一个 x 的局部变量：

```rust
// Create a macro that assumes the existence of a local `x`.
macro_rules! increment_x {
    {} => { x += 1; };
}
```

这样的用法将会造成编译错误：

```rust
let mut x = 2;
increment_x!();
println!("x = {}", x);
```

```rust
error[E0425]: cannot find value `x` in this scope
   --> src/main.rs:55:13
    |
55  |     {} => { x += 1; };
    |             ^ not found in this scope
...
314 |     increment_x!();
    |     -------------- in this macro invocation
    |
    = note: this error originates in the macro `increment_x`
```

这种“干净”的特性意味着 Rust 的宏比 C 的基于预处理替换的宏要安全很多。然而，仍有一些需要在使用时注意的内容。

第一，尽管一个宏*看起来*很像是函数的声明，它并不是。宏将会在调用的地方进行代码展开，而且可以随着传入参数的不同进行不同形式的展开：

```rust
macro_rules! inc_item {
    { $x:ident } => { $x.contents += 1; }
}
```

这就意味着常规意义上的参数被移动（moved）或者 & （被引用）的情形没有发生：

```rust
let mut x = Item { contents: 42 }; // type is not `Copy`

// Item is *not* moved, despite the (x) syntax,
// but the body of the macro *can* modify `x`.
inc_item!(x);

println!("x is {x:?}");
```

```rust
x is Item { contents: 43 }
```

如果我们还记得宏只是在调用它的地方进行展开的话，上述示例就会变得清楚了 —— 在这个示例中，调用宏的地方只相当于添加了一行增加`x.contents`值的代码。借助[cargo-expand]可以很清晰地看到宏展开后编译器看到的代码：

```rust
let mut x = Item { contents: 42 };
x.contents += 1;
{
    ::std::io::_print(format_args!("x is {0:?}\n", x));
};
```

展开的代码中可以看到直接使用了变量本身，而非其引用。（有意思的是，我们可以看到`println!`的展开中，依赖了`format_args!`宏，这一点会在后面进行讨论。）

所以，宏里的`!`起到了一个警示的作用：展开的代码可能会对参数做一些任性的事情。

展开的代码也可能会包含一些在调用代码中无法访问的控制流，可能包括循环、判断、返回值甚至使用`?`操作符。显然，这里会和[最小惊讶原则]相冲突，所以在使用宏时，应当考虑封装常规的 Rust 语句。（另一方面，如果使用宏的*目的*是实现一些奇怪的控制流，请确保这些控制流在文档中都给用户提供了！）

举例来说，考虑这样一个宏（用来校验 HTTP 状态码）包含了一个`return`语句：

```rust
/// Check that an HTTP status is successful; exit function if not.
macro_rules! check_successful {
    { $e:expr } => {
        if $e.group() != Group::Successful {
            return Err(MyError("HTTP operation failed"));
        }
    }
}
```

用这段宏来校验一些 HTTP 行为的代码可能会以一些很晦涩的控制流来结束：

```rust
let rc = perform_http_operation();
check_successful!(rc); // may silently exit the function

// ...
```

另一种可以实现上述功能的宏是产生一个`Result`：

```rust
/// Convert an HTTP status into a `Result<(), MyError>` indicating success.
macro_rules! check_success {
    { $e:expr } => {
        match $e.group() {
            Group::Successful => Ok(()),
            _ => Err(MyError("HTTP operation failed")),
        }
    }
}
```

而这样依赖，代码就很好理解了：

```rust
let rc = perform_http_operation();
check_success!(rc)?; // error flow is visible via `?`

// ...
```

对于声明式宏来说，第二件需要注意的事情是和 C 的预编译宏同样的问题：如果宏的参数是一个存在副作用的表达式，当心在宏里多次使用的情况。比如我们在早先定义的`square!`宏输入了较为随意的表达式来作为参数，然后使用两次，这将会造成奇怪的结果：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
let mut x = 1;
let y = square!({
    x += 1;
    x
});
println!("x = {x}, y = {y}");
// output: x = 3, y = 6
```

假设这种行为并非有意的，一种修复的方法是尝试仅执行给定的表达式一次，然后将结果赋值给一个本地的变量：

```rust
macro_rules! square_once {
    { $e:expr } => {
        {
            let x = $e;
            x*x // Note: there's a detail here to be explained later...
        }
    }
}
// output now: x = 2, y = 4
```

另一种可选的方案是不允许将表达式作为宏的输入。如果将[expr]替换为`indent`，那么这个宏就仅会接受标志符作为入参，而使用类似任意的表达式将不再能编译通过。

## 格式化数值（？）

声明式宏的一种常见的使用模式包含了汇聚多个参数的一个信息（待调整）。比如


---

### 注释


原文[点这里]查看。

<!-- 参考链接 -->

[On Lisp (Prentice Hall)]: https://www.paulgraham.com/onlisp.html
[干净的]: https://en.wikipedia.org/wiki/Hygienic_macro
[声明式宏]: https://doc.rust-lang.org/reference/macros-by-example.html
[cargo-expand]: https://github.com/dtolnay/cargo-expand
