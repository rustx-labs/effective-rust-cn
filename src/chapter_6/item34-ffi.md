# 第 34 条：控制跨越 FFI 边界的内容

虽然 Rust 已经具备了能力丰富的[标准库][standard library]，并且还有迅速发展的[crate 生态系统][crate eco]，但现实中还是存在大量的非 Rust 的代码。

与其他较新的语言一样，Rust 提供了 *外部函数接口（foreign function interface, FFI）* 机制，该机制使得 Rust 可以与其他语言编写的代码以及数据结构进行互操作。虽然 FFI 的名字中带有“函数”，实际上这种互操作的能力并不局限于函数调用。这使得 Rust 程序可以使用其他语言编写的已有的库，无需“使用 Rust 重写”。

Rust 的默认目标是可以与 C 程序互操作，许多其他提供跨语言互通的语言也是首先提供与 C 互通的能力。一部分原因是因为 C 的库普遍存在，另外也是因为 C 的简洁性：C 作为互操作性的“最小公分母”，它不依赖具备高级功能的工具链。如果与其他语言互通可能需要考虑更多高级功能（例如：Java 或 Go 中的垃圾收集、C++ 的异常处理和模板类、Java 和 C++ 的函数重载等）。

但是，这并不是说 Rust 和 C 互操作就是非常简单的事情。由于引入了其他语言编写的代码，Rust 提供的安全保证和保护将不再使用，尤其是涉及到内存安全的部分。

所以，Rust 中的 FFI 代码都是 `unsafe` 的，[第 16 条]的建议将不再适用于此场景。本章节提供一些针对 FFI 的替代建议，[第 35 条]探讨了用来解决使用 FFI 时遇到的某些问题（但不是全部）的工具，《[Rustonomicon][Rustonomicon]》 中的 [FFI][FFI chapter] 一章也提供了很有帮助的建议和信息。

## 从 Rust 调用 C 函数

最简单的 FFI 交互就是从Rust 代码调用 C 函数，并且参数都是不涉及指针、引用或者内存地址的“直接”类型：

```c
/* 文件：lib.c */
#include "lib.h"

/* C 函数定义。 */
int add(int x, int y) {
  return x + y;
}
```

C 代码中*定义*了一个函数，通常还伴随一个头文件来*声明*这个函数以方便其他 C 代码使用它：

```C
/* 文件：lib.h */
#ifndef LIB_H
#define LIB_H

/* C 函数声明。 */
int add(int x, int y);

#endif  /* LIB_H */
```

该声明大致如下：在某处存在一个名为 `add` 的函数，它接受两个整数作为参数，并且返回另外一个整数。这使得其他 C 代码可以使用这个函数，但是实际上该函数的实现代码会在后续的链接阶段提供。

如果要在 Rust 代码中使用这个 `add` 函数，也需要类似的声明，来描述函数的签名且对应的实现代码会在后续环节提供：

```rust
use std::os::raw::c_int;
extern "C" {
    pub fn add(x: c_int, y: c_int) -> c_int;
}
```

通过在函数声明中使用 `extern "C"` 标记表示未来会有一个 C 的库提供函数代码 [^1]，同时，这个标记也会自动将函数标记为 [`#[no_mangle]`][no_mangle]，下一节会详细讲解这个属性。

### 链接过程

C 工具链是如何生成外部库以及该库的格式，和平台环境相关，这些细节超出了本书范畴。然而，在类 Unix 系统上，*静态库*文件是常见的简单形态。静态库文件可以使用 [`ar`][ar] 工具生成，文件名通常是 *lib&lt;something&gt;.a* 的格式，例如：*libcffi.a*。

Rust 的构建系统需要知道对于所声明的外部函数，在哪个库文件中包含其对应代码。可以通过在代码中使用 [`link` 属性][link]指明对应的库文件：

```rust
#[link(name = "cffi")] // 需要名为 `libcffi.a` 的外部库文件
extern "C" {
    // ...
}
```

或者，可以使用[构建脚本][build script]向 `cargo` 发起 [`cargo:rustc-link-lib`][rustc-link-lib] 指令 [^2]：

```rust
// 文件： build.rs
fn main() {
    // 需要名为 `libcffi.a` 的外部库文件
    println!("cargo:rustc-link-lib=cffi");
}
```

后者更加灵活，因为构建脚本可以检查所处的环境，然后根据所找到的内容采取不同的行为。

无论是哪种方案，如果所需的 C 的库不在系统的库路径中，Rust 构建系统都要有一种能够找到这个库文件的机制。可以在构建脚本中向 `cargo` 发起 [`cargo:rustc-link-search`][rustc-link-search] 指令，其中包含了库文件所在的路径：

```rust
// 文件：build.rs
fn main() {
    // ...

    // 获取 `Cargo.toml` 文件所在目录。
    let dir = std::env::var("CARGO_MANIFEST_DIR").unwrap();
    // 在上级目录中查找库文件。
    println!(
        "cargo:rustc-link-search=native={}",
        std::path::Path::new(&dir).join("..").display()
    );
}
```

### 代码层面的考量

让我们回到代码本身，即使是最简单的 FFI 调用都可能存在一些陷阱。首先，使用 FFI 的函数会被自动标注为 `unsafe` 的，需要将其包裹在 `unsafe { }` 块中：

```rust
let x = add(1, 1);
```

```
error[E0133]: call to unsafe function is unsafe and requires unsafe function
              or block
   --> src/main.rs:176:13
    |
176 |     let x = add(1, 1);
    |             ^^^^^^^^^ call to unsafe function
    |
    = note: consult the function's documentation for information on how to
            avoid undefined behavior
```

另一个需要注意的问题是 C 的 `int` 类型，在 Rust 中对应的是 [`std::os::raw::c_int`][c_int ]。一个 `int` 是多大？*有可能*下面两个值是一样的：

- 用来编译 C 代码的工具链中的 `int` 类型大小
- Rust 工具链中的 `std::os::raw::c_int`  大小

但是我们不能冒险假设此二者一定一样大。因此，尽量**在 FFI 边界选择已知大小的类型** —— 在 C 代码中，使用 `<stdint.h>` 中的类型，例如：`uint32_t`。然而，如果是一个已经使用了 `int` / `long` / `size_t` 类型的现有 C 代码库，那这个要求就难以满足了。

最后一个实际问题是，C 代码和 Rust 代码中的函数声明要完全匹配。更糟糕的是，如果它们不匹配，构建工具不会给出任何警告，而是默默的生成错误的代码。

[第 35 条]中提到，可以使用 `bindgen` 工具来避免类似问题，*为什么*构建工具无法检测这种问题，其背后的原因是什么？这值得我们花时间去搞搞清楚，特别是，了解*名称重整*的基本原理。

### 名称重整

编译型语言通常支持*独立编译*：先将程序的不同部分分别转换成机器代码块（目标文件），最后再由*链接器*将其整合成一个完整的程序。这就意味着，如果只改动了一小部分代码，仅需重新编译对应的目标文件即可，然后由链接器将变动过的和未变动过的目标文件合并起来重建程序。

粗略来讲，[链接步骤就像“按点连线”游戏][join the dots]那样，一部分目标文件提供函数或者变量的定义，另一些目标文件中包含占位符，表示期望在其他目标文件中找到在编译期间尚未提供的对应定义，链接器会将二者合并起来，确保每个占位符都会被对应的具体定义替换。

链接器通过简单的名称匹配机制来查找占位符和定义之间的关系，这就意味着所有的关联关系都存在于一个全局命名空间中。

这种方式对于链接 C 程序来说是没有问题的，因为一个名字不能以任何方式来重用 —— 函数的名字就是它在目标文件中的名字。因此，C 库的一个常见的约定是，通过在符号前增加前缀以避免命名冲突，例如：`lib1_process` 和 `lib2_process` 。

但是，对于允许重载定义的 C++ 语言来说，这样是行不通的：

```cpp
// C++ 代码
namespace ns1 {
int32_t add(int32_t a, int32_t b) { return a+b; }
int64_t add(int64_t a, int64_t b) { return a+b; }
}
namespace ns2 {
int32_t add(int32_t a, int32_t b) { return a+b; }
}
```

为了解决这个问题，引入了*名称重整*机制：编译器将[重载函数的签名和类型信息编码][compiler encodes]到输出到目标文件中，链接器还是保持原来的处理方式：在占位符和定义之间一一匹配。

在类 Unix 系统上，可以使用 [`nm`][nm] 命令行工具查看目标文件：

```
% nm ffi-lib.o | grep add  # C 链接器看到的
0000000000000000 T _add

% nm ffi-cpp-lib.o | grep add  # C++ 链接器看到的
0000000000000000 T __ZN3ns13addEii
0000000000000020 T __ZN3ns13addExx
0000000000000040 T __ZN3ns23addEii
```

在本例中，有 3 个经过重整的符号，都指向其对应的代码。（`T` 表示二进制输出文件中的*文本段*，也就是代码所在的区域）。

[`c++filt`][c++filt] 工具可以将重整后的名称还原到代码中的名称：

```
% nm ffi-cpp-lib.o | grep add | c++filt  # what the programmer sees
0000000000000000 T ns1::add(int, int)
0000000000000020 T ns1::add(long long, long long)
0000000000000040 T ns2::add(int, int)
```

由于重整后的名称中包含了类型信息，所以链接器可以检测占位符和定义之间是否匹配，这可以保证类型安全：如果定义发生了改变，但是占位符中尚未进行相应的更新，链接器会报错。

回到 Rust，标记为 `extern "C"` 的外部函数被隐式地加上了 `#[no_mangle]` 的标记，所以在输出的目标文件中，函数将会保持原始的名称，就像 C 的处理方式一样。这就意味着函数签名的类型安全能力的丢失：链接器只能看到函数名字，所以即使函数的定义和使用之间在类型上的期望是不一致的，链接器也无法感知这一点，问题只会到运行程序的时候才会显现。

## 从 Rust 访问 C 的数据

前面所示的 `add` 函数在 Rust 和 C 之间交换的都是简单的数据类型：一个可以存储到寄存器中的整数。即使如此，仍然有一些细节需要注意。所以，不难想象当处理负责数据结构时会有怎样的棘手问题。

C 和 Rust 都使用 `struct` 将一系列相关的数据合并到一个数据结构之内。但是，当在内存中表示一个 `struct` 时，这两种语言就会有区别了，它们会将字段放到不同的位置，甚至是按照不同的顺序来存放数据（即[*布局*][layout]）。位了防止不匹配问题，**对在 FFI 使用的 Rust 中的类型使用 `#[repr(C)]` 标记**，这种表示方式是专门为与 C 互操作设计的：

```c
/* C 结构体定义 */
/* 这里的变动要同步映射到 lib.rs 中。 */
typedef struct {
    uint8_t byte;
    uint32_t integer;
} FfiStruct;
```

```rust
// 对应的 Rust 数据结构。
// 这里的变动要同步映射到 lib.h / lib.c。
#[repr(C)]
pub struct FfiStruct {
    pub byte: u8,
    pub integer: u32,
}
```

上面所示的结构体定义中，有一行注释专门提醒程序员此两处的定义一定要保持同步。长期来看，完全依靠程序员来保证两处的同步是不可靠的，所以应该借助类似 `bindgen` 这样的工具来实现两种语言代码之间的自动化同步（见[第 35 条]）。

在 FFI 互操作场景中，要特别小心字符串类型。C 和 Rust 中默认的字符串类型是完全不同的：

- Rust [`String`][String] 是已知长度的 UTF-8 编码的数据，可能包括值为 0 的字节。
- C 字符串（`char *`）保存的是字节值（可能有符号的，也可能无符号），它的长度由数据中的第一个值为 0 （`\0`） 的字节决定。

幸运的是，鉴于 Rust 库的设计者已经完成了底层的繁重工作，我们可以在 Rust 中简单明了的使用 C 字符串。在和 C 的互操作过程中，如果需要拥有字符串值，可以**使用 [`CString`][CString] 类型**，如果需要借用字符串值，可以使用 [`CStr`][Cstr] 类型。当你需要向 FFI 函数传递 `const char*` 类型的字符串时，可以使用 `CStr` 的 [`as_ptr()`][as_ptr] 方法。注意，这里的 `const` 很重要，如果 FFI 函数需要修改字符串内容（`char *`），就不可以这样使用了。

## 生命周期

大部分的数据都比较大，以至于无法存储到寄存器，只能存储到内存中。也就是说，访问数据实际上访问的是内存地址。在 C 中对应的是*指针*：一个无任何其他附加语义信息的、代表内存地址的数值（见[第 8 条]）。

在 Rust 中，表示内存地址的概念叫做*引用*，其数值可以提取为一个*裸指针*，方便传递给 FFI ：

```rust
extern "C" {
    // C 函数操作
    // `FfiStruct` 的内容
    pub fn use_struct(v: *const FfiStruct) -> u32;
}
```

```rust
let v = FfiStruct {
    byte: 1,
    integer: 42,
};
let x = unsafe { use_struct(&v as *const FfiStruct) };
```

但是，正如[第 14 条]所述，Rust 中的引用包含所关联内存段*生命周期*相关的额外约束，当将引用转换成裸指针时，这种约束将会丢失。

因此，使用裸指针本质上是不安全的，`unsafe` 标记表明这里存在风险：FFI 边界另一侧的 C 代码可以做出一些破坏 Rust 内存安全性的操作：

- C 代码可能保留指针的值，并在后续的代码中使用。当关联的内存已经从堆上释放，或者在栈上重用了，就会发生*使用已释放的内存*问题。
- C 代码可能会抛弃传递给它的指针的 `const` 限定符，然后修改指针指向的数据，但是 Rust 一侧期望这段数据是不可变的。
- C 代码不受限于 Rust 的 `Mutex` 保护，因此引发数据竞争（见[第 17 条]）问题。
- C 代码可能错误地（例如，调用 `free()` 函数）将关联的堆内存地址返回给内存分配器，意味着 Rust 代码面临使用已释放内存的问题。

这些风险是通过 FFI 机制重用现有代码以节约成本的时候不可避免的。优点是，你只需编写或者自动生成相应的声明就可以重用现有的、大概率可以正常工作的代码；缺点是你失去了使用 Rust 的最大优势 —— 内存保护。

避免 FFI 中的内存问题的首要法则是：**在同一侧分配和释放内存**。例如，下面这段代码包含一对对称的函数：

```C
/* C 函数。 */

/* 为 `FfiStruct` 分配内存 */
FfiStruct* new_struct(uint32_t v);
/* 释放前面为 `FfiStruct` 分配的内存 */
void free_struct(FfiStruct* s);
```

对应的 Rust FFI 声明：

```rust
extern "C" {
    // 为 `FfiStruct` 分配内存的 C 代码。
    pub fn new_struct(v: u32) -> *mut FfiStruct;
    // 释放前面为 `FfiStruct` 分配的内存的 C 代码。
    pub fn free_struct(s: *mut FfiStruct);
}

```

为了确保分配内存的代码有对应的释放内存的代码，建议实现一个 RAII 包装来自动避免 C 一侧分配的内存泄漏问题（见[第 11 条]）。用作包装器的结构体持有 C 一侧分配的内存：

```rust
/// 包装器结构体拥有 C 一侧分配的内存
struct FfiWrapper {
    // 不可变量: inner 是非空的。
    inner: *mut FfiStruct,
}
```

然后为这个结构体实现 `Drop` trait，将内存地址返回给 C 代码库来避免内存泄漏风险：

```rust
/// 手动实现 [`Drop`]
/// 以确保从 C 代码分配的内存能够正确释放
impl Drop for FfiWrapper {
    fn drop(&mut self) {
        // 安全的: `inner` 是非空的
        // 同时，`free_struct()` 还处理了空指针的情况
        unsafe { free_struct(self.inner) }
    }
}
```

**为 FFI 派生的资源实现 `Drop` trait 以实现 RAII** 这条法则同样适用于除内存之外的其他资源：打开的文件、数据库连接等（见[第 11 条]）。

将与 C 的互操作包装到一个结构体中，还可以捕获一些其他潜在的陷阱，例如：可以将原本不可见的失败转换成 `Result` ：

```rust
type Error = String;

impl FfiWrapper {
    pub fn new(val: u32) -> Result<Self, Error> {
        let p: *mut FfiStruct = unsafe { new_struct(val) };
        // 裸指针不能保证是非空的。
        if p.is_null() {
            Err("Failed to get inner struct!".into())
        } else {
            Ok(Self { inner: p })
        }
    }
}
```

包装后的结构体对外提供安全的使用 C 函数的方法：

```rust
impl FfiWrapper {
    pub fn set_byte(&mut self, b: u8) {
        // 安全的：依赖于 `inner` 是非空的。
        let r: &mut FfiStruct = unsafe { &mut *self.inner };
        r.byte = b;
    }
}
```

或者，如果底层的 C 数据结构有一个等价的 Rust 映射，并且可以安全的直接操作该数据，那么实现 `AsRef` 和 `AsMut` trait （见[第 8 条]）用起来更直接：

```rust
impl AsMut<FfiStruct> for FfiWrapper {
    fn as_mut(&mut self) -> &mut FfiStruct {
        // 安全：`inner` 非空
        unsafe { &mut *self.inner }
    }
}
```

```rust
let mut wrapper = FfiWrapper::new(42).expect("real code would check");
// 直接修改在 C 一侧分配的数据
wrapper.as_mut().byte = 12;
```

上面的示例展示了使用 FFI 时一条非常有帮助的法则：**将访问 `unsafe` FFI 库的代码封装到安全的 Rust 代码之内**。这可以使得其他代码遵循[第 16 条]的建议，避免编写 `unsafe` 代码。它还可以将危险的代码集中在一个地方，便于仔细研究和测试以发现问题，并在出现问题的时候将这些代码视为首要怀疑点来处理。

## 从 C 调用 Rust

哪一侧算是“外部”，这取决于你的观察角度：如果你正在使用 C 开发应用，那么 *Rust* 就可以视为通过 FFI 访问的“外部”。

将 Rust 库导出给 C 的基本原理类似，只不过是相反方向的：

- Rust 中导出的函数需要 `extern "C"` 标记来确保和 C 兼容。
- 默认情况下，Rust 中的符号名称会被重整，类似 C++ 的行为 [^3] ，所以，在函数定义上也需要 `#[no_mangle]` 属性来保持原始名字。但是这同样意味着函数名称作为全局命名空间的一部分，可能和其他程序中的其他符号冲突。所以，**建议在导出的名称前增加前缀**避免混淆，例如：`mylib_...`。
- 给需要导出的结构体增加 `#[repr(C)]` 标记来确保其内存布局和 C 数据类型相同。

同样的，当处理指针、引用或者生命周期的时候，也会有一些潜在的问题。毕竟，C 指针和 Rust 的引用是有所不同的，如果忘记这一点，可能会引发严重问题。

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
#[no_mangle]
pub extern "C" fn add_contents(p: *const FfiStruct) -> u32 {
    // 将调用者传入的裸指针
    // 转换成 Rust 的引用
    let s: &FfiStruct = unsafe { &*p }; // Ruh-roh
    s.integer + s.byte as u32
}
```

```C
/* C 调用 Rust。 */
uint32_t result = add_contents(NULL); // 出错啦！
```

切记你有责任保证遵循 Rust 引用的方式使用裸指针：

```rust
#[no_mangle]
pub extern "C" fn add_contents_safer(p: *const FfiStruct) -> u32 {
    let s = match unsafe { p.as_ref() } {
        Some(r) => r,
        None => return 0, // 如果 C 代码给我们空指针的时候
    };
    s.integer + s.byte as u32
}
```

在上面的示例代码中，C 代码给 Rust 代码传入了一个裸指针，Rust 代码将其转换成一个引用来操作结构体。但是，这个指针从哪里来？Rust 的引用到底引用了什么？

在[第 8 条]的示例中，演示了 Rust 的内存安全机制会防止返回对栈上过期对象的引用。当你把引用作为裸指针返回的时候，就会出现类似的问题：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
impl FfiStruct {
    pub fn new(v: u32) -> Self {
        Self {
            byte: 0,
            integer: v,
        }
    }
}

// 这里不会发生编译错误
#[no_mangle]
pub extern "C" fn new_struct(v: u32) -> *mut FfiStruct {
    let mut s = FfiStruct::new(v);
    &mut s // 返回指向即将过期的栈上对象的裸指针！
}
```

任何从 Rust 返回给 C 一侧的指针都应该是指向堆内存的地址，而非栈地址。但是，尝试通过 `Box` 来把对象放到堆上是不能解决这个问题的：

<div class="ferris"><img src="../images/ferris/not_desired_behavior.svg" width="75" height="75" /></div>

```rust
// 这里不会发生编译错误
#[no_mangle]
pub extern "C" fn new_struct_heap(v: u32) -> *mut FfiStruct {
    let s = FfiStruct::new(v); // 在栈上创建 `FfiStruct`
    let mut b = Box::new(s); // 将 `FfiStruct` 移入堆中
    &mut *b // 返回指向即将过期的堆上对象的裸指针！
}
```

拥有这个值的 `Box` 是在栈上的，所以当它超出作用范围，将会被释放，同时，堆上对象也将被释放，此时返回了无效的指针。

[`Box::into_raw`][box-into-raw] 可以解决这个问题，它放弃了对堆上对象的拥有责任，“忘记”了它：

```rust
#[no_mangle]
pub extern "C" fn new_struct_raw(v: u32) -> *mut FfiStruct {
    let s = FfiStruct::new(v); // 在栈上创建 `FfiStruct`
    let b = Box::new(s); // 将 `FfiStruct` 移入堆中

    // 消费 `Box`，并接管堆上内存
    Box::into_raw(b)
}
```

但是这样做引发了另外一个问题：堆上对象如何释放？之前我们建议在同一侧代码中申请和释放内存，那么就是说，Rust 一侧负责释放内存。对应的工具是 [`Box::from_raw`][box-from-raw]，它可以从裸指针构建一个 `Box`：

```rust
#[no_mangle]
pub extern "C" fn free_struct_raw(p: *mut FfiStruct) {
    if p.is_null() {
        return; // 如果 C 代码给了空指针
    }
    let _b = unsafe {
        // 安全：p 一定非空
        Box::from_raw(p)
    };
} // `_b` 在作用范围结束后被抛弃，释放 `FfiStruct` 内存
```

但是这仍然使得 Rust 代码受制于 C 代码。如果 C 代码出现混乱，两次请求 Rust 释放同一指针，会导致 Rust 的分配器出现致命问题。

以上表明了本章的主题：使用 FFI 会让你面对标准 Rust 中不存在的风险。只要你能够意识到其中的风险和成本，那也是值得的。控制跨越 FFI 边界内容的细节有助于降低风险，但是无法完全消除它。

当使用 C 代码调用 Rust 代码的时候，还有一点需要关注的：如果你的 Rust 代码忽略了[第 18 条]的建议，你应该**防止 `panic!` 跨越 FFI 边界**，因为这会导致未定义的、糟糕的行为 [^4]。

## 牢记

- 与其他语言的代码接口使用 C 作为最小公分母，这意味着符号都存在于一个全局命名空间中。
- 尽可能减少跨越 FFI 边界时可能发生的错误：
  - 使用安全的包装器包裹 `unsafe` 的 FFI 代码
  - 分配和释放内存应该在 FFI 的同一侧完成，无论是哪一侧都可以
  - 让数据结构的内存布局是 C 兼容的
  - 使用已知大小的整数类型
  - 使用标准库中提供的 FFI 相关助手函数或者类型
  - 防止 `panic!` 跨越 FFI 边界

原文[点这里][origin]查看

----

## 注释

[^1]: 如果所用的 FFI 函数来自 C 标准库， [`libc`][libc] crate 已经具备这些声明了，无需重复编写。
[^2]: 在 *Cargo.toml* 中使用 [`links`][links] 键可以让这个依赖对 Cargo 可见。
[^3]: Rust 中用来将重整后的名称变回阅读友好名称的工具叫做 [`rustfilt`][rustfilt]，这个工具基于 [`rustc-demangle`][rustc-demangle] 命令构建，类似于 `c++filt` 工具。
[^4]: Rust 1.71 版本中包含 [C-unwind ABI][c-unwind]，可以实现跨语言的错误回退功能。

<!-- 参考链接 -->

[第 8 条]: ../chapter_1/item8-references&pointer.md
[第 11 条]: ../chapter_2/item11-impl-drop-for-RAII.md
[第 14 条]: ../chapter_3/item14-lifetimes.md
[第 16 条]: ../chapter_3/item16-unsafe.md
[第 17 条]: ../chapter_3/item17-deadlock.md
[第 35 条]: item35-bindgen.md

[origin]: https://www.lurklurk.org/effective-rust/ffi.html
[standard library]: https://doc.rust-lang.org/std/index.html
[crate eco]: https://crates.io/
[Rustonomicon]: https://doc.rust-lang.org/nomicon/
[FFI chapter]: https://doc.rust-lang.org/nomicon/ffi.html
[libc]: https://docs.rs/libc
[no_mangle]: https://doc.rust-lang.org/reference/abi.html?highlight=no_mangle#the-no_mangle-attribute
[ar]: https://man7.org/linux/man-pages/man1/ar.1.html
[link]: https://doc.rust-lang.org/reference/items/external-blocks.html#the-link-attribute
[build script]: https://doc.rust-lang.org/cargo/reference/build-scripts.html
[rustc-link-lib]: https://doc.rust-lang.org/cargo/reference/build-scripts.html#rustc-link-lib
[rustc-link-search]: https://doc.rust-lang.org/cargo/reference/build-scripts.html#rustc-link-search
[c_int]: https://doc.rust-lang.org/std/os/raw/type.c_int.html
[join the dots]: https://lurklurk.org/linkers/linkers.html
[compiler encodes]: https://lurklurk.org/linkers/linkers.html#namemangling
[nm]: https://en.wikipedia.org/wiki/Nm_%28Unix%29
[c++filt]: https://man7.org/linux/man-pages/man1/c%2b%2bfilt.1.html
[layout]: https://doc.rust-lang.org/reference/type-layout.html
[repr]: https://doc.rust-lang.org/reference/type-layout.html#the-c-representation
[String]: https://doc.rust-lang.org/alloc/string/struct.String.html
[CString]: https://doc.rust-lang.org/alloc/ffi/struct.CString.html
[CStr]: https://doc.rust-lang.org/core/ffi/struct.CStr.html
[as_ptr]: https://doc.rust-lang.org/core/ffi/struct.CStr.html#method.as_ptr
[box-into-raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.into_raw
[box-from-raw]: https://doc.rust-lang.org/std/boxed/struct.Box.html#method.from_raw
[bad]: https://doc.rust-lang.org/nomicon/unwinding.html
[links]: https://doc.rust-lang.org/cargo/reference/build-scripts.html#the-links-manifest-key
[rustfilt]: https://crates.io/crates/rustfilt
[rust-demangle]: https://github.com/rust-lang/rustc-demangle
[c-unwind]: https://github.com/rust-lang/rfcs/pull/2945









