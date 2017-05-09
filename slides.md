---
title: Rust Lang Ergonomics Initiative
---

## The Rust Language Ergonomics Initiative

_@withoutboats_

---

## Who Am I?

* Rust user since late 2014

* Lang team member since October 2016

* Since March, supported by a modest contract from Mozilla

---

## What does the language team do?

Note:
Explain project structure, how lang team's role is different

---

## My lang team projects

* RFC shepherding - especially const parameters and associated type constructors

* chalk - a refactor of the trait system based on logic programming

* The Ergonomics Initiative

---

## RFCs

(Const generics)
```rust
impl<T: Clone, const LEN: usize> Clone for [T; LEN] {
    ...
}
```

(Associated type constructors)
```rust
trait Iterable {
    type Item;
    type Iter<'a>: Iterator<Item = &'a Self::Item>;
    fn iter<'a>(&'a self) -> Self::Iter<'a>;
}
```

---

## chalk

Rust:

```rust
impl Clone for i32 { }

impl<T: Clone> Clone for Vec<T> { }
```

Chalk:

```
i32: Clone;

forall<T> { (Vec<T>: Clone) :- T: Clone; }
```

Note:
Explain operating on the AST vs lowering to chalk

Advantages:

* much better caching
* more theoretically grounded, easier to extend to new features

---

# The Ergonomics Initiative

Note:
Start with philosophical comments, then we'll go through some of the features we're considering

---

> Explicit is better than implicit

Note:
this statement is from the Zen of Python

I want to interrogate this idea

---

## Explicit is better than implicit

```rust
fn main() {
    println!("Hello, world.")
}
```

Note:
Let's transform this program to be more explicit.

We can start by expanding the macro to regular Rust code, since
macros always hide information.

---

## Explicit is better than implicit

```rust
use std::io::{self, Write};

fn main() {
    io::stdout().write(b"Hello, world.\n").unwrap();
}
```

Note:
So this is now more explicit. It contains more information
that we had hidden in println - that we're writing to stdout,
and we insert a newline character.

But method syntax is a kind of implicitness. We can make it
more explicit by using the cannonical method names.

---

## Explicit is better than implicit

```rust
use std::io::{self, Write};

fn main() {
    let result = <io::Stdout as Write>::write(
        &mut io::stdout(),
        b"Hello, world.\n",
    );
    <Result<usize, io::Error>>::unwrap(result);
}
```

Note:
So now this is more explicit now (explain syntax).

But the thing about it is that namespacing is also implicit.
You have to look at the top of the file to find out what `io`
actually means. So we could transform this to use absolute
paths for everything.

---

## Explicit is better than implicit

```rust
fn main() {
    let result = <::std::io::Stdout as ::std::io::Write>::write(
        &mut ::std::io::stdout(),
        b"Hello, world.\n",
    );
    <::std::result::Result<usize, ::std::io::Error>>::unwrap(result);
}
```

Note:
Now this is a very explicit program. The only thing that's still
a bit implicit is this byte string syntax - it implicitly converts
those characters into ASCII bytes. So let's fix that...

---

## Explicit is better than implicit

```rust
fn main() {
    let result = <::std::io::Stdout as ::std::io::Write>::write(
        &mut ::std::io::stdout(),
        &[0x48u8, 0x65, 0x6c, 0x70, 0x20, 0x6d, 0x65, 0x0a] as &[u8],
    );
    <::std::result::Result<usize, ::std::io::Error>>::unwrap(result);
}
```

Note:
Now this is a very explicit program.

Since explicit is better than implicit, this is clearly a better program
than the original.

Of course not! If any of you have the ASCII table memorized and paid close
attention, you'd notice that this program doesn't print "Hello, world."

---

## Explicit is better than implicit

```
rustc 1.17.0 (56124baa9 2017-04-24)

----------------------------------------

Help me!
```

Note:
It prints "Help me!", because this would be a terrible way to program.

So I want to introduce some tools the lang team uses to make a more nuanced
judgment about when implicitness is okay.

---

## Reasoning Footprint

> How much [implicit] information do you need to understand what a particular line of code is doing? 

Note:
Reasoning footprint, from a blog post by Aaron Turon. Read quote.

---

## Reasoning Footprint

* **Applicability:** Is it easy to remember when this elision applies and when it doesn't?

* **Power:** How significantly does the information elided change what the program does?

* **Contextuality:** How much does the text around the elision determine what is elided? (as opposed to a general rule)

Note:
These are the three factors we consider, explain each of them.

---

## Scale of Rigor

* **Pre-rigorous:** A mental model of the system from user experience alone.

* **Rigorous:** A formal education in the real underlying rules of the system.

* **Post-rigorous:** An intuition for the system combining experience & formal understanding.

Note:
Another tool we use is the idea of 'building a mental model.' We think about it in terms
of these three phases of learning.

---

## Explicitness also costs

* The more information a source text contains, the more you need to know to understand what it does.

* Elisions make it easier to gain a *pre-rigorous* understanding of a piece of code.

Note:
And from this perspective, its important to remember that explicitness also costs.

Explicitness favors the rigorous understanding over the pre-rigorous and post-rigorous. It tends
to favor intermediate users over newer and expert users.

---

## Explicitness also costs

```rust
fn main() {
    println!("Hello, world.");
}
```

What is the most surprising thing about this program?

Note:
To give an example: hello world, most surprising? Its the bang.

Wrong intuition about side effects.

The real answer is macro, users may not even know what macros are.

---

# The Ergonomics Initative

* Ownership & Borrowing

* Traits

* Modules

Note:
Having laid that ground work, for the rest of this I'm going to go
through some of the features we're considering, divided into three
areas.

---

# Ownership & Borrowing

---

## Ownership & Borrowing

> **Goal:** reduce the degree to which new users 'fight with the borrowchecker'

> **Goal:** improve edit-compile-debug cycle by creating less opportunities for silly errors

---

## Non-lexical lifetimes

```rust
// error[E0502]: cannot borrow `*map` as mutable
// because it is also borrowed as immutable

fn get_default(map: &mut HashMap<i32, String>, key: i32) -> &String {
    if let Some(value) = map.get(&key) {
//                       --- immutable borrow occurs here       
        value
    } else {
        map.insert(key, String::default());
//      ^^^ mutable borrow occurs here
        map.get(&key).unwrap()
    }
//  - immutable borrow ends here
}
```

Note:
Get-or-insert hash map problem.

Explain code on the screen.

Mention the entry API.

---

## Non-lexical lifetimes

```rust
// error[E0502]: cannot borrow `*map` as mutable
// because it is also borrowed as immutable

fn push_len(vec: &mut Vec<usize>) {
    vec.push(vec.len());
//  ---      ^^^      - mutable borrow ends here
//  |        |
//  |        immutable borrow occurs here
//  mutable borrow occurs here
}
```

Note:
Push length of a vector into the vector

Explain code on the screen.

---

## Lifetime elisions

```rust
fn foo(slice: &[i32], idx: &usize) -> &'slice i32 {
    &slice[*idx]
}
```

Note:
Lifetime elisions, the 'identifier syntax.

---

## Lifetime elisions

```rust
struct Ref<', T> {
    inner: &T,
}

struct Foo<'> {
    bar: &str,
    baz: Ref<', i32>,
}

impl Foo<'> {
    ...
}
```

Note:
Eliding lifetimes in struct and impl blocks with the `'`
or other syntax.

---

## Lifetime elisions

```rust
struct Ref&<T> {
    inner: &T,
}

struct Foo& {
    bar: &str,
    baz: Ref&<i32>,
}

impl Foo& {
    ...
}
```

Note:
A more radical syntactic idea: make these 'reference types'
which end with an ampersand.

---

## Refs in, refs out

Note:
This is a principle that could help make borrowing easier to learn.

If you start with a reference and then do something to it, you should get a reference out.

---

## for loops

```rust
fn greet(guests: &[String]) {
    for name in guests {
        // This works. name is a &String

        println!("Greetings, {}!", name):
    }
}
```

Note:
An example where this works right now - for loops.

Explain the code.

If you loop over a reference to something, the elements are references.

---

## Patterns

```rust
fn greet(guest: &Option<String>) {
    if let Some(name) = guest {
//         ^^^^^^^^^^ expected reference, found enum `Option`
//              |
//              |
//              | cannot move out of borrowed context
        println!("Greetings, {}!", name);
    }
}
```

(RFC 1944)

Note:
Here's an example where this doesn't work.

Explain the code and the error.

---

## Patterns

* When a **&T** is destructured with a **T** pattern, bindings are made *by reference*.

* You can override this with a **move** binding (similar to **ref** bindings.

(RFC 1944)

Note:
We could make this work.

---

## Field access

```rust
struct Foo {
    bar: String,
}

impl Foo {
    fn bar(&self) -> &String {
        self.bar
//      ^^^^^^^^ expected reference, found struct `String`
    }
}
```

Note:
The same thing applies to field access - you can't move a field out
of a reference, but that's what the syntax does by defualt.

Changing this is a breaking change if the field is Copy.

---

## References to Copy types

```rust
fn increment(x: &i32) -> i32 {
    // This works (Because &i32 implements Add<i32>.)
    x + 1
}

fn identity(x: &i32) -> i32 {
    x
//  ^ expected i32, found &i32
}
```

Note:
But we could also fix that by automatically dereferencing Copy types.

This has its own uses.

---

## String literals

```rust
// If len would be greater than capacity, we reallocate data.
String = {
    data: *ptr [u8], // Into the heap
    len: usize,
    capacity: usize,
}

&'static str = {
    data: *ptr [u8], // Into the rodata section of the binary
    len: usize,
}
```

Note:
Strings are another common problem - new users have a lot of
trouble understanding the two types of strings.

This is the memory layout of the two types. (Explain).

We could coerce string literals to String types.

---

## String literals

* We add a capacity of `0`, keep the pointer to rodata

* Whenever we mutate we reallocate into the heap.

---

## Argument coercions

```rust
fn sum(x: &Vec<i32>) -> i32 {
    x.iter().sum()
}

fn main() {
    sum(vec![0, 1, 2, 3])
//      ^^^^^^^^^^^^^^^^ expected reference, found struct `Vec`
}
```

Note:
Explain code, Autoref arguments, similar to methods.

---

## Argument coercions

* If a function takes **&T**, accept **T** (like methods do today).

* Unlike methods, *move* the **T**, even though the function doesn't take ownership.

Note:
Explain ownership caveat.

---

# Traits

---

## Traits

> **Goal:** reduce syntactic overhead of generics, making them easier to understand and use

---

## Implied bounds

```rust
struct HashMap<K: Hash + Eq, V> {
    ...
}

impl<K, V> HashMap<K, V> 
//         ^^^^^^^^^^^^^ the trait `Eq` is not implemented for `K`
//         ^^^^^^^^^^^^^ the trait `Hash` is not implemented for `K`

impl<K: Clone, V: Clone> HashMap<K, V>
//                       ^^^^^^^^^^^^^ 
//                       | the trait `Eq` is not implemented for `K`
//                       | the trait `Hash` is not implemented for `K`
```


---

## Trait aliases

```rust
trait Convert<T> = Into<T> + AsRef<T> + AsMut<T>;

trait HttpService = Service<Request = http::Request,
                            Response = http::Response,
                            Error = http::Error>;

```

(RFC 1733)

---

## impl Trait

```rust
fn process<I: Iterator<Item=u32>(iter: I) -> Box<Iterator<Item=u32>> {
    Box::new(iter.filter(|x| x % 2 == 0).map(|x| x * 2))
}

impl Option<T> {
    fn map<U, F: FnOnce(T) -> U>(self, f: F) -> U) -> Option<U> {
        match self {
            Some(t) => Some(f(t)),
            None    => None,
        }
    }
}
```

---

## impl Trait

```rust
fn process(iter: impl Iterator<Item=u32>) -> impl Iterator<Item=u32> {
    iter.filter(|x| x % 2 == 0).map(|x| x * 2)
}

impl Option<T> {
    fn map<U>(self, f: impl FnOnce(T) -> U) -> Option<U> {
        match self {
            Some(t) => Some(f(t)),
            None    => None,
        }
    }
}
```

---

## virtual Trait

```rust
impl<'a> MyTrait for Iterator<Item = &'a str> {
//       ^^^^^^^ the trait `Sized` is not implemented
//               for `Iterator<Item=&'a str>`
}

fn process(iter: Iterator<Item = u32>) -> Iterator<Item = u32> {
//         ^^^^ the trait `Sized` is not implemented
//              for `Iterator<Item=&'a str>`
}
```

---

## virtual Trait

```rust
impl<'a> MyTrait for Iterator<Item = &'a str> {
    // bare "Iterator" means "T where T: Iterator"
}

fn process(iter: Iterator<Item = u32>) -> Iterator<Item = u32> {
    // (the same as impl Trait)
}

fn process_virtual(iter: Box<virtual Iterator<Item = u32>>) {
    // the "virtual" keyword makes it into a trait object
}
```

---

# Modules

---

## Modules

> **Goal:** Reduce module boilerplate.

> **Goal:** Make it easier to form a mental model of the module system.

---

## File name flexibility

```
src
├─ lib.rs
├─ foo.rs
└─ foo
    ├─ bar.rs
    └─ baz.rs
```

```rust
// src/foo.rs
pub mod bar;
pub mod baz;
// error: cannot declare a new module at this location
```

---

## Absolute vs relative paths

```rust
// src/main.rs
fn main() {
    let x = std::collections::HashMap::new();
}
```

```rust
// src/foobar.rs
fn foobar() {
    let x = std::collections::HashMap::new();
            ^^^ Use of undeclared type or module `std
}
```

---

## pub(restricted)

```rust
// Visible anywhere inside this crate.
pub(crate) struct Foo {
    ...
}

// Visible only in the super-module.
pub(super) struct Bar {
    ...
}
```

(Stable in 1.18.)

---

## extern crate

```toml
[dependencies]
serde = "1.0.0"
```

```rust
use serde::Serialize;
//  ^^^^^^^^^^^^^^^^ Maybe a missing `extern crate serde;`?

fn foo<T: Serialize>(arg: T) {
    ...
}
```

---

## mod declarations

```
src
├─ lib.rs
└─ foo.rs
```

```rust
// src/foo.rs

pub struct Foo;
```

```rust
// src/lib.rs

use foo::Foo;

pub struct Bar(Foo);
```

---

## mod declarations

The problem is re-exports:

```rust
// src/lib.rs

pub use foo::Foo;
// many users `facade`, and don't want Foo
// to be accessible at foo::Foo?

pub struct Bar(Foo);
```

---

# Epochs

---

## Epoch declaration

* Make some breaking changes, but *only* if you specify the new epoch.

```toml
[package]
name = "my-awesome-package"
authors = ["Me"]
version = "1.2.7"
epoch = "2018"
```

* Breaking changes should be easy to upgrade across, ideally automatically.

---

Thanks 💖
