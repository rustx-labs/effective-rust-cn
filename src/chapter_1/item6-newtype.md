# 第 6 条：拥抱 newtype 模式

[第 1 条]描述了*元组结构体*，它的字段没有名字，而是通过数字（`self.0`）来引用。本条款着重介绍的是只包含一个类型的元组结构体。它是一个新的类型，可以包含和内置类型一样的值。在 Rust 中，这个模式非常普遍，它叫做：*newtype* 模式。

newtype 模式的最简单用法，是在类型原有行为的基础上，提供[额外的语义]。想象有一个将卫星送往火星的项目。[^1]这是一个大项目，不同的团队已经构建了项目的不同部分。其中一个小组负责火箭引擎的代码：

```rust
/// 点燃推进器。返回产生的脉冲，单位为磅/秒。
pub fn thruster_impulse(direction: Direction) -> f64 {
    // ...
    return 42.0;
}
```

另一个团队负责惯性导航系统：

```rust
/// 根据推力（单位：牛顿/秒）更新轨迹模型。
pub fn update_trajectory(force: f64) {
    // ...
}
```

最终，结合这些不同部分：

```rust
let thruster_force: f64 = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

糟糕（Ruh-roh）。[^2]

Rust 有类型别名（`type alias`）的特性，让不同的团队能够更清楚地表达他们的意图：

```rust
/// 推力的单位。
pub type PoundForceSeconds = f64;

/// 点燃推进器。返回产生的脉冲。
pub fn thruster_impulse(direction: Direction) -> PoundForceSeconds {
    // ...
    return 42.0;
}
```
```rust
/// 推力的单位。
pub type NewtonSeconds = f64;

/// 根据推力更新轨迹模型。
pub fn update_trajectory(force: NewtonSeconds) {
    // ...
}
```

然而，类型别名实际上只是文档：它们比前面的文档注释有更强的提示，但不能阻止 `PoundForceSeconds` 值被使用在希望使用 `NewtonSeconds` 值的地方。

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

再次出现问题了。

这就是 newtype 模式能带来帮助的地方：

```rust
/// 推力的单位。
pub struct PoundForceSeconds(pub f64);

/// 点燃推进器。返回产生的脉冲。
pub fn thruster_impulse(direction: Direction) -> PoundForceSeconds {
    // ...
    return PoundForceSeconds(42.0);
}
/// 推力的单位。
pub struct NewtonSeconds(pub f64);

/// 根据推力更新轨迹模型。
pub fn update_trajectory(force: NewtonSeconds) {
    // ...
}
```

如名称所示，newtype 是一个新类型。因此，当类型不匹配时，编译器会报错。在这里，我们尝试将 `PoundForceSeconds` 值传递给期望使用 `NewtonSeconds` 值的地方：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

```rust
let new_direction = update_trajectory(thruster_force);
error[E0308]: mismatched types
  --> src/main.rs:76:43
   |
76 |     let new_direction = update_trajectory(thruster_force);
   |                         ----------------- ^^^^^^^^^^^^^^ expected
   |                         |        `NewtonSeconds`, found `PoundForceSeconds`
   |                         |
   |                         arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:66:8
   |
66 | pub fn update_trajectory(force: NewtonSeconds) {
   |        ^^^^^^^^^^^^^^^^^ --------------------
help: call `Into::into` on this expression to convert `PoundForceSeconds` into
      `NewtonSeconds`
   |
76 |     let new_direction = update_trajectory(thruster_force.into());
   |                                                         +++++++
```

如在[第 5 条]中所述，添加标准的 `From` trait 的实现：

```rust
impl From<PoundForceSeconds> for NewtonSeconds {
    fn from(val: PoundForceSeconds) -> NewtonSeconds {
        NewtonSeconds(4.448222 * val.0)
    }
}
```

这样就能用 `.into()` 执行单位和类型的转换：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force.into());
```

使用 newtype，除了能附加「单位」语义，还可以使布尔参数更清晰。回顾[第 1 条]的例子，使用 newtype 可以清晰地说明参数的含义：

```rust
struct DoubleSided(pub bool);

struct ColorOutput(pub bool);

fn print_page(sides: DoubleSided, color: ColorOutput) {
    // ...
}
```

```rust
print_page(DoubleSided(true), ColorOutput(false));
```

如果需要考虑大小或二进制兼容性，那么 <code>#[repr(transparent)]</code> 属性能确保 newtype 在内存中的表示与内部类型相同。

这个来自[第 1 条]的例子，是 newtype 的简单用法 —— 将语义编码到类型系统中，以让编译器负责管理这些语义。

## 绕过 trait 的孤儿规则

另一个[常见]但更巧妙的需要 newtype 模式的场景，是 Rust 的孤儿规则。这个规则意味着，在一个包里，以下条件之一满足时，才能为某个类型实现 trait：

• 包定义了该 trait
• 包定义了该类型

我们来尝试为一个外部类型实现一个外部 trait：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
use std::fmt;

impl fmt::Display for rand::rngs::StdRng {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        write!(f, "<StdRng instance>")
    }
}
```

编译器会出错（它指出要使用 newtype）：

```text
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
   --> src/main.rs:146:1
    |
146 | impl fmt::Display for rand::rngs::StdRng {
    | ^^^^^^^^^^^^^^^^^^^^^^------------------
    | |                     |
    | |                     `StdRng` is not defined in the current crate
    | impl doesn't use only types from inside the current crate
    |
    = note: define and implement a trait or new type instead
```

这种限制的原因是可能发生歧义：如果依赖关系图中的两个不同的包（[第 25 条]）都要实现 `impl std::fmt::Display for rand::rngs::StdRng`，那么编译器/链接器不知道选择哪个。

这经常会带来挫败感：例如，如果你试图序列化包含来自其他包的类型的数据，孤儿规则会阻止你写 `impl serde::Serialize for somecrate::SomeType`。[^3]

但是 newtype 模式意味着你定义了一个*新*类型，这是当前包的一部分，所以就满足了孤儿规则的第二点。现在就能够实现一个外部 trait：

```rust
struct MyRng(rand::rngs::StdRng);

impl fmt::Display for MyRng {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        write!(f, "<MyRng instance>")
    }
}
```

## newtype 的限制

newtype 模式解决了两类问题 —— 阻止单位转换和绕过孤儿原则。但它也有一些不足 —— 每个 newtype 的操作都需要转发到内部类型。

这意味着必须在所有地方都使用 `thing.0`，而不是使用 `thing`。不过这很容易做到，而且编译器会告诉你在哪里需要。

比较麻烦的是，内部类型的任何 trait 实现都会丢失：因为 newtype 是一个新类型，所以现有的内部实现都不适用。

对于能派生的 trait，只需要在 newtype 的声明上使用 `derive`：

```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq, Ord, PartialOrd)]
pub struct NewType(InnerType);
```

然而，对于更复杂的 trait，需要一些样板代码来恢复内部类型的实现，例如：

```rust
use std::fmt;
impl fmt::Display for NewType {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        self.0.fmt(f)
    }
}
```

## 注释

[^1]: 具体来说，是火星气候轨道器。

[^2]: 参见维基百科上的“火星气候轨道器”[条目](https://en.wikipedia.org/wiki/Mars_Climate_Orbiter)，了解更多关于失败原因的信息。译者注:这句话 "Ruh-roh" 通常用来表达轻微的麻烦或问题即将出现。它起源于美国动画片《史酷比》(Scooby-Doo)，是其中角色 Shaggy Rogers 的口头禅，用来表达他们遇到了一些麻烦或即将面临挑战。

[^3]: 对于serde来说，这是一个足够常见的问题，因此它包含了一种帮助机制。


原文[点这里](https://www.lurklurk.org/effective-rust/newtype.html)查看

<!-- 参考链接 -->

[第 1 条]: item1-use-types.md
[第 5 条]: item5-casts.md
[第 25 条]: ../chapter_4/item25-dep-graph.md

[常见]: https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#using-the-newtype-pattern-to-implement-external-traits-on-external-types
[额外的语义]: https://doc.rust-lang.org/book/ch19-04-advanced-types.html#using-the-newtype-pattern-for-type-safety-and-abstraction
[repr(transparent)]: https://doc.rust-lang.org/reference/type-layout.html#the-transparent-representation
