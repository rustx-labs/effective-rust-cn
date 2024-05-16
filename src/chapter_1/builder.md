# 第 7 条：对于复杂的类型，使用构造器

这条款项描述了构造器模式：对于复杂的数据类型提供对应的构造器类型 `builder type`，使得用户可以方便地创造该数据数据类型的实例。

Rust 要求开发者在创建一个新的 `struct` 实例的时候，必须填入 `struct` 的所有字段。这样可以保证结构体中永远不会存在未初始化的值，从而保证了代码的安全，然而这会比理想的情况下产生更多的冗余的代码片段。

例如，任何可选的字段都必须显式地使用 `None` 来标记为缺失：

```rust
/// Phone number in E164 format.
#[derive(Debug, Clone)]
pub struct PhoneNumberE164(pub String);

#[derive(Debug, Default)]
pub struct Details {
    pub given_name: String,
    pub preferred_name: Option<String>,
    pub middle_name: Option<String>,
    pub family_name: String,
    pub mobile_phone: Option<PhoneNumberE164>,
}

// ...

let dizzy = Details {
    given_name: "Dizzy".to_owned(),
    preferred_name: None,
    middle_name: None,
    family_name: "Mixer".to_owned(),
    mobile_phone: None,
};
```

这样的样板式代码也很脆弱，因为将来要向 `struct` 中添加一个新字段的时候需要更改所有创建这个结构体的地方。

通过使用和实现 [Default] trait 可以显著地减少这种样板代码，如[第 10 条]中所述：

```rust
let dizzy = Details {
    given_name: "Dizzy".to_owned(),
    family_name: "Mixer".to_owned(),
    ..Default::default()
};
```

使用 `Default` 还有助于减少结构体新增字段时候导致的修改，前提是新的字段本身的类型也实现了 `Default`。

还有一个更普遍的问题：仅当所有的字段类型都实现了 `Default` trait 的时候，结构体才能使用自动派生的 `Default` 实现。如果有任何一个字段不满足，那么 `derive` 就会失败了：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
#[derive(Debug, Default)]
pub struct Details {
    pub given_name: String,
    pub preferred_name: Option<String>,
    pub middle_name: Option<String>,
    pub family_name: String,
    pub mobile_phone: Option<PhoneNumberE164>,
    pub date_of_birth: time::Date,
    pub last_seen: Option<time::OffsetDateTime>,
}
```

```shell
error[E0277]: the trait bound `Date: Default` is not satisfied
  --> src/main.rs:48:9
   |
41 |     #[derive(Debug, Default)]
   |                     ------- in this derive macro expansion
...
48 |         pub date_of_birth: time::Date,
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Default` is not
   |                                       implemented for `Date`
   |
   = note: this error originates in the derive macro `Default`
```

由于孤儿规则的存在，代码没办法为 `chrono::Utc` 实现 `Default`；但就算可以，也无济于事 —— 给出生日期赋一个值默认值几乎总是一个错误的选择。

缺少 `Default` 意味着所有字段都必须手动填写：

```rust
let bob = Details {
    given_name: "Robert".to_owned(),
    preferred_name: Some("Bob".to_owned()),
    middle_name: Some("the".to_owned()),
    family_name: "Builder".to_owned(),
    mobile_phone: None,
    date_of_birth: time::Date::from_calendar_date(
        1998,
        time::Month::November,
        28,
    )
    .unwrap(),
    last_seen: None,
};
```

如果你**为复杂的数据结构实现了构造器模式**，那么就可以提高这里的效率和体验。

构造器模式最简单的一种实现方式就是用一个额外的 `struct` 来保存构造原始复杂数据类型所需的数据。简单起见，这里的实例会直接保存一个该类型的实例：

```rust
pub struct DetailsBuilder(Details);

impl DetailsBuilder {
    /// Start building a new [`Details`] object.
    /// 开始构造一个新的 [`Details`] 对象
    pub fn new(
        given_name: &str,
        family_name: &str,
        date_of_birth: time::Date,
    ) -> Self {
        DetailsBuilder(Details {
            given_name: given_name.to_owned(),
            preferred_name: None,
            middle_name: None,
            family_name: family_name.to_owned(),
            mobile_phone: None,
            date_of_birth,
            last_seen: None,
        })
    }
}
```

随后，我们可以给构造器类型增添辅助函数来填充新的字段。每一个这种函数都会消费 `self` 同时产生一个新的 `Self`，以允许对不同的构造方法进行链式调用。

这些辅助函数会比简单的 `setter` 函数有用多了：

```rust
/// Update the `last_seen` field to the current date/time.
/// 把 `last_seen` 字段更新成当前日期/时间
pub fn just_seen(mut self) -> Self {
    self.0.last_seen = Some(time::OffsetDateTime::now_utc());
    self
}
```

构造器被调用的最后一个函数会消费它自身并输出所构造的对象：

```rust
/// Consume the builder object and return a fully built [`Details`]
/// object.
/// 消费构造器对象并返回最后创建的 [`Details`] 对象
pub fn build(self) -> Details {
    self.0
}
```

总而言之，这让构造器的使用者拥有了更符合工程学的体验：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let also_bob = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
)
.middle_name("the")
.preferred_name("Bob")
.just_seen()
.build();
```

构造器“消费自己”的性质也导致了一些问题。首先，对象的构造过程不能独立完成：

```rust
let builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
if informal {
    builder.preferred_name("Bob");
}
let bob = builder.build();
```

```shell
error[E0382]: use of moved value: `builder`
   --> src/main.rs:256:15
    |
247 |     let builder = DetailsBuilder::new(
    |         ------- move occurs because `builder` has type `DetailsBuilder`,
    |                 which does not implement the `Copy` trait
...
254 |         builder.preferred_name("Bob");
    |                 --------------------- `builder` moved due to this method
    |                                       call
255 |     }
256 |     let bob = builder.build();
    |               ^^^^^^^ value used here after move
    |
note: `DetailsBuilder::preferred_name` takes ownership of the receiver `self`,
      which moves `builder`
   --> src/main.rs:60:35
    |
27  |     pub fn preferred_name(mut self, preferred_name: &str) -> Self {
    |                               ^^^^
```

这个问题可以通过把被消费的构造器重新赋值给同一个变量来解决：

```rust
let mut builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
if informal {
    builder = builder.preferred_name("Bob");
}
let bob = builder.build();
```

构造器的性质带来的另一个问题是你只能构造一个最终对象，对同一个构造器重复调用 `build()` 函数来创建多个实例会违反编译器的检查规则，如同你能想到的那样：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let smithy = DetailsBuilder::new(
    "Agent",
    "Smith",
    time::Date::from_calendar_date(1999, time::Month::June, 11).unwrap(),
);
let clones = vec![smithy.build(), smithy.build(), smithy.build()];
```

```shell
error[E0382]: use of moved value: `smithy`
   --> src/main.rs:159:39
    |
154 |   let smithy = DetailsBuilder::new(
    |       ------ move occurs because `smithy` has type `base::DetailsBuilder`,
    |              which does not implement the `Copy` trait
...
159 |   let clones = vec![smithy.build(), smithy.build(), smithy.build()];
    |                            -------  ^^^^^^ value used here after move
    |                            |
    |                            `smithy` moved due to this method call
```

另一种实现构造器的途径是让构造器的方法接受 `&mut self` 并返回一个 `&mut Self`：

```rust
/// Update the `last_seen` field to the current date/time.
/// 把 `last_seen` 字段更新成当前日期/时间
pub fn just_seen(&mut self) -> &mut Self {
    self.0.last_seen = Some(time::OffsetDateTime::now_utc());
    self
}
```

这可以让代码免于分步构造场景下的自赋值：

```rust
let mut builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
if informal {
    builder.preferred_name("Bob"); // no `builder = ...`
}
let bob = builder.build();
```

然而，这个版本的实现使得构造器的构造方法和它的 `setter` 函数无法被链式调用：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
)
.middle_name("the")
.just_seen();
let bob = builder.build();
```

```shell
error[E0716]: temporary value dropped while borrowed
   --> src/main.rs:265:19
    |
265 |       let builder = DetailsBuilder::new(
    |  ___________________^
266 | |         "Robert",
267 | |         "Builder",
268 | |         time::Date::from_calendar_date(1998, time::Month::November, 28)
269 | |             .unwrap(),
270 | |     )
    | |_____^ creates a temporary value which is freed while still in use
271 |       .middle_name("the")
272 |       .just_seen();
    |                   - temporary value is freed at the end of this statement
273 |       let bob = builder.build();
    |                 --------------- borrow later used here
    |
    = note: consider using a `let` binding to create a longer lived value
```

如同编译器错误所示，你可以通过 `let` 为构造器指定一个名字来解决这个问题：

```rust
let mut builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
builder.middle_name("the").just_seen();
if informal {
    builder.preferred_name("Bob");
}
let bob = builder.build();
```

这种修改自身的构造器实现允许你构造多个最终对象。`build()` 方法的签名*不*需要消费 `self`，因此必须如下所示：

```rust
/// Construct a fully built [`Details`] object.
/// 生成一个构造完毕的 [`Details`] 对象。
pub fn build(&self) -> Details {
    // ...
}
```

这个可重复调用的 `build()` 的实现必须在每次被调用的时候构造一个全新的实例。如果底层类型实现了 `Clone`，这就很简单了 —— 构造器可以持有一个模板然后在每一次 `build()` 的时候执行一次 `clone()`。如果底层类型*没有*实现 `Clone`，那么构造器需要保留足够的状态信息，在每一次 `build()` 的时候手动创建一个实例返回。

不管是哪种构造器模式的实现，样板代码都集中在一个地方 —— 构造器本身 —— 而不是每个需要操作底层类型的地方。

剩下的样板代码或许还可以通过宏（[第 28 条]）进一步减少，但如果你打算在这条路上走下去，你应该看看是否有现成的包（尤其是 [derive_builder]）已经提供了你需要的功能——如果你愿意添加一个依赖的话（[第 25 条]）。


原文[点这里](https://www.lurklurk.org/effective-rust/builders.html)查看

<!-- 参考链接 -->

[第 10 条]:https://www.lurklurk.org/effective-rust/std-traits.html
[第 25 条]:https://www.lurklurk.org/effective-rust/dep-graph.html
[第 28 条]:https://www.lurklurk.org/effective-rust/macros.html

[Default]:https://doc.rust-lang.org/std/default/trait.Default.html
[derive_builder]:https://docs.rs/derive_builder/latest/derive_builder/