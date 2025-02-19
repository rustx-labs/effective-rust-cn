# 第 10 条：熟悉标准库的 traits

Rust 通过一系列细粒度的、描述类型行为的标准库 `traits` 将类型系统自身的关键行为特征编码到了类型系统本身（参考[第 2 条]）。

其中的许多 `traits` 对于 C++ 程序员来说会感觉很熟悉，类比于拷贝构造函数、析构函数、相等性判断和赋值运算符等等。

和在 C++ 中一样，为用户的自定义类型实现标准库当中的许多 `traits` 是个不错的选择；Rust 编译器会在用户的自定义类型需要某些 `traits` 而类型又缺少对应实现的时候给出有用的错误信息。

实现这么多的 `traits` 看起来有点吓人，但当中绝大多数的 `traits` 都可以通过 [`derive` 宏][derive macros] 自动应用到用户的自定义类型上。 `derive` 宏会基于类型生成相应的实现（例如：对于 `struct` 的字段逐一进行 `Eq` 判断）；这通常要求结构体的组成部分也实现了对应的 `trait`。自动生成的实现*通常*就是你会需要的，但也有例外，我们会在后面讨论具体 `trait` 的时候提到。

使用 `derive` 宏会让类型的定义看着像这样：

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, PartialOrd, Ord, Hash)]
enum MyBooleanOption {
    Off,
    On,
}
```

这里就触发了对于 8 个不同的 `trait` 实现的自动生成。

这种细粒度的行为规范一开始可能会让人觉得不适应，但熟悉最常见的标准库 `traits` 非常重要，这样才能快速了解到一个类型的定义中给它赋予的各种可用行为。

## 常见的标准库 traits

这一节讨论最常遇到的标准库 traits。以下是对这些 traits 粗略的一句话总结：

- [Clone]：需要时，该类型的实例可以通过执行用户定义的代码来创建自身的一个副本。
- [Copy]：如果编译器对类型实例的内存表示数据执行按比特拷贝，会得到一个有效的新副本。
- [Default]：可以使用合理的默认值创建该类型的实例。
- [PartialEq]：该类型的实例之间存在[部分等价关系][partial equivalence relation] —— 任意两个实例可以明确地进行比较，但 `x == x` 并不总为真。
- [Eq]：该类型的实例之间存在[等价关系][equivalence relation] —— 任意两个实例可以明确地进行比较，且 `x == x` 总为真。
- [PartialOrd]：该类型的某些实例之间可以进行比较和排序。
- [Ord]：该类型的所有实例之间可以进行比较和排序。
- [Hash]：该类型的实例可以在需要的时候生成一个稳定的散列值（哈希值）。
- [Debug]：该类型的实例可以对程序员显示（调试信息）。
- [Display]：该类型的实例可以对用户显示。

除了 `Display` （因为与 `Debug` 功能有重叠）以外，这些 `traits` 都可以通过 `derive` 为用户自定义类型派生。然后有些时候手动实现 —— 或者说不实现这些 `traits` 可能是个更好的选择。

下面的小节会更详细地讨论这些常见的 `traits`。

### [Clone]

`Clone` `trait` 表示可以通过调用 [clone()] 函数来创建一个对象的新副本。这跟 C++ 的拷贝函数大致相同，但是表意更加明确：编译器不会默默地调用这个函数（下一节会更详细地说明）。

如果一个类型的所有字段都实现了 `Clone` ，那么可以通过 `derive` 为这个类型自动派生 `Clone`。`derive` 派生获得的实现会对类型的每个成员依次执行克隆操作；再说一次，这跟 C++ 的构造函数大致相同。这个 `trait` 需要显式地启用（通过添加 `#[derive(Clone)]`），这与 C++ 中需要显式禁止（`MyType(const MyType&) = delete;`）恰恰相反。

派生 `Clone` 是一个常见且有用的操作，以至于更应该去了解哪些情况下不应该或不能实现 `Clone`，或者默认的派生实现是否符合实际要求。

- 如果一个类型的实例持有某些资源的唯一访问权（例如[第 11 条]提到的 `RAII` 类型 ），或者有其他原因限制拷贝（例如对象持有了加密密钥），那么你*不应该*实现 `Clone`。
- 如果类型的某些部分不是 `Clone` 的，那么你也*无法*实现 `Clone`：
  - 字段是可变引用（`&mut T`），因为借用检查器（[第 15 条]）在同一时刻只允许一个对同一资源的可变引用的存在。
  - 属于上述类别的标准库类型，例如 [MutexGuard]（体现唯一访问权）或者 [Mutex]（出于线程安全限制拷贝）。
- 如果类型的任何部分不能通过（递归的）字段拷贝，或者有生命周期相关的簿记动作需要执行，那么你需要*手动*实现 `Clone`。举个例子，考虑一个在运行时跟踪所有现存实例数量的度量类型，你需要手动实现 `Clone` 来保证计数器的准确。

### [Copy]

`Copy` `trait` 有一个简单的声明：

```rust
pub trait Copy: Clone { }
```

这个 `trait` 里面没有方法，这意味这是一个标记 `trait`（如同[第 2 条]中所述）：这是用来表示对类型的某些约束，而这种约束本身没有直接在类型系统中表达。

对于 `Copy` 而言，这个标记表示，对类型实例的内存进行按比特的拷贝，可以得到一个正确的新对象。实际上，这个 `trait` 是一个标记，表示一个类型是 [plain old data]（POD）类型。

这也意味着 `Clone` 特性可能会有点令人困惑：尽管实现了 `Copy` 的类型需要实现 `Clone`，但是当一个实例被拷贝的时候，`clone()` 方法并*没有*被调用 —— 编译器在不使用任何用户定义代码的情况下生成了一个新的对象。

跟其他用户自定义的标记 `trait`（[第 2 条]）相比，`Copy` 对编译器有着比作为 `trait` 约束以外更特殊的意义（和 `std::marker` 中的其他几个 `trait` 一样）—— 它使编译器在处理类型的时候从*移动语意*变成*拷贝语义*。

在移动语义的场景下，赋值运算符会把运算符右侧的内容，拿走并赋值给左边：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
#[derive(Debug, Clone)]
struct KeyId(u32);

let k = KeyId(42);
let k2 = k; // value moves out of k into k2
println!("k = {k:?}");
```

```shell
error[E0382]: borrow of moved value: `k`
  --> src/main.rs:60:23
   |
58 |         let k = KeyId(42);
   |             - move occurs because `k` has type `main::KeyId`, which does
   |               not implement the `Copy` trait
59 |         let k2 = k; // value moves out of k into k2
   |                  - value moved here
60 |         println!("k = {k:?}");
   |                       ^^^^^ value borrowed here after move
   |
   = note: this error originates in the macro `$crate::format_args_nl`
help: consider cloning the value if the performance cost is acceptable
   |
59 |         let k2 = k.clone(); // value moves out of k into k2
   |                   ++++++++
```

而使用拷贝语义的话，被用于赋值的变量在赋值过后依然存在：

```rust
#[derive(Debug, Clone, Copy)]
struct KeyId(u32);

let k = KeyId(42);
let k2 = k; // value bitwise copied from k to k2
println!("k = {k:?}");
```

这使得 `Copy` 成为了最需要注意的 `trait` 之一：它从根本上改变了赋值的行为 —— 包括方法调用时候的传参。

在这方面，这跟 C++ 的拷贝构造函数又有相似了，但是值得强调的一个关键区别在于：在 Rust 里面没有办法让编译器隐式调用用户定义的代码 —— 要调用的话必须显式指定（比如使用 `.clone()`），或者让编译器执行并非由用户定义的代码（比如按比特位的拷贝动作）。

因为 `Copy` 具有 `Clone` `trait` 的特性，所以是可以 `.clone()` 任意一个满足 `Copy` 的对象的。但是这不是一个好的主意：按位拷贝总是会比调用 `trait` 方法要快。Clippy（[第 29 条]）会提示你：

```rust
let k3 = k.clone();
```

```shell
warning: using `clone` on type `KeyId` which implements the `Copy` trait
  --> src/main.rs:79:14
   |
79 |     let k3 = k.clone();
   |              ^^^^^^^^^ help: try removing the `clone` call: `k`
   |
```

跟讨论 `Clone` 的时候类似，何时应该或不应该实现 `Copy` 也是值得探讨的：
- 显而易见：**如果按比特位拷贝不能生成有效的新对象，不要实现 `Copy`**。如果 `Clone` 需要手动而不是通过 `derive` 实现，那么很有可能是这种情况。
- 如果你的类型比较大，实现 `Copy` 可能是个坏主意。`Copy` 的基本约定是按比特位拷贝是合法的，然而这也通常隐含一个假设：拷贝是很快速的。如果你的类不符合这样的要求，不实现 `Copy` 可以阻止意外的耗时拷贝的发生。
- 如果你的类型中某些部分不是 `Copy` 的那么你也*不能*为类型实现 `Copy`。
- 如果你的类型中所有部分都是 `Copy` 的，那么通常也值得为你的类型派生一个 `Copy`。编译器有一个默认的提示项 [missing_copy_implementations] 来指出这种情况。

### [Default]

`Default` `trait` 通过 [default()] 方法定义了一个*默认构造函数*。如果用户定义类型的内含类型都有 `Default` 的实现，那么类型可以通过 `derive` 来实现这个 `trait`；如果内含类型并非都实现了 `Default`，那么用户需要手动为类型实现这个 `trait`。还是跟 C++ 做比较：在 Rust 中需要显式地定义默认构造函数 —— 编译器不会自动帮你创建。

`enum` 类型也可以通过 `derive` 实现 `Default` `trait`，只要给编译器提供一个 `#[default]` 属性来提示编译器哪一个分支是默认值即可：

```rust
#[derive(Default)]
enum IceCreamFlavor {
    Chocolate,
    Strawberry,
    #[default]
    Vanilla,
}
```

`Default` `trait` 最有用的一个场景是和[结构体更新语法][struct update syntax]组合使用。这个语法允许 `struct` 实例通过从同一种 `struct` 的实例中拷贝或者移动内容来初始化自身尚未显式初始化的字段。要被拷贝的模板在初始化结束的时候给出，放在 `..` 之后，`Default` `trait` 提供了一个理想的模板供使用：

```rust
#[derive(Default)]
struct Color {
    red: u8,
    green: u8,
    blue: u8,
    alpha: u8,
}

let c = Color {
    red: 128,
    ..Default::default()
};
```

这使得初始化具有大量字段，但仅有部分字段不是使用默认值的结构体变得更加容易。（[第 7 条]提到的构造器模式对于这种场景或许也是适用的）。

### [PartialEq] 和 [Eq]

`PartialEq` 和 `Eq` `trait` 允许你为用户定义的类型定义相等性。这些 `trait` 尤其重要，因为如果这些 `trait` 被定义了，编译器会自动在相等性（`==`）判断的时候调用相关逻辑，类比于 C++ 中的 `operator==`。`derive` 来的实现会执行逐字段的相等性判断。

`Eq` 版本只是一个标记 `trait`，用于扩展 `PartialEq`，它添加了*自反性*：任何声明了 `Eq` 的类型 `T`，对于任意的实例 `x: T` 都应该保证有 `x == x`。

这很奇怪，你会立即提出这个问题：什么时候 `x == x` 是不成立的？对相等性的这种拆分主要跟[浮点数][floating point numbers] [^1] 有关，尤其是涉及到“不是数字”这个 NaN 值（对应 Rust 中的 `f32:NAN`/`f64:NAN`）。浮点数的标准要求任何东西不会等于 NaN，*包括 NaN 自身*；`PartialEq` `trait` 的存在就是这种要求的连带反应。

对于没有任何浮点数相关特性的用户自定义类型，**你应该在实现 `PartialEq` 的同时也实现 `Eq`**。如果你要把类型当作 [HashMap] 类型的 key，完整的 `Eq` `trait` 也是需要实现的（同样还有 `Hash` `trait`）。

如果类型包含一些不影响区分相等性的字段（如内部缓存或者其他类型的性能优化），则应该手动实现 `PartialEq`。如果 `Eq` 也被定义了，实现也将用于 `Eq`，因为 `Eq` 只是一个标记 `trait`，它内部并没有任何方法。

### [PartialOrd] 和 [Ord]

`PartialOrd` 和 `Ord` 这两个排序 `trait` 允许比较同一类型的两个项，并返回 `Less`、`Greater` 或 `Equal` 之一。这两个 `trait` 需要对应的相等性 `trait` 有对应的实现（`PartialOrd` 要求 `PartialEq`; `Ord` 要求 `Eq`），并且对应的两个必须逻辑自洽（手动实现的时候尤其要注意）。

跟相等性 `trait` 一样，比较 `trait` 也很重要，因为编译器会在比较运算中使用到它们（`<`，`>`，`<=`，`>=`）。

`derive` 产生的默认实现会按照字段（或 `enum` 的不同变体）定义的顺序，按照字典序进行比较。如果这不符合预期结果则需要手动实现 `trait`（或者对字段进行重新排序）。

跟 `PartialEq` 不同的是，`PartialOrd` `trait` 确实对应各种真实发生的场景。比如说，它可以用于表示集合之间的子集关系 [^2]：`{1, 2}` 是 `{1, 2, 4}` 的子集，但 `{1, 3}` 不是 `{2, 4}` 的子集，反过来也是不成立的。

但是，即使偏序关系准确地描述了你的类型的行为，**要小心仅仅实现了 `PartialOrd` 而没有实现 `Ord` 的场景**（这种情况很少见，它与[第 2 条]中将行为编码到类型系统中的建议相违背）—— 它可能会导致令人惊讶的结果：

```rust
// 从 `f32` 继承 `PartialOrd`
#[derive(PartialOrd, PartialEq)]
struct Oddity(f32);

// NaN 的输入值可能会给出意想不到的结果。
let x = Oddity(f32::NAN);
let y = Oddity(f32::NAN);

// 一个看起来应该总是成立的自比较也可能不成立。
if x <= x {
    println!("This line doesn't get executed!");
}

// 程序员也不太可能写出覆盖所有可能的比较分支；如果涉及的类型实现了 `Ord`，那么后两种情况可以合并。
if x <= y {
    println!("y is bigger"); // Not hit.
} else if y < x {
    println!("x is bigger"); // Not hit.
} else {
    println!("Neither is bigger");
}
```

### [Hash]

`Hash` `trait` 用于给某个对象生成一个大概率与其他对象不相同的值。这个哈希值通常用于一些基于哈希桶的数据结构，比如 [HashMap] 和 [HashSet]；因此，这些数据结构中的 key 类型必须实现 `Hash`（和 `Eq`）。

反过来说，“相同”的项（以 `Eq` 来说）必须产生一样的的哈希值：如果 `x == y` （通过 `Eq`），那么 `hash(x) == hash(y)` 必须始终为真。**如果你手动实现了 `Eq`，那么要检查确认你是否也需要手动实现 `Hash`**以满足上述要求。

### [Debug] 和 [Display]

`Debug` 和 `Display` `trait` 允许类型定义它在输出中应该怎么显示，这包括常规显示（`{}` 格式化参数）和调试（`{:?}` 格式化参数）目的，这大致类似于 C++ 中对 `iostream` 的 `operator<<` 重载。

这两个 `trait` 的设计意图差异不仅仅在于格式化占位符的不同，还包括：

- `Debug` 可以通过 `derive` 自动派生 获得，而 `Display` 只能手动实现。
- `Debug` 的输出格式在不同的 Rust 版本下可能会不一样。如果输出需要被其他的代码做解析，那么使用 `Display`。
- `Debug` 是面向程序员的， `Display` 是面向用户的。一个有助于理解这个场景的思想实验是：如果程序被本地化到程序作者不懂的语言会发生什么 —— 如果显示的内容应该被翻译，那么使用 `Display` 是合适的，否则就应该使用 `Debug`。

通常来说，**给你的类型添加一个自动生成的 `Debug` 实现是个不错的选择**，除非类型里面包含一些敏感信息（个人详细信息、密码相关的内容等）。为了更容易遵守这个规则，Rust 编译器有一个 [missing_debug_implementations ] 提示可以指出没有实现 `Debug` 的类型。这个提示默认是禁用的，但可以通过以下任一方式在你的代码中启用：

```rust
#![warn(missing_debug_implementations)]
```

```rust
#![deny(missing_debug_implementations)]
```

如果自动生成的 `Debug` 实现输出了太多细节信息，那么或许手动实现并且汇总一下类型里面的内容会更好一点。

如果你的类型要作为文本向最终用户显示，那么就实现 `Display`。

## 其他条款描述的标准库 trait

除了前面章节描述的常见 `trait`，标准库还包含其他没那么常见的 `trait`。在这些之中，以下是最重要的，但它们都在其他条款中有所介绍，因此这里不会详细地介绍：

- [Fn]，[FnOnce] 和 [FnMut]：实现了这些 `trait` 的对象代表它们是可以被调用的闭包。见[第 2 条]。
- [Error]：实现了这个 `trait` 的对象可以向用户或程序员呈现错误信息，并且可能包含嵌套的子错误信息。见[第 4 条]。
- [Drop]：实现了这个 `trait` 的对象会在它们被销毁的时候执行动作，这对于 RAII 模式来说是至关重要的。见[第 11 条]。
- [From] 和 [TryFrom]：实现了这些 `trait` 的对象，可以自动从别的类型的实例中转换过来，但后一个 `trait` 表示转换可能会失败。见[第 5 条]。
- [Deref] 和 [DerefMut]：实现了这些 `trait` 的对象是类似于指针的对象，它们可以被解引用以获得对内部对象的访问。见[第 8 条]。
- [Iterator] 及相关：实现了这些 `trait` 的对象表示可以它是一个可以被迭代的集合。见[第 9 条]。
- [Send]：实现了这个 `trait` 的对象可以在多个线程之间安全地传输。见[第 17 条]。
- [Sync]：实现了这个 `trait` 的对象可以在多个线程之间被安全地引用。见[第 17 条]。

这些 `trait` 都不能通过 `derive` 直接派生获得。

## 运算符重载

标准库 `trait` 的最后一个类别跟运算符重载有关，Rust 允许用户自定义类型通过实现 [std::ops] 模块里面的标准库 `trait`，对内置的单目和双目运算符进行重载。这些 `trait` 不能通过 `derive` 获得，通常仅用于表示“代数”类型的对象，对于这些类型来说这些运算符都有很自然的解释。

然而，C++ 中的经验表明，最好**避免对不相关的类型进行运算符重载**，因为它通常会导致代码难以维护，也可能会出现一些意外的性能问题（比如，一个 `x + y` 操作调用了一个开销为 O(N) 的方法）。

为了遵循最小惊讶原则，如果你实现了任何一个运算符重载，那么你应该**实现一系列相关的运算符的重载**。打个比方，如果 `x + y` 有一个重载（[Add]），并且 `-y`（[Neg]）也有，那么你应该实现 `x - y`（[Sub]）并确保它给出和 `x + (-y)` 一样的结果。

传递给运算符重载 `trait` 方法的对象会被移动掉，这意味着非 `Copy` 的类型默认会被消耗掉。为 `&'a MyType` 实现这些 `trait` 可以帮助解决这个问题，但需要更多的样板代码来覆盖所有的可能性（比如，对于双目运算符，入参类型可以是引用/非引用，就有 4 = 2 × 2 种可能性）。

## 总结

这个条款已经涵盖了很多方面，下面按序给出了一些表格，总结了我们谈论到的标准库 `trait`。首先，表格 2-1 涵盖了这个条款深入讲述过的 `trait`，除了 `Display` 以外，所有这些 `trait` 都能通过 `derive` 获得实现。

*表格 2-1. 常见的标准库 `trait`*

| `Trait`      |      编译器使用       |        约束         | 方法          |
| :----------- | :------------------: | :-----------------: | :------------ |
| [Clone]      |                      |                     | [clone]       |
| [Copy]       |     `let y = x;`     |       `Clone`       | 标记 `trait`  |
| [Default]    |                      |                     | [default]     |
| [PartialEq]  |       `x == y`       |                     | [eq]          |
| [Eq]         |       `x == y`       |     `PartialEq`     | 标记 `trait`  |
| [PartialOrd] | `x < y`, `x <= y`, … |     `PartialEq`     | [partial_cmp] |
| [Ord]        | `x < y`, `x <= y`, … |  `Eq + PartialOrd`  | [cmp]         |
| [Hash]       |                      |                     | [hash]        |
| [Debug]      | `format!("{:?}", x)` |                     | [fmt]         |
| [Display]    |  `format!("{}", x)`  |                     | [fmt]         |

运算符重载相关的 `trait` 在表格 2-2 [^3] 中总结了。它们都不能通过 `derive` 获得。

*表格 2-2. 运算符重载 `trait`*

| `Trait`        | 编译器使用  | 约束 | 方法            |
| :------------- | :--------: | :--: | :-------------- |
| [Add]          |  `x + y`   |      | [add]           |
| [AddAssign]    |  `x += y`  |      | [add_assign]    |
| [BitAnd]       |  `x & y`   |      | [bitand]        |
| [BitAndAssign] |  `x &= y`  |      | [bitand_assign] |
| [BitOr]        |  `x \| y`  |      | [bitor]         |
| [BitOrAssign]  |  `x \|= y` |      | [bitor_assign]  |
| [BitXor]       |  `x ^ y`   |      | [bitxor]        |
| [BitXorAssign] |  `x ^= y`  |      | [bitxor_assign] |
| [Div]          |  `x / y`   |      | [div]           |
| [DivAssign]    |  `x /= y`  |      | [div_assign]    |
| [Mul]          |  `x * y`   |      | [mul]           |
| [MulAssign]    |  `x *= y`  |      | [mul_assign]    |
| [Neg]          |    `-x`    |      | [neg]           |
| [Not]          |    `!x`    |      | [not]           |
| [Rem]          |  `x % y`   |      | [rem]           |
| [RemAssign]    |  `x %= y`  |      | [rem_assign]    |
| [Shl]          |  `x << y`  |      | [shl]           |
| [ShlAssign]    | `x <<= y`  |      | [shl_assign]    |
| [Shr]          |  `x >> y`  |      | [shr]           |
| [ShrAssign]    | `x >>= y`  |      | [shr_assign]    |
| [Sub]          |  `x - y`   |      | [sub]           |
| [SubAssign]    |  `x -= y`  |      | [sub_assign]    |

为完整起见，在其他条款中提及的 `trait` 在表格 2-3 中涵盖了。这些 `trait` 都不能通过 `derive` 获得（但是 `Send` 和 `Sync` 可能由编译器自动实现）。

*表格 2-3. 在其他条款中提及的 `trait`*

| `Trait`               |    条款    |     编译器使用       |       约束          | 方法             |
| :-------------------- | :--------: | :-----------------: | :-----------------: | :-------------- |
| [Fn]                  | [第 2 条]  |       `x(a)`        |       `FnMut`       | [call]          |
| [FnMut]               | [第 2 条]  |       `x(a)`        |      `FnOnce`       | [call_mut]      |
| [FnOnce]              | [第 2 条]  |       `x(a)`        |                     | [call_once]     |
| [Error]               | [第 4 条]  |                     | `Display + Debug`   | [source]        |
| [From]                | [第 5 条]  |                     |                     | [from]          |
| [TryFrom]             | [第 5 条]  |                     |                     | [try_from]      |
| [Into]                | [第 5 条]  |                     |                     | [into]          |
| [TryInto]             | [第 5 条]  |                     |                     | [try_into]      |
| [AsRef]               | [第 8 条]  |                     |                     | [as_ref]        |
| [AsMut]               | [第 8 条]  |                     |                     | [as_mut]        |
| [Borrow]              | [第 8 条]  |                     |                     | [borrow]        |
| [BorrowMut]           | [第 8 条]  |                     |      `Borrow`       | [borrow_mut]    |
| [ToOwned]             | [第 8 条]  |                     |                     | [to_owned]      |
| [Deref]               | [第 8 条]  |     `*x`, `&x`      |                     | [deref]         |
| [DerefMut]            | [第 8 条]  |   `*x`, `&mut x`    |       `Deref`       | [deref_mut]     |
| [Index]               | [第 8 条]  |      `x[idx]`       |                     | [index]         |
| [IndexMut]            | [第 8 条]  |   `x[idx] = ...`    |       `Index`       | [index_mut]     |
| [Pointer]             | [第 8 条]  | `format("{:p}", x)` |                     | [fmt]           |
| [Iterator]            | [第 9 条]  |                     |                     | [next]          |
| [IntoIterator]        | [第 9 条]  |    `for y in x`     |                     | [into_iter]     |
| [FromIterator]        | [第 9 条]  |                     |                     | [from_iter]     |
| [ExactSizeIterator]   | [第 9 条]  |                     |     `Iterator`      | （[size_hint]） |
| [DoubleEndedIterator] | [第 9 条]  |                     |     `Iterator`      | [next_back]     |
| [Drop]                | [第 11 条] | `}` （作用域结束）   |                     | [drop]          |
| [Sized]               | [第 12 条] |                     |                     | 标记 `trait`    |
| [Send]                | [第 17 条] |     跨线程传递      |                     | 标记 `trait`    |
| [Sync]                | [第 17 条] |     跨线程使用      |                     | 标记 `trait`    |


## 注释

[^1]: 当然，比较浮点数总是一个危险的游戏，因为通常情况下没法保证精度舍入计算会产生跟最初设想的数字（按比特值存储）完全相同的结果。

[^2]: 更一般地说，任何“[格结构][lattice structure]”都具有偏序性质。

[^3]: 这里的一些名称有点隐晦 —— 例如 `Rem` 是求余数，`Shl` 是按位左移 —— 但是 [std::ops] 的文档清楚第说明了它们的预期行为。

原文[点这里](https://www.lurklurk.org/effective-rust/std-traits.html)查看

<!-- 参考链接 -->

[第 2 条]: ../chapter_1/item2-use-types-2.md
[第 4 条]: ../chapter_1/item4-errors.md
[第 5 条]: ../chapter_1/item5-casts.md
[第 7 条]: ../chapter_1/item7-builder.md
[第 8 条]: ../chapter_1/item8-references&pointer.md
[第 9 条]: ../chapter_1/item9-iterators.md
[第 11 条]: ../chapter_2/item11-impl-drop-for-RAII.md
[第 12 条]: ../chapter_2/item12-generics&trait-objects.md
[第 15 条]: ../chapter_3/item15-borrows.md
[第 17 条]: ../chapter_3/item17-deadlock.md
[第 29 条]: ../chapter_5/item29-listen-to-clippy.md

[derive macros]: https://doc.rust-lang.org/reference/procedural-macros.html#derive-macros
[add_assign]: https://doc.rust-lang.org/std/ops/trait.AddAssign.html#tymethod.add_assign
[Add]: https://doc.rust-lang.org/std/ops/trait.Add.html
[AddAssign]: https://doc.rust-lang.org/std/ops/trait.AddAssign.html
[as_mut]: https://doc.rust-lang.org/std/convert/trait.AsMut.html#tymethod.as_mut
[as_ref]: https://doc.rust-lang.org/std/convert/trait.AsRef.html#tymethod.as_ref
[AsMut]: https://doc.rust-lang.org/std/convert/trait.AsMut.html
[AsRef]: https://doc.rust-lang.org/std/convert/trait.AsRef.html
[bitand_assign]: https://doc.rust-lang.org/std/ops/trait.BitAndAssign.html#tymethod.bitand_assign
[BitAnd]: https://doc.rust-lang.org/std/ops/trait.BitAnd.html
[bitand]: https://doc.rust-lang.org/std/ops/trait.BitAnd.html#tymethod.bitand
[BitAndAssign]: https://doc.rust-lang.org/std/ops/trait.BitAndAssign.html
[bitor_assign]: https://doc.rust-lang.org/std/ops/trait.BitOrAssign.html#tymethod.bitor_assign
[BitOr]: https://doc.rust-lang.org/std/ops/trait.BitOr.html
[bitor]: https://doc.rust-lang.org/std/ops/trait.BitOr.html#tymethod.bitor
[BitOrAssign]: https://doc.rust-lang.org/std/ops/trait.BitOrAssign.html
[bitxor_assign]: https://doc.rust-lang.org/std/ops/trait.BitXorAssign.html#tymethod.bitxor_assign
[BitXor]: https://doc.rust-lang.org/std/ops/trait.BitXor.html
[bitxor]: https://doc.rust-lang.org/std/ops/trait.BitXor.html#tymethod.bitxor
[BitXorAssign]: https://doc.rust-lang.org/std/ops/trait.BitXorAssign.html
[borrow_mut]: https://doc.rust-lang.org/std/borrow/trait.BorrowMut.html#tymethod.borrow_mut
[Borrow]: https://doc.rust-lang.org/std/borrow/trait.Borrow.html
[borrow]: https://doc.rust-lang.org/std/borrow/trait.Borrow.html#tymethod.borrow
[BorrowMut]: https://doc.rust-lang.org/std/borrow/trait.BorrowMut.html
[call_mut]: https://doc.rust-lang.org/std/ops/trait.FnMut.html#tymethod.call_mut
[call_once]: https://doc.rust-lang.org/std/ops/trait.FnOnce.html#tymethod.call_once
[call]: https://doc.rust-lang.org/std/ops/trait.Fn.html#tymethod.call
[clone()]: https://doc.rust-lang.org/std/clone/trait.Clone.html#tymethod.clone
[Clone]: https://doc.rust-lang.org/std/clone/trait.Clone.html
[clone]: https://doc.rust-lang.org/std/clone/trait.Clone.html#tymethod.clone
[cmp]: https://doc.rust-lang.org/std/cmp/trait.Ord.html#tymethod.cmp
[Copy]: https://doc.rust-lang.org/std/marker/trait.Copy.html
[Debug]: https://doc.rust-lang.org/std/fmt/trait.Debug.html
[default()]: https://doc.rust-lang.org/std/default/trait.Default.html#tymethod.default
[Default]: https://doc.rust-lang.org/std/default/trait.Default.html
[default]: https://doc.rust-lang.org/std/default/trait.Default.html#tymethod.default
[deref_mut]: https://doc.rust-lang.org/std/ops/trait.DerefMut.html#tymethod.deref_mut
[Deref]: https://doc.rust-lang.org/std/ops/trait.Deref.html
[DerefMut]: https://doc.rust-lang.org/std/ops/trait.DerefMut.html
[Display]: https://doc.rust-lang.org/std/fmt/trait.Display.html
[div_assign]: https://doc.rust-lang.org/std/ops/trait.DivAssign.html#tymethod.div_assign
[Div]: https://doc.rust-lang.org/std/ops/trait.Div.html
[div]: https://doc.rust-lang.org/std/ops/trait.Div.html#tymethod.div
[DivAssign]: https://doc.rust-lang.org/std/ops/trait.DivAssign.html
[DoubleEndedIterator]: https://doc.rust-lang.org/core/iter/trait.DoubleEndedIterator.html
[Drop]: https://doc.rust-lang.org/std/ops/trait.Drop.html
[Eq]: https://doc.rust-lang.org/std/cmp/trait.Eq.html
[equivalence relation]: https://en.wikipedia.org/wiki/Equivalence_relation
[Error]: https://doc.rust-lang.org/std/error/trait.Error.html
[ExactSizeIterator]: https://doc.rust-lang.org/core/iter/trait.ExactSizeIterator.html
[floating point numbers]: https://en.wikipedia.org/wiki/Single-precision_floating-point_format
[fmt]: https://doc.rust-lang.org/std/fmt/trait.Debug.html#tymethod.fmt
[Fn]: https://doc.rust-lang.org/std/ops/trait.Fn.html
[FnMut]: https://doc.rust-lang.org/std/ops/trait.FnMut.html
[FnOnce]: https://doc.rust-lang.org/std/ops/trait.FnOnce.html
[from_iter]: https://doc.rust-lang.org/core/iter/trait.FromIterator.html#tymethod.from_iter
[From]: https://doc.rust-lang.org/std/convert/trait.From.html
[FromIterator]: https://doc.rust-lang.org/core/iter/trait.FromIterator.html
[Hash]: https://doc.rust-lang.org/std/hash/trait.Hash.html
[HashMap]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[HashSet]: https://doc.rust-lang.org/std/collections/struct.HashMap.html
[index_mut]: https://doc.rust-lang.org/std/ops/trait.IndexMut.html#tymethod.index_mut
[Index]: https://doc.rust-lang.org/std/ops/trait.Index.html
[index]: https://doc.rust-lang.org/std/ops/trait.Index.html#tymethod.index
[IndexMut]: https://doc.rust-lang.org/std/ops/trait.IndexMut.html
[into_iter]: https://doc.rust-lang.org/core/iter/trait.IntoIterator.html#tymethod.into_iter
[Into]: https://doc.rust-lang.org/std/convert/trait.Into.html
[into]: https://doc.rust-lang.org/std/convert/trait.Into.html#tymethod.into
[IntoIterator]: https://doc.rust-lang.org/core/iter/trait.IntoIterator.html
[Iterator]: https://doc.rust-lang.org/core/iter/trait.Iterator.html
[lattice structure]: https://en.wikipedia.org/wiki/Lattice_(order)
[missing_copy_implementations]: https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#missing-copy-implementations
[missing_debug_implementations]: https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#missing-debug-implementations
[mul_assign]: https://doc.rust-lang.org/std/ops/trait.MulAssign.html#tymethod.mul_assign
[Mul]: https://doc.rust-lang.org/std/ops/trait.Mul.html
[mul]: https://doc.rust-lang.org/std/ops/trait.Mul.html#tymethod.mul
[MulAssign]: https://doc.rust-lang.org/std/ops/trait.MulAssign.html
[Mutex]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[MutexGuard]: https://doc.rust-lang.org/std/sync/struct.MutexGuard.html
[Neg]: https://doc.rust-lang.org/std/ops/trait.Neg.html
[next_back]: https://doc.rust-lang.org/core/iter/trait.DoubleEndedIterator.html#tymethod.next_back
[next]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#tymethod.next
[Not]: https://doc.rust-lang.org/std/ops/trait.Not.html
[not]: https://doc.rust-lang.org/std/ops/trait.Not.html#tymethod.not
[Ord]: https://doc.rust-lang.org/std/cmp/trait.Ord.html
[partial equivalence relation]: https://en.wikipedia.org/wiki/Partial_equivalence_relation
[partial_cmp]: https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html#tymethod.partial_cmp
[PartialEq]: https://doc.rust-lang.org/std/cmp/trait.PartialEq.html
[PartialOrd]: https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html
[plain old data]: https://en.wikipedia.org/wiki/Passive_data_structure
[Pointer]: https://doc.rust-lang.org/std/fmt/trait.Pointer.html
[rem_assign]: https://doc.rust-lang.org/std/ops/trait.RemAssign.html#tymethod.rem_assign
[Rem]: https://doc.rust-lang.org/std/ops/trait.Rem.html
[rem]: https://doc.rust-lang.org/std/ops/trait.Rem.html#tymethod.rem
[RemAssign]: https://doc.rust-lang.org/std/ops/trait.RemAssign.html
[Send]: https://doc.rust-lang.org/std/marker/trait.Send.html
[shl_assign]: https://doc.rust-lang.org/std/ops/trait.ShlAssign.html#tymethod.shl_assign
[Shl]: https://doc.rust-lang.org/std/ops/trait.Shl.html
[shl]: https://doc.rust-lang.org/std/ops/trait.Shl.html#tymethod.shl
[ShlAssign]: https://doc.rust-lang.org/std/ops/trait.ShlAssign.html
[shr_assign]: https://doc.rust-lang.org/std/ops/trait.ShrAssign.html#tymethod.shr_assign
[Shr]: https://doc.rust-lang.org/std/ops/trait.Shr.html
[shr]: https://doc.rust-lang.org/std/ops/trait.Shr.html#tymethod.shr
[ShrAssign]: https://doc.rust-lang.org/std/ops/trait.ShrAssign.html
[size_hint]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.size_hint
[Sized]: https://doc.rust-lang.org/std/marker/trait.Sized.html
[source]: https://doc.rust-lang.org/std/error/trait.Error.html#method.source
[std::ops]: https://doc.rust-lang.org/std/ops/index.html
[struct update syntax]: https://doc.rust-lang.org/reference/expressions/struct-expr.html#functional-update-syntax
[sub_assign]: https://doc.rust-lang.org/std/ops/trait.SubAssign.html#tymethod.sub_assign
[Sub]: https://doc.rust-lang.org/std/ops/trait.Sub.html
[SubAssign]: https://doc.rust-lang.org/std/ops/trait.SubAssign.html
[Sync]: https://doc.rust-lang.org/std/marker/trait.Sync.html
[to_owned]: https://doc.rust-lang.org/std/borrow/trait.ToOwned.html#tymethod.to_owned
[ToOwned]: https://doc.rust-lang.org/std/borrow/trait.ToOwned.html
[try_from]: https://doc.rust-lang.org/std/convert/trait.TryFrom.html#tymethod.try_from
[try_into]: https://doc.rust-lang.org/std/convert/trait.TryInto.html#tymethod.try_into
[TryFrom]: https://doc.rust-lang.org/std/convert/trait.TryFrom.html
[TryInto]: https://doc.rust-lang.org/std/convert/trait.TryInto.html
