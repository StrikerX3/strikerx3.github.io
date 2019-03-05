---
layout: post
title:  "Typesafe Enum Class Bitmasks in C++"
date:   2019-02-27 19:34:11 -0300
categories: cpp
---
I've been working on [virt86](https://github.com/StrikerX3/virt86) for a few weeks now, and some of the things I decided to focus on were type safety and making good use of modern C++ features. I did many things to reduce the chances for users to shoot their own foot with the library, such as [deleting move and copy constructors and assignment operators](https://github.com/StrikerX3/virt86/blob/d050883ee3931e6a0a74d3da9f6b948ee3cd0533/modules/core/include/virt86/platform/platform.hpp#L77) from classes to make sure there is only one instance obtained from a known source, [deleting](https://github.com/StrikerX3/virt86/blob/d050883ee3931e6a0a74d3da9f6b948ee3cd0533/modules/core/include/virt86/vp/vp.hpp#L108) or [hiding](https://github.com/StrikerX3/virt86/blob/d050883ee3931e6a0a74d3da9f6b948ee3cd0533/modules/core/include/virt86/vm/vm.hpp#L326) the address operator `&` so that users cannot get a pointer to objects that are not meant to be `delete`d and using `enum class` for type-safe enumerations.

I figured it would be interesting to use `enum class`es as bitmasks for their type-safety. Unfortunately, you cannot use them as is with bitwise operators, but you can use other C++ features to make them behave like a bitmask type.

<!-- more -->

## A basic version

I knew very little about `enum class`es at the time, so my first idea was to add methods to the type, assuming they would work like a `class`, however C++ disallows that. The next solution I thought of was to define operator overloads outside them, which I found out to be allowed by the language. Templates would make that even better.

While researching a bit about using `enum class` as bitmasks I came across a [blog post](http://blog.bitwigglers.org/using-enum-classes-as-type-safe-bitmasks/) by Andre Haupt, which the author says is "a reiteration of a [blog post](https://www.justsoftwaresolutions.co.uk/cplusplus/using-enum-classes-as-bitfields.html) by Anthony Williams", which is exactly the kind of solution I envisioned. The post progressively expands from a motivating example based on `unsigned` values and regular `enums`, to a simple set of overloaded operators for an specific type of enumeration, to templates which encompass every `enum class` type, finishing with SFINAE to ensure only types tagged as bitmasks actually have access to the bitwise operators.

I took his code and modernized it a bit by introducing an alias template for the bitmask enum type trait value and using the alias template `std::underlying_type_t`, both of which saves a bit of typing overall. Here's my version:

```cpp
#define ENABLE_BITMASK_OPERATORS(x)  \
template<>                           \
struct is_bitmask_enum<x> {          \
    static const bool enable = true; \
};

template<typename Enum>
struct is_bitmask_enum {
    static const bool enable = false;
};

template<class Enum>
inline constexpr bool is_bitmask_enum_v = is_bitmask_enum<Enum>::enable;

// ----- Bitwise operators ----------------------------------------------------

template<typename Enum>
typename std::enable_if_t<is_bitmask_enum_v<Enum>, Enum>
operator |(Enum lhs, Enum rhs) {
    using underlying = typename std::underlying_type_t<Enum>;
    return static_cast<Enum> (
        static_cast<underlying>(lhs) |
        static_cast<underlying>(rhs)
    );
}

// ... similarly for & and ^ and ~

// ----- Bitwise assignment operators -----------------------------------------

template<typename Enum>
typename std::enable_if_t<is_bitmask_enum_v<Enum>, Enum>
operator |=(Enum& lhs, Enum rhs) {
    using underlying = typename std::underlying_type_t<Enum>;
    lhs = static_cast<Enum> (
        static_cast<underlying>(lhs) |
        static_cast<underlying>(rhs)
    );
    return lhs;
}

// ... similarly for &= and ^=
```

This is how the author demonstrated its usage, which is unchanged with my version:

```cpp
enum Permissions {
    Readable = 0x4, Writable = 0x2, Executable = 0x1
};

Permissions p = Permissions::Readable | Permissions::Writable;  
p |= Permissions::Executable;  
p &= ~Permissions::Writable;
```

## Limitations

The original version falls short when it comes to another common use case for bitmasks: checking if a bitmask contains a particular bit. You have to do some convoluted and repetitive code to achieve that:

```cpp
if ((p & Permissions::Executable) == Permissions::Executable) {
    // p has Executable
}
```

And what if we need to check if the bitmask contains at least one bit of a set of multiple bits?

```cpp
Permissions rw = Permissions::Readable | Permissions::Writable;
if ((p & rw) ... ??) {
    // p has at least one of Readable, Writable
}
```

One way to solve this problem is to add a `None = 0` entry to the enum. Or you could resort to `static_cast`:

```cpp
Permissions rw = Permissions::Readable | Permissions::Writable;
if ((p & rw) != static_cast<Permissions>(0)) {
    // p has at least one of Readable, Writable
}
```

But this is still too cumbersome. Wouldn't it be great if you could write those conditions like this?

```cpp
if (p.AllOf(Permissions::Executable)) {
    // p has Executable
}
if (p.AnyOf(Permissions::Readable | Permissions::Writable)) {
    // p has at least one of Readable, Writable
}
```

We could even introduce new checks, such as:

```cpp
if (p.NoneOf(Permissions::Writable)) {
    // p does not have Writable, but may have other bits
}
if (p.AnyExcept(Permissions::Executable)) {
    // p does not have Executable and has at least one of the other bits
}
```

Unfortunately, we hit the same wall again: `enum class`es cannot have methods.

## Enter the struct

The solution I came up with is to introduce a `struct` template which wraps an `enum class` and provides the check methods.

```cpp
template<typename Enum>
struct BitmaskEnum {
    const Enum value;
    static const Enum none = static_cast<Enum>(0);

    using underlying = typename std::underlying_type_t<Enum>;

    BitmaskEnum(Enum value) : value(value) {
        static_assert(is_bitmask_enum_v<Enum>);
    }

    // ... operations here
};
```

The basic checks we want to implement are:

- `Any()`: `true` if the bitmask contains any bits
- `None()`: `true` if the bitmask is empty
- `AnyOf(Enum)`: `true` if the bitmask contains one or more bits from the given bitmask
- `AllOf(Enum)`: `true` if the bitmask contains all bits from the given bitmask
- `NoneOf(Enum)`: `true` if the bitmask doesn't contain any bit from the given bitmask
- `AnyExcept(Enum)`: `true` if the bitmask contains any bits excluding those from the given bitmask
- `NoneExcept(Enum)`: `true` if the bitmask doesn't contain any bits excluding those from the given bitmask

It would be very handy to have an `EnumBitmask` to convert back into the `Enum` value or a `bool`, the latter of which is a very common use case with regular `enum`s or even `int`s to check if the bitmask has any bits set.

The implementations of these operations are left as an exercise to the reader.

Here's how to use the `BitmaskEnum`:

```cpp
Permissions p = Permissions::Readable | Permissions::Writable;
auto pBM = BitmaskEnum(p);
if (pBM.AllOf(Permissions::Executable)) {
    // p has Executable
}
if (pBM.AnyOf(Permissions::Readable | Permissions::Writable)) {
    // p has at least one of Readable, Writable
}
if (pBM.NoneOf(Permissions::Writable)) {
    // p does not have Writable, but may have other bits
}
if (pBM.AnyExcept(Permissions::Executable)) {
    // p does not have Executable and has at least one of the other bits
}
```

Much nicer and easier to understand. The best part of it? It's a [zero overhead abstraction](http://www.stroustrup.com/ETAPS-corrected-draft.pdf). No runtime costs at all. Here's a [gist](https://gist.github.com/StrikerX3/46b9058d6c61387b3f361ef9d7e00cd4) containing a header file taken straight out of [virt86](https://github.com/StrikerX3/virt86/blob/master/modules/core/include/virt86/util/bitmask_enum.hpp), along with a cpp file you can use in [Compiler Explorer](https://godbolt.org/) to check that this is indeed free. (Except MSVC, which still compiles every method in the struct, even though none of them are used, but the `test()` function is essentially the same.)

[![Optimized down to a simple return!](/assets/2.1-compiler-explorer.png)](/assets/2.1-compiler-explorer.png){:target="_blank"}  
**Optimized down to a simple return!**
{: style="text-align: center;"}

## Caveats

One thing to keep in mind, however, is that it is illegal to specialize a template in a different namespace. This will happen if you define an `enum class` in a namespace and immediately use the macro still within the namespace:

```cpp
namespace foo {

enum class Bar {
    ...
}
ENABLE_BITMASK_OPERATORS(Bar)
// error: specialization of ‘template<class Enum> struct is_bitmask_enum’ in different namespace [-fpermissive]

}
```

One solution to this is to use the macro after the namespace block, referring to the `enum class` via its namespace:

```cpp
namespace foo {

enum class Bar {
    ...
}

}

ENABLE_BITMASK_OPERATORS(foo::Bar)
// good to go

```

I hope this helps you use the type-safe `enum class` as bitmasks more comfortably!
