# 第 28 条：在合适的时候使用宏

> “在一些场景下，我们会很容易来决定应该使用宏（macro）而非函数（function），因为只有宏才能满足我们的需求。” - Paul Graham，“[On Lisp (Prentice Hall)]”

Rust 的宏能够让你实现元编程（metaprogramming）：在项目中使用代码来生成代码。这一特性在需要编写很多确定性、重复性都很强的“样板代码”时会很有用，不借助宏的话我们就只能手动维护这些代码了。

程序员接触 Rust 之前可能已经预先了解了 C/C++ 中通过预处理器（preprocessor）提供的宏，这种方式是通过对输入文本的符号进行文本替换来实现的。而 Rust 的宏则有一些不同，它是在已解析的程序符号（parsed tokens of the program）或者在抽象语法树（abstract syntax tree, AST）的基础上实现的宏，而非在文本处理阶段。

这就意味着 Rust 的宏是能够理解代码结构并且规避掉一系列的文本替换方式实现的宏所存在的意外情况。比如说，在接下来的内容中，我们可以看到 Rust 所声明的宏是[卫生的] —— 在宏里不会意外引用（或者捕获）宏的上下文代码中的局部变量。

> 译者注：上面这段内容较为晦涩。引用一段维基百科上的内容来说明文本替换方式实现的宏所带来的问题。

```c
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
 * 这显然是不符合预期的。产生这一结果的原因是由于文本替换过于粗暴，而无法进行实际语义上的理解。
 *
 * 本注释内容为译者添加。
 */
```

一种理解宏的方法是将其视为代码的不同抽象方式。函数是代码的一种简单抽象：它将同一类型的不同值的差异抽象出来，实现代码可以使用这一类型的任意特性及方法，无论当前正在操作的值是多少。而泛型是另外一个层面的抽象：它是对符合同一 trait 约束的不同类型的差异进行抽象，实现代码可以使用该 trait 约束提供的任意方法，无论当前正在操作的类型是什么。

宏可以对不同程序中扮演相同角色（类型、标识符、表达式等）的程序片段进行抽象，实现代码可以在同样的角色里使用抽象出来的程序片段。

Rust 提供了两种方式来定义宏：
* 声明宏，也被成为“示例宏”。声明宏允许基于宏的输入参数（根据在抽象语法树里的角色进行分类）将任意的 Rust 代码插入到程序中。
* 过程宏，同样可以将任意的 Rust 代码插入到程序中，不过是基于源码中的解析符号。这最常被 `derive` 宏所使用。`derive` 宏可以基于数据结构的定义来生成代码。

## 声明宏

虽然本条款没必要重复[声明宏]的文档内容，但还是有必要来提醒下声明宏中需要关注的内容。

首先，需要注意的是声明宏的作用域范围和其他的 Rust 语言项是不同的。如果一个声明宏在源代码中被定义了，只有*跟在宏后面*的代码能够使用它：

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

```shell
error: cannot find macro `square` in this scope
 --> src/main.rs:4:45
  |
4 |     println!("[before] square {} is {}", 2, square!(2));
  |                                             ^^^^^^
  |
  = help: have you added the `#[macro_use]` on the module/import?

```

`#[macro_export]` 特性让宏可以在更大的范围里可见，但是也有一个奇怪的地方：尽管宏是在模块中定义的，但它看上去出现在了 Crate 的最顶层：

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

Rust 的声明宏是卫生的（hygienic）：宏内部展开的代码无法使用所在作用域的局部变量。比如，宏内部使用了局部变量 `x` 时：

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

```shell
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

这种“卫生的”特性意味着 Rust 的宏比 C 的基于预处理替换的宏要安全很多。然而，仍有一些需要在使用时注意的内容。

第一，尽管一个宏*看起来*很像是函数的声明，它并不是。宏将会在调用的地方进行代码展开，而且展开的代码可以对其参数进行操作：

```rust
macro_rules! inc_item {
    { $x:ident } => { $x.contents += 1; }
}
```

这就意味着常规意义上的参数被移动（moved）或者被 `&` 引用的情形没有发生：

```rust
let mut x = Item { contents: 42 }; // type is not `Copy`

// Item is *not* moved, despite the (x) syntax,
// but the body of the macro *can* modify `x`.
inc_item!(x);

println!("x is {x:?}");
```

```shell
x is Item { contents: 43 }
```

如果我们还记得宏只是在调用它的地方进行展开的话，上述示例就会变得清楚了 —— 在这个示例中，调用宏的地方只相当于添加了一行增加 `x.contents` 值的代码。借助 [cargo-expand] 工具可以很清晰地看到编译器将宏进行展开后的代码：

```rust
let mut x = Item { contents: 42 };
x.contents += 1;
{
    ::std::io::_print(format_args!("x is {0:?}\n", x));
};
```

展开的代码中可以看到直接使用了变量本身，而非其引用。（一个有意思的事情是，我们可以看到 `println!` 的展开中依赖了 `format_args!` 宏。）[^1]

所以，宏里的 `!` 起到了一个警示的作用：展开的代码可能会对参数做一些任性的事情。

展开的代码也可能会包含一些在调用代码中无法访问的控制流，可能包括循环、判断、返回值甚至 `?` 操作符的使用。显然，这里会和[最小惊讶原则]相冲突，所以只要有可能并且是合适的，应当尽量让宏的行为与常规 Rust 代码的行为对齐。（另一方面，如果使用宏的*目的*是实现一些奇怪的控制流，请确保这些奇怪的控制流都清晰地文档化了！）

举例来说，考虑一个包含了 `return` 语句的宏（用来校验 HTTP 状态码）：

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

用这段宏来校验一些 HTTP 行为的代码可能会以很隐晦的控制流被结束：

```rust
let rc = perform_http_operation();
check_successful!(rc); // may silently exit the function

// ...
```

另一种可以实现上述功能的宏是产生一个 `Result`：

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

使用这个宏的代码就很好理解了：

```rust
let rc = perform_http_operation();
check_success!(rc)?; // error flow is visible via `?`

// ...
```

对于声明宏来说，第二件需要注意的事情是和 C 的预编译宏同样的问题：如果宏的参数是一个存在副作用的表达式，要注意参数在宏里被多次使用的情况。我们在早先定义的 `square!` 宏可以接受任意的表达式来作为参数，并且使用了两次该参数，这就有可能造成奇怪的结果：

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

假设这种行为并非有意的，一种修复的方法是尝试仅执行给定的表达式一次，并将结果赋值给一个局部的变量：

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

另一种可选的方案是不允许将表达式作为宏的输入。如果将 [`expr`] 替换为 `ident`，那么这个宏就仅会接受标志符作为入参，而使用任意的表达式将不再能编译通过。

### 格式化参数

声明宏的一种常见的使用模式将当前代码状态里的多个值汇聚成一个消息。比如，标准库中的 `format!` 用来拼接一个字符串，`println!` 用来输出到标准输出，`eprintln!` 用来输出到标准错误输出。[fmt 文档]中阐述了 `format!` 的语法，和 `C` 中的 `printf` 使用大致是等同的。当然，Rust 中的 `format!` 参数是类型安全并且会在编译时进行检查的，并且 `format!` 宏在实现时使用了[第 10 条]描述的 `Display` 以及 `Debug` trait 来格式化个体的值。[^2]

你可以（也应该）在项目中的宏中使用相同的格式化语法。比如，`log` crate 中的 `logging` 宏就使用了和 `format!` 相同的语法。要做到这个，*应使用 `format_args!` 来实现参数的格式化*而不是重复造轮子。

```rust
/// Log an error including code location, with `format!`-like arguments.
/// Real code would probably use the `log` crate.
macro_rules! my_log {
    { $($arg:tt)+ } => {
        eprintln!("{}:{}: {}", file!(), line!(), format_args!($($arg)+));
    }
}
```

```rust
let x = 10u8;
// Format specifiers:
// - `x` says print as hex
// - `#` says prefix with '0x'
// - `04` says add leading zeroes so width is at least 4
//   (this includes the '0x' prefix).
my_log!("x = {:#04x}", x);
```

```shell
src/main.rs:331: x = 0x0a
```

## 过程宏

Rust 也支持了*过程宏*，也被称为 `proc macros`。和声明宏类似，[过程宏]能够任意的 Rust 代码插入到程序的源代码中。不同的是，过程宏的输入不再仅限制在特定的传入参数。过程宏可以访问一些源代码中的解析符号（parsed tokens）。这就让过程宏一定程度上具备了类似动态语言（比如 Lisp）的表达能力 —— 但是仍然会在编译时进行检查。这也帮助缓解了 Rust 中反射的局限，如在[第 19 条]中所讨论过的。

过程宏需要和其被使用的代码定义在不同的 crate 中（crate 类型需要被声明为 `proc_macro`），并且 crate 往往需要引入对 [proc-macro]（官方工具链中提供）或者 [proc-macro2]（由 David Tolnay 提供）的依赖，这两个依赖支持宏对输入的符号进行操作。

实际上，有三种不同的过程宏：
* 类函数宏（Function-like macros）：通过传入的参数调用。
* 属性宏（Attribute macros）：附加到程序里的一些语法片段中。
* 派生宏（Derive macros）：附加到数据结构的定义中。

### 类函数宏

类函数宏会通过一个传递的参数来被调用，宏的实现中可以访问参数的解析符号（parsed tokens），并且返回任意的符号。注意在先前的表述中，我们使用的是单数的参数 —— 即使函数式的宏调用的时候看起来传入了很多参数：

```rust
my_func_macro!(15, x + y, f32::consts::PI);
```

但是宏本身只接收到了一个解析后的符号流。一个将符号流输出（在编译时）的宏实现示例如下：

```rust
use proc_macro::TokenStream;

// Function-like macro that just prints (at compile time) its input stream.
#[proc_macro]
pub fn my_func_macro(args: TokenStream) -> TokenStream {
    println!("Input TokenStream is:");
    for tt in args {
        println!("  {tt:?}");
    }
    // Return an empty token stream to replace the macro invocation with.
    TokenStream::new()
}
```

其运行结果如下：

```shell
Input TokenStream is:
  Literal { kind: Integer, symbol: "15", suffix: None,
            span: #0 bytes(10976..10978) }
  Punct { ch: ',', spacing: Alone, span: #0 bytes(10978..10979) }
  Ident { ident: "x", span: #0 bytes(10980..10981) }
  Punct { ch: '+', spacing: Alone, span: #0 bytes(10982..10983) }
  Ident { ident: "y", span: #0 bytes(10984..10985) }
  Punct { ch: ',', spacing: Alone, span: #0 bytes(10985..10986) }
  Ident { ident: "f32", span: #0 bytes(10987..10990) }
  Punct { ch: ':', spacing: Joint, span: #0 bytes(10990..10991) }
  Punct { ch: ':', spacing: Alone, span: #0 bytes(10991..10992) }
  Ident { ident: "consts", span: #0 bytes(10992..10998) }
  Punct { ch: ':', spacing: Joint, span: #0 bytes(10998..10999) }
  Punct { ch: ':', spacing: Alone, span: #0 bytes(10999..11000) }
  Ident { ident: "PI", span: #0 bytes(11000..11002) }
```

输入流的低层级本质意味着宏的实现需要能够解析传入的参数。例如，通过查找用来表示分隔参数的逗号的 `TokenTree::Punct` 符号来将输入给宏的不同的参数区分开来。David Tolnay开发的 [syn 包]提供了一个解析的库来辅助这些事情，如[派生宏](#派生宏)章节所述。

正因为这些解析的工作，使用声明宏往往比类函数的过程宏要简单，因为在声明宏里对宏的输入的预期结构可以用匹配的模式来表示。

这种需要手动处理繁杂的另一面是类函数的宏可以更加灵活的接受那些无法像一般 Rust 代码那样被解析的输入。这种特性并非是经常需要的，所以类函数宏相对较少出现。

### 属性宏

属性宏是通过将其放置在程序的一些片段前调用的，而这些片段的解析符号会被传入到宏的内部进行处理。属性宏也可以将任意的结果作为返回值，但是返回值一般是对输出进行转换处理后的结果。

比如，属性宏可以用来对函数体进行封装：

```rust
#[log_invocation]
fn add_three(x: u32) -> u32 {
    x + 3
}
```

之后，在调用这个被封装的函数时，就会有日志输出：

```rust
let x = 2;
let y = add_three(x);
println!("add_three({x}) = {y}");
```

```shell
log: calling function 'add_three'
log: called function 'add_three' => 5
add_three(2) = 5
```

这个宏的实现体量比较大，这里并不打算将其细节附上。在实现时，宏需要检查输入符号的结构并构建出新的输出符号。当然，这一过程仍然可以使用 `syn` 包来辅助实现。

### 派生宏

最后一种过程宏是派生宏。派生宏可以为其修饰的数据（`struct`、`enum` 或者 `union`）自动地生成代码。这一点和属性宏有些像，但是有一些派生宏的特点需要注意下。

首先，派生宏会附加到输入的符号中，而非将其替换。这就意味着原始的数据结构的定义会被保留，而派生宏将在原始数据结构的基础上附加代码。

其次，派生宏可以用来声明一些辅助属性。当数据需要用作一些特殊的处理时，可以使用派生宏来标注。比如，[serde 库]的 [Deserialize] 派生宏有一个辅助属性 `serde` 可以提供元信息来引导反序列化的过程：

```rust
fn generate_value() -> String {
    "unknown".to_string()
}

#[derive(Debug, Deserialize)]
struct MyData {
    // If `value` is missing when deserializing, invoke
    // `generate_value()` to populate the field instead.
    #[serde(default = "generate_value")]
    value: String,
}
```

关于派生宏的最后一点是，[syn 包]可以完成将输入符号解析到相应的语法树结点的工作。[syn::parse\_macro\_input!] 宏可以将符号转换成 [syn::DeriveInput] 数据结构，该数据结构描述了被修饰对象的内容，并且 `DeriveInput` 操作起来远比原始的符号流要好处理。

实际上，派生宏是所有过程宏中最常使用的宏 —— 这种逐字段（对于结构体）或逐变体（对于枚举）操作的能力能够让程序员轻易地实现许多的功能 —— 比如，仅通过添加一行类似 `#[derive(Debug, Clone, PartialEq, Eq)]` 的代码，即可实现相应的功能。

由于派生宏的代码插入是自动实现的，这也意味着这些插入的代码可以和结构体的实现保持一致。比如，如果你向 `struct` 中插入了一个新的字段，如果采用手动实现 `Debug` trait 的话，你就需要在插入后对结构体进行更新以使其满足 trait 的需求。而对于自动插入代码的派生宏来说，你并不需要做任何调整（当然了，如果插入的字段不满足派生宏的实现要求，编译时会报错）。

## 什么时候使用宏

使用宏的首要原因当然是避免重复的代码 —— 尤其是那些需要人工确保和其他代码关联正确性的重复代码。从这一点来说，使用宏仅是编程活动中日常进行的封装抽象操作的一种扩展：
* 如果需要重复一段处理同一类型的不同值的代码，将代码封装为一个通用的函数，并在所有重复的地方调用这个函数。
* 如果需要重复一段处理不同类型的代码，将代码封装成一个有 trait 约束的泛型，并在所有重复的地方使用这个泛型。
* 如果需要在多个地方重复同一段结构的代码，将代码封装成一个宏，并且在所有重复的地方使用这个宏。

比如，可以使用宏来避免重复处理不同枚举变体的代码：

```rust
enum Multi {
    Byte(u8),
    Int(i32),
    Str(String),
}

/// Extract copies of all the values of a specific enum variant.
#[macro_export]
macro_rules! values_of_type {
    { $values:expr, $variant:ident } => {
        {
            let mut result = Vec::new();
            for val in $values {
                if let Multi::$variant(v) = val {
                    result.push(v.clone());
                }
            }
            result
        }
    }
}

fn main() {
    let values = vec![
        Multi::Byte(1),
        Multi::Int(1000),
        Multi::Str("a string".to_string()),
        Multi::Byte(2),
    ];

    let ints = values_of_type!(&values, Int);
    println!("Integer values: {ints:?}");

    let bytes = values_of_type!(&values, Byte);
    println!("Byte values: {bytes:?}");

    // Output:
    //   Integer values: [1000]
    //   Byte values: [1, 2]
}
```

另一个宏的使用场景是用来避免将一个数据值集合的信息分散到代码的不同区域。

比如，假设一个数据结构封装了 HTTP 的状态码，通过宏可以帮助将这些相关的信息保持在一起：

```rust
// http.rs module

#[derive(Debug, PartialEq, Eq, Clone, Copy)]
pub enum Group {
    Informational, // 1xx
    Successful,    // 2xx
    Redirection,   // 3xx
    ClientError,   // 4xx
    ServerError,   // 5xx
}

// Information about HTTP response codes.
http_codes! {
    Continue           => (100, Informational, "Continue"),
    SwitchingProtocols => (101, Informational, "Switching Protocols"),
    // ...
    Ok                 => (200, Successful, "Ok"),
    Created            => (201, Successful, "Created"),
    // ...
}
```

通过使用宏，可以将每个 HTTP 状态码的所有相关联的信息 —— 数值、分组以及描述信息 —— 都聚集起来，看起来就像是使用一种领域特定语言（domain-specific language, DSL）来保存数据一样。

宏的定义描述了要生成的代码，每一行类似 `$( ... )+` 的代码都会被扩展成多行的代码，多行代码中的每一行都对应宏的一个输入参数：

```rust
macro_rules! http_codes {
    { $( $name:ident => ($val:literal, $group:ident, $text:literal), )+ } => {
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
        #[repr(i32)]
        enum Status {
            $( $name = $val, )+
        }
        impl Status {
            fn group(&self) -> Group {
                match self {
                    $( Self::$name => Group::$group, )+
                }
            }
            fn text(&self) -> &'static str {
                match self {
                    $( Self::$name => $text, )+
                }
            }
        }
        impl core::convert::TryFrom<i32> for Status {
            type Error = ();
            fn try_from(v: i32) -> Result<Self, Self::Error> {
                match v {
                    $( $val => Ok(Self::$name), )+
                    _ => Err(())
                }
            }
        }
    }
}
```

这样处理后，宏就可以依据输入的参数来派生如下的代码：
* 一个 `enum` 定义用来保存所有的数值。
* 一个 `group()` 方法用来返回一个 HTTP 状态码的分组归属。
* 一个 `text()` 方法用来将状态码映射到对应的文字描述中。
* 一个 `TryFrom<i32>` 的 trait 实现用来将数值转换成状态枚举值。

如果需要新增一个状态码，只需要添加这样的一行代码：

```rust
ImATeapot => (418, ClientError, "I'm a teapot"),
```

如果不使用宏的话，就需要对四部分代码分别进行手工的更新。编译器可能会有一些提示信息（`match` 表达式需要覆盖所有的场景），但也不是所有的地方 —— `TryFrom<i32` 就很容易被遗忘。

由于宏可以在调用的地方对代码进行展开，所以它们也可以用来自动生成一些诊断信息 —— 尤其是，在使用了标准库中的 [file!()] 以及 [line!()] 宏之后，可以生成代码的位置信息：

```rust
macro_rules! log_failure {
    { $e:expr } => {
        {
            let result = $e;
            if let Err(err) = &result {
                eprintln!("{}:{}: operation '{}' failed: {:?}",
                          file!(),
                          line!(),
                          stringify!($e),
                          err);
            }
            result
        }
    }
}
```

当出现报错时，日志文件中就会自动地包含报错的内容和位置等细节信息：

```rust
use std::convert::TryInto;

let x: Result<u8, _> = log_failure!(512.try_into()); // too big for `u8`
let y = log_failure!(std::str::from_utf8(b"\xc3\x28")); // invalid UTF-8
```

## 宏的缺点

使用宏的最大缺点是引入之后代码的可读性及可维护性。之前在声明宏的示例中介绍了宏允许我们创建一个领域特定语言来简明地描述代码及数据的关键特性。但是，这也意味着任何阅读这段代码的人除了理解 Rust 之外，还需要理解这个领域特定语言以及它在宏定义里的实现。比如，在 `http_codes!` 宏的示例中使用了一个名为 `Status` 的 Rust 枚举，但是它在进行宏调用的领域特定语言中并不可见。

这种使用宏而引入的不可知性还有更大的影响范围：很多分析 Rust 或者和 Rust 交互的工具无法理解这样的代码，因为它不再遵循 Rust 代码的语法惯例。先前展示的 `square_once!` 宏就是一个直观的例子：宏的主体并没有按照 `rustfmt` 的规则来格式化：

```rust
{
    let x = $e;
    // The `rustfmt` tool doesn't really cope with code in
    // macros, so this has not been reformatted to `x * x`.
    x*x
}
```

另一个例子是已经提到的 `http_codes!` 宏，领域特定语言直接使用了 `Group` 枚举的变体名称，比如 `Informational`，而没有使用 `Group::` 前缀或 `use` 语句。这一点会让一些代码导航工具感到困惑。

甚至编译器本身也无法提供更多的帮助：编译器提供的报错信息没有完全跟随宏的定义及使用链。当然，还是有一些工具（参照[第 31 条]）可以辅助宏的使用，比如早先使用的 David Tolnay 的 [cargo-expand]。

另一个使用宏的缺点是可能会导致代码的膨胀 —— 一个简单的宏调用就可能引入数百行的生成代码，并且在进行代码分析时是无法直观看到的。这在代码第一次编写时可能不会成为问题，因为彼时这些代码是需要的，并且帮助开发者节约了大量的代码编写时间。但是，如果这些代码随后不再需要了，仅从数行的宏调用中可能并不能看到将其删除的必要性。

## 建议

尽管上节我们列举了很多宏的缺点，但是当我们需要让一些代码段保持一致，但又没有其他可用的方式时，宏仍然是完成这类工作的正确工具：*当宏是确保不同代码保持一致的唯一方式时，使用它！*

当我们需要压缩一些样板代码时，宏也是可以使用的工具：*使用宏来处理样板代码*，当它们无法用函数或泛型来进行合并时。

为了降低宏对可读性的影响，请尽量避免在宏中使用和 Rust 的常规语法规则相冲突的语法。要么让宏在调用时和一般的代码表现的一致；要么让宏在调用时和一般的代码足够不同，以至于没有用户会混淆宏和一般的代码。特别地，可以遵循如下的准则：
* *尽可能避免在宏展开时插入引用操作* —— 类似 `my_macro!(&list)` 的宏调用就比 `my_macro!(list)` 更符合 Rust 的常规代码。
* *尽量避免在宏中引入非局部的控制流*，这样阅读这段代码的人就可以在不了解宏的细节的情况下，正确理解上下文中的控制流。

这种倾向于类似常规 Rust 代码的可读性偏好有时会影响对声明宏或者过程宏的选择。如果你需要给一个结构体的每一个字段或者枚举的每一个枚举值都生成代码，*应该使用派生宏而不是会生成新类型的过程宏来处理*（暂时忽略在[什么时候使用宏](#什么时候使用宏)中的例子） —— 这样会更加符合语言习惯并且读起来更加简单。

然而，如果要添加的派生宏并非是项目中所独有的功能，可以检查下外部的库中是否已经提供了所需要的宏（参照[第 25 条]）。比如，类似将数值类型转换为合适的 C 风格的枚举值的需求：在[enumn::N]、[num_enum::TryFromPrimitive]、[num_derive::FromPrimitive] 以及 [strum::FromRepr] 中都一定程度的实现了这个需求。

## 注释

[^1]: 眼神好的读者可能已经注意到了 `format_arg!` 仍然像是一个宏的调用，尽管它在 `println!` 宏的展开代码里。这是因为它是编译器的内建宏。

[^2]: 在 [std::fmt 模块]中也包含了很多其他展示特定格式数据时会使用的 trait。比如，当需要一个 `x` 格式的特殊说明符来输出小写的十六进制输出时，就会使用 [LowerHex] trait。


原文[点这里](https://www.lurklurk.org/effective-rust/macros.html)查看。

<!-- 参考链接 -->

[第 10 条]: ../chapter_2/item10-std-traits.md
[第 19 条]: ../chapter_3/item19-reflection.md
[第 25 条]: ../chapter_4/item25-dep-graph.md
[第 31 条]: ../chapter_5/item31-use-tools.md

[On Lisp (Prentice Hall)]: https://www.paulgraham.com/onlisp.html
[卫生的]: https://en.wikipedia.org/wiki/Hygienic_macro
[声明宏]: https://doc.rust-lang.org/reference/macros-by-example.html
[cargo-expand]: https://github.com/dtolnay/cargo-expand
[最小惊讶原则]: https://en.wikipedia.org/wiki/Principle_of_least_astonishment
[`expr`]: https://doc.rust-lang.org/reference/macros-by-example.html#metavariables
[fmt 文档]: https://doc.rust-lang.org/std/fmt/index.html
[过程宏]: https://doc.rust-lang.org/reference/procedural-macros.html
[proc-macro]: https://doc.rust-lang.org/proc_macro/index.html
[proc-macro2]: https://docs.rs/proc-macro2
[serd 库]: https://docs.rs/serde/latest/serde/
[Deserialize]: https://docs.rs/serde/latest/serde/derive.Deserialize.html
[syn 包]: https://docs.rs/syn/latest/syn/
[syn::parse\_macro\_input!]: https://docs.rs/syn/latest/syn/macro.parse_macro_input.html
[syn::DeriveInput]: https://docs.rs/syn/latest/syn/struct.DeriveInput.html
[file!()]: https://doc.rust-lang.org/std/macro.file.html
[line!()]: https://doc.rust-lang.org/std/macro.line.html
[enumn::N]: https://docs.rs/enumn/latest/enumn/derive.N.html
[num_enum::TryFromPrimitive]: https://docs.rs/enumn/latest/enumn/derive.N.html
[num_derive::FromPrimitive]: https://docs.rs/num-derive/latest/num_derive/derive.FromPrimitive.html
[strum::FromRepr]: https://docs.rs/strum/latest/strum/derive.FromRepr.html
[std::fmt 模块]: https://doc.rust-lang.org/std/fmt/index.html
[LowerHex]: https://doc.rust-lang.org/std/fmt/trait.LowerHex.html
