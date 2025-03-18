# 第 13 条：使用默认实现来减少必需实现的 trait 方法

在设计 trait 时，需要考虑两种不同的受众：实现 trait 的人和使用 trait 的人。在 trait 设计中，这两种受众之间存在一定的冲突：

- 为了让实现者实现 trait 更轻松，最好让 trait 具有实现其目的所需的最少方法。
- 为了让使用者使用 trait 更方便，最好提供一系列覆盖所有常见用法的方法。

通过涵盖更多方便使用者的方法，同时为那些可以从接口上其他更基础的操作构建的方法提供默认实现，可以缓解这种冲突。

来看一个简单的例子，[`ExactSizeIterator`] 是一个知道确切迭代内容的 `Iterator` [^1]，它的 [`is_empty()`] 方法有一个依赖于 [`len()`] 方法的默认实现：

```rust
fn is_empty(&self) -> bool {
    self.len() == 0
}
```

存在默认的实现仅仅意味着它有一个默认值。如果 trait 的实现有不同的方法来判断迭代器是否为空，它也可以用自己的 `is_empty()` 替换默认实现。

这种方法使得 trait 定义具有少量必需的方法，以及大量默认实现的方法。实现者只需实现前者，即可随意使用所有后者。

Rust 标准库广泛采用了这种方法；[`Iterator`] trait 就是一个很好的例子，它只有一个必需方法（[`next()`]），但包含了大量预提供的方法（[第 9 条]），撰写本文时已经超过 50 个。

Trait 方法可以添加 trait 约束，这意味着只有在相关类型实现特定 trait 时，目标方法才可用。这在结合默认方法实现时非常有用，`Iterator` 也印证了这点。例如，[`cloned()`] 的迭代器方法有一个 trait 约束和一个默认实现：

```rust
fn cloned<'a, T>(self) -> Cloned<Self>
where
    T: 'a + Clone,
    Self: Sized + Iterator<Item = &'a T>,
{
    Cloned::new(self)
}
```

换句话说，`cloned()` 方法只有在 `Item` 的类型实现了 [`Clone`] trait 时才可用；一旦实现 `Clone` trait，`cloned()` 方法也会自动实现。

关于带有默认实现的 trait 方法，最后一个要点是，即使在 trait 的初始版本发布之后，通常也可以安全地向 trait 添加新方法。只要新方法名不与类型实现的其他 trait 方法名冲突，就能保持向后兼容性（详见[第 21 条]）。[^2]

因此，请参照标准库的示例，通过添加带有默认实现的方法（并根据需要添加 trait 约束），为实现者提供最少的 API 接口，但为使用者提供方便且全面的 API。

原文[点这里](https://www.lurklurk.org/effective-rust/default-impl.html)查看

## 注释

[^1]: `is_empty()` 方法当前还是[每日构建里的实验性功能](https://github.com/rust-lang/rust/issues/35428)。

[^2]: 如果新方法恰好与*具体*类型中的方法同名，则具体方法（称为[固有实现](https://doc.rust-lang.org/reference/items/implementations.html#inherent-implementations)）将优先于 trait 方法被使用，但也可以通过类型转换 `<具体类型 as Trait>::method()` 来显式地选择使用 trait 方法。

<!-- 参考链接 -->

[第 9 条]: ../chapter_1/item9-iterators.md
[第 21 条]: ../chapter_4/item21-semver.md

[`is_empty()`]: https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html#method.is_empty
[`ExactSizeIterator`]: https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html
[`len()`]: https://doc.rust-lang.org/std/iter/trait.ExactSizeIterator.html#method.len
[`Iterator`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html
[`next()`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#tymethod.next
[`cloned()`]: https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.cloned
[`Clone`]: https://doc.rust-lang.org/std/clone/trait.Clone.html