# Lifetime Annotations

_I'm yet another annoying Rust enthusiast not qualified to write tutorials but is going to anyways.
This is a mix of Crust of Rust and my personal intuition + scattered thoughts about lifetimes._

## Toy problem

Be able to split a string by a delimiter, similar to Python's `.split(delim)`.

`"hello world" => ["hello", "world"]`

## API

Let's start by sketching out an API.

### First, let's envision the API usage:

```rust
fn main() {
    let str_split = StrSplit {}
    for s in str_split {
        println!("{}", s);
    }
}
```

### Now, let's define the skeleton enabling the usage:
```rust
pub struct StrSplit<'a> {
    s: &'a str,
    delimiter: &'a str,
}

impl<'a> Iterator for StrSplit<'a> {
    type Item = ???
    ...
}
}
```

## Question 1: Why are we defining a custom struct vs. returning an existing collection type?

Consider the alternative of reusing existing collection types (the same way Python returns a `list`
when you do `.split()`):
- returning a `Vec`
- returning a `slice (&[])`

Both those options return concrete collection types. So the caller is forced to use whatever concrete
type you, the API designer, specified. Who gave _you_ that authority??

Also, what if it's a very long `&str` we're splitting? Indexed collection types like `Vec` require 
all the data to be in memory at once.

Let's keep things general and instead return an `Iterator`. But `Iterator` is a `trait`, not a 
concrete type, so here we define our own structs `StrSplit` and `impl Iterator`.

In fact, this is a common idiom when designing Rust APIs: you define a `struct` specifically for
the return type of your API function.

> #### Reification
> * The process of transforming a function (computation) into a struct (data) is known as **reification**. _Code_ becomes _data_.
> * The inverse is **Church encoding**. _Data_ becomes _code_.
> * Read more [here](https://underscore.io/blog/posts/2017/06/02/uniting-church-and-state.html) if this excites you.


## Question 2: Why do we need a lifetime?

In Rust, all values have a `lifetime`. Essentially, Rust (the borrow checker) needs to know how
long a value lives, so it knows when to `drop` (`free`) a value.

This is to prevent both memory leaks and double-frees: any C programmer is familiar with that pain.

Whenever you take out a reference with `&` you are _borrowing_ that value. So the borrow checker
needs to verify that value is remains "alive" while you borrow it.

Often, the lifetimes of references are implied and don't need to be explicitly annotated when 
making function calls. **Yet, here we need to explicitly define a lifetime when defining the 
struct.**

# Why? Pause here and try to answer before you continue reading.

One intuition for lifetimes is they are a value's "scope."

The reason functions often don't need to explicitly annotate lifetimes, when taking in
references as arguments, is that the value's scope is tied to the length of the function body.

Specifically, values are dropped at the point the function returns -- which is also when the stack 
frames get popped off and the memory becomes invalid!

If a function calls another function, then the variables in the calling function live 
longer than the variables in the callee function. Why?

> From a low-level perspective, the (calling) function's stack frames are below (`0xFFF...` 🦊🦊🦊 
> at the top) the inner (callee) function's stack frames.

> From a high-level perspective, a caller's variables outlive the callee's by definition, since
> the nested function finishes executing before returning control to the calling function
> (barring threads or concurrency, which is another can of worms).

But if you think about it, when you define a `struct StrSplit`, _there is no function_ associated
with that struct. Right? And so what scope is tied with that lifetime?

If you answered "it's not known at the time of the struct's definition time", you'd be right!

> Indeed, this might remind you of a very familiar construct: generics!
> Let's build some intuition: when you write a generic function
> `fn do_something<T>(t: T)`, the concrete _argument_ of `T` is not known at the definition site.
> It's only known once you eventually make a call like `do_something(42)` later in your code.

It's only at later usage sites that Rust (the borrow checker) knows what to pass in for `T`.
The intuition is exactly the same for _lifetimes_. When you define a `struct`, the 
lifetimes (scopes) associated with that struct are _generic_ aka. _unknown at the definition site_.

It's only at the scope where you _use_ the `struct` that Rust now knows, "oh, programmer wants to 
associate a _reference with this lifetime (scope)_ with the struct!"

So you have to explicitly tell Rust, "hey, this thing is generic over _some lifetime_ `<'a>`.
You don't know what it is yet, but once I write the rest of the code you'll see."

While it is theoretically possible for the compiler to _elide_ lifetimes in structs, it would
arguably lead to more confusion. Specifically, now the compiler is going to have
to generate some lifetime for your struct, and it all becomes implicit the fact there is an
underlying lifetime (and scope).

So compiler error messages can quickly become very confusing, especially when you have structs
taking references to other structs (which in turn hold references).

In fact, you could make a similarly argument for why you have to explicitly annotate types for
function arguments and return types. In theory, the Rust compiler could do _type inference_ on your 
entire program beginning from your main function... but that would become unwieldy: slow and lead 
to very confusing error messages.

> #### N.B.
> The struct `'a` is _not_ the struct's lifetime. Rather, it is the lifetime of the fields of the
> struct, which _may happen_ to live as long as the struct itself.
> 
> Later, we will discuss different lifetimes for different fields in a struct.


Similarly, as a rule, a `struct` has to explicitly be declared as _generic_ with respect to _some 
lifetime_. That's what you're doing with `<'a>`.

> Aside: when using generics like `T`, you are implementing
[parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism), aka. the
[universal quantifier](https://en.wikipedia.org/wiki/Universal_quantification)
(there's deep and beautiful relationships between predicate logic and programming if you're into
that).

To be continued...

```rust
impl<'a> Iterator for StrSplit<'a> {
    type Item = &'a str;

    fn next(&mut self) -> Option<Self::Item> {
        if let Some(i) = self.s.find(self.delimiter) {
            let s = &self.s[..i];
            self.s = &self.s[i + self.delimiter.len()..];
            Some(s)
        } else if self.s.is_empty() {
            None
        } else {
            let s = self.s;
            self.s = "";
            Some(s)
        }
    }
}
```