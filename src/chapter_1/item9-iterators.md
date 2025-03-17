# 第 9 条：考虑使用迭代器转换代替显式循环

编程语言中简陋朴素的循环经历了一段漫长的发展，逐步提高了使用的便利性和抽象性。[B 语言]（C 语言的前身）当时仅有 `while (condition) { ... }` 这种结构，但随着 C 语言的到来，`for` 循环的加入使通过数组下标进行遍历这种高频行为变得越来越方便：

```c
// C code
int i;
for (i = 0; i < len; i++) {
  Item item = collection[i];
  // body
}
```

C++ 的早期版本通过允许将循环变量的声明嵌入到 `for` 语句中，进一步提高了便利性（这也被 C 语言的 C99 标准所纳入）：

```c++
// C++98 code
for (int i = 0; i < len; i++) {
  Item item = collection[i];
  // ...
}
```

大多数现代编程语言把循环的概念做了更进一步的抽象：循环的核心功能通常是移动到某个容器的下一项进行访问。维护访问容器元素所需要的逻辑（`index++` or `++it`）通常是无关紧要的细节。基于这种认识，产生了两个核心的概念：

- 迭代器（`Iterators`）：一种类型，它存在的目的是重复地释出容器的下一个元素，直到把容器耗尽 [^1]。
- `For-each` 循环：一种紧凑的循环表达式，用于迭代容器中的所有元素，表达式会将容器元素绑定到循环变量上，而不是提供一些让你访问到元素的细节信息。

这些概念允许循环代码变得更简短，（更重要的是）更加清晰地表明意图：

```c++
// C++11 code
for (Item& item : collection) {
  // ...
}
```

一旦有了这些概念，它们的强大显而易见，因此它们很快地就被移植到了那些本没有这些概念的语言中（例如，`for-each` 循环在 [Java 1.5] 和 C++11 中被引入）。

Rust 包括迭代器和 `for-each` 风格的循环，但它还包括了更进一步的抽象：允许整个循环体通过迭代器转换（`iterator transform`，有时候也被称作迭代器适配器 `iterator adaptor`）来表达。如我们在[第 3 条]中讨论的 `Option` 和 `Reuslt` 一样，这条条款会尝试展示如何使用这些迭代器转换来替换显式的循环，并就应该何时使用给出相应的指导。特别要注意的是，迭代器转换或许会比显式的循环更高效，因为编译器可以跳过那些潜在的边界检查。

在看完这条条款后，一个 C 风格的，用于求向量 `vector` 前五个偶数项的平方和的循环：

```rust
let values: Vec<u64> = vec![1, 1, 2, 3, 5 /* ... */];

let mut even_sum_squares = 0;
let mut even_count = 0;
for i in 0..values.len() {
    if values[i] % 2 != 0 {
        continue;
    }
    even_sum_squares += values[i] * values[i];
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

应该能被很自然地表达为函数式表达式的样子：

```rust
let even_sum_squares: u64 = values
    .iter()
    .filter(|x| *x % 2 == 0)
    .take(5)
    .map(|x| x * x)
    .sum();
```

像这样的迭代器转换表达式可以粗略被分解为三个部分：

- 一个源迭代器，来自于一个实现了 Rust 的任意一种 `iterator` trait 的类型实例；
- 一个由若干迭代器转换组成的操作序列；
- 一个最终的消费者方法（`consumer method`）将迭代的结果合并为最后的值。

其中前两部分有效地将循环的内部功能从循环体内移到 `for` 表达式中，最后一部分则完全消除了对 `for` 语句的依赖。

> 译者注：这里的原文表达的意思有点绕。不熟悉这种编程范式的朋友可以尝试这么理解：`for` 循环版本代码中的逻辑，通过 `filter().take().map()` 这一系列调用“表达”了。而 `iter()` 和 `sum()` 相当于是这个循环的“启动”和“结束”，对应 `for` 循环版本中的 `for` 和 `even_sum_squares += ...`。于是我们最终摆脱了对 `for` 这个表达式的使用。

## 迭代器 Trait

标准库中这个核心的 [Iterator] trait 有一个十分简单的接口：一个 [next] 方法用于生成 `Some` 元素，直到没法再继续生成（返回 `None`）。迭代过程中生成元素的类型通过 trait 的关联类型 `Item` 定义。

那些允许用户对其中元素进行遍历的容器 —— 在其他语言中被称为可迭代对象  `iterable`）—— 实现了 [IntoIterator] trait；trait 中定义的 [into_iter] 方法会消耗掉 `Self` 并返回一个 `Iterator`。编译器会自动对以下形式的表达式自动应用这个 trait：

```rust
for item in collection {
    // body
}
```

并高效地转换成类似如下的代码：

```rust
let mut iter = collection.into_iter();
loop {
    let item: Thing = match iter.next() {
        Some(item) => item,
        None => break,
    };
    // body
}
```

或者更简洁和惯用的形式：

```rust
let mut iter = collection.into_iter();
while let Some(item) = iter.next() {
    // body
}
```

为了让不同场景下的使用更流畅，对于任何的 `Iterator` 都有实现 `IntoIterator`，而实现就是返回 `self`；毕竟要把 `Iterator` 转成 `Iterator` 不费吹灰之力！

这种初始形式生成的是一个消耗型迭代器，在创建的时候会把容器消耗掉。

```rust
let collection = vec![Thing(0), Thing(1), Thing(2), Thing(3)];
for item in collection {
    println!("Consumed item {item:?}");
}
```

任何在迭代后完成后对容器进行的操作都会失败：

```rust
println!("Collection = {collection:?}");
```

```shell
error[E0382]: borrow of moved value: `collection`
   --> src/main.rs:171:28
    |
163 |   let collection = vec![Thing(0), Thing(1), Thing(2), Thing(3)];
    |       ---------- move occurs because `collection` has type `Vec<Thing>`,
    |                  which does not implement the `Copy` trait
164 |   for item in collection {
    |               ---------- `collection` moved due to this implicit call to
    |                           `.into_iter()`
...
171 |   println!("Collection = {collection:?}");
    |                          ^^^^^^^^^^^^^^ value borrowed here after move
    |
note: `into_iter` takes ownership of the receiver `self`, which moves
      `collection`
```

虽然容易理解，但这种消耗整个容器的行为通常不是我们想要的；我们需要对被迭代的元素进行某种*借用*。

为了确保展示内容的清晰，这里的例子使用了一个*没有*实现 `Copy`（[第 10 条]） 的 `Thing` 类型，因为 `Copy` 会掩盖掉所有权（[第 15 条]）的问题 —— 编译器会偷偷四处拷贝：

```rust
// 特地不实现 `Copy`
#[derive(Clone, Debug, Eq, PartialEq)]
struct Thing(u64);

let collection = vec![Thing(0), Thing(1), Thing(2), Thing(3)];
```

如果被迭代的集合以 `&` 作为前缀：

```rust
for item in &collection {
    println!("{}", item.0);
}
println!("collection still around {collection:?}");
```

那么编译器会寻找 `&Collection` 类型的 [IntoIterator] 实现。正确设计的集合都会提供这样的一个实现；这个实现仍然会消耗 `Self`，不过此时 `Self` 是 `&Collection` 类型而不是 `Collection`，并且对应的关联类型 `Item` 将会是一个引用类型 `&Thing`。

这使得在迭代之后容器仍然保持完整，等效的扩展代码如下：

```rust
let mut iter = (&collection).into_iter();
while let Some(item) = iter.next() {
    println!("{}", item.0);
}
```

在可以遍历可变引用的场景下 [^2]，`for item in &mut collection` 也有类似的模式：编译器寻找 `&mut Collection` 的 `IntoIterator` trait，此时关联类型 `Item` 是 `&mut Thing` 类型。

按照惯例，标准库容器会提供一个 `iter()` 方法返回对底层元素的引用，以及如果可以的话，一个等效的 `iter_mut()` 方法，其行为与上面提到的相同。这些方法可以在 `for` 循环中使用，但在用作迭代器转换的场景下有更明显的好处：

```rust
let result: u64 = (&collection).into_iter().map(|thing| thing.0).sum();
```

可以变成：

```rust
let result: u64 = collection.iter().map(|thing| thing.0).sum();
```

## 迭代器转换

[Iterator] trait 只有一个必须的 [next] 方法，但也提供了大量的在迭代器上执行转换计算的默认方法实现（[第 13 条]）。

其中一些转换会影响到整个迭代的过程：

- [take(n)]：限制迭代器最多只能产生 `n` 个元素。
- [skip(n)]：跳过迭代器的前 `n` 个元素。
- [step_by(n)]：转换迭代器，让它每隔 `n` 个元素生成一个元素。
- [chain(other)]：将两个迭代器粘合在一起构造一个组合迭代器，它会在遍历完第一个迭代器的内容后开始遍历第二个迭代器。
- [cycle()]：将迭代器转换为一个永久循环的迭代器，当遍历到头后再次从头开始遍历。（迭代器需要实现 `Clone` 来支持这个方法。）
- [rev()]：反转迭代器的方向。（迭代器需要实现 [DoubleEndedIterator] trait，这个 trait 有一个额外的 [next_back] 方法。）

其他的转换会影响到 `Iterator` 对应的 `Item` 的性质/属性：

- [map(|item| {...})]：重复应用闭包依次转换迭代的元素。这是最通用的转换，这个列表中的以下若干个方法都可以用 `map` 等价地实现。
- [clone()]：产生原始迭代器中元素的一个克隆；这个方法在遍历 `&Item` 这种引用的时候十分有用。（显然这需要底层类型 `Item` 实现 `Clone`。）
- [copied()]：产生原始迭代器中元素的一个拷贝；这个方法在遍历 `&Item` 这种引用的时候十分有用。（显然这需要底层类型 `Item` 实现 `Copy`，如果是这样的话，那么有可能会比 `cloned()` 要快一些。）
- [enumerate()]：将迭代器转换成迭代 `(usize, Item)` 值对的迭代器，提供了迭代器中元素的索引。
- [zip(it)]：将一个迭代器和另一个迭代器联结，构建一个组合迭代器，用于产生值对，每一个值对里面的元素分别来自于两个迭代器，组合迭代器会一直产生元素直到元素较少的迭代器迭代完毕。

还有一些转换可以对 `Iterator` 产生的 `Item` 进行过滤：
- [filter(|item| {...})]：对每个元素的引用应用一个返回布尔值的闭包，来判断这个元素是否应该要被迭代器提供。
- [take_while()]：基于谓词提供迭代器初始区间中的元素。是 `skip_while` 的镜像。
- [skip_while()]：基于谓词提供迭代器末端区间中的元素。是 `take_while` 的镜像。

[flatten] 方法用于处理元素类型还是迭代器类型的迭代器，用于展平结果。单就这个方法来看这好像没有什么用，但是我们发现当 [Option] 和 [Result] 类型用作迭代器的时候，这就很有用了：这两个类型会产生零（`None`，`Err(e)`）或者一（`Some(v)`，`Ok(v)`）。这意味着 `flatten` 一个 `Option` 或者 `Result` 的流是一个提取其中有效值的简单方式。

从整体上看，上面提到的方法允许对迭代器进行转换，以便迭代器精确地生成大多数情况下所需要的元素序列。

## 迭代器消费者

前面两节介绍了如何获取迭代器，以及如何对其进行合适的转换来进行精准的迭代。这种目的明确的迭代也可以通过显式循环的方式来实现：

```rust
let mut even_sum_squares = 0;
for value in values.iter().filter(|x| *x % 2 == 0).take(5) {
    even_sum_squares += value * value;
}
```

但同时 [Iterator] 提供的方法集里面还包含了许多可以消费掉整个迭代以获得结果的方法，从而可以让我们消除显式的 `for` 循环。

这些方法中最常见的是 [for_each(|item| {...})]，它会对 `Iterator` 产生的每个元素应用一个闭包。这可以完成*绝大多数*显式 `for` 循环可以完成的工作（除了少量例外，我们会在后面的部分中提到）。但它的普适性也让它用起来有点尴尬 —— 闭包需要捕获对外部状态的可变引用才能“返回”结果：

```rust
let mut even_sum_squares = 0;
values
    .iter()
    .filter(|x| *x % 2 == 0)
    .take(5)
    .for_each(|value| {
        // closure needs a mutable reference to state elsewhere
        even_sum_squares += value * value;
    });
```

但是如果 `for` 循环的循环体跟一些常见的模式之一匹配，那么就有更特化的方法来“消费”迭代器，这些方法往往更清晰、简短且符合惯用法。

这些模式包括从一个集合中生成一个值的便捷方法：

- [sum()]：对数值（整型或浮点型）类型的集合求和。
- [product()]：将数值类型集合中的元素相乘。
- [min()]：使用 `Item` 的 `Ord` 实现，寻找集合中的最小值。
- [max()]：使用 `Item` 的 `Ord` 实现，寻找集合中的最大值。
- [min_by(f)]：使用用户提供的比较函数 `f`，寻找集合中的最小值。
- [max_by(f)]：使用用户提供的比较函数 `f`，寻找集合中的最大值。
- [reduce(f)]：通过在每次迭代中执行闭包来计算 `Item` 的求和值，闭包会接收截止目前的求和值和当前遍历的元素作为参数。这是一个更通用的操作，可以用于实现前面提到的一些方法。
- [fold(f)]：通过在每次迭代中执行闭包来计算任意类型（而不限于 `Iterator::Item` 类型）的求和值，闭包会接收截止目前的求和值和当前遍历的元素作为参数。这是 `reduce` 的更泛化的版本。
- [scan(init, f)]：通过在每次迭代中执行闭包来计算某个特定类型的求和值，闭包会接收某种内部状态的可变引用和当前遍历的元素作为参数。这是一个稍特别的 `reduce` 的泛化版本。

还有一些方法可以用从集合中*选择*一个值：

- [find(p)]：查找第一个满足谓词的元素。
- [position(p)]：也是查找第一个满足谓词的元素，不过返回元素对应的索引。
- [nth(n)]：如果有，返回迭代的第 `n` 的元素。

还有一些方法可以针对集合中的每个元素进行测试：

- [any(p)]：返回谓词是否对集合中的*任一*元素成立。
- [all(p)]：返回谓词是否对集合中的*所有*元素成立。

对于上面两个方法之一，如果遍历过程中找到一个反例，迭代都会提前终止。

有一些方法允许闭包在对元素操作的时候返回失败。在这种场景下，如果闭包对某个元素的操作返回失败，迭代将终止，并返回第一个导致操作失败的错误：

- [try_for_each(f)]：行为类似于 `for_each`，但闭包操作可能会失败。
- [try_fold(f)]：行为类似于 `fold`，但闭包操作可能会返回失败。
- [try_find(f)]：行为类似于 `find`，但闭包操作可能会返回失败。

最后，还有一些方法可以把所有迭代的元素累积到新的集合中。当中最重要的就是 [collect()] 方法，它可以用于创建任意一种实现了 [FromIterator] trait 的集合类型。

`FromIterator` trait 在所有的标准库集合类型上都有实现（[Vec]、[HashMap] 和 [BTreeSet] 等），但这种普遍性也意味着你总是要显式地指定集合的类型，否则编译器无法推断出你想要组装一个（比如说）`Vec<i32>` 还是 `HashSet<i32>`：

```rust
use std::collections::HashSet;

// 创建一个只有偶数的集合。你必须指定集合的类型，因为对于两个集合来说构造的表达式一模一样。
let myvec: Vec<i32> = (0..10).into_iter().filter(|x| x % 2 == 0).collect();
let h: HashSet<i32> = (0..10).into_iter().filter(|x| x % 2 == 0).collect();
```

这个例子也展示了如何使用[范围表达式]来生成要迭代的初始数据。

还有一些其他（更加晦涩）的集合生成方法：

- [unzip()]：将一个 pair 的迭代器拆分到两个集合中。
- [partition(p)]：通过应用谓词到元素上，把一个迭代器迭代的内容切分到两个集合中。

本条款涉及了很多 `Iterator` 相关的方法，但这仅仅是所有可用方法的子集；要想了解更多，可以参考文档 [iterator documentation] 或者 *Programming Rust* 第二版（[O'Reilly] [^3]）的 15 章，书里更详细地介绍了各种可能的用法。

丰富的迭代器转换方法集合在日常中就这么触手可及，这些方法可以让代码变得更符合惯用法、更紧凑，同时更好地表达代码的意图。

将循环表达式转化成迭代器转换还有助于生成更高效的代码。为了安全起见，Rust 在访问诸如 `vector` 和切片这种连续的容器时会对访问执行*边界检查*；任何尝试访问越界的元素的操作都会导致 `panic` 而不是访问无效的数据。传统的访问容器值的方法（如 `values[i]`）*可能*会受到这些运行时检查的介入，而一个逐步提供值的迭代器可以被认为是不会越界的。

但是，与迭代器转换等效的传统循环表达式也可能*不会*受到额外的边界检查的影响。Rust 的编译器和优化器很擅长分析切片访问的上下文代码来决定跳过边界检查是否安全可行的；Sergey "Shnatsel" Davidoff 的文章 [2023 article] 探讨了其中的细节。

## 从 Result 值构建集合

上一节我们介绍了如何用 `collect()` 从迭代器构建集合，但同时 `collect()` 对于处理 `Result` 值的场景也有特别有用的特性。

考虑一个例子，将一个 `u64` 的 vector 转成字节 `u8`，并期望它们都满足条件：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
// In the 2021 edition of Rust, `TryFrom` is in the prelude, so this
// `use` statement is no longer needed.
use std::convert::TryFrom;

let inputs: Vec<i64> = vec![0, 1, 2, 3, 4];
let result: Vec<u8> = inputs
    .into_iter()
    .map(|v| <u8>::try_from(v).unwrap())
    .collect();
```

这是可以的，直到有一些意外的输入：

```rust
let inputs: Vec<i64> = vec![0, 1, 2, 3, 4, 512];
```

这将导致运行时的失败：

```shell
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value:
TryFromIntError(())', iterators/src/main.rs:266:36
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

根据[第 3 条]的建议，我们希望保留 `Result` 类型并且使用 `?` 运算符让任何一个失败的操作作为调用代码的问题返回。修改让代码返回 `Result` 达不到我们的目的：

```rust
let result: Vec<Result<u8, _>> =
    inputs.into_iter().map(|v| <u8>::try_from(v)).collect();
// Now what?  Still need to iterate to extract results and detect errors.
```

但是还有另一种 `collect()` 的版本，可以组装一个持有 `Vec` 的 `Result`，而不是一个装着 `Result` 的 `Vec`。

想要强制使用这个版本就要用到 turbofish 语法（`::<Result<Vec<_>, _>>`）：

```rust
let result: Vec<u8> = inputs
    .into_iter()
    .map(|v| <u8>::try_from(v))
    .collect::<Result<Vec<_>, _>>()?;
```

将其与问号运算符结合使用实现了一种有用的行为：

- 如果迭代遇到了错误，错误的值会返回到调用方，并且迭代终止。
- 如果没有遇到错误，则余下代码能处理返回一个合理、类型正确的值的集合。

## 转化显式循环

这个条款的目的是让你相信很多显式循环都可以被转化成迭代器转换。这对于不习惯的程序员来说可能会让人觉得不太自然，所以我们来一步步完成一次转换吧。

我们从一个很 C 语言风格的循环开始，对一个 `vector` 的头五个偶数项求平方和：

```c
let mut even_sum_squares = 0;
let mut even_count = 0;
for i in 0..values.len() {
    if values[i] % 2 != 0 {
        continue;
    }
    even_sum_squares += values[i] * values[i];
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

第一步是把 `vector` 的索引换成 `for-each` 循环的迭代器：

```rust
let mut even_sum_squares = 0;
let mut even_count = 0;
for value in values.iter() {
    if value % 2 != 0 {
        continue;
    }
    even_sum_squares += value * value;
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

使用 `continue` 来跳过某些元素的原始代码分支可以用 `filter()` 来很自然地表达：

```rust
let mut even_sum_squares = 0;
let mut even_count = 0;
for value in values.iter().filter(|x| *x % 2 == 0) {
    even_sum_squares += value * value;
    even_count += 1;
    if even_count == 5 {
        break;
    }
}
```

接下来，一旦我们有 5 个元素了，我们就提前退出循环，这里用 `take(5)`：

```rust
let mut even_sum_squares = 0;
for value in values.iter().filter(|x| *x % 2 == 0).take(5) {
    even_sum_squares += value * value;
}
```

每次迭代我们都只要元素的平方值 `value * value`，这就是 `map()` 的理想使用场景：

```rust
let mut even_sum_squares = 0;
for val_sqr in values.iter().filter(|x| *x % 2 == 0).take(5).map(|x| x * x)
{
    even_sum_squares += val_sqr;
}
```

对原始循环的重构最后成为了 `sum()` 方法大锤下一颗完美的钉子：

```rust
let even_sum_squares: u64 = values
    .iter()
    .filter(|x| *x % 2 == 0)
    .take(5)
    .map(|x| x * x)
    .sum();
```

## 什么时候显式循环更好

这条条款强调了迭代器转换的优势，特别是在简洁性和清晰度方面。那么什么时候迭代器转换*不是*那么适合或者不符合惯用法呢？

- 如果循环体很大、或者功能很多，那么保留在一个显式的循环体里面会比把逻辑压缩到闭包中更合理。
- 如果循环体包含很多会导致功能提前终止的错误条件，最好还是把它们保留在显式的循环体中 —— `try...()` 之类的方法也不会帮上很多忙。但是，`collect()` 能把一个值类型为 `Result` 的集合转换成一个持有集合类型的 `Result` 类型的能力，在配合 `?` 运算符的场景下还是可以进行错误条件的处理。
- 如果性能至关重要，包含闭包的迭代器转换*理应*跟显式的代码[一样快]。但如果代码中一个核心的循环很重要，*测量*不同的实现方法并进行适当的调优。

  - 请确保你的测试能反映实际的性能 —— 编译器的优化可能会对测试数据给出过于乐观的结果（如[第 30 条]所述）。
  - [Godbolt compiler explorer] 是一个了不得的工具，你可以看到编译器都生成了什么。

最重要的，如果转换的过程是强行的或者生搬硬套的，那么就不要把显式循环转成迭代器转换了。这不过是一种编程风格口味的取向而已 —— 但注意到，随着你对函数式的风格越来越熟悉，你的口味也很可能会发生变化。


## 注释

[^1]: 事实上，迭代器可以更通用 —— 在直到结束之前不停地产生下一个元素，这种想法不必跟某种容器强关联。

[^2]: 如果对容器元素的修改可能会导致容器内部的一些约束被打破，那么这个方法就不能提供了。比如说：导致元素的 [Hash] 值发生变化的修改，就可能会导致 `HashMap` 内部数据结构的失效。

[^3]: 国内的图灵社区已引进翻译并上市：[《Rust 程序设计（第 2 版）》](https://www.ituring.com.cn/book/2846)。

原文[点这里](https://www.lurklurk.org/effective-rust/iterators.html)查看

<!-- 参考链接 -->

[第 3 条]: item3-transform.md
[第 10 条]: ../chapter_2/item10-std-traits.md
[第 13 条]: ../chapter_2/item13-use-default-impl.md
[第 15 条]: ../chapter_3/item15-borrows.md
[第 30 条]: ../chapter_5/item30-write-more-than-unit-tests.md

[B 语言]: https://web.archive.org/web/20150611114427/https://www.bell-labs.com/usr/dmr/www/kbman.pdf
[Java 1.5]: https://docs.oracle.com/javase/1.5.0/docs/guide/language/foreach.html
[Iterator]: https://doc.rust-lang.org/core/iter/trait.Iterator.html
[next]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#tymethod.next
[IntoIterator]: https://doc.rust-lang.org/core/iter/trait.IntoIterator.html
[into_iter]: https://doc.rust-lang.org/core/iter/trait.IntoIterator.html#tymethod.into_iter
[take(n)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.take
[skip(n)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.skip
[step_by(n)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.step_by
[chain(other)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain
[cycle()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.cycle
[rev()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.rev
[DoubleEndedIterator]: https://doc.rust-lang.org/core/iter/trait.DoubleEndedIterator.html
[next_back]: https://doc.rust-lang.org/core/iter/trait.DoubleEndedIterator.html#tymethod.next_back
[map(|item| {...})]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map
[clone()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.cloned
[copied()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.copied
[enumerate()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.enumerate
[zip(it)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip
[filter(|item| {...})]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter
[take_while()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.take_while
[skip_while()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.skip_while
[flatten]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flatten
[Option]: https://doc.rust-lang.org/std/option/enum.Option.html#method.iter
[Result]: https://doc.rust-lang.org/std/result/enum.Result.html#method.iter
[for_each(|item| {...})]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.for_each
[sum()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.sum
[product()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.product
[min()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.min
[max()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max
[min_by(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.min_by
[max_by(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max_by
[reduce(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.reduce
[fold(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold
[scan(init, f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.scan
[find(p)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.find
[position(p)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.position
[nth(n)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.nth
[any(p)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.any
[all(p)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.all
[try_for_each(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_for_each
[try_fold(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_fold
[try_find(f)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.try_find
[collect()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect
[FromIterator]: https://doc.rust-lang.org/core/iter/trait.FromIterator.html
[Vec]: https://doc.rust-lang.org/std/vec/struct.Vec.html#impl-FromIterator%3CT%3E
[HashMap]: https://doc.rust-lang.org/std/collections/struct.HashMap.html#impl-FromIterator%3C(K%2C%20V)%3E
[BTreeSet]: https://doc.rust-lang.org/std/collections/struct.BTreeSet.html#impl-FromIterator%3CT%3E
[范围表达式]: https://doc.rust-lang.org/reference/expressions/range-expr.html
[unzip()]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.unzip
[partition(p)]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.partition
[iterator documentation]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[O'Reilly]: https://learning.oreilly.com/library/view/programming-rust-2nd/9781492052586/ch15.html
[2023 article]: https://shnatsel.medium.com/how-to-avoid-bounds-checks-in-rust-without-unsafe-f65e618b4c1e
[一样快]: https://doc.rust-lang.org/book/ch13-04-performance.html
[Godbolt compiler explorer]: https://rust.godbolt.org/
[Hash]: https://doc.rust-lang.org/std/hash/trait.Hash.html
