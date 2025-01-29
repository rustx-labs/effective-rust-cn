# 第 20 条：避免过度优化的诱惑

> Rust让你能写出既安全又高效的零拷贝算法，但这不意味着你写的每个算法都得这样。—— [trentj](https://oreil.ly/fQMfu) 

这本书里的大部分内容都是帮助程序员熟悉Rust及其惯用法。但本章节主要讨论另一个问题，当程序员过度追求Rust的性能潜力时，可能会牺牲代码的易用性和可维护性。

## 数据结构与分配

Rust的引用就像其他语言里的指针，可以让你不复制数据就能重用它。不同的是，Rust的引用生命周期和借用规则让你能安全地这么做。但是，要遵循这些规则（[第 15 条]里有讲），可能会写出更难用的代码。

这对数据结构来说尤其重要。你可以选择是分配一个数据结构里的新副本，还是引用一个已存在的副本。

例如，可以考虑这样的一段代码，它解析一个字节数据流，提取出类型-长度-值（TLV）结构的数据，数据是这样传输的：[^1]

* 一个描述值类型的字节，保存在type_code字段里。
* 一个描述值长度的字节，创建指定长度的切片。
* 接着是值的指定字节数，保存在 value 字段里：

```rust
/// A type-length-value (TLV) from a data stream.
#[derive(Clone, Debug)]
pub struct Tlv<'a> {
    pub type_code: u8,
    pub value: &'a [u8],
}

pub type Error = &'static str; // Some local error type.

/// Extract the next TLV from the `input`, also returning the remaining
/// unprocessed data.
pub fn get_next_tlv(input: &[u8]) -> Result<(Tlv, &[u8]), Error> {
    if input.len() < 2 {
        return Err("too short for a TLV");
    }
    // The TL parts of the TLV are one byte each.
    let type_code = input[0];
    let len = input[1] as usize;
    if 2 + len > input.len() {
        return Err("TLV longer than remaining data");
    }
    let tlv = Tlv {
        type_code,
        // Reference the relevant chunk of input data
        value: &input[2..2 + len],
    };
    Ok((tlv, &input[2 + len..]))
}
```

这个Tlv数据结构很高效，因为它引用了输入数据的相关部分，而不是复制任何数据。Rust的内存安全性保证了引用始终有效。这对于某些场景来说很完美，但如果需要保留数据结构实例（如[第 15 条]所述），事情就会变得尴尬。

例如，考虑一个以TLV数据格式接收消息的网络服务器。接收到的数据可以解析为Tlv实例，但那些实例的生命周期将与传入消息的生命周期相匹配 —— 这可能是堆上的Vec<u8> 临时变量，也可能是某个缓冲区，该缓冲区被多次用于接收消息。

如果服务器代码曾经想要存储传入消息以便稍后查询，这就会引起问题：

```rust
pub struct NetworkServer<'a> {
    // ...
    /// Most recent max-size message.
    max_size: Option<Tlv<'a>>,
}

/// Message type code for a set-maximum-size message.
const SET_MAX_SIZE: u8 = 0x01;

impl<'a> NetworkServer<'a> {
    pub fn process(&mut self, mut data: &'a [u8]) -> Result<(), Error> {
        while !data.is_empty() {
            let (tlv, rest) = get_next_tlv(data)?;
            match tlv.type_code {
                SET_MAX_SIZE => {
                    // Save off the most recent `SET_MAX_SIZE` message.
                    self.max_size = Some(tlv);
                }
                // (Deal with other message types)
                // ...
                _ => return Err("unknown message type"),
            }
            data = rest; // Process remaining data on next iteration.
        }
        Ok(())
    }
}
```

这段代码可以编译，但实际上无法使用：NetworkServer的生命周期必须小于被其process() 方法处理的数据的生命周期。这意味着一个简单的处理循环：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let mut server = NetworkServer::default();
while !server.done() {
    // Read data into a fresh vector.
    let data: Vec<u8> = read_data_from_socket();
    if let Err(e) = server.process(&data) {
        log::error!("Failed to process data: {:?}", e);
    }
}
```

这段代码编译失败，因为临时数据的生命周期被错误地绑定在运行时间更长的的服务器上：

```rust
error[E0597]: `data` does not live long enough
   --> src/main.rs:375:40
    |
372 |     while !server.done() {
    |            ------------- borrow later used here
373 |         // Read data into a fresh vector.
374 |         let data: Vec<u8> = read_data_from_socket();
    |             ---- binding `data` declared here
375 |         if let Err(e) = server.process(&data) {
    |                                        ^^^^^ borrowed value does not live
    |                                              long enough
...
378 |     }
    |     - `data` dropped here while still borrowed
```

改变代码，使其重用一个生命周期更长的缓冲区，同样没有帮助：

<div class="ferris"><img src="../images/ferris/does_not_compile.svg" width="75" height="75" /></div>

```rust
let mut perma_buffer = [0u8; 256];
let mut server = NetworkServer::default(); // lifetime within `perma_buffer`

while !server.done() {
    // Reuse the same buffer for the next load of data.
    read_data_into_buffer(&mut perma_buffer);
    if let Err(e) = server.process(&perma_buffer) {
        log::error!("Failed to process data: {:?}", e);
    }
}
```

这次，编译器抱怨代码试图在保持对同一缓冲区的引用的同时，也提供了一个可变引用：

```rust
error[E0502]: cannot borrow `perma_buffer` as mutable because it is also
              borrowed as immutable
   --> src/main.rs:353:31
    |
353 |         read_data_into_buffer(&mut perma_buffer);
    |                               ^^^^^^^^^^^^^^^^^ mutable borrow occurs here
354 |         if let Err(e) = server.process(&perma_buffer) {
    |                         -----------------------------
    |                         |              |
    |                         |              immutable borrow occurs here
    |                         immutable borrow later used here
```

核心问题是Tlv结构引用了临时的数据 —— 这对于临时处理是没问题的，但与存储状态以备后用根本不兼容。然而，如果将Tlv数据结构转换为拥有其内容：

```rust
#[derive(Clone, Debug)]
pub struct Tlv {
    pub type_code: u8,
    pub value: Vec<u8>, // owned heap data
}
```

并且相应地调整 get_next_tlv() 代码，包括对 .to_vec() 的额外调用：

```rust
// ...
let tlv = Tlv {
    type_code,
    // Copy the relevant chunk of data to the heap.
    // The length field in the TLV is a single `u8`,
    // so this copies at most 256 bytes.
    value: input[2..2 + len].to_vec(),
};
```

这样一来，服务器代码的工作就轻松多了。拥有数据的Tlv结构没有生命周期参数，所以服务器数据结构也不需要，两种处理循环的变体都能正常工作。

## 谁害怕可怕的复制？

程序员过于迷恋减少复制的一个原因是，Rust通常使复制和内存分配显式化。像 .to_vec() 或 .clone() 这样的方法的显式调用，或像 Box::new() 这样的函数的调用，清楚地表明了复制和内存分配正在发生。这与C++ 形成了鲜明的对比，在C++ 中，很容易无意中写出在复制构造函数或赋值运算符下偷偷进行内存分配的代码。

使分配或复制操作可见而不是隐藏，并不是将其优化掉的好理由，尤其是如果这样做是以牺牲可用性为代价的话。在许多情况下，首先关注可用性更有意义，只有在性能真正成为问题并且基准测试（参考[第 30 条]）表明减少复制将产生显著影响时，才进行微调以达到最佳效率。

此外，代码的效率通常只有在需要扩展大规模使用时，才显得重要。如果事实证明代码的权衡是错误的，并且无法很好地应对数百万用户的使用 —— 好吧，那确实是一个不错的问题。

然而，还有几个具体的要点需要记住。首先是隐藏在“通常”这个词后面的，当指出复制通常是可见的时候。一个重要例外是Copy类型，编译器会无声地随意进行复制，从移动语义切换到复制语义。因此，[第 10 条]中的建议在这里值得重复：除非位复制是有效且快速的，否则不要实现Copy。但反过来说也成立：如果位复制是有效且快速的，那么请考虑实现Copy。例如，如果不携带额外数据的枚举类型派生自Copy，通常会更容易使用。

第二个可能相关的点是与no_std使用的潜在权衡。[第 33 条]建议，通常只需进行少量修改就可以编写与no_std兼容的代码，完全避免内存分配的代码使这一点更加简单。然而，针对支持堆分配的no_std环境（通过[第 33 条]中描述的alloc库）可能是可用性和no_std支持之间最佳平衡。

## 引用和智能指针

> 直到最近，我试着做了一个实验，不去纠结假设中的完美代码。相反，当我需要时，我会调用 .clone()，并使用Arc将本地对象更平滑地传递给线程和futures。
> 
> 这感觉非常美妙。
> ——乔希·特里普莱特（https://oreil.ly/1ViCT）


设计一个数据结构，使其拥有自己的内容，确实可以带来更好的用户体验，但如果多个数据结构需要使用相同的信息，仍然可能存在问题。如果数据是不可变的，那么每个地方都有自己的副本是可以的，但如果信息可能会改变（这在非常常见的情况下），那么多个副本意味着有多个需要同步更新的地方。

使用 Rust 的智能指针，例如 Rc 和 Arc，能有效地解决一些问题。它们允许设计从单一所有者模型转变为共享所有者模型。Rc（用于单线程代码）和 Arc（用于多线程代码）智能指针通过引用计数支持这种共享所有权模型。如果需要可变性，它们通常与一个允许内部可变性的内部类型配对，这一过程独立于 Rust 的借用检查规则：

* RefCell: 用于实现单线程代码中的内部可变性，常见的组合形式是 Rc<RefCell<T>> 。
* Mutex: 用于实现多线程代码中的内部可变性时（参考[第 17 条]），常见的组合形式是 Arc<Mutex<T>>。

这个转换在[第 15 条]的 GuestRegister 示例中有更详细的介绍，但这里的重点是，你不必将 Rust 的智能指针视为最后的手段。如果你的设计使用智能指针而不是复杂的相互连接的引用生命周期，这并不意味着承认失败 —— 智能指针可以带来更简单、更可维护、更易用的设计。

<!-- 参考链接 -->

[^1]: 在 Rust 语言中，你不能将字段命名为 “type”，因为这是一个被保留的关键字。如果你确实需要使用这个名称，可以通过在前面加上 r# 来绕过这个限制（https://oreil.ly/oC8VO），比如将字段命名为 r#type: u8。但大多数情况下，更简单的方法是选择一个不同的字段名称。

[第 10 条]: ../chapter_2/item10-std-traits.md
[第 15 条]: ../chapter_3/item15-borrows.md
[第 17 条]: item17-deadlock.md
[第 30 条]: ../chapter_5/item30-write-more-than-unit-tests.md
[第 33 条]: ../chapter_6/item33-no-std.md
