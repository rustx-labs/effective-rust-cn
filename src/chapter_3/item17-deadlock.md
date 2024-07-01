# 第 17 条：对共享状态的并行性保持警惕

> "即使是最大胆的共享数据形式在 Rust 中也能保证安全。"-[Aaron Turon](https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html)

官方文档描述说 Rust 实现了[“无畏并发”]，但本条将探讨为什么（令人遗憾的是）即使是在 Rust 中，仍有一些理由需要对并发保持警惕。

本条特别关注*共享状态*的并行性：正在执行的不同线程通过共享内存相互通信。无论是哪种语言，线程之间共享状态通常会带来*两个*可怕的问题：

- _数据竞争_：这可能导致数据损坏。
- _死锁_：这可能导致你的程序陷入停滞。

上述两个问题都很可怕（“引起或可能导致恐慌”），因为他们在实际调试中会变得非常困难：错误的发生是不固定的，并且通常更有可能在有负载的情况下发生——这意味着他们可能不会在单元测试，集成测试或其他任何类型测试中被发现（[第 30 条]），但他们会在生产环境中出现。

Rust 已经向前迈出了一大步，因为它完全解决了上述两个问题之一。然而，正如我们所见，另外一个问题仍然存在。

## 数据竞争

让我们先通过探索*数据竞争*和 Rust 来看一个好消息。数据竞争的精确定义因语言而异，但我们可以将关键部分总结如下：

> 当两个不同的线程在以下条件访问内存中同一位置时，会发生数据竞争：
>
> - 至少有一个线程在写入。
> - 没有强制规定访问顺序的同步机制。

### C++ 中的数据竞争

通过一个例子可以很好地说明这这些基础知识。考虑一个跟踪银行账户的数据结构：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```cpp
// C++ code.
class BankAccount {
 public:
  BankAccount() : balance_(0) {}

  int64_t balance() const {
    if (balance_ < 0) {
      std::cerr << "** Oh no, gone overdrawn: " << balance_ << "! **\n";
      std::abort();
    }
    return balance_;
  }
  void deposit(uint32_t amount) {
    balance_ += amount;
  }
  bool withdraw(uint32_t amount) {
    if (balance_ < amount) {
      return false;
    }
    // 如果此时其他线程更改了 `balance_` 会发生什么？
    std::this_thread::sleep_for(std::chrono::milliseconds(500));

    balance_ -= amount;
    return true;
  }

 private:
  int64_t balance_;
};
```

这个例子用 C++ 写的，不是 Rust，原因很快就会明了。然而，相同的一般概念也适用于许多其他（非 Rust）语言——Java，Go，Python 等。

该类在单线程中工作正常，但是考虑多线程下情况：

```cpp
BankAccount account;
account.deposit(1000);

// 启动一个线程，用来监视余额不足的情况，并为账户充值。
std::thread payer(pay_in, &account);

// 启动三个线程，每个线程尝试重复取款。
std::thread taker(take_out, &account);
std::thread taker2(take_out, &account);
std::thread taker3(take_out, &account);
```

这里有若干线程反复尝试从账户中取款，并且有一个额外的线程在账户余额不足时为账户充值：

```cpp
// 持续监控 `account` 余额，如果余额不足则进行充值。
void pay_in(BankAccount* account) {
  while (true) {
    if (account->balance() < 200) {
      log("[A] Balance running low, deposit 400");
      account->deposit(400);
    }
    // (此带有 sleep 的无限循环只是为了示范/模拟耗时操作的目的)
    std::this_thread::sleep_for(std::chrono::milliseconds(5));
  }
}

// 反复尝试从 `account` 取款。
void take_out(BankAccount* account) {
  while (true) {
    if (account->withdraw(100)) {
      log("[B] Withdrew 100, balance now " +
          std::to_string(account->balance()));
    } else {
      log("[B] Failed to withdraw 100");
    }
    std::this_thread::sleep_for(std::chrono::milliseconds(20));
  }
}
```

最终，程序会出错：

```
** Oh no, gone overdrawn: -100! **
```

这个问题不难发现，特别是 `withdraw()` 方法写了有用的注释：当涉及多个线程时，余额的值在检查和修改之间可能发生变化。然而，现实世界的此类 bug 会更难被发现——尤其是如果允许编译器在幕后执行各种小技巧以及代码重新排序（如 C++这样）。

包含了 `sleep` 的调用是为了人为地提高这种错误被命中，并尽早发现的机会；当此类问题实际发生时，他们很可能很少出现或间歇性发生——这使得他们很难被调试。

`BankAccount` 类是*线程兼容的*，这意味着它可以在多线程环境中使用，但使用该类的人要确保对其访问需要受到某种外部同步机制的控制。

通过添加内部同步操作，可以将该类转换为*线程安全*类，这意味着可以安全地从多个线程使用该类：[^1]

```cpp
// C++ code.
class BankAccount {
 public:
  BankAccount() : balance_(0) {}

  int64_t balance() const {
    // 对该作用域上锁 mu_。
    const std::lock_guard<std::mutex> with_lock(mu_);
    if (balance_ < 0) {
      std::cerr << "** Oh no, gone overdrawn: " << balance_ << " **!\n";
      std::abort();
    }
    return balance_;
  }
  void deposit(uint32_t amount) {
    const std::lock_guard<std::mutex> with_lock(mu_);
    balance_ += amount;
  }
  bool withdraw(uint32_t amount) {
    const std::lock_guard<std::mutex> with_lock(mu_);
    if (balance_ < amount) {
      return false;
    }
    balance_ -= amount;
    return true;
  }

 private:
  mutable std::mutex mu_; // 保护 balance_
  int64_t balance_;
};
```

内部字段 `balance_` 现在由锁 `mu_` 进行保护：这是一种可以确保同一时刻只有一个线程可以持有该锁的同步对象。调用者可以通过调用 `std::mutex::lock()` 来获取锁；第二个以及后续调用者调用 `std::mutex::lock()` 都会被阻塞，直到最开始的调用者调用了 `std::mutex::unlock()`，然后被阻塞的线程*之一*会解除阻塞并继续执行 `std::mutex::lock()`。

现在，对余额的所有访问都在持有锁的情况下进行，这确保了此值在检查和修改之间保持一致。[`std::lock_guard`] 也值得强调一下：它是一个 RAII 类（参考[第 11 条]），创建时调用 `lock()` 并在销毁时调用 `unlock()`。这确保了锁在离开作用域时被 unlock，从而减少了在手动调用 `lock()` 和 `unlock()` 时出错的概率。

然而，这里的线程安全仍然非常脆弱；摧毁这种安全只需要对类进行一个错误的修改：

```cpp
// 添加一个新的 C++ 方法...
void pay_interest(int32_t percent) {
  // ...但是忘记关于 mu_ 的事情了
  int64_t interest = (balance_ * percent) / 100;
  balance_ += interest;
}
```

然后线程安全被摧毁了。[^2]

### Rust 中的数据竞争

对于一本关于 Rust 的书来说，本条已经写了够多 C++了，所以考虑将这个类直接转换成 Rust：

```rust
pub struct BankAccount {
    balance: i64,
}

impl BankAccount {
    pub fn new() -> Self {
        BankAccount { balance: 0 }
    }
    pub fn balance(&self) -> i64 {
        if self.balance < 0 {
            panic!("** Oh no, gone overdrawn: {}", self.balance);
        }
        self.balance
    }
    pub fn deposit(&mut self, amount: i64) {
        self.balance += amount
    }
    pub fn withdraw(&mut self, amount: i64) -> bool {
        if self.balance < amount {
            return false;
        }
        self.balance -= amount;
        true
    }
}
```

以及尝试永久向账户付款或取款的功能：

```rust
pub fn pay_in(account: &mut BankAccount) {
    loop {
        if account.balance() < 200 {
            println!("[A] Running low, deposit 400");
            account.deposit(400);
        }
        std::thread::sleep(std::time::Duration::from_millis(5));
    }
}

pub fn take_out(account: &mut BankAccount) {
    loop {
        if account.withdraw(100) {
            println!("[B] Withdrew 100, balance now {}", account.balance());
        } else {
            println!("[B] Failed to withdraw 100");
        }
        std::thread::sleep(std::time::Duration::from_millis(20));
    }
}
```

这在单线程上下文中工作正常——即使该线程不是主线程：

```rust
{
    let mut account = BankAccount::new();
    let _payer = std::thread::spawn(move || pay_in(&mut account));
    // 在该作用域结尾，`_payer` 线程开始独立运行
    // 并且成为 `BankAccount` 的唯一所有者。
}
```

但如果简单地尝试跨多个线程使用 `BankAccount`：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
{
    let mut account = BankAccount::new();
    let _taker = std::thread::spawn(move || take_out(&mut account));
    let _payer = std::thread::spawn(move || pay_in(&mut account));
}
```

会立即编译失败：

```
error[E0382]: use of moved value: `account`
   --> src/main.rs:102:41
    |
100 | let mut account = BankAccount::new();
    |     ----------- move occurs because `account` has type
    |                 `broken::BankAccount`, which does not implement the
    |                 `Copy` trait
101 | let _taker = std::thread::spawn(move || take_out(&mut account));
    |                                 -------               ------- variable
    |                                 |                         moved due to
    |                                 |                         use in closure
    |                                 |
    |                                 value moved into closure here
102 | let _payer = std::thread::spawn(move || pay_in(&mut account));
    |                                 ^^^^^^^             ------- use occurs due
    |                                 |                        to use in closure
    |                                 |
    |                                 value used here after move
```

借用检查器规则（[第 15 条]）能告知我们原因：对同一项目有两个可变引用，其中一个超出了允许的范围。借用检查器的规则是，你可以对某个项目有单个可变引用，或者多个（不可变）引用，但是不能同时有二者。

这与本条开头的数据竞争的定义有一个奇怪的相同点：强制只有一个写入者，或多个读取者（但不能两者同时），这意味着不能出现数据竞争。通过强制执行内存安全，[Rust “免费” 获得了线程安全]。

与 C++ 一样，需要某种同步来使得 `struct` 线程安全。最常见的机制也称为 [`Mutex`] 互斥锁，但 Rust 版本的 Mutex “包装” 受保护的数据，而不是变成一个独立的对象（如 C++中）：

```rust
pub struct BankAccount {
    balance: std::sync::Mutex<i64>,
}
```

`Mutex` 泛型上的 [`lock()`] 方法返回具有 RAII 行为的 [`MutexGuard`] 对象，如 C++ 的 `std::lock_guard` 一样：在作用域结束时，guard 被 drop，互斥锁会自动释放。（与 C++ 相比，Rust 的互斥锁并没有手动获取或释放的方法，因为它们会让开发者陷入忘记保持这些调用完全同步的风险中。）

更准确地说，`lock()` 实际上返回了一个持有 `MutexGuard` 的 `Result`，以应对 `Mutex` 被*中毒*的可能性。如果线程在持有锁时失败，就会发生中毒，因为这可能意味着任何被互斥锁保护的不变量已经不再可靠。实际上，锁中毒是非常罕见的（并且当它发生时让程序终止是可取的），因此通常会直接调用 `.unwarp()` 来处理 `Result`（尽管这与[第 18 条]相违背）。

`MutexGuard` 对象还通过实现 `Deref` 和 `DerefMut` trait [第 8 条]来充当 `Mutex` 所包含数据的代理，允许它可以进行读取操作。

```rust
impl BankAccount {
    pub fn balance(&self) -> i64 {
        let balance = *self.balance.lock().unwrap();
        if balance < 0 {
            panic!("** Oh no, gone overdrawn: {}", balance);
        }
        balance
    }
}
```

对于写入操作：

```rust
impl BankAccount {
    // 注意：不再需要 `&mut self`。
    pub fn deposit(&self, amount: i64) {
        *self.balance.lock().unwrap() += amount
    }
    pub fn withdraw(&self, amount: i64) -> bool {
        let mut balance = self.balance.lock().unwrap();
        if *balance < amount {
            return false;
        }
        *balance -= amount;
        true
    }
}
```

这些方法的签名中隐藏一个有趣的细节：尽管他们正在修改 `BankAccount` 的余额，但是这些方法参数是 `&self` 而不是 `&mut self`。这是不可避免的：如果多个线程想要保存对同一个 `BankAccount` 的引用，根据借用检查器规则，这些引用最好是不可变的。这也是[第 8 条]中描述的*内部可变性*模式的另一个实例：借用检查实际上从编译时移动到运行时，但是此处具有了同步跨线程行为。如果可变引用已经存在，则尝试获取第二个引用将被阻止，直到第一个引用已被删除。

把共享状态包装在 `Mutex` 中可以安抚借用检查器，但仍存在生命周期问题（[第 14 条]）需要修复：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
{
    let account = BankAccount::new();
    let taker = std::thread::spawn(|| take_out(&account));
    let payer = std::thread::spawn(|| pay_in(&account));
    // 在该作用域末尾，`account` 被 drop，但是
    // `_taker` 和 `_payer` 线程仍在运行，所以
    // 仍持有对 `account` 的（不可变）引用。
}
```

```
error[E0373]: closure may outlive the current function, but it borrows `account`
              which is owned by the current function
   --> src/main.rs:206:40
    |
206 |     let taker = std::thread::spawn(|| take_out(&account));
    |                                    ^^           ------- `account` is
    |                                    |                     borrowed here
    |                                    |
    |                                    may outlive borrowed value `account`
    |
note: function requires argument type to outlive `'static`
   --> src/main.rs:206:21
    |
206 |     let taker = std::thread::spawn(|| take_out(&account));
    |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
help: to force the closure to take ownership of `account` (and any other
      referenced variables), use the `move` keyword
    |
206 |     let taker = std::thread::spawn(move || take_out(&account));
    |                                    ++++
error[E0373]: closure may outlive the current function, but it borrows `account`
              which is owned by the current function
   --> src/main.rs:207:40
    |
207 |     let payer = std::thread::spawn(|| pay_in(&account));
    |                                    ^^         ------- `account` is
    |                                    |                  borrowed here
    |                                    |
    |                                    may outlive borrowed value `account`
    |
note: function requires argument type to outlive `'static`
   --> src/main.rs:207:21
    |
207 |     let payer = std::thread::spawn(|| pay_in(&account));
    |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
help: to force the closure to take ownership of `account` (and any other
      referenced variables), use the `move` keyword
    |
207 |     let payer = std::thread::spawn(move || pay_in(&account));
    |                                    ++++
```

给出的错误消息清晰展示问题所在：`BankAccount` 将在该块末尾被 `drop`，但是这里有两个线程引用了它，并可能在这之后继续运行。（编译器给出关于如何修改的建议并没有太大帮助——如果 `BankAccount` 数据被移动到第一个闭包中，则第二个闭包将无法再接收对它的引用！）

用来确保对象保持活动状态，直到对它的所有引用都消失，这种标准工具是引用计数指针，Rust 用于多线程使用的变体是 [`std::sync::Arc`]：

```rust
let account = std::sync::Arc::new(BankAccount::new());
account.deposit(1000);

let account2 = account.clone();
let _taker = std::thread::spawn(move || take_out(&account2));

let account3 = account.clone();
let _payer = std::thread::spawn(move || pay_in(&account3));
```

每个线程都会获得自己的引用计数指针的副本，并移动至闭包中，并且仅当引用计数降至零时，才会使底层数据 `BankAccount` 被 `drop`。`Arc<Mutex<T>>` 这种组合在使用共享状态并行性的 Rust 程序中很常见。

从技术细节退一步来看，Rust 完全避免了困扰其他语言的多线程编程的数据竞争问题。当然，这个好消息仅限于 _safe_ Rust——`unsafe` 代码（[第 16 条]）和尤其是 FFI 绑定（[第 34 条]）可能不会避免数据竞争——但是这仍是一个值得注意的现象。

### 标准库 trait 标记

有两个标准库 trait 会影响线程之间 Rust 对象的使用。这两个 trait 都是*标记 trait*（[第 10 条]），他们没有关联的方法，但在多线程场景中对于编译器具有特殊含义：

- [`Send`] trait 表明某种类型的数据可以安全地跨线程传输；这种类型的数据的所有权可以从一个线程传递到另一个线程。
- [`Sync`] trait 表明某种类型的数据可以由多个线程安全地访问，但必须遵守借用检查器规则。

换句话说，我们可以发现，`Send` 意味着 `T` 可以在线程间传输，`Sync` 意味着 `&T` 可以在线程间传输。

这两个 trait 都是[自动 trait]：编译器会自动把他们派生（derive）为新类型，只需要该类型的组成部分也实现了 `Send` / `Sync`。

大多数安全类型都实现了 `Send` 和 `Sync`，这些类型太多了，所以我们需要清楚的了解哪些类型*没有*实现这些 trait（以 `impl !Sync for Type` 来表示）。

没有实现 `Send` 的类型只能在单个线程使用。一个典型的例子是非同步引用计数指针 [`Rc<T>`]（[第 8 条]）。这种类型在实现上就明确假定使用单线程（为了速度）；它没有尝试同步内部引用计数来供多线程使用。因此，不允许在线程之间传输 `Rc<T>`；为此应该用 `Arc<T>`（以及额外性能开销）。

未实现 `Sync` 的类型无法安全地从多个线程通过*非*`mut`引用来使用（因为借用检查器会确保永远不会有多个 `mut` 引用）。典型的例子是，以不同步方式提供*内部可变性*的类型，例如 [`Cell<T>`] 和 [`RefCell<T>`]。使用 `Mutex<T>` 或 [`RwLock<T>`] 来在多线程环境中提供内部可变性。

原始指针类型，比如 `*const T` 和 `*mut T` 也都没实现 `Send` 和 `Sync`；相见[第 16 条]和[第 34 条]。

## 死锁

现在有个坏消息。虽然 Rust 已经解决了数据竞争问题（如前所述），但对于具有共享状态的多线程代码来说，它仍会受到*第二个*可怕问题的影响：_死锁_。

考虑一个简化的多人服务器，它是用多线程应用来实现的，可以并行地为许多玩家提供服务。有两个核心的数据结构，可能是玩家的集合（按用户名索引），以及正在进行游戏的集合（按某个唯一的标识符进行索引）：

```rust
struct GameServer {
    // 从玩家名字到玩家信息的映射
    players: Mutex<HashMap<String, Player>>,
    // 当前游戏，由唯一的 game ID 来索引。
    games: Mutex<HashMap<GameId, Game>>,
}
```

这两种数据结构都有 `Mutex` 来保护，所以不会出现数据竞争。然而，操作*这两种*数据结构的代码可能有潜在的问题。两者之间的单一交互可以正常运行：

```rust
impl GameServer {
    /// 添加新玩家并将他们加入当前游戏。
    fn add_and_join(&self, username: &str, info: Player) -> Option<GameId> {
        // 添加新玩家
        let mut players = self.players.lock().unwrap();
        players.insert(username.to_owned(), info);

        // 找到一个未满的游戏房间来让他们加入
        let mut games = self.games.lock().unwrap();
        for (id, game) in games.iter_mut() {
            if game.add_player(username) {
                return Some(id.clone());
            }
        }
        None
    }
}
```

然而，两个相互独立且锁定的数据结构之间的第二次交互，就会开始产生问题：

```rust
impl GameServer {
    /// 通过 `username` 来封禁掉玩家，把他们从任何当前游戏中移除
    fn ban_player(&self, username: &str) {
        // 找到该用户所在所有的游戏房间，并移除。
        let mut games = self.games.lock().unwrap();
        games
            .iter_mut()
            .filter(|(_id, g)| g.has_player(username))
            .for_each(|(_id, g)| g.remove_player(username));

        // 从用户列表删除他们。
        let mut players = self.players.lock().unwrap();
        players.remove(username);
    }
}
```

为了理解该问题，想象有两个单独的线程使用这两种方法，他们的执行顺序如表 3-1 所示。

_表 3-1. 线程死锁顺序_

| **线程 1**                                                  | **线程 2**                                                    |
| ----------------------------------------------------------- | ------------------------------------------------------------- |
| 进入 `add_and_join()` 并立即获取 `players` 锁。             |                                                               |
|                                                             | 进入 `ban_player()` 并立即获取 `games` 锁。                   |
| 尝试获取 `games` 锁；但目前由线程 2 所有，所以线程 1 阻塞。 |                                                               |
|                                                             | 尝试获取 `players` 锁；但目前由线程 1 所有，所以线程 2 阻塞。 |

此时，程序陷入死锁：两个线程都不会继续运行，任何其他线程也不会对两个 `Mutex` 保护的数据结构中的任何一个执行任何操作。

其根本原因是*锁反转*：一个函数按照 `player` 然后 `games` 的顺序获取锁，而另外一个函数使用相反的顺序（`games` 然后 `players`）。这只是一个普遍问题的简单示例；更长的嵌套锁链也会出现这种情况（线程 1 取得锁 A，然后 B，然后尝试获取 C；线程 2 获取 C，然后尝试获取 A）以及跨更多线程（线程 1 给 A 上锁，然后 B；线程 2 给 B 上锁，然后 C；线程 3 给 C 上锁，然后 A）。

解决此问题的尝试的简单方法有：缩小锁的范围，因此我没有必要同时持有两个锁。

```rust
/// 添加新玩家，并把他们加入到当前游戏
fn add_and_join(&self, username: &str, info: Player) -> Option<GameId> {
    // 添加新玩家。
    {
        let mut players = self.players.lock().unwrap();
        players.insert(username.to_owned(), info);
    }

    // 找到一个未满的游戏房间来让他们加入
    {
        let mut games = self.games.lock().unwrap();
        for (id, game) in games.iter_mut() {
            if game.add_player(username) {
                return Some(id.clone());
            }
        }
    }
    None
}
/// 通过 `username` 来封禁掉玩家，把他们从任何当前游戏中移除
fn ban_player(&self, username: &str) {
    // 找到该用户所在所有的游戏房间，并移除。
    {
        let mut games = self.games.lock().unwrap();
        games
            .iter_mut()
            .filter(|(_id, g)| g.has_player(username))
            .for_each(|(_id, g)| g.remove_player(username));
    }

    // 从用户列表删除他们。
    {
        let mut players = self.players.lock().unwrap();
        players.remove(username);
    }
}
```

（更好的方法是将 `players` 数据结构的操作封装到 `add_player()` 和 `remove_player()` 辅助方法中，来减少忘记关闭作用域范围的可能性。）

这解决了死锁问题，但是又多了数据一致性的问题：如果执行顺序如表 3-2 所示，`players` 和 `games` 的数据结构可能会彼此不同步。

_表 3-2. 状态不一致顺序_

| **线程 1**                                                                                   | **线程 2**                                                                        |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| 进入 `add_and_join("Alice")` 并且添加 Alice 到 `players` 数据结构中（然后释放 `players` 锁） |                                                                                   |
|                                                                                              | 进入 `ban_player("Alice")` 并且从所有 `games` 中移除 Alice（然后释放 `games` 锁） |
|                                                                                              | 从 `players` 数据结构中删除 Alice；线程 1 已经释放了锁，所以不会阻塞              |
| 继续并获取 `games` 锁（已由线程 2 释放）。持有锁后，添加 "Alice" 到正在进行的游戏中          |                                                                                   |

此时，根据 `player` 数据结构，有一个游戏包含不存在的玩家！

问题的核心是，有两个数据结构都需要保持彼此同步。做到这一点的最好方法是使用一个覆盖二者的同步原语：

```rust
struct GameState {
    players: HashMap<String, Player>,
    games: HashMap<GameId, Game>,
}

struct GameServer {
    state: Mutex<GameState>,
    // ...
}
```

## 建议

为了避免共享状态并行性所出现的问题，最明显的建议就是避免共享状态的并行性。[Rust 程序设计语言]中引用了 [Go 语言文档]：“不用通过共享内存进行通信；相反，通过通信来共享内存。”

Go 语言[内置了](https://go.dev/ref/spec#Channel_types)用于这种操作的*管道*；对 Rust 来说，相同的功能可以在标准库 [`std::sync::mpsc` 模块]中找到：函数 `channel()` 返回一个 `(Sender, Receiver)` 元组对，允许特性类型的值在线程之间进行通信。

如果共享状态进行并发无法避免，那么有一些方法可以减少编写容易出现死锁问题的代码：

- **将必须保持一致的数据结构包含在单个互斥锁中。**
- **保持互斥锁的范围越小越明显越好**；尽可能使用辅助方法来获取和设置锁所包含的内容。
- **避免调用持有锁的闭包**；这会使得代码受到将来可能添加到代码库中的任何闭包的影响。
- 同样，**避免将 `MutexGuard` 返回给调用者**：从死锁的角度看，这就像是分发一把已经上膛的枪。
- 在 CI 系统（[第 32 条]）中加入*死锁检测工具*，例如 [`no_deadlocks`]，[ThreadSanitizer]，或[`parking_lot::deadlock`]。
- 最后手段：设计、记录、测试并严格执行一个*上锁的层次结构*，该结构描述了允许\需求的锁定顺序。这应该作为最后的手段，因为任何依赖于工程师从不犯错策略从长远来看都很可能失败。

更抽象地说，多线程代码应该应用于以下一般建议的理想场所：倾向于编写明显没有错误的简单代码，而不是编写复杂到不明显有错误的代码。

#### 注释

[^1]: 第三种类的行为是*线程对立（thread-hostile）*的：*即使*对它的所有访问都是对外同步的，代码在多线程环境中也是危险的。
[^2]: Clang C++ 编译器包含一个 [`-Wthread-safety`](https://clang.llvm.org/docs/ThreadSafetyAnalysis.html) 选项，有时也称为*注释*，它允许通过关于哪一个互斥锁保护该数据的信息来注释该数据，并通过关于该函数获取锁的信息来注释该函数。当这些不变量被破坏时，会在编译期产生错误，就像 Rust 一样；然而，并没有强制使用这些注释——例如，当一个线程兼容的库第一个在多线程环境中使用时。

<!-- 参考链接 -->

[第 8 条]: https://www.lurklurk.org/effective-rust/references.html
[第 10 条]: https://www.lurklurk.org/effective-rust/std-traits.html
[第 11 条]: https://www.lurklurk.org/effective-rust/raii.html
[第 14 条]: https://www.lurklurk.org/effective-rust/lifetimes.html
[第 15 条]: https://www.lurklurk.org/effective-rust/borrows.html
[第 16 条]: https://www.lurklurk.org/effective-rust/unsafe.html
[第 34 条]: https://www.lurklurk.org/effective-rust/ffi.html
[第 18 条]: https://www.lurklurk.org/effective-rust/panic.html
[第 30 条]: https://www.lurklurk.org/effective-rust/testing.html
[第 32 条]: https://www.lurklurk.org/effective-rust/ci.html
[“无畏并发”]: https://doc.rust-lang.org/book/ch16-00-concurrency.html
[`std::lock_guard`]: https://en.cppreference.com/w/cpp/thread/lock_guard
[Rust “免费” 获得了线程安全]: https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency.html
[`Mutex`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[`lock()`]: https://doc.rust-lang.org/std/sync/struct.Mutex.html#method.lock
[`MutexGuard`]: https://doc.rust-lang.org/std/sync/struct.MutexGuard.html
[`std::sync::Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[`Send`]: https://doc.rust-lang.org/std/marker/trait.Send.html
[`Sync`]: https://doc.rust-lang.org/std/marker/trait.Sync.html
[自动 trait]: https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits
[`Rc<T>`]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[`Cell<T>`]: https://doc.rust-lang.org/std/cell/struct.Cell.html
[`RefCell<T>`]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[`RwLock<T>`]: https://doc.rust-lang.org/std/sync/struct.RwLock.html
[Rust 程序设计语言]: https://doc.rust-lang.org/book/ch16-02-message-passing.html
[Go 语言文档]: https://golang.org/doc/effective_go.html#concurrency
[`std::sync::mpsc` 模块]: https://doc.rust-lang.org/std/sync/mpsc/index.html
[`no_deadlocks`]: https://docs.rs/no_deadlocks
[ThreadSanitizer]: https://clang.llvm.org/docs/ThreadSanitizer.html
[`parking_lot::deadlock`]: https://amanieu.github.io/parking_lot/parking_lot/deadlock/index.html
