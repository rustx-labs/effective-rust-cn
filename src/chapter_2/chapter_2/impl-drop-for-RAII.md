# 第 11 条：为RAII模式实现Drop trait

> “永远不要让人去做机器的工作。” —— 史密斯特工（出自电影《黑客帝国》）

RAII 即“资源获取即初始化”（Resource Acquisition Is Initialization）是一种编程模式，其中值的生命周期与某些附加资源的生命周期完全相关。 RAII 模式由 C++ 编程语言普及，是 C++ 对编程的最大贡献之一。

值的生命周期与资源的生命周期之间的关联被编码在RAII类型中：

- 该类型的构造函数获取对某些资源的访问权
- 该类型的析构函数释放对这些资源的访问权

其结果是 RAII 类型具有一个不变性：仅当对象存在时，才能访问底层资源。因为编译器确保局部变量在作用域退出时会被销毁，这就意味着底层资源也会在退出作用域时被释放。

这对于程序的可维护性很有帮助：如果对代码的后续改动改变了控制流，对象和资源的生命周期仍然是正确的。为了说明这点，来看一些没有使用RAII模式，手动锁定、解锁互斥锁的代码；以下代码是用C++编写的，因为Rust的`Mutex`不允许这种易出错的用法！
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