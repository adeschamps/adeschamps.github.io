---
layout: post
title: "Nested match expressions on enums"
permalink: /nested-enum-match
---

I recently encountered a situation in Rust where I had two enums, and
I wanted to do something different for each combination of values. In
my case, each enum had three variants (some of which carried
additional data) for a total of nine possibilites.

I could think of two ways to write that function. One way is to match
on the first enum, and then in each branch match on the second. The
other way is to make a tuple out of the two enums and then match on
that. I was curious if there would be a performance difference.

# A simple example

Let's start with a simple case, where each enum has two variants and
neither of them carry any additional data:

```rust
pub enum Vertical {
    Up,
    Down,
}

pub enum Horizontal {
    Left,
    Right,
}
```

Next, we'll write the simplest possible function that returns a
different result for each possible input. There are two ways to do
this. In the first version, `match_nested`, We match on the `Vertical`
enum first, and then in each branch we match on the `Horizontal`
enum. In the second version, `match_tuple`, we use a tuple to match on
both enums at once.

```rust
pub fn match_nested(v: Vertical, h: Horizontal) -> usize {
    match v {
        Vertical::Up => match h {
            Horizontal::Left => 0,
            Horizontal::Right => 1,
        },
        Vertical::Down => match h {
            Horizontal::Left => 2,
            Horizontal::Right => 3,
        },
    }
}

pub fn match_tuple(v: Vertical, h: Horizontal) -> usize {
    match (v, h) {
        (Vertical::Up, Horizontal::Left) => 0,
        (Vertical::Up, Horizontal::Right) => 1,
        (Vertical::Down, Horizontal::Left) => 2,
        (Vertical::Down, Horizontal::Right) => 3,
    }
}
```

I wrote a simple [benchmarked][simple-benchmark] for these two
functions. Here are the results for calling each function 1000 times:

```text
test simple::tests::match_nested     ... bench:       1,599 ns/iter (+/- 701)
test simple::tests::match_tuple      ... bench:       1,562 ns/iter (+/- 135)
```

There turned out to be a very good reason for that - when compiled
with optimizations, they both produce the exact same
[assembly][simple-assembly]. Now, I'm not that great at reading
assembly, but I can tell from the following that there are no
branches. Also, I may be wrong, but it seems like it's using the enum
discriminant itself as part of the calculation.

```asm
example::match_nested:
  push rbp
  mov rbp, rsp
  xor ecx, ecx
  test sil, sil
  setne cl
  lea rax, [rcx + 2]
  test dil, dil
  cmove rax, rcx
  pop rbp
  ret

example::match_tuple:
  push rbp
  mov rbp, rsp
  xor ecx, ecx
  test sil, sil
  setne cl
  lea rax, [rcx + 2]
  test dil, dil
  cmove rax, rcx
  pop rbp
  ret
```

Obviously these functions are too simple to meausure what we want. A
match statement implies that there should be some branching, yet all
the branching has been optimized away. Let's try something else.

# A slightly more complicated example

Let's try with some more complicated enums.  In this example, we'll
use an overly contrived way of representing movements on a grid:

```rust
pub enum Vertical {
    Up(usize),
    Zero,
    Down(usize),
}

pub enum Horizontal {
    Left(usize),
    Zero,
    Right(usize),
}
```

Next, we'll define two versions of a function to calculate the
Manhattan distance of those movements. Similar to before, the first
version, `distance_nested`, matches on the `Vertical` enum and then in
each branch matches on the `Horizontal` enum. In the second version,
`distance_tuple`, we tie the two variables together and write a single
match statement that handles all the cases.

```rust
pub fn distance_nested(v: Vertical, h: Horizontal) -> usize {
    match v {
        Vertical::Up(dv) => match h {
            Horizontal::Left(dh) => dv + dh,
            Horizontal::Zero => dv,
            Horizontal::Right(dh) => dv + dh,
        },
        Vertical::Zero => match h {
            Horizontal::Left(dh) => dh,
            Horizontal::Zero => 0,
            Horizontal::Right(dh) => dh,
        },
        Vertical::Down(dv) => match h {
            Horizontal::Left(dh) => dv + dh,
            Horizontal::Zero => dv,
            Horizontal::Right(dh) => dv + dh,
        },
    }
}

pub fn distance_tuple(v: Vertical, h: Horizontal) -> usize {
    match (v, h) {
        (Vertical::Up(dv), Horizontal::Left(dh)) => dv + dh,
        (Vertical::Up(dv), Horizontal::Zero) => dv,
        (Vertical::Up(dv), Horizontal::Right(dh)) => dv + dh,
        (Vertical::Zero, Horizontal::Left(dh)) => dh,
        (Vertical::Zero, Horizontal::Zero) => 0,
        (Vertical::Zero, Horizontal::Right(dh)) => dh,
        (Vertical::Down(dv), Horizontal::Left(dh)) => dv + dh,
        (Vertical::Down(dv), Horizontal::Zero) => dv,
        (Vertical::Down(dv), Horizontal::Right(dh)) => dv + dh,
    }
}
```

I think it's rather subjective whether one version is more pleasant to
read than the other. I personally prefer the tuple version, especially
if the logic inside each branch is less trivial.

When I [benchmarked][complex-benchmark] these (again, by calling them
1000 times), I got some different results. I ran it a number of times,
and sometimes the standard deviations were quite a bit larger, but the
tuple version was always faster than the nested version.

```text
test complex::tests::distance_nested ... bench:       2,732 ns/iter (+/- 124)
test complex::tests::distance_tuple  ... bench:       2,347 ns/iter (+/- 196)
```

I'm hesitant to trust my benchmarks. There seemed to be quite a bit of
variability between runs, which makes me wonder whether I was really
measuring what I intended to. I know that benchmarking is a difficult
art, and I haven't done a lot of it.

Let's look at the [assembly][complex-assembly] though. At this point
I'm pushing the limits of my knowledge of assembly, but I can infer a
few things.

The first is that the two functions definitely produce different assembly.

The second thing I can tell is that the nested function has a lot more
branching in it. There are a total of six jump instructions, compared
to two jump expressions in the tuple version. Furthermore, there are
several execution paths through the nested version that take two
jumps, whereas in the tuple version no execution path takes more than
one jump.

This is interesting to me, because based on the source code I would
say that there are two branch points in each function. However, when
matching on a tuple it seems that the compiler was able to reduce the
amount of branching.

On the other hand, if I trace through it by hand, it seems to me that
all the paths through the code wind up executing roughly the same
number of instructions.  Just because there's more code doesn't mean
that the function will take longer to execute. In fact, it's possible
for the opposite to be true.

```asm
example::distance_nested:
  push rbp
  mov rbp, rsp
  mov al, byte ptr [rdi]
  cmp al, 2
  je .LBB0_6
  cmp al, 1
  jne .LBB0_4
  mov al, byte ptr [rsi]
  cmp al, 1
  je .LBB0_3
  cmp al, 2
  mov rax, qword ptr [rsi + 8]
  pop rbp
  ret
.LBB0_6:
  mov rax, qword ptr [rdi + 8]
  mov cl, byte ptr [rsi]
  cmp cl, 2
  je .LBB0_10
  cmp cl, 1
  je .LBB0_8
.LBB0_10:
  add rax, qword ptr [rsi + 8]
  pop rbp
  ret
.LBB0_4:
  mov rax, qword ptr [rdi + 8]
  mov cl, byte ptr [rsi]
  cmp cl, 1
  je .LBB0_8
  cmp cl, 2
  add rax, qword ptr [rsi + 8]
  pop rbp
  ret
.LBB0_8:
  pop rbp
  ret
.LBB0_3:
  xor eax, eax
  pop rbp
  ret

example::distance_tuple:
  push rbp
  mov rbp, rsp
  mov rcx, qword ptr [rdi]
  mov rdx, qword ptr [rsi]
  mov rsi, qword ptr [rsi + 8]
  cmp rcx, 1
  je .LBB1_4
  mov rax, qword ptr [rdi + 8]
  cmp rcx, 2
  cmp rdx, 1
  je .LBB1_3
  cmp rdx, 2
  add rax, rsi
.LBB1_3:
  pop rbp
  ret
.LBB1_4:
  xor eax, eax
  cmp rdx, 1
  cmove rsi, rax
  mov rax, rsi
  pop rbp
  ret
```

# Conclusion

We've reached the point where I don't know enough about assembly to
really make any meaningful observations.

What have I learned from this? I certainly can't conclude that
matching on a tuple is faster than nesting match expressions. I can't
even conclude that it's faster in this particualr case, because I
didn't benchmark a large domain of inputs.

I think I can conclude that, at least for simple cases like this, the
two methods for matching on a pair of enums are not sufficiently
different for me to let performance influence my decision. Going
forward, I will prefer whichever one I think results in clearer code.

[simple-benchmark]: https://github.com/adeschamps/nested-enum-match/blob/master/src/simple.rs
[complex-benchmark]: https://github.com/adeschamps/nested-enum-match/blob/master/src/complex.rs
[simple-assembly]: https://godbolt.org/g/QGcQeM
[complex-assembly]: https://godbolt.org/g/TN6VS7
