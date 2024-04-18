# Item 1: Use the type system to express your data structures

"who called them programers and not type writers" – @thingskatedid

The basics of Rust's type system are pretty familiar to anyone coming from another statically typed programming language (such as C++, Go or Java). There's a collection of integer types with specific sizes, both signed (i8, i16, i32, i64, i128) and unsigned (u8, u16, u32, u64, u128).

There's also signed (isize) and unsigned (usize) integers whose size matches the pointer size on the target system. Rust isn't a language where you're going to be doing much in the way of converting between pointers and integers, so that characterization isn't really relevant. However, standard collections return their size as a usize (from .len()), so collection indexing means that usize values are quite common – which is obviously fine from a capacity perspective, as there can't be more items in an in-memory collection than there are memory addresses on the system.

The integral types do give us the first hint that Rust is a stricter world than C++ – attempting to put a quart (i32) into a pint pot (i16) generates a compile-time error.

```rust
        let x: i32 = 42;
        let y: i16 = x;
error[E0308]: mismatched types
  --> use-types/src/main.rs:14:22
   |
14 |         let y: i16 = x;
   |                ---   ^ expected `i16`, found `i32`
   |                |
   |                expected due to this
   |
help: you can convert an `i32` to an `i16` and panic if the converted value doesn't fit
   |
14 |         let y: i16 = x.try_into().unwrap();
   |                       ++++++++++++++++++++
```

This is reassuring: Rust is not going to sit there quietly while the programmer does things that are risky. It also gives an early indication that while Rust has stronger rules, it also has helpful compiler messages that point the way to how to comply with the rules. The suggested solution raises the question of how to handle situations where the conversion would alter the value, and we'll have more to say on both error handling (Item 4) and using panic! (Item 18) later.

Rust also doesn't allow some things that might appear "safe":

```rust
        let x = 42i32; // Integer literal with type suffix
        let y: i64 = x;
error[E0308]: mismatched types
  --> use-types/src/main.rs:23:22
   |
23 |         let y: i64 = x;
   |                ---   ^ expected `i64`, found `i32`
   |                |
   |                expected due to this
   |
help: you can convert an `i32` to an `i64`
   |
23 |         let y: i64 = x.into();
   |                       +++++++
```

Here, the suggested solution doesn't raise the spectre of error handling, but the conversion does still need to be explicit. We'll discuss type conversions in more detail later (Item 6).

Continuing with the unsurprising primitive types, Rust has a bool type, floating point types (f32, f64) and a unit type () (like C's void).

More interesting is the char character type, which holds a Unicode value (similar to Go's rune type). Although this is stored as 4 bytes internally, there are again no silent conversions to or from a 32-bit integer.

This precision in the type system forces you to be explicit about what you're trying to express – a u32 value is different than a char, which in turn is different than a sequence of UTF-8 bytes, which in turn is different than a sequence of arbitrary bytes, and it's up to you to specify exactly which you mean1. Joel Spolsky's famous blog post can help you understand which you need.

Of course, there are helper methods that allow you to convert between these different types, but their signatures force you to handle (or explicitly ignore) the possibility of failure. For example, a Unicode code point2 can always be represented in 32 bits, so 'a' as u32 is allowed, but the other direction is trickier (as there are u32 values that are not valid Unicode code points):

char::from_u32 returns an Option<char> forcing the caller to handle the failure case
char::from_u32_unchecked makes the assumption of validity, but is marked unsafe as a result, forcing the caller to use unsafe too (Item 16).
Aggregate Types
Moving on to aggregate types, Rust has:

Arrays, which hold multiple instances of a single type, where the number of instances is known at compile time. For example [u32; 4] is four 4-byte integers in a row.
Tuples, which hold instances of multiple heterogeneous types, where the number of elements and their types are known at compile time, for example (WidgetOffset, WidgetSize, WidgetColour). If the types in the tuple aren't distinctive – for example (i32, i32, &'static str, bool) – it's better to give each element a name and use…
Structs, which also hold instances of heterogeneous types known at compile time, but which allows both the overall type and the individual fields to be referred to by name.
The tuple struct is a cross-breed of a struct with a tuple: there's a name for the overall type, but no names for the individual fields – they are referred to by number instead: s.0, s.1, etc.

```rust
    struct TextMatch(usize, String);
    let m = TextMatch(12, "needle".to_owned());
    assert_eq!(m.0, 12);
```

This brings us to the jewel in the crown of Rust's type system, the enum.

In its basic form, it's hard to see what there is to get excited about. As with other languages, the enum allows you to specify a set of mutually exclusive values, possibly with a numeric or string value attached.

```rust
    enum HttpResultCode {
        Ok = 200,
        NotFound = 404,
        Teapot = 418,
    }
    let code = HttpResultCode::NotFound;
    assert_eq!(code as i32, 404);
```

Because each enum definition creates a distinct type, this can be used to improve readability and maintainability of functions that take bool arguments. Instead of:

```rust
        print_page(/* both_sides= */ true, /* colour= */ false);
```

a version that uses a pair of enums:

```rust
    pub enum Sides {
        Both,
        Single,
    }

    pub enum Output {
        BlackAndWhite,
        Colour,
    }

    pub fn print_page(sides: Sides, colour: Output) {
        // ...
    }
```

is more type-safe and easier to read at the point of invocation:

```rust
        print_page(Sides::Both, Output::BlackAndWhite);
```

Unlike the bool version, if a library user were to accidentally flip the order of the arguments, the compiler would immediately complain:

```rust
error[E0308]: mismatched types
  --> use-types/src/main.rs:89:20
   |
89 |         print_page(Output::BlackAndWhite, Sides::Single);
   |                    ^^^^^^^^^^^^^^^^^^^^^ expected enum `enums::Sides`, found enum `enums::Output`
error[E0308]: mismatched types
  --> use-types/src/main.rs:89:43
   |
89 |         print_page(Output::BlackAndWhite, Sides::Single);
   |                                           ^^^^^^^^^^^^^ expected enum `enums::Output`, found enum `enums::Sides`
```

(Using the newtype pattern (Item 7) to wrap a bool also achieves type safety and maintainability; it's generally best to use that if the semantics will always be Boolean, and to use an enum if there's a chance that a new alternative (e.g. Sides::BothAlternateOrientation) could arise in the future.)

The type safety of Rust's enums continues with the match expression:

This code does not compile!

```rust
        let msg = match code {
            HttpResultCode::Ok => "Ok",
            HttpResultCode::NotFound => "Not found",
            // forgot to deal with the all-important "I'm a teapot" code
        };
error[E0004]: non-exhaustive patterns: `Teapot` not covered
  --> use-types/src/main.rs:65:25
   |
51 | /     enum HttpResultCode {
52 | |         Ok = 200,
53 | |         NotFound = 404,
54 | |         Teapot = 418,
   | |         ------ not covered
55 | |     }
   | |_____- `HttpResultCode` defined here
...
65 |           let msg = match code {
   |                           ^^^^ pattern `Teapot` not covered
   |
   = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
   = note: the matched value is of type `HttpResultCode`
```

The compiler forces the programmer to consider all of the possibilities3 that are represented by the enum, even if the result is just to add a default arm _ => {}. (Note that modern C++ compilers can and do warn about missing switch arms for enums as well.)

enums With Fields
The true power of Rust's enum feature comes from the fact that each variant can have data that comes along with it, making it into an algebraic data type (ADT). This is less familiar to programmers of mainstream languages; in C/C++ terms it's like a combination of an enum with a union – only type-safe.

This means that the invariants of the program's data structures can be encoded into Rust's type system; states that don't comply with those invariants won't even compile. A well-designed enum makes the creator's intent clear to humans as well as to the compiler:

```rust
pub enum SchedulerState {
    Inert,
    Pending(HashSet<Job>),
    Running(HashMap<CpuId, Vec<Job>>),
}
```

Just from the type definition, it's reasonable to guess that Jobs get queued up in the Pending state until the scheduler is fully active, at which point they're assigned to some per-CPU pool.

This highlights the central theme of this Item, which is to use Rust's type system to express the concepts that are associated with the design of your software.

A dead giveaway for when this is not happening is a comment that explains when some field or parameter is valid:

```rust
struct DisplayProps {
    x: u32,
    y: u32,
    monochrome: bool,
    // `fg_colour` must be (0, 0, 0) if `monochrome` is true.
    fg_colour: RgbColour,
}
```

This is a prime candidate for replacement with an enum holding data:

```rust
#[derive(Debug)]
enum Colour {
    Monochrome,
    Foreground(RgbColour),
}

struct DisplayProperties {
    x: u32,
    y: u32,
    colour: Colour,
}
```

This small example illustrates a key piece of advice: make invalid states inexpressible in your types. Types that only support valid combinations of values mean that whole classes of error are rejected by the compiler, leading to smaller and safer code.

Options and Errors
Returning to the power of the enum, there are two concepts that are so common that Rust includes built-in enum types to express them.

The first is the concept of an Option: either there's a value of a particular type (Some(T)), or there isn't (None). Always use Option for values that can be absent; never fall back to using sentinel values (-1, nullptr, …) to try to express the same concept in-band.

There is one subtle point to consider though. If you're dealing with a collection of things, you need to decide whether having zero things in the collection is the same as not having a collection. For most situations, the distinction doesn't arise and you can go ahead and use Vec<Thing>: a count of zero things implies an absence of things.

However, there are definitely other rare scenarios where the two cases need to be distinguished with Option<Vec<Thing>> – for example, a cryptographic system might need to distinguish between "payload transported separately" and "empty payload provided". (This is related to the debates around the NULL marker columns in SQL.)

One common edge case that's in the middle is a String which might be absent – does "" or None make more sense to indicate the absence of a value? Either way works, but Option<String> clearly communicates the possibility that this value may be absent.

The second common concept arises from error processing: if a function fails, how should that failure be reported? Historically, special sentinel values (e.g. -errno return values from Linux system calls) or global variables (errno for POSIX systems) were used. More recently, languages that support multiple or tuple return values (such as Go) from functions may have a convention of returning a (result, error) pair, assuming the existence of some suitable "zero" value for the result when the error is non-"zero".

In Rust, always encode the result of an operation that might fail as a Result<T, E>. The T type holds the successful result (in the Ok variant), and the E type holds error details (in the Err variant) on failure. Using the standard type makes the intent of the design clear, and allows the use of standard transformations (Item 3) and error processing (Item 4); it also makes it possible to streamline error processing with the ? operator.

1: The situation gets muddier still if the filesystem is involved, since filenames on popular platforms are somewhere in between arbitrary bytes and UTF-8 sequences: see the std::ffi::OsString documentation.

2: Technically, a Unicode scalar value rather than a code point

3: This also means that adding a new variant to an existing enum in a library is a breaking change (Item 21): clients of the library will need to change their code to cope with the new variant. If an enum is really just an old-style list of values, this behaviour can be avoided by marking it as a non_exhaustive enum; see Item 21.