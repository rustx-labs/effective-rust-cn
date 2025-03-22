# 第 11 条：为 RAII 模式实现 Drop trait

> “永远不要让人去做机器的工作。” —— 史密斯特工（出自电影《黑客帝国》）

RAII 代表“资源获取即初始化”（Resource Acquisition Is Initialization）是一种编程模式，其中值的生命周期与某些附加资源的生命周期完全相关。 RAII 模式由 C++ 编程语言普及，是 C++ 对编程的最大贡献之一。

值的生命周期与资源的生命周期之间的关联体现在 RAII 类型中：

- 该类型的构造函数获取对某些资源的访问权
- 该类型的析构函数释放对这些资源的访问权

其结果是 RAII 类型具有一个恒定的特性：当且仅当对象存在时，才能访问底层资源。因为编译器确保局部变量在作用域退出时会被销毁，这就意味着底层资源也会在退出作用域时被释放。

这对于程序的可维护性很有帮助：如果对代码的后续改动改变了控制流，对象和资源的生命周期仍然是正确的。为了说明这点，来看一些没有使用 RAII 模式，手动锁定、解锁互斥锁的代码；以下代码是用 C++ 编写的，因为 Rust 的 `Mutex` 不允许这种易出错的用法！

```C++
// C++ code
class ThreadSafeInt {
 public:
  ThreadSafeInt(int v) : value_(v) {}

  void add(int delta) {
    mu_.lock();
    // ... more code here
    value_ += delta;
    // ... more code here
    mu_.unlock();
  }
```

如果修改程序以在错误发生时提前退出函数，将会导致互斥锁保持锁定状态：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```C++
// C++ code
void add_with_modification(int delta) { 
  mu_.lock();
  // ... more code here
  value_ += delta;
  // Check for overflow.
  if (value_ > MAX_INT) {
    // Oops, forgot to unlock() before exit
    return;
  }
  // ... more code here
  mu_.unlock();
}
```

然而，如果我们把锁定、解锁的行为放到 RAII 类中：

```C++
// C++ code (real code should use std::lock_guard or similar)
class MutexLock {
 public:
  MutexLock(Mutex* mu) : mu_(mu) { mu_->lock(); }
  ~MutexLock()                   { mu_->unlock(); }
 private:
  Mutex* mu_;
};
```

对于同样的改动，代码就是安全的：

```C++
// C++ code
void add_with_modification(int delta) {
  MutexLock with_lock(&mu_);
  // ... more code here
  value_ += delta;
  // Check for overflow.
  if (value_ > MAX_INT) {
    return; // Safe, with_lock unlocks on the way out
  }
  // ... more code here
}
```

在 C++ 中，RAII 模式最初常用于内存管理，以确保手动分配（`new`，`malloc()`）和释放（`delete`，`free()`）操作保持同步。C++11 标准库中加入了一个通用版本的内存管理：`std::unique_ptr<T>` 类型确保只有一个指针独享内存的“所有权”，但允许指向该内存的指针被“借用”用于临时使用（`ptr.get()`）。

在 Rust 中，内存指针的这种行为被内置在语言中（[第 15 条]），但 RAII 的一般原则对于其他类型的资源仍然有用。**我们应该对任何持有必须释放资源的类型实现 `Drop` trait**，例如以下情况：

- 访问操作系统资源。对于类 Unix 系统，这通常意味着持有[文件描述符]的类型对象；未能正确释放这些资源将会占用系统资源（并最终导致程序的每个进程获取文件描述符受限）。
- 访问同步资源。标准库已经包括内存同步原语，但其他资源（例如文件锁、数据库锁等）可能需要类似的封装。
- 访问原始内存，对于处理低级内存管理的 `unsafe` 类型（例如，用于外部函数接口 [FFI] 功能）。

Rust 标准库中最明显的 RAII 实例是由 [`Mutex::lock()`] 操作返回的 [`MutexGuard`]，它通常用于[第 17 条]中讨论的通过共享状态实现并行的程序。这大致类似于之前提到的 C++ 示例，但在 Rust 中，`MutexGuard` 不仅作为持有锁的 RAII 对象，还充当对互斥锁保护的数据的代理：

```rust
use std::sync::Mutex;

struct ThreadSafeInt {
    value: Mutex<i32>,
}

impl ThreadSafeInt {
    fn new(val: i32) -> Self {
        Self {
            value: Mutex::new(val),
        }
    }
    fn add(&self, delta: i32) {
        let mut v = self.value.lock().unwrap();
        *v += delta;
    }
}

```

[第 17 条]建议不要在大段代码中持有锁；为确保这点，**可以使用代码块来限制 RAII 对象的作用域**。虽然这样会导致奇怪的缩进，但为了增加安全性和确保生命周期的精确性，这是值得的：

```rust
impl ThreadSafeInt {
    fn add_with_extras(&self, delta: i32) {
        // ... more code here that doesn't need the lock
        {
            let mut v = self.value.lock().unwrap();
            *v += delta;
        }
        // ... more code here that doesn't need the lock
    }
}
```

在推崇了 RAII 模式的用法之后，有必要解释一下如何实现它。[`Drop`][drop_trait] trait 允许你在对象销毁时添加用户自定义的行为。这个 trait 只有一个方法，[`drop`][drop_method]，编译器会在释放持有对象的内存之前运行这个方法：

```rust
#[derive(Debug)]
struct MyStruct(i32);

impl Drop for MyStruct {
    fn drop(&mut self) {
        println!("Dropping {self:?}");
        // Code to release resources owned by the item would go here.
    }
}
```

`drop` 方法是专门为编译器保留的，不允许手动调用：

```rust
x.drop();
```

```shell
error[E0040]: explicit use of destructor method
  --> src/main.rs:70:7
   |
70 |     x.drop();
   |     --^^^^--
   |     | |
   |     | explicit destructor calls not allowed
   |     help: consider using `drop` function: `drop(x)`
```

在这里，我们需要了解一些技术细节。请注意，`Drop::drop` 方法的签名是 `drop(&mut self)` 而不是 `drop(self)` ：它接收的是对象的可变引用，而不是将对象移动到方法中。如果 `Drop::drop` 像普通方法那样运行，就意味着对象在方法执行后仍然可用 —— 尽管它的所有内部状态已经被清理完毕，资源也已释放！

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
{
    // If calling `drop` were allowed...
    x.drop(); // (does not compile)

    // `x` would still be available afterwards.
    x.0 += 1;
}
// Also, what would happen when `x` goes out of scope?
```

编译器提供了一种简单的替代方案，即调用 [`drop()`] 函数手动销毁对象。该函数接收一个参数移动到函数内，其实 `drop(_item: T)` 函数的实现只是一个空的函数体 `{}` —— 所以当该作用域到右括号时，被移动的对象会就被销毁。

另外，`drop(&mut self)` 方法的签名没有返回类型，这意味着它无法传递失败信息。如果释放资源可能会失败，那么你可能需要一个单独的 `release` 方法来返回一个 `Result`，以便用户检测详情。

无论技术细节如何，`drop` 方法仍然是实现 RAII 模式的关键；它是实现释放与对象相关资源的最佳位置。

原文[点这里](https://www.lurklurk.org/effective-rust/raii.html)查看
  
<!-- 参考链接 -->

[第 15 条]: ../chapter_3/item15-borrows.md
[第 17 条]: ../chapter_3/item17-deadlock.md

[文件描述符]: https://en.wikipedia.org/wiki/File_descriptor
[`Mutex::lock()`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html#method.lock
[`MutexGuard`]: https://doc.rust-lang.org/std/sync/struct.MutexGuard.html
[drop_trait]: https://doc.rust-lang.org/std/ops/trait.Drop.html
[drop_method]: https://doc.rust-lang.org/std/ops/trait.Drop.html#tymethod.drop
[`drop()`]: https://doc.rust-lang.org/std/mem/fn.drop.html
