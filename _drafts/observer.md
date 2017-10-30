---
layout: post
title:  "Mutating observers in Rust"
permalink: /observer
---

I started this blog because I used to enjoy writing
but I haven't had many opportunities to do so in the last few years.
In my mind, I've set an arbitrary goal of publishing one post per month,
but I've been busy lately, and October is drawing to an end.

However, having just attended Rust Belt Rust and listened to Chris Krycho's excellent talk on becoming a contributor,
I've decided to write this anyway, even though I'm still working through it myself.

# The Observer Pattern

The [observer pattern][observer-wiki] is basically when we ask someone else to notify us when something interesting happens.
Usually, "notify me" means "call one of my member functions".
This is easily expressible in a lot of languages,
but there are ways to get it wrong and the Rust compiler catches some of them.

Timmy Jose wrote an anticle on [The Observer Pattern in Rust][observer-rust],
which was a helpful starting point for me.
I'd recommend reading that first.

We'll be using the following `Observer` trait,
which is similar to the one Timmy defined.
Notice that it takes an _immutable_ reference to `self`.

```rust
pub trait Observer<T> {
    fn notify(&self, value: &T);
}
```

Instead of using an `Observable` trait, I'm going to simplify things by making it a struct,
since we'll only be using one concrete implementation:

```rust
pub struct Counter {
    value: usize,
    observers: Vec<Box<Observer<usize>>>,
}

impl Counter {
    pub fn new() -> Counter {
        Counter { value: 0, observers: vec![] }
    }

    pub fn register(&mut self, observer: Box<Observer<usize>>) {
        self.observers.push(observer);
    }

    pub fn run(&mut self) {
        loop {
            self.value += 1;
            for observer in self.observers.iter() {
                observer.notify(&self.value);
            }
        }
    }
}
```

We're using [`Box`][Box] here because the `Counter` doesn't know the concrete types of its observers.
They could be anything, as long as they implement the `Observer` trait.

# Mutability

Suppose we want our observers to be mutable.
This is easy!
We just change our trait's `notify` function to take `&mut self`:

```rust
pub trait Observer<T> {
    fn notify(&mut self, value: &T);
}
```

We also modify our counter's `run` function to use [`iter_mut`][Vec::iter_mut] instead of [`iter`][Vec::iter]:

```rust
impl Counter {
    // ...

    pub fn run(&mut self) {
        loop {
            self.value += 1;
            for observer in self.observers.iter_mut() {
                observer.notify(&self.value); // Changed this line
            }
        }
    }
}
```

This didn't require us to change our `Counter` struct.
Since it owns its observers, it's free to mutate them as it pleases.
Other than being explicit about which iterator we use, there's no difference.


# Non-Ownership

In the example above, the `Counter` owns its observers.
This is probably the simplest situation when it comes to memory safety.
The observers will live as long as the `Counter`, and no longer.

Suppose we don't want our observers to be owned by the `Counter`.
Rather than using `Box`, we'll have to use plain references.
For now, we'll forget about the observers being mutable.

```rust
pub struct Counter<'a> {
    value: usize,
    observers: Vec<&'a Observer<usize>>,
}

impl<'a> Counter<'a> {
    // ...

    pub fn register(&mut self, observer: &'a Observer<usize>) {
        self.observers.push(observer);
    }
}
```

The lifetime specifier is necessary to tell the compiler that the elements of our `Vec` will live at least as long as `'a`.
If we try to use our new function like this, then Rust won't

```rust
struct Foo;

impl Observer<usize> for Foo {
    fn notify(&self, value: &usize) {}
}

fn main() {
    let observer = Foo;
    let mut counter = Counter::new();
    counter.register(&observer);
    counter.run();
}
```

In the example above, it is important that the observer be initialized before the counter.
Otherwise, the compiler will stop us because the counter is still holding a reference to the observer when the observer goes out of scope.
At the risk of going too far on a tangent, this is where one could use reference counted pointers like [`Rc`][Rc].

Since this example uses non-mutable references, we can safely register our observer with as many counters as we want:

```rust
fn main() {
    let observer = Foo;

    let mut counter_1 = Counter::new();
    let mut counter_2 = Counter::new();

    counter_1.register(&observer);
    counter_2.register(&observer);

    counter_1.run();
    counter_2.run();
}
```

# Mutability and Non-Ownership

So far, we've taken our initial example in two different directions.
In one case, we allowed our observers to mutate their state when notified.
In the other case, we moved the owneship of the observers outside of the counter, so that they could be shared by multiple counters.
Can we do both?

Take a moment to consider what we're asking for: multiple mutable references to the same data.
Isn't that exactly what Rust forbids us from having?
Well, yes, but there's always an escape hatch.
In this case, it's called [`RefCell`][RefCell].

`RefCell` moves the borrow checking from compile-time to runtime.
It provides _interior mutability_ - while the compiler treats the `RefCell` itself as immutable,
it can be used to obtain both immutable and mutable references using functions
such as [`borrow`][RefCell::borrow] and [`borrow_mut`][RefCell::borrow_mut].

__Be careful!__
Rather than getting a compiler error when we make a mistake, those functions will panic instead.
There are related functions, [`try_borrow`][RefCell::try_borrow] and [`try_borrow_mut`][RefCell::try_borrow_mut], which return `Result`s instead of panicking.
In any case, we'll know as soon as something goes wrong and we'll get a clear error message,
rather than some subtle undefined behaviour due to memory corruption.
Note, however, that this comes with the runtime cost of a reference count.

Modifying our `Counter` to use `RefCell` is fairly straightforward:

```rust
pub struct Counter<'a> {
    value: usize,
    observers: Vec<&'a RefCell<Observer<usize>>>,
}

impl<'a> Counter<'a> {
    // ...

    pub fn run(&mut self) {
        loop {
            self.value += 1;
            for observer in self.observers.iter() {
                observer.borrow_mut().notify(&self.value); // Changed this line
            }
        }
    }
}
```

# Do we really want this?

`RefCell` should not be the first tool we reach for.
Although we haven't sacrificed memory safety
(trying to concurrently access the same memory will cause a panic, not undefined behaviour),
it moves those safety guarantees from compile time to runtime.
If we're thinking of using `RefCell`, we should probably consider alternative designs as well.
Consider some form of message passing;
perhaps the standard library's [thread safe queue][mpsc] would be useful.

That said, there will be times when it _is_ the tool we need.
That's why it's in the standard library in the first place.
In my case, I have a C library that I'm interfacing with.
The library allows us to register callbacks for various events.
While I know that the callbacks will be invoked in a thread safe way,
the Rust compiler can't prove that.
This is a case where it's okay to have multiple mutable references to the same memory.
We just have to be sure that they won't be used incorrectly,
or we can use the `RefCell::try_` variants if we can't be sure.






[Box]: https://doc.rust-lang.org/std/boxed/struct.Box.html
[Rc]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[RefCell::borrow]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow
[RefCell::borrow_mut]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow_mut
[RefCell::try_borrow]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.try_borrow
[RefCell::try_borrow_mut]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.try_borrow_mut
[Vec::iter]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter
[Vec::iter_mut]: https://doc.rust-lang.org/std/vec/struct.Vec.html#method.iter_mut
[mpsc]: https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html
[observer-rust]: https://z0ltan.wordpress.com/2017/06/23/the-observer-pattern-in-rust/
[observer-wiki]: https://en.wikipedia.org/wiki/Observer_pattern
