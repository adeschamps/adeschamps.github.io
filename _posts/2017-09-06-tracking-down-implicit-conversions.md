---
layout: post
title:  "Tracking down implicit conversions in C++"
permalink: /implicit
---

In C++, a _converting constructor_ is a constructor that may be implicitly invoked
as a means of converting from one type to another.
Take, for example, the following function:

```c++
void greet(std::string const & name) {
    std::cout << "Hello, " << name << "\n";
}
```

If you didn't have converting constructors,
you would have to always make sure you pass it a `std::string` argument.
That means that the following wouldn't work:

```c++
int main() {
    greet("Anthony");
}
```

The compiler would complain that `greet` takes a `std::string`,
but you passed it a `char const *`, which is definitely not what it wanted.
Instead, you would have to invoke the `std::string` constructor and pass the result to `greet`:

```c++
int main() {
    greet(std::string("Anthony"));
}
```

The compiler assumes that this is what you meant when you wrote the first version,
so you don't actually have to invoke the `std::string` constructor explicitly.
That is, it does an _implict conversion_.

# Preventing implicit conversions

Implicit conversions can be nice, but they can be dangerous too.
When you look at the first example of calling `greet`,
it's not immediately clear that there's a function besides `greet` which is being called.
If the constructor being invoked does a lot of work or has some side effects,
you may be in for a surprise.

This behaviour can be prevented by using the `explicit` specifier:

```c++
struct MyClass {
    explicit MyClass(int v) : value(v) {}
    int value;
};

int main() {
    MyClass a = 10; // will not compile
}
```

# When would you want to prevent this?

A while ago I working on a project using the Unreal Engine,
and I needed to log a bunch of events along with their timestamps.
I did the obvious thing, and represented times as unix milliseconds using 64 bit integers:

```c++
void log_something(
    int64 timestamp,
    std::string const & description);

// ...

// No, I didn't actually hard code the timestamps
log_something(1234, "something interesting");
```

Not long after, I realized that UE4 already has a [`FDateTime`][FDateTime] struct with all sorts of useful functions,
and I decided that I should use that instead.
I went ahead and changed the types of my timestamps, recompiled my project, and carried on.

```c++
void log_something(
    FDateTime timestamp, // changed the type of timestamp
    std::string const & description);

// ...

log_something(1234, "something interesting");
```

Wait, how did that compile?
If I changed the type of my timestamp, shouldn't I get compilation errors
from all the places where I'm passing an integer to something that expects a struct?
Let's take a look at the [`FDateTime` constructor][FDateTime::FDateTime]:

```c++
FDateTime(int64 InTicks);
```

There's no `explicit` specifier,
so the compiler is free to invoke this constructor without us telling it to.
But what does `InTicks` mean?
Is it unix milliseconds?
If so, everything should work like it did before.
But what if it means "milliseconds since the game was started",
so that it starts at `0` when you launch the game?
That seems like a valid interpretation, but it would cause my logs to have the wrong timestamps.

In any case, I'd rather make my intentions clear by using [this function][FDateTime::FromUnixTimestamp] instead:

```c++
static FDateTime FromUnixTimestamp(int64 UnixTime);
```

But how do I find all the places where an implicit conversion is happening?
I'd like to turn them into errors, but I wasn't the author of `FDateTime`,
so I can't just modify its constructor and mark it `explicit`.
Well, I could recompile the engine, but I'd probably cause something to break in the process,
and UE4 takes more than a few minutes to build.

# Turning implicit conversions into errors

I said I can't modify `FDateTime`, but this is C++.
There's always a way around things.
Why else would something like `const_cast` exist?
In this case, we can solve our problem with a sprinkling of variadic templates and a touch of the preprocessor.

First, we define a new class that does nothing on its own,
but uses a variadic constructor that passes all arguments to its base class.
The only difference is that it makes all constructors `explicit`.

```c++
template <class Base>
struct Explicit : public Base {
    template <typename ... Ts>
    explicit Explicit(Ts && ... ts)
        : Base(ts...) {}
};
```

Now all we have to do is use the preprocessor to replace instances of `FDateTime` with our new class,
and the following turns into an error:

```c++
#define FDateTime Explicit<FDateTime>

void log_something(
    FDateTime timestamp,
    std::string const & description);

// ...

// Will not compile!
log_something(1234, "something interesting");
```

Now we can make sure that we're perfectly clear that the timestamp we give to the log function is a unix timestamp:

```c++
log_something(
    FDateTime::FromUnixTimestamp(1234),
    "something interesting");
```

The nice thing about this technique is that we can control the scope of this forced `explicit`ness
by changing where we put that `#define`.
This is useful, because there may be other code that does an implicit conversion but doesn't need to be changed.

You could keep this code around if you really wanted to.
I expect it would have a small effect on a debug build,
and that the compiler would optimize it away in a release build.
In the end, though, it's really just a trick to aid in refactoring,
and should probably be removed when you're done with it.



[FDateTime]: https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Misc/FDateTime/index.html
[FDateTime::FDateTime]: https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Misc/FDateTime/__ctor/2/index.html
[FDateTime::FromUnixTimestamp]: https://docs.unrealengine.com/latest/INT/API/Runtime/Core/Misc/FDateTime/FromUnixTimestamp/index.html
