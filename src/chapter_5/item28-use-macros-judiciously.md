# 第 28 条：在合适的时候使用宏

> “在一些场景下，我们会很容易来决定应该使用宏（macro）而非函数（function），因为只有宏才能满足我们的需求。” - Paul Graham，“[On Lisp (Prentice Hall)]”

Rust 的宏能够让你实现元编程（metaprogramming）：在项目中使用代码来生成代码。这一特性在需要编写很多确定性、重复性都很强的“样板代码”时会很有用，不借助宏的话我们就只能手动维护这些代码了。

程序员接触 Rust 之前可能已经预先了解了 C/C++ 中通过预处理（preprocessor）来实现的宏，这种方式是在预处理阶段通过文本替换来展开宏定义。而 Rust 的宏则有一些不同，它是在符号流（parsed tokens of the program）或者在抽象语法树（abstract syntax tree, AST）的基础上实现的宏，而非在文本处理阶段。

这就意味着 Rust 的宏是能够理解代码结构并且规避掉一系列的文本替换方式实现的宏所存在的意外情况。比如说，在接下来的内容中，我们可以看到 Rust 所声明的宏是[卫生的] —— 在宏里不会意外引用（或者捕获）宏所嵌入代码中的变量信息。

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

一种理解宏的方法是将其视为代码的不同抽象方式。函数也是代码的一种简单抽象：它将同一类型的不同值的不同抽象出来，实现了针对这一类型，而非特定的值，会做的操作及方法。而宏中的生成则是另外一个层面的抽象：宏是对符合同一特性的不同类型进行抽象，使用针对这些不同类型所具备的相同特性，而非特性的类型，进行代码的实现。

宏可以对不同程序中扮演相同角色（类型、标记、表达式等）的代码抽象出来，然后这些程序就可以以同一种方式来使用抽象出的逻辑。

Rust 提供了两种方式来定义宏：
* 声明宏，也被成为“示例宏”。声明宏允许将输入到宏中任意的 Rust 程序，基于抽象语法树中的结果，集成到代码中。
* 过程宏。过程宏同样可以将任意的 Rust 程序集成到代码中，不过是基于源码中的解析符号。`derive`宏就是常见的过程宏。`derive`宏可以基于代码的结构定义来展开代码。

## 声明宏

虽然这篇文章不是为了重复[声明宏]的内容，但还是有必要来提醒下声明宏中需要关注的内容。

首先，需要注意的是声明宏的作用域范围和直觉上的理解的是不同的（对比 C 里的预处理宏）。如果一个声明宏在源代码中被定义了，就只有*跟在*宏里的代码能够使用：

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

`#[macro_export]`特性让宏可以访问更多的数据，但是也存在一些奇怪的事情：尽管宏并没有在模块中定义，它还是出现在了模块的顶层。

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

Rust 的声明宏是卫生（hygienic）的：宏内部展开的代码无法使用所在作用域的局部变量。比如，宏内部使用了局部变量 x 时：

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

这种“卫生”的特性意味着 Rust 的宏比 C 的基于预处理替换的宏要安全很多。然而，仍有一些需要在使用时注意的内容。

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

如果我们还记得宏只是在调用它的地方进行展开的话，上述示例就会变得清楚了 —— 在这个示例中，调用宏的地方只相当于添加了一行增加`x.contents`值的代码。借助[cargo-expand]可以很清晰地看到编译器将宏进行展开后的代码：

```rust
let mut x = Item { contents: 42 };
x.contents += 1;
{
    ::std::io::_print(format_args!("x is {0:?}\n", x));
};
```

展开的代码中可以看到直接使用了变量本身，而非其引用。（一个有意思的事情是，我们可以看到`println!`的展开中，依赖了`format_args!`宏[^1]。）

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

对于声明宏来说，第二件需要注意的事情是和 C 的预编译宏同样的问题：如果宏的参数是一个存在副作用的表达式，当心在宏里多次使用的情况。比如我们在早先定义的`square!`宏输入了较为随意的表达式来作为参数，然后使用两次，这将会造成奇怪的结果：

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

## 格式化参数

声明宏的一种常见的使用模式将多个值汇聚成一个消息。比如，标准库中的`format!`用来拼接一个字符串，`println!`用来输出到标准输出，`eprintln!`用来输出到标准错误输出。[fmt 文档]中阐述了`format!`的语法，和`C`中的`printf`使用几乎是相同的。当然，Rust 中的`format!`参数是类型安全并且会在编译时进行检查的，并且`format!`宏实现时使用了`Display`以及`Debug`特性用来约束宏的参数。`Display`以及`Debug`宏的使用参见[第 10 条][^2]。

你可以（同时也建议）在项目中的宏中使用相同的格式化语法。比如，一个`log`库中的`logging`宏就可以使用和`format!`相同的语法。在实践中，使用`format_args!`来实现参数的格式化而不是重复造轮子。

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

```rust
src/main.rs:331: x = 0x0a
```

## 过程宏

Rust 也支持了*过程宏*，也被称为`proc macros`。和声明宏类似，[过程宏]能够任意的 Rust 代码插入到程序的源代码中。不同的时，过程宏的输入不再仅限制在特定的传入参数。过程宏可以访问一些源代码中的解析符号（parsed tokens）。这就过程宏一定程度上类似动态语言，比如 Lisp，的非常富有表达力的能力 —— 但是仍然在编译时进行检查。这也帮助缓解了 Rust 中反射的局限，这在[第 19 条]中讨论了。

过程宏需要和其被使用的代码定义在不同的包中（并且包需要被声明为`proc_macro`），并且包中往往需要引入[proc-macro]（官方工具链中提供）或者[proc-macro2]（由 David Tolnay 提供）的依赖，这两个依赖可以宏能够操作输入的符号。

实际上，有三种不同的过程宏：
* 类函数宏（Function-like macros）：通过传入的参数调用。
* 类属性宏（Attribute macros）：附加到程序中的某些特定语法的代码中。
* 派生宏（Derive macros）：附加到特定的数据结构中。

### 类函数宏

函数式的宏会通过传递参数来调用，宏的实现中可以访问参数的解析符号（parsed tokens），并且返回任意的符号。注意在先前的表述中，我们使用的是单数的参数， —— 即使函数式的宏调用的时候看起来传入了很多参数：

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

```rust
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

由于输入流涉及到的底层特性式的这段宏在实现时必须要能够解析所传入的参数。比如，宏中分隔那些需要分隔的参数需要使用`TokenTree:Punct`。[syn 包]（David Tolnay开发）提供了一个解析的库来辅助这些事情，下一节会进行介绍。

正因为这些解析的工作，使用声明宏往往比函数式的过程宏要简单，因为声明宏所处理的是匹配所定义的结构。

这种需要手动处理繁杂的另一面是函数是的宏可以更加灵活的接受那些无法像一般 Rust 代码解析的输入。这种特性并非经常需要的，所以函数式的宏相对较少出现。

### 类属性宏

类属性宏通过将其放置在程序的一些片段前调用的，而这些片段的解析符号会被传入到宏的内部进行处理。类属性宏也可以将任意的结果作为返回值，但是一般返回值是对输出的一些转换处理。

比如，下面是一个用来封装函数体的类属性宏：

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

```rust
log: calling function 'add_three'
log: called function 'add_three' => 5
add_three(2) = 5
```

这个宏的实现是极为复杂的，这里并不打算将其细节附上。在实现时，需要校验输入符号的结构以便构建新的输出符号。当然，这一过程仍然可以使用`syn`包来辅助实现。

### 派生宏

最后一种过程宏是派生宏。派生宏可以为其修饰的数据（struct, enum 或者 union 均可）自动地生成代码。这一点和类属性宏有些像，但是会多一些派生的操作 —— 请注意理解这里的派生概念。

首先，派生宏会附加到输入的符号中，而非将其替换。这就意味着原始的数据结构的定义会被保留，而派生宏将在原始数据结构的基础上附加代码。

其次，派生宏可以用来声明一些辅助性的特征。当数据需要用作一些特殊的处理时，可以使用派生宏来辅助标注。比如，[serde 库]的[Deserialize]派生宏有一个`serde`的辅助特性，用户可以使用派生宏来声明这些结构体符合某种特性：

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

关于派生宏的最后一个概念是，[syn 包]可以完成将输入符号解析到相应的语法树的工作。[syn::parse\_macro\_input!]宏可以将符号转换成[syn::DeriveInput]数据结构，这种结构描述被修饰对象的主要内容，并且`DeriveInput`操作起来远比原始的符号流要好处理。

特别地，`derive`宏是所有过程宏中最常使用的宏 —— 这种逐字段或逐变量操作的能力能够让程序员最简单地实现最多的功能 —— 比如，仅通过添加一行类似`#[derive(Debug, Clone, PartialEq, Eq)]`的代码，即可实现预期的目的。

由于派生宏的代码插入是自动实现地，这也意味着这些插入的代码可以同时和结构体的实现保持一致。比如，如果你向`struct`中插入了一个新的字段，如果采用手动实现`Debug`特征的话，你就需要在插入后对结构体进行更新以使其满足特征的需求。而对于自动插入代码的派生宏来说，你并不需要做任何调整（当然了，如果插入的字段不满足派生宏的实现要求，编译时会报错）。

## 什么时候使用宏

使用宏的首要原因当然是避免重复的代码 —— 尤其是那些需要人工确保和其他代码关联正确性的重复代码。从这一点来说，使用宏仅是编程常用的封装抽象的扩展：
* 如果需要重复一段处理同一类型的不同值的代码，将其封装为一个函数并在所有需要这段逻辑的地方使用它。
* 如果需要重复一段处理不同类型的代码，构建一个`trait`并且使用该`trait`来封装逻辑并在所有满足该特性的要求的地方进行使用。
* 如果需要重复一段结构相同的代码，将其封装成一个宏并且在所有满足类似结构的代码中进行使用。

举例如下：如果希望规避重复处理不同`enum`的代码，使用宏即可：

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

另一个宏的使用场景是，规避同一结构体中的数据被分散在代码的不同区域。

比如，假设一个结构体封装了 HTTP 的状态码。通过宏可以避免实现这些信息时代码的分散：

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

通过使用宏，可以将每个 HTTP 状态码的所有相关联的信息 —— 数值、元组以及描述信息 —— 都聚集起来，看起来就像是使用一种领域特定语言（domain-specifix language, DSL）来保存数据一样。

汇聚之后，宏就可以生成代码。每一行类似`$( ... )+`中的代码都会被扩展成特定的代码：

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
* 一个`enum`枚举用来保存所有的数值。
* 一个`group()`方法来返回一个 HTTP 状态码的分组归属。
* 一个`text()`方法来将状态码映射到对应的文字描述中。
* 一个`TryFrom<i32>`的特征实现来将数值转换成`enum`中的枚举值。

如果需要新增一个状态码，只需要添加这样的一行代码：

```rust
ImATeapot => (418, ClientError, "I'm a teapot"),
```

如果不使用宏的话，就需要对四部分代码分别更新。编译器可能会有一些提示信息（`match`表达式需要覆盖所有的场景），但是存在一些遗漏 —— `TryFrom<i32` 就很容易被遗忘。

由于宏可以在调用的地方对代码进行展开，所有它们也可以用来自动生成一些提示信息 —— 尤其是，在使用了标准库中的[file!()]以及[line!()]宏了之后，可以生成代码的位置信息：

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

当报错出现时，日志文件中就会自动地包含报错内容、位置等细节：

```rust
use std::convert::TryInto;

let x: Result<u8, _> = log_failure!(512.try_into()); // too big for `u8`
let y = log_failure!(std::str::from_utf8(b"\xc3\x28")); // invalid UTF-8
```

## 宏的缺点

使用宏的最大缺点是引入之后代码的可读性及可维护性。之前在声明宏小结中介绍了宏允许我们创建一个特定的语言来简明地描述代码及数据的关键特性。但是，这也意味着任何阅读这段代码的人将不得不理解这段使用 Rust 实现的特定的语句 —— 而且这还是使用宏来定义的。比如，在`http_codes!`宏的示例中使用了一个名为`Status`的 Rust `enum`，但是在使用的时候并不能察觉到。

这种使用宏而引入的不可知性远超一般程序员所能带来的影响：很多分析或和 Rust 交互的工具无法理解这样晦涩的代码，因为它不在遵循 Rust 代码交互语法。先前展示的`square_once!`宏就是一个直观的例子：宏的主体并没有按照`rustfmt`的规则来格式化：

```rust
{
    let x = $e;
    // The `rustfmt` tool doesn't really cope with code in
    // macros, so this has not been reformatted to `x * x`.
    x*x
}
```

另一个例子是已经提到的`http_codes!`宏，这里使用了`Group`枚举了诸如`Informational`的值，而没有使用`Group::`前缀或`use`语句。这一点会让代码的补全工具感到混淆。

甚至编译器本身也无法提供更多的帮助：编译器提供的报错信息没有完全符合宏的定义及使用。当然，还是有一些工具（参照[第 31 条]）可以辅助宏的使用，比如早先使用的 David Tolnay 的 [cargo-expand]。

使用宏也可能会导致代码的膨胀 —— 一个简单的宏调用就可能引入数百行的生成代码，并且在进行代码分析时是无法直观看到的。这在代码第一次编写时可能不会成为问题，因为彼时这些代码是需要的，并且帮助开发者节约了大量的代码编写时间。但是，如果这些代码随后不再需要了，考虑实际生成的数百行代码，仅从数行的宏调用中可能并不能看到将其删除的必要性。

## 建议

尽管上节我们列举了很多宏的缺点，但是当我们需要合并存在一些存在一致性的代码，但是没有其他可用的方式时，使用宏仍然是完成这样工作的正确工具：*当宏是确保不同代码保持一致的唯一方式时，使用它！*

当我们需要合并一些模版化的代码时，宏也是可以使用的工具：*使用宏来处理模版代码*，当它们无法合并为一个函数或者一个特性时。

为了降低宏对可读性的影响，请尽量避免在宏中使用和 Rust 的一般语法规则相冲突的语法。要么让宏在调用时和一般的代码表现的一致；要么让宏在调用时和一般的代码完全不同，这样就没有用户会混淆宏和一般的代码。特别地，可以遵循如下的准则：
* *尽可能的避免向宏传递参数的引用* —— 类似`my_macro!(list)`的使用就比`my_macro!(&list)`要好。
* *尽量避免在宏中引入非局部的控制流*，这样所有阅读这段代码的人都可以在不了解宏的细节的情况下，正确理解上下文中的控制流。

这种倾向于类似 Rust 一般代码的可读性偏好有时会影响声明宏或者过程式宏的选择。如果你需要给一个`struct`的每一个字段或者`enum`中的每一个枚举值都生成代码，*尽量使用派生宏来处理*（暂时忽略在上一节中列举的问题） —— 这样会更加符合语言习惯并且读起来更加简单。

然而，如果要添加的派生宏并非是项目中所独有的功能，可以检查下外部的库中是否已经提供了所需要的宏（参照[第 25 条]）。比如，类似将数值类型转换为合适的 C 风格的枚举值的需求：在[enumn::N]、[num\_enum::TryFromPrimitive]、[num\_derive::FromPrimitive]以及[strum::FromRepr]中都一定程度的实现了这个需求。

## 注释

[^1]: 眼神儿好的读者可能已经注意到了`format_arg!`仍然像是一个宏的调用，尽管它在`println!`宏的展开代码里。这是因为它是编译器的内建宏。

[^2]: 在[std::fmt 模块]中也包含了很多其他展示特定格式数据是会使用的特性。比如，当需要一个 x 格式的特殊说明符来输出小写的十六进制输出时，就会使用[LowerHex]特性。


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
[fmt 文档]: https://doc.rust-lang.org/std/fmt/index.html
[过程宏]: https://doc.rust-lang.org/reference/procedural-macros.html
[serd 库]: https://docs.rs/serde/latest/serde/
[Deserialize]: https://docs.rs/serde/latest/serde/derive.Deserialize.html
[syn 包]: https://docs.rs/syn/latest/syn/
[syn::parse\_macro\_input!]: https://docs.rs/syn/latest/syn/macro.parse_macro_input.html
[syn::DeriveInput]: https://docs.rs/syn/latest/syn/struct.DeriveInput.html
[file!()]: https://doc.rust-lang.org/std/macro.file.html
[line!()]: https://doc.rust-lang.org/std/macro.line.html
[enumn::N]: https://docs.rs/enumn/latest/enumn/derive.N.html
[num_enum::TryFromPrimitive]: https://docs.rs/enumn/latest/enumn/derive.N.html
[num\_derive::FromPrimitive]: https://docs.rs/num-derive/latest/num_derive/derive.FromPrimitive.html
[strum::FromRepr]: https://docs.rs/strum/latest/strum/derive.FromRepr.html
[std::fmt 模块]: https://doc.rust-lang.org/std/fmt/index.html
[LowerHex]: https://doc.rust-lang.org/std/fmt/trait.LowerHex.html

