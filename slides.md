---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
#background: https://cover.sli.dev
background: https://images.unsplash.com/photo-1656863678565-b7576b9363bb?q=80&w=3270&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
# some information about your slides, markdown enabled
title: Ownership in rust-lang
info: |
  ## Slidev Starter Template
# apply any unocss classes to the current slide
class: text-center
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# https://sli.dev/guide/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/guide/syntax#mdc-syntax
mdc: true
---

# Ownership in rust-lang

<br>
<br>
or.. 'How I learned to love the borrow checker'

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    <carbon:arrow-right class="inline"/>
  </span>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: slide-left
---

# Table of contents

<Toc v-click minDepth="1" maxDepth="2"></Toc>

---
transition: slide-up
---

# Scope

Before considering ownership, we must first understand what a _scope_ is.

<v-clicks class="text-x1">

- Typically demarcated by '\{ ... \}'
- Two varieties of scope
  - 'Global' (or _static_)
  - 'Execution' (within some '\{ ... \}')

```rust {*|2}
// first line of a source file
static FOO:&'static = "foo";
// ...
```

```rust {*|3}
// ...
fn foobar() {
  let bar: usize = 0;
}
// ...
```

This is true when considering a single source file; Namespaces make this slightly different.

</v-clicks>

---
transition: slide-up
---

# Ownership

All variable bindings in rust have _ownership_ of what they're bound to.

```rust {all|2}
fn sum() -> u32 {
  let numbers = vec![0xcafe, 0xbeef];
  // ...
}
```

<v-clicks class="text-x1">

Consider ``numbers``.

What happens when ``numbers`` comes in to scope?

What happens when we leave that scope?

- A new ``Vector`` is created on the stack
- Space for its' elements are allocated on the heap
- Upon leaving that scope..
  - .. heap-allocated elements are cleaned up 
  - .. followed by the stack allocated ``Vector``

</v-clicks>

---
transition: slide-left
level: 2
---

# Ownership::example

Let's move ``numbers`` outside of ``sum()`` and accept it as a parameter.

<div v-click="5">

Now let's take another binding of ``numbers``

</div>

````md magic-move {lines: true}

```rust {*|2|*}
fn sum() -> u32 {
  let numbers = vec![0xcafe, 0xbeef];
  // ...
}
```

```rust {*|6,7}
fn sum(numbers: Vec<u32>) -> u32 {
  // ...
}

fn main() {
  let numbers = vec![0xcafe, 0xbeef];
  let rv = sum(numbers);
}
```

```rust {*|6|6,7}
fn sum(numbers: Vec<u32>) -> u32 {
  // ...
}

fn main() {
  let numbers = vec![0xcafe, 0xbeef];
  let other_numbers = numbers;
  let rv = sum(numbers);
}
```

```bash {*|6,7|8,9|6,7,8,9|*}
error[E0382]: use of moved value: `numbers`
 --> /tmp/foo.rs:8:18
  |
6 |     let numbers = vec![0,1,2,3,4];
  |         ------- move occurs because `numbers` has type `Vec<u32>`, which does not implement the `Copy` trait
7 |     let other_numbers = numbers;
  |                         ------- value moved here
8 |     let rv = sum(numbers);
  |                  ^^^^^^^ value used here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
7 |     let other_numbers = numbers.clone();
  |                                ++++++++
```

```rust {*|7,8,9}
fn sum(numbers: Vec<u32>) -> u32 {
  // ...
}

fn main() {
  let numbers = vec![0xcafe, 0xbeef];
  {
    let other_numbers = numbers;
  }
  let rv = sum(numbers);
}
```

````

<div v-click="8">

Why does the toolchain emit an error when rebinding ``numbers``?

</div>
<div v-click="12">

Let's fix it.

</div>


---
level: 2
---

# Ownership::rules

.

There are three 'key' rules to be aware of:

<v-clicks>

1. Every value has an owner
2. **There can be only one**
3. When the owner goes out of scope, the value it owned is ``drop``ped

</v-clicks>

<div v-click="4">

```rust
fn sum(numbers: Vec<u32>) -> u32 {
  // ...
}

fn main() {
  let numbers = vec![0xcafe, 0xbeef];
  {
    let other_numbers = numbers;
  }
  let rv = sum(numbers);
  println!("{:?}", numbers);
}
```

</div>

---
level: 2
---

# Ownership::rules continued

Making it work

````md magic-move {lines: true}

```rust {*|*|10|11|10,11}
fn sum(numbers: Vec<u32>) -> u32 {
  // ...
}

fn main() {
  let numbers = vec![0xcafe, 0xbeef];
  {
    let other_numbers = numbers;
  }
  let rv = sum(numbers);
  println!("{:?}", numbers);
}
```

```rust {*|*|10|11|10,11}
fn sum(numbers: Vec<u32>) -> Vec<u32> {
  // ...
}

fn main() {
  let numbers = vec![0xcafe, 0xbeef];
  {
    let other_numbers = numbers;
  }
  numbers = sum(numbers);
  println!("{:?}", numbers); // Ok!
}
```
````

<div v-click="9">

Obviously this feels wrong; We can solve this by using references, and _borrowing_.

</div>

---
transition: slide-right
---

# Borrowing

.. and references

<v-clicks>

References allow us to keep ownership of a value while still allowing some other scope access to it.

Can be generally thought of as C/C++ pointers, **not C/C++ references**.

References come in two forms:

Immutable
```rust
  let num = 0;
  let num_ref = &num;
```

Mutable
```rust
  let num = 0;
  let num_ref = &mut num;
```

</v-clicks>

---
level: 2
---

# Borrowing::example_1

Printing a vector

````md magic-move {lines: true}
```rust {*|1,2,3,4|6-10|*}
fn print_vec(vec: Vector<u32>) -> Vector<u32> {
  println!("{:?}", vec); // Ok, we have ownership
  vec
}

fn main() {
  let vec = vec![0,1,2];
  let vec = print_vec(vec); // Shadow the previous binding
  println!("{:?}", vec); // Ok! `print_vec()` returns ownership
}
```

```rust {*|1-3|7|*}
fn print_vec(vec: &Vector<u32>) -> () {
  println!("{:?}", vec); // Ok, we are borrowing ownership
}

fn main() {
  let vec = vec![0,1,2];
  print_vec(&vec);
  println!("{:?}", vec); // Ok! `print_vec()` borrow has finished
}
```
````

---
level: 2
---

# Borrowing::example_2

Modifying a vector

````md magic-move {lines: true}
```rust {*|1,2,3,4|6-10|*}
fn push_vec(vec: Vector<u32>, val: u32) -> Vector<u32> {
  vec.append(val); // Ok, we have taken ownership
  vec
}

fn main() {
  let vec = vec![0,1,2];
  let vec = push_vec(vec, 3); // Shadow the previous binding
  println!("{:?}", vec); // Ok! `push_vec()` returns ownership, we'll see §[0, 1, 2, 3]§
}
```

```rust {*|1-3|7|8|*}
fn push_vec(vec: &mut Vector<u32>, val: u32) -> () {
  vec.append(val); // Ok, we have borrowed ownership via mutable reference
}

fn main() {
  let mut vec = vec![0,1,2];
  vec_push(&mut vec, 3);
  println!("{:?}", vec); // Ok! `push_vec()` borrow has finished, we'll see §[0, 1, 2, 3]§
}
```

````

---
level: 2
---

# Borrowing::rules

Simple, right?

<v-clicks>

1. Any borrow must last for no longer than the scope of the owner
2. You may have one of these two kinds of borrow, but not both at the same time:
   - one or more references (``&T``) to a value
   - exactly one mutable reference (``&mut T``) to a value

In other words:

- As many things can read from a resource at once as long as they're not writing to it.
- Only one thing can write to a resource at a time.

This is very similar to the definition of a _data race_:

> There is a ‘data race’ when two or more pointers access the same memory location at the same time, where at least one of them is writing, and the operations are not synchronized.

</v-clicks>

---
level: 2
---

# Borrowing::rules continued

..

<v-clicks>

Examples given previously are all single threaded;

Rust provides these rules such that it can check _at compile time_ that the same is true in a multi-threaded context.

</v-clicks>

---
transition: slide-down
---

# Lifetimes

A language construct to reason about scopes

Scopes tell us when a value is no longer being used;

When the owner is no longer in scope, its' value is ``drop``ed much like a destructor in C++.

The drop behaviour is customisable for non-trivial data types by implementing the ``Drop`` trait

How can we reason about the lifetime of a parameter to a function?

How do we describe that a function accepts two parameters that may have separate lifetimes?
 
---
level: 2
---

# Lifetimes::continued

..

Lending references to a resource can be complicated; Consider the following:

1. I acquire a handle to some kind of resource.
2. I lend you a reference to the resource.
3. I decide I’m done with the resource, and deallocate it, while you still have your reference.
4. You decide to use the resource.

This is often referred to as a '_dangling pointer_'.

Rust's ownership model makes sure that step ``4.`` _never happens before step ``3.``_ via _lifetimes_.

---
level: 2
---

# Lifetimes::syntax

.. 

Lifetimes are present in all examples given in this presentation, however they have been _implied_.

We can also explicitly denote the lifetimes of references:

```rust {*|1-3|5-7|*}
// implicit
fn foo(x: &i32) {
}

// explicit
fn bar<'a>(x: &'a i32) {
}
```

.. where ``'a`` is read as "the lifetime of ``a``".

---
level: 2
---

# Lifetimes::syntax continued

.. 

<v-clicks>

If your struct contains a reference, you'll likely need to denote the lifetime:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5; // this is the same as `let _y = 5; let y = &_y;`
    let f = Foo { x: y };

    println!("{}", f.x);
}
```

</v-clicks>

---
level: 2
---

# Lifetimes::syntax continued

.. 

<v-clicks>

In brief:

```rust
struct Foo<'a> {
```

declares a lifetime, and:

```rust
x: &'a i32,
```

uses it.

Why do we need a lifetime here?

.. to ensure that any reference to a ``Foo`` cannot outlive the reference to an ``i32`` it contains.
</v-clicks>