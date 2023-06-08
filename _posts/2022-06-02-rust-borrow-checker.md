---
title: Rust ownership model and the borrow checker.
author: Thomas Luijken
date: 2022-06-02 21:35:00 +0200
categories: [Borrow checker, Rust]
tags: [Rust, heap, stack, borrow checker, lifetimes, scopes]
---

Rust is a systems programming language that has gained popularity in recent
years due to its strong focus on safety, performance, and concurrency. One of
the key features that sets Rust apart from other languages is its borrow
checker. The borrow checker is a type of static analysis tool that helps prevent
common errors such as null pointer dereferences, use-after-free bugs, and data
races. In this blog post, we'll dive deep into the Rust borrow checker and
explore how it works, what benefits it provides, and how it can sometimes be
challenging to work with. Whether you're a seasoned Rustacean or just getting
started with the language, this post will provide valuable insights into one of
Rust's most powerful and unique features.

# The stack vs the heap.

It's important to note the distinction between memory that's allocated on the
stack and memory that's allocated on the heap.

## The stack.
Imagine the following code

```rust
fn main() {
    let var_a: i8 = 6;
    let var_b: f32 = 7.0;
    let var_c: bool = true;

    println!("{}, {}, {}", var_a, var_b, var_c);
}
```

You can see this as a sheet of paper, where containing the following items.
![image](/assets/img/getting_started_with_rust/01.png)


The stack now looks like this.
![image](/assets/img/getting_started_with_rust/02.png)

If we add a method and then call it, there will be a change.

```rust
fn main() {
    let var_a: i8 = 6;
    let var_b: f32 = 7.0;
    let var_c: bool = true;

    println!("{}, {}, {}", var_a, var_b, var_c);
}

fn function_something(){
    let var_x: bool = true;
    let var_y: i8 = 123;
}
```

When a method is called, a new scope is created. You can think of each new scope
as a fresh sheet of paper being added to the top of the stack.
![image](/assets/img/getting_started_with_rust/03.png)

And the stack now looks like this:
![image](/assets/img/getting_started_with_rust/04.png)

When a method is called, two new items are added to the stack memory to
represent the method's arguments and local variables. However, as soon as we
exit the method's scope by reaching the closing curly brace, everything inside
that scope is destroyed, including any values that were allocated on the stack
such as x and y. These values are no longer considered alive and cannot be
accessed or used after the scope has ended.
![image](/assets/img/getting_started_with_rust/05.png)

The stack now looks like this again.
![image](/assets/img/getting_started_with_rust/06.png)

It's worth noting that the compiler can preallocate memory for the stack since
all items we assign to have a fixed size. For example, the value var_a is of
type `i8`, so it will always use 8 bits in memory no matter how large the number
is. However, the stack can only be used for fixed-sized allocations, and items
on the stack are placed next to each other, leaving no room for expansion at
runtime. Accessing and copying values from/to the stack is therefore very fast
and cheap.

Rust always defaults to allocating memory on the stack whenever possible, unless
the programmer specifies otherwise. This contributes to Rust's memory safety and
performance gains. However, collections like String, which is a collection of
u8's, can grow in size and cannot be placed on the stack. For this reason, we
need to use the heap.

## The heap
The heap is used for values that can grow or shrink in size and need to be
passed around between scopes. You can think of the heap as a row of lockers that
can be used to store items of various sizes. When we allocate memory on the
heap, we request a block of memory of a certain size, and the operating system
finds a suitable spot to place that block.

Unlike the stack, which is managed automatically by the Rust compiler, heap
memory must be manually allocated and deallocated using special functions. This
introduces some complexities, such as the possibility of memory leaks or
dangling pointers, which can cause bugs and security vulnerabilities. However,
Rust's ownership and borrowing rules help to prevent these issues and ensure
that memory is managed safely and efficiently.

```rust
fn main() {
    let var_a: i8 = 6;
    let var_b: String = String::from("Hello");

    println!("{}, {}", var_a, var_b);
}
```

The String value is different from other fixed-size values like `i8` because it
can be modified, causing the size of the allocated memory to increase or
decrease. Therefore, it cannot be placed on the stack. Instead, `var_b` is placed
on the heap, where it can be represented as a pointer to the start of the
allocated memory block, along with metadata that specifies the size and capacity
of the block.

Heap-allocated values like `var_b` must be explicitly deallocated when they are no
longer needed. If a heap-allocated value is not properly deallocated, it can
lead to memory leaks or other issues. Rust's ownership and borrowing rules help
to prevent these issues by ensuring that heap-allocated values are properly
managed and deallocated when they are no longer needed.
![image](/assets/img/getting_started_with_rust/07.png)

As you can see, `var_b` now contains a pointer that references memory on the heap,
along with metadata that specifies the length and capacity of the allocated
block. If we were to change "Hello" to "Hell", the length of the allocated
memory for that item would be altered to 4.

Memory allocations on the heap are more expensive than memory allocations on the
stack because they require more management. In most modern programming
languages, a garbage collector takes care of this management. However, garbage
collection can introduce pauses in the program execution, making it unsuitable
for system programming.

Rust doesn't have a garbage collector, but it does take care of memory
management for you. This is where Rust's flagship feature, the ownership model,
comes in. The ownership model ensures that each value in memory has a unique
owner, and that ownership can be transferred between different parts of the
program. By doing so, Rust ensures that memory is managed efficiently and
without introducing any run-time overhead, making it a great choice for systems
programming.

# Rust ownership model.
The Rust ownership model has a few key features that make it unique and
powerful:

* There is only one owner for allocated memory, whether it's on the stack or the
  heap.
* Memory is always freed as soon as the owner is removed from the stack.

This simplicity allows Rust to guarantee memory safety without the need for a
garbage collector. When an owner is removed from the stack, Rust automatically
frees the memory associated with that owner. This makes Rust code both safe and
efficient, as memory is always managed correctly without any overhead from a
garbage collector.

The ownership model is enforced through Rust's borrowing system, which allows
for flexible and safe sharing of memory between different parts of the program.

In the next sections, we'll dive deeper into the details of Rust's ownership
model and borrowing system, and how they work together to make Rust code safe
and efficient.

Consider the following code:

```rust
fn main() {
    let var_a: i8 = 6;
    let var_b: i8 = var_a;
    println!("{}, {}", var_a, var_b);
}
```

Question: who is the owner of the initial value 6?

In this case a copy is made when assigning `var_b` of the value of `var_a`
Remember: it's so cheap and fast to have values on the stack, rust will just
copy the value to a new value to the stack whenever possible. So now, we have 2
values on the stack. `var_a` owns a value 6, and `var_b` also owns another value
of 6.

![image](/assets/img/getting_started_with_rust/08.png)

This also applies when moving stack allocations to other scopes like a method
call:

```rust
fn main() {
    let var_a = 8;
    some_other_function(var_a);
    println!("var_a = {}", var_a);
}

fn some_other_function(mut val: i8) {
    val += 9;
    println!("val = {}", val);
}
```

the output will be:

```shell
val = 17
var_a = 8
```

This is because a clone is made of `var_a`, and passed in to
`some_other_function` and given ownership to parameter `val`. There is no
correlation between `var_a` and `val` in this case. Remember this copy action,
we will see this later on some more.

# A more complex example

So if we have the following example:

```rust
fn main() {
    let var_a: i8 = 6;
    let var_b: String = String::from("Hello");

    println!("{}, {}", var_a, var_b);
}

fn some_other_function() {
    let var_x = String::from("World");
}

```

Our stack and heap memory now looks like this:
![image](/assets/img/getting_started_with_rust/09.png)

But as soon as we hit the end of the `some_other_function()` scope (at the curly braces),

```rust
fn main() {
    let var_a: i8 = 6;
    let var_b: String = String::from("Hello");

    println!("{}, {}", var_a, var_b);
}

fn some_other_function() {
    let var_x = String::from("World");
} // <--- ends the scope of some_other_function()

```
`var_x` is removed from the stack, and therefore its corresponding allocated
memory on the heap is also removed. This process of deallocating memory on the
heap is straightforward in Rust.

In Rust, ownership of allocated memory can be transferred or moved between
variables.

See the following example:

```rust
fn main() {
    let var_b: String = String::from("Hello");
    some_other_function(var_b);
}

fn some_other_function(input: String) {
    println!("{}", input);
}
```

Nothing new here, right? As expected, the string 'Hello' will be printed to the console.

However, something changed here. The ownership of the allocated memory moved from `var_b` to input.
Let's see what happens if we try to use `var_b` after the `some_other_function()` call'.

```rust
fn main() {
    let var_b: String = String::from("Hello");
    some_other_function(var_b);
    println!("{}", var_b);
}

fn some_other_function(input: String) {
    println!("{}", input);
}
```

After the ownership of the allocated memory for `var_b` is moved to the input
variable in the `some_other_function()` call, Rust does not allow `var_b` to be used
anymore. This is because the ownership of `var_b` has been transferred and it no
longer owns the memory. When trying to use `var_b` after the function call, the
compiler detects that it has been moved and will complain about a borrow after
the move error.

```shell
error[E0382]: borrow of moved value: `var_b`
 --> src/main.rs:4:20
  |
2 |     let var_b: String = String::from("Hello");
  |         ----- move occurs because `var_b` has type `String`, which does not implement the `Copy` trait
3 |     some_other_function(var_b);
  |                         ----- value moved here
4 |     println!("{}", var_b);
  |                    ^^^^^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0382`.
error: could not compile `moving_ownership` due to previous error
```

**WHAT???**

As mentioned earlier, in Rust, items are removed when their owner is removed. In
the example, we moved the ownership of "Hello" from the `var_b` variable to input
parameter. The input parameter only lives during the scope of the
`some_other_function()` method. As soon as we hit the end of the scope, the `input`
parameter no longer lives, and this "Hello" is removed from the heap.

Additionally, Rust's String object does not implement the `Copy` trait, which
means that it cannot automatically be copied when passing it in as a reference.
This is different from basic types like integers or booleans, which can be
copied easily.

## Sooooo, how do we fix this?

We could do a couple of things here.

* We could clone the "Hello" (a) value. We get another instance of "Hello" (b)
  on the heap, which is only alive during scope of `some_other_function` making
  the `input` parameter the owner of that reference. `var_b` will stay the owner
  of it's own copy (a) and a will be alive during the 3rd line where the output
  get's printed. However, this is expensive, and not necessary.

  ```rust
  fn main() {
      let var_b: String = String::from("Hello");
      some_other_function(var_b.clone());
      println!("{}", var_b);
  }

  fn some_other_function(input: String) {
      println!("{}", input);
  }
  ```

* We could pass back the ownership of "Hello" to `var_b`. We don't have 2 copies
  on the heap, and the item will live long enough as the ownership is moved back
  and forth. Note that we're modifying `var_b` and thus it should be marked as
  mutable.

  > It is good to mention that in rust, everything is immutable by default.

  ```rust
  fn main() {
      let mut var_b: String = String::from("Hello");
      var_b = some_other_function(var_b);
      println!("{}", var_b);
  }

  fn some_other_function(input: String) -> String {
      println!("{}", input);
      input
  }
  ```

# Borrowing!
Although the examples above work, they are not recommended as they are not
idiomatic Rust. Instead, Rust has a much better mechanism for dealing with
allocated memory that is used by other variables and scopes. This is done using
the borrowing mechanism in Rust. A borrowed value is indicated by using the `&`
sign.

```rust
fn main() {
    let var_b: String = String::from("Hello");
    some_other_function(&var_b);
    println!("{}", var_b);
}

fn some_other_function(input: &String) {
    println!("{}", input);
}
```

In this case, the `input` temporary borrows the allocated memory from `var_b`
and returns the ownership once it is done with it. Note that we don't have to
declare the `var_b` variable anymore as mutable.

This works as expected, but let's try something more interesting. Let's add a
struct with some members and try borrowing the ownership while modifying some
values.

```rust
#[derive(Debug)]
struct Person {
    age: u8,
    name: String,
}

fn main() {
    let person_a = Person {
        age: 38,
        name: String::from("Thomas"),
    };
    some_other_function(&person_a);
    println!("{:?}", person_a);
}

fn some_other_function(input: &Person) {
    println!("{:?}", input);
    // it's my birthday!
    input.age = 39;
}
```

Although this may seem fine to most of you, the compiler won't have any of it.

```shell
error[E0594]: cannot assign to `input.age`, which is behind a `&` reference
  --> src/main.rs:18:5
   |
16 | fn some_other_function(input: &Person) {
   |                               ------- help: consider changing this to be a mutable reference: `&mut Person`
17 |     println!("{:?}", input);
18 |     input.age = 39;
   |     ^^^^^^^^^^^^^^ `input` is a `&` reference, so the data it refers to cannot be written

For more information about this error, try `rustc --explain E0594`.
error: could not compile `moving_ownership` due to previous error
```
Remember, in Rust, everything is immutable by default, including borrowed
values. However, you can get a mutable borrowed value by using the `&mut` syntax
instead of `&`.

> In Rust, you can have as many immutable borrowed values as you'd like, but you
> can only have one mutable borrowed value at a time. This is a hard restriction
> that prevents us from running into various memory management issues and makes
> memory management more understandable.

> For simplicity's sake, a struct was used instead of a String value. When using
> boxed allocations on the heap, we also have to work with lifetimes, making
> things even more complex. However, for now, let's focus on borrowing.

We can fix the error by marking the `input` parameter as mutable using `&mut`.

```rust
#[derive(Debug)]
struct Person {
    age: u8,
    name: String,
}

fn main() {
    let mut person_a = Person {
        age: 38,
        name: String::from("Thomas"),
    };
    some_other_function(&mut person_a);
    println!("{:?}", person_a);
}

fn some_other_function(input: &mut Person) {
    println!("{:?}", input);
    input.age = 39;
}
```
# Some more examples.

Consider the following code example:

```rust
fn main() {
    let r;
    // lets define a scope
    {
        // 'x' only lives in this scope, and will be dropped after it
        // Therefore 'r' can not have a reference to x, as the lifetime of 'r' exceeds that of 'x'
        let x = 5;
        r = &x;
    }
    println!("R: {}", r);
}
```

When attempting to compile the code, the Rust compiler is able to detect that
memory is being released prematurely. It will issue a compile-time error,
preventing us from running into issues at runtime. This is a powerful feature of
Rust's ownership system, as it helps to catch potential bugs before they can
cause problems.

```shell
➜ rust-ownership-test (master) ✗ cargo build
   Compiling rust-ownership-test v0.1.0 (/home/thomas/projects/tests/rust/rust-ownership-test)
error[E0597]: `x` does not live long enough
  --> src/main.rs:8:13
   |
8  |         r = &x;
   |             ^^ borrowed value does not live long enough
9  |     }
   |     - `x` dropped here while still borrowed
10 |     println!("R: {}", r);
   |                       - borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `rust-ownership-test` due to previous error
```

# Lifetimes
Another great example of the borrow checker in action is how it handles
lifetimes

```rust
fn main() {
    let string1 = String::from("Hello, ");
    let string2 = String::from("world");

    let longest = longest(string1.as_str(), string2.as_str());
    println!("{}", longest);
}

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        return x; // returns x
    }
    y // returns y
}
```

Although this code might appear normal, it will encounter compilation issues.

```shell
➜ rust-ownership-test (master) ✗ cargo build
   Compiling rust-ownership-test v0.1.0 (/home/thomas/projects/tests/rust/rust-ownership-test)
error[E0106]: missing lifetime specifier
 --> src/main.rs:9:33
  |
9 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
9 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++

For more information about this error, try `rustc --explain E0106`.

error: could not compile `rust-ownership-test` due to previous error
```
>  This function's return type contains a borrowed value, but the signature does
>  not indicate whether it is borrowed from `x` or `y`.

The reason is that `x` and `y` could have different lifetimes. The `longest`
function doesn't know about the lifetimes, and we need to know which lifetime to
return. The compiler already told us how to fix this.

We will add the lifetime annotations to our method, as the compiler suggested.

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        return x; // returns x
    }
    y // returns y
}
```
What we are now specifying is a relationship between `a`, `b` and the return
value. This means that the lifetime of the return value is the same as the
shortest lifetime of the arguments. So if `x` has a smaller lifetime than `y`, then
the lifetime of the return value is the same as `x`. Conversely, if the lifetime
of `y` is smaller than `x`, the lifetime of the returned value is the same as `y`.

Going back to our `main` function, we can see that we print out the longest value
at the end. The borrow checker will validate if the shortest lifetime is still
valid.

```rust
fn main() {
    let string1 = String::from("Hello, ");
    let string2 = String::from("world");

    let longest = longest(string1.as_str(), string2.as_str());
    println!("{}", longest);
}
```

But lets say I want to do something different, let's say like this:
```rust
fn main() {
    let string1 = String::from("Hello, ");
    {
        let string2 = String::from("world");

        let result = longest(string1.as_str(), string2.as_str());
        println!("{}", result);
    }
}
```

Although `string1` has a longer lifetime than `string2`, when we print out the
longest value, the lifetime of `string2` is still valid, and our program can
compile and execute without any issues.

However, if we were to do the following:

```rust
fn main() {
    let string1 = String::from("Hello, ");
    let result: &str;
    {
        let string2 = String::from("world");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("{}", result);
}
```
The compiler will throw an error, indicating that the lifetime of the returned
value is not guaranteed to be long enough when calling the `println!` macro.

```shell
➜ rust-ownership-test (master) ✗ cargo build
   Compiling rust-ownership-test v0.1.0 (/home/thomas/projects/tests/rust/rust-ownership-test)
error[E0597]: `string2` does not live long enough
 --> src/main.rs:6:44
  |
6 |         result = longest(string1.as_str(), string2.as_str());
  |                                            ^^^^^^^^^^^^^^^^ borrowed value does not live long enough
7 |     }
  |     - `string2` dropped here while still borrowed
8 |     println!("{}", result);
  |                    ------ borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `rust-ownership-test` due to previous error
```

# Fighting the borrow checker.
As you may have noticed, working with lifetimes and borrowing can be challenging
in Rust. However, if you understand these concepts well, you will be able to
write safe and efficient code. Even experienced developers can sometimes find it
frustrating to work with the borrow checker, which has led to the creation of
the meme "Fighting the borrow checker".

# Conclusion.
In conclusion, Rust's borrow checker is a powerful tool for ensuring memory
safety in Rust programs, but it can also be challenging to work with. However,
with practice and a good understanding of lifetimes, developers can become
proficient in writing Rust code that is not only safe, but also performant.
