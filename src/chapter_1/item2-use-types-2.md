# 第 2 条：使用类型系统表达常见行为

[第1条]讨论了如何在类型系统中表达数据结构；本节继续讨论在 Rust 的类型系统中行为的编码。

本条目所描述的机制通常会让人感觉熟悉，因为它们在其他语言中都有直接的类似物：

- 函数：将一段代码与一个名称和参数列表相关联的通用机制。
- 方法：与特定数据结构的实例相关联的函数。在面向对象成为一种编程范式之后创建的编程语言中，方法很常见。
- 函数指针：C 语言家族中的大多数语言都支持，包括 C++ 和 Go，作为一种在调用其他代码时允许额外间接层次的机制。
- 闭包：最初在 Lisp 语言家族中最为常见，但已经被移植到许多流行的编程语言中，包括 C++（从 C++11 开始）和 Java（从 Java 8 开始）。
- 特征（Traits）：描述适用于同一基础项的相关功能的集合。在许多其他语言中都有大致等效的概念，包括 C++ 中的抽象类以及 Go 和 Java 中的接口。

当然，所有这些机制都有Rust特定的细节，本条将会一一介绍。
在前面列出的内容中，特征（traits）对于本书来说意义最为重大，因为它们描述了 Rust 编译器和标准库提供的很多行为。[第2章] 重点讨论了关于设计和实现特征的相关内容，但它们的普遍性意味着它们在本章的其他条目中也会频繁出现。

## 函数和方法（ Function and Methods ）

跟多数其他编程语言一样，Rust 使用函数将代码组织到一个带名称的代码块，方便重用。代码块的输入用参数的形式表达。跟其他静态类型语言相同，参数和返回值的类型必须明确地被定义：
```rust
/// Return `x` divided by `y`.
fn div(x: f64, y: f64) -> f64 {
    if y == 0.0 {
        // Terminate the function and return a value.
        return f64::NAN;
    }
    // The last expression in the function body is implicitly returned.
    x / y
}

/// Function called just for its side effects, with no return value.
/// Can also write the return value as `-> ()`.
fn show(x: f64) {
    println!("x = {x}");
}

```

如果一个函数与特定的数据结构密切相关，它就表现为一个方法。方法通过 self 标识，对该类型实例进行操作，并包含在 impl DataStructure 块中。这以类似于其他语言的面向对象方式将相关数据和代码封装在一起；然而，在 Rust 中，方法不仅可以添加到`结构体`类型上，也可以添加到`枚举`类型上，这与 Rust 枚举的普遍性质相符（[第1条]）。

```rust
enum Shape {
    Rectangle { width: f64, height: f64 },
    Circle { radius: f64 },
}

impl Shape {
    pub fn area(&self) -> f64 {
        match self {
            Shape::Rectangle { width, height } => width * height,
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        }
    }
}
```

方法的名称为其编码的行为提供了一个标签，而方法签名提供了其输入和输出的类型信息。方法的第一个输入是 `self` 的某种变体，指示该方法可能对数据结构执行的操作：

- `&self` 参数表示可以从数据结构中读取内容，但不会修改它。
- `&mut self` 参数表示该方法可能会修改数据结构的内容。
- `self` 参数表示该方法会消耗数据结构。

### 函数指针

前面的章节描述了如何将一个名称（或者一个参数列表）与某些代码块相关联。实际上，调用方法总是会导致相同的代码被执行；从一次调用到下一次调用，所有的改变就只是该方法操作的数据，这涵盖了许多可能的场景，但是，如果在运行时需要代码发生变化呢？

能实现这个功能的最简单的行为抽象就是[函数指针]：一个仅指向某些代码的指针，其类型反映了函数的签名。

```rust
fn sum(x: i32, y: i32) -> i32 {
    x + y
}
// Explicit coercion to `fn` type is required...
let op: fn(i32, i32) -> i32 = sum;
```

类型在编译时进行检查，所以到程序运行时，这个值只是指针的大小。函数指针没有与之关联的其他数据，因此，可以以各种方式将它们视为值：

```rust
// `fn` types implement `Copy`
let op1 = op;
let op2 = op;
// `fn` types implement `Eq`
assert!(op1 == op2);
// `fn` implements `std::fmt::Pointer`, used by the {:p} format specifier.
println!("op = {:p}", op);
// Example output: "op = 0x101e9aeb0"
```

<!-- <div class="ferris-border"> -->

一个需要注意的技术细节：需要显式地将函数强制转换为 `fn` 类型，因为仅仅使用函数的名称并不能得到 `fn` 类型的值；

这段代码无法编译！

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let op1 = sum;
let op2 = sum;
// Both op1 and op2 are of a type that cannot be named in user code,
// and this internal type does not implement `Eq`.
assert!(op1 == op2);
````

```rust
error[E0369]: binary operation `==` cannot be applied to type `fn(i32, i32) -> i32 {main::sum}`
   --> use-types-behaviour/src/main.rs:117:21
    |
117 |         assert!(op1 == op2);
    |                 --- ^^ --- fn(i32, i32) -> i32 {main::sum}
    |                 |
    |                 fn(i32, i32) -> i32 {main::sum}
    |
help: you might have forgotten to call this function
    |
117 |         assert!(op1( /* arguments */ ) == op2);
    |                    +++++++++++++++++++
help: you might have forgotten to call this function
    |
117 |         assert!(op1 == op2( /* arguments */ ));
    |                           +++++++++++++++++++

```

相反，编译器错误表明该类型类似于 `fn(i32, i32) -> i32 {main::sum}`，一种完全内部于编译器的类型（即不能在用户代码中编写），它同时标识了特定的函数及其签名。

换句话说，`sum` 的类型既编码了函数的签名又编码了其位置（出于优化原因）；这种类型可以自动强制转换为 `fn` 类型（[第6条]）。
<!-- </div> -->

### 闭包

裸函数指针的使用是有限的，因为被调用函数唯一可以使用的输入是那些明确作为参数值传递的内容。

例如，考虑一些使用函数指针修改切片中每个元素的代码。


```rust
// In real code, an `Iterator` method would be more appropriate.
pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
    for value in data {
        *value = mutator(*value);
    }
}
```
这对于对切片进行简单的修改是有效的：

```rust
fn add2(v: u32) -> u32 {
    v + 2
}
let mut data = vec![1, 2, 3];
modify_all(&mut data, add2);
assert_eq!(data, vec![3, 4, 5,]);
```

然而，如果修改依赖于任何额外的状态，那么无法隐式地将这些状态传递给函数指针。

这段代码无法编译！

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let amount_to_add = 3;
fn add_n(v: u32) -> u32 {
    v + amount_to_add
}
let mut data = vec![1, 2, 3];
modify_all(&mut data, add_n);
assert_eq!(data, vec![3, 4, 5,]);
```

```rust
error[E0434]: can't capture dynamic environment in a fn item
   --> use-types-behaviour/src/main.rs:142:17
    |
142 |             v + amount_to_add
    |                 ^^^^^^^^^^^^^
    |
    = help: use the `|| { ... }` closure form instead
```

错误信息指向了正确的工具：闭包。闭包是一段看起来像函数定义体（`lambda` 表达式）的代码，不同之处在于：
- 它可以作为表达式的一部分构建，因此，不需要与一个名称相关联
- 输入参数以竖线 `|param1, param2|` 给出（它们的关联类型通常可以由编译器自动推导）
- 它可以捕获其周围环境的一部分。


```rust
let amount_to_add = 3;
let add_n = |y| {
    // a closure capturing `amount_to_add`
    y + amount_to_add
};
let z = add_n(5);
assert_eq!(z, 8);
```

为了（大致）理解捕获是如何工作的，可以想象编译器创建了一个一次性的、内部的类型，它包含了 `lambda` 表达式中提到的环境所有部分。当闭包被创建时，这个临时类型的一个实例被创建来保存相关的值，当闭包被调用时，这个实例被用作额外的上下文使用。


```rust
let amount_to_add = 3;
// *Rough* equivalent to a capturing closure.
struct InternalContext<'a> {
    // references to captured variables
    amount_to_add: &'a u32,
}
impl<'a> InternalContext<'a> {
    fn internal_op(&self, y: u32) -> u32 {
        // body of the lambda expression
        y + *self.amount_to_add
    }
}
let add_n = InternalContext {
    amount_to_add: &amount_to_add,
};
let z = add_n.internal_op(5);
assert_eq!(z, 8);
```

在这个概念性的上下文中持有的值通常是引用（[第9条]），就像这里的例子，但它们也可以是环境中事物的可变引用，或者是通过在输入参数前使用 `move` 关键字而从环境中完全移出的值。

回到 `modify_all` 的例子，闭包不能用在期望函数指针的地方。

```rust
error[E0308]: mismatched types
   --> use-types-behaviour/src/main.rs:165:31
    |
165 |         modify_all(&mut data, |y| y + amount_to_add);
    |                               ^^^^^^^^^^^^^^^^^^^^^ expected fn pointer, found closure
    |
    = note: expected fn pointer `fn(u32) -> u32`
                  found closure `[closure@use-types-behaviour/src/main.rs:165:31: 165:52]`
note: closures can only be coerced to `fn` types if they do not capture any variables
   --> use-types-behaviour/src/main.rs:165:39
    |
165 |         modify_all(&mut data, |y| y + amount_to_add);
    |                                       ^^^^^^^^^^^^^ `amount_to_add` captured here
```

相反，接收闭包的代码必须接受一个实现了 `Fn*` 特征的实例。

```rust
pub fn modify_all<F>(data: &mut [u32], mut mutator: F)
where
    F: FnMut(u32) -> u32,
{
    for value in data {
        *value = mutator(*value);
    }
}
```

Rust 有三种不同的 `Fn*` 特征，它们之间表达了关于环境捕获行为的一些区别。
- `FnOnce` 描述了一个只能被调用一次的闭包。如果环境的某些部分被移动到闭包的上下文中，并且闭包的主体随后将其从闭包的上下文中移出，那么这种移动只能发生一次 —— 因为源项没有其他副本可以移动 —— 因此，闭包只能被调用一次。
- `FnMut` 描述了一个可以被多次调用的闭包，它能够改变其环境，因为它会可变地借用环境。
- `Fn` 描述了一个可以被多次调用的闭包，它只从环境中不可变地借用值。

编译器会为代码中的任何 `lambda` 表达式自动实现这些 `Fn*` 特征的适当子集；不可能手动实现这些特征中的任何一个[^1]（与 `C++` 的 `operator()` 重载不同）。

回到上面关于闭包的粗略心理模型，编译器自动实现的特征大致对应于捕获的环境上下文是否具有：
- `FnOnce`: 任何被移动的值
- `FnMut`: 任何对值的可变引用（`&mut T`）
- `Fn`: 只是对值的普通引用（`&T`）。

上面列表中的后两个特征各自具有前一个特征的特征约束，当你考虑使用闭包时，这是有意义的。
- 如果某事物只期望调用一次闭包（通过接收 `FnOnce` 表示），那么传递给它一个能够被多次调用的闭包（`FnMut`）是可以的。
- 如果某事物期望重复调用一个可能改变其环境的闭包（通过接收 `FnMut` 表示），那么传递给它一个不需要改变其环境的闭包（`Fn`）是可以的。

裸函数指针类型 `fn` 也名义上属于这个列表的末尾；任何（非不安全的）`fn` 类型自动实现所有 `Fn*` 特征，因为它不借用任何环境。

因此，在编写接受闭包的代码时，**使用最通用的 `Fn*` 特征，以允许调用者最大的灵活性** —— 例如，对于只使用一次的闭包，接受 `FnOnce`。同样的道理：**建议优先使用 `Fn*` 特征约束而不是裸函数指针（`fn`）**。

### 特征（`Traits`）

`Fn*` 特征比裸函数指针更灵活，但它们仍然只能描述单个函数的行为，并且只能在函数签名的基础上描述。
然而，它们本身就是 Rust 类型系统中描述行为的另一种机制的例子，即特征。特征定义了一组相关的方法，这些方法由一些底层项公开提供。此外，这些函数通常是（但不一定是）方法，将 self 的某个变体作为它们的第一个参数。

特征中的每个方法也有一个名称，这允许编译器区分具有相同签名的方法，更重要的是，它允许程序员推断方法的目的。

Rust 的特征大致类似于 Go 和 Java 中的“接口”，或者 C++ 中的“抽象类”（所有虚拟方法，没有数据成员）。特征的实现必须提供所有方法（但请注意特征定义可以包括默认的实现，[第13条]），并且还可以有相关联的数据，那些实现会使用这些数据。这意味着代码和数据在共同的抽象中以某种面向对象的方式一起封装。

接受结构体并调用其方法的代码被限制只能与特定类型一起工作。如果有多个类型实现了公共行为，那么定义一个特征来封装这种行为，并让代码使用特征的方法而不是特定结构体的方法会更加灵活。

这导致了与其他受面向对象[^2]影响的语言相同的建议：**如果预期未来需要灵活性，请优先接受特征类型而不是具体类型**。

有时，你希望在某些行为中使用类型系统来区分，但这些行为无法表达为特征定义中的特定方法签名。例如，考虑一个用于排序集合的特征；一个实现可能是稳定的（比较相同的元素在排序前后的顺序不变），但没有办法在排序方法参数中表达这一点。

在这种情况下，使用标记特征（`marker trait`）在类型系统中跟踪这个要求仍然是值得的。

```rust
pub trait Sort {
    /// Re-arrange contents into sorted order.
    fn sort(&mut self);
}

/// Marker trait to indicate that a [`Sortable`] sorts stably.
pub trait StableSort: Sort {}
```

标记特征（`marker trait`）没有方法，但实现仍然需要声明它正在实现该特征 —— 这被视为实现者的承诺：“我庄严宣誓，我的实现有稳定排序的能力。”依赖于稳定排序的代码可以指定 `StableSort` 特征约束，依赖诚信制度来保持其不变性。**使用标记特征来区分无法在特征方法签名中表达的行为**。

一旦行为被封装到 Rust 的类型系统中作为一个特征，它可以以两种方式被使用：
- 作为特征约束（`trait bound`），它在编译时限制了哪些类型可以接受一个泛型数据类型或方法，或者
- 作为特征对象（`trait object`），它在运行时限制了哪些类型可以存储或传递给一个方法。
[第12条] 更详细地讨论了这两种方式的权衡。

## 特征约束

特征约束表明，当某个类型 `T` 实现了某个特定特征时，参数化为该类型 `T` 的泛型代码才能被使用。特征约束的存在意味着泛型的实现可以使用来自该特征的方法，确信编译器将确保任何可以编译的 `T` 确实具有那些方法。这种检查发生在编译时，当泛型被单态化 ———— 即从处理任意 T 类型的泛型代码转换为处理特定某类型的代码（Rust 对 C++ 中所谓的“模板实例化”的术语）。

对目标类型 `T` 的这种限制是明确的，编码在特征约束中：只有满足特征约束的类型才能实现该特征。这与 C++ 中的等价情况形成对比，在 C++ 中，`template<typename T>` 中使用的类型 `T` 的约束是隐式的[^3]：C++ 模板代码仍然只有在所有引用的方法在编译时都可用时才会编译，但检查纯粹基于方法和签名。（这种“[鸭子类型]”可能导致混淆；一个使用 `t.pop()` 的 C++ 模板可能为 `Stack` 或 `Balloon` 的 `T` 类型参数编译 —— 这不太可能是期望的行为。）

对显式特征约束的需求也意味着大部分泛型使用特征约束。要了解为什么会这样，反过来考虑一下在没有 `T` 的特征约束的情况下 `struct Thing<T>` 可以做什么。没有特征约束，`Thing` 只能执行适用于任何类型 `T` 的操作；这允许`容器`、`集合`和`智能指针`，但除此之外并不多。任何使用类型 `T` 的东西都需要一个特征约束。


```rust
pub fn dump_sorted<T>(mut collection: T)
where
    T: Sort + IntoIterator,
    T::Item: Debug,
{
    // Next line requires `T: Sort` trait bound.
    collection.sort();
    // Next line requires `T: IntoIterator` trait bound.
    for item in collection {
        // Next line requires `T::Item : Debug` trait bound
        println!("{:?}", item);
    }
}
```

因此，这里的建议是**使用特征约束来表达对泛型中使用的类型的要求**，这很容易遵循 —— 编译器将迫使你遵守它。

## 特征对象

特征对象是利用特征定义的封装的另一种方式，但在这里，不同的特征实现是在运行时而不是编译时选择的。这种动态分派类似于 C++ 中虚拟函数的使用，在底层，Rust 有 '`vtable`' 对象，它们与 C++ 中的类似。

特征对象的这种动态方面也意味着它们必须始终通过间接方式处理，通过引用（`&dyn Trait`）或指针（`Box<dyn Trait>`）。这是因为实现特征的对象大小在编译时是未知的 —— 它可能是一个巨大的结构体或一个微小的枚举 —— 因此无法为裸特征对象分配正确数量的空间。

不知道具体对象的大小也意味着用作特征对象的特征不能有返回 Self 类型的方法或使用 Self 的参数（除了接收者 —— 正在调用方法的对象）。原因是使用特征对象的预先编译的代码将不知道 Self 的大小。

具有泛型方法 fn some_fn<T>(t: T) 的特征允许对于可能存在的所有不同类型 T 有无穷多个已实现的方法的可能性。对于用作特征约束的特征来说这是没问题的，因为在编译时可能调用的无穷集合的泛型函数在编译时会变成实际调用的有限集合的泛型方法。对于特征对象则不是这样：在编译时可用的代码必须应对在运行时可能出现的所有可能的 T。

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
trait Foo {
    fn method<T>(&self, t: T);
}

struct Bar;

impl Bar {
    fn new() -> Self {
        Self {}
    }
}

impl Foo for Bar {
    fn method<T>(&self, t: T) {
        println!("Bar impl trait Foo!");
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::hash_map::VacantEntry;

    #[test]
    fn as_trait_bound() {
        let bar = Bar::new();
        bar.method(0u8);
    }

    #[test]
    fn as_trait_obj() {
        let bar = Bar::new();
        let mut v: Vec<&dyn Foo> = vec![];
        v.push(&bar);
    }
}
```

fn as_trait_bound() 测试可以通过，没有错误。但是as_trait_obj()会报错：

```rust
error[E0038]: the trait `Foo` cannot be made into an object
  --> src/lib.rs:33:20
   |
33 |         let mut v: Vec<&dyn Foo> = vec![];
   |                    ^^^^^^^^^^^^^ `Foo` cannot be made into an object
   |
note: for a trait to be "object safe" it needs to allow building a vtable to allow the call to be resolvable dynamically; for more information visit <https://doc.rust-lang.org/reference/items/traits.html#object-safety>
  --> src/lib.rs:2:8
   |
1  | trait Foo {
   |       --- this trait cannot be made into an object...
2  |     fn method<T>(&self, t: T);
   |        ^^^^^^ ...because method `method` has generic type parameters
   = help: consider moving `method` to another trait
   = help: only type `Bar` implements the trait, consider using it directly instead
```

这两个限制 —— 不能返回 `Self` 和不能有泛型方法 —— 结合成了对象安全的概念。只有对象安全的特征才能用作特征对象。

---

#### 注释

[^1]: 至少，在撰写本文时的稳定 Rust 中是这样。实验性功能 `unboxed_closures` 和 `fn_traits` 可能在未来改变这一点。

[^2]: 例如，Effective Java 第64条：通过它们的接口引用对象

[^3]: C++20 中添加的概念允许对模板类型上的约束进行显式指定，但检查仍然只在模板实例化时执行，而不是在声明时执行。

原文[点这里](https://www.lurklurk.org/effective-rust/use-types-2.html)查看

<!-- 参考链接 -->

[第1条]: item1-use-types.md
[第2章]: /chapter_2//item10-std-traits.md
[第6条]: item6-newtype.md
[第9条]: item9-iterators.md
[第12条]: /chapter_2/item12-generics&trait-objects.md
[第13条]: /chapter_2/item13-use-default-impl.md

[函数指针]: https://doc.rust-lang.org/std/primitive.fn.html
[鸭子类型]: https://en.wikipedia.org/wiki/Duck_typing
