---
layout: post
title: "True Macro Recursion in C"
date: 2023-04-26
---

Macros are a very useful feature of the C preprocessor for reducing boilerplate.
It is well-known that they have limited processing power; most importantly, they
cannot recurse, and cannot iterate over lists.  Libraries for meta-programming
(e.g. [Boost.Preprocessor][] and [metalang99][]) work around this, but are only
capable of executing a limited number of iterations, and have complex
implementations.  However, an obscure GCC feature can be abused to allow for
"true" recursion in macros in a relatively elegant manner, allowing more complex
and useful macros to be implemented.

[Boost.Preprocessor]: https://www.boost.org/doc/libs/release/libs/preprocessor/
[metalang99]: https://github.com/Hirrolot/metalang99

## The Background

C is a relatively simple language, in that it does not provide many facilities
for reducing boilerplate in code.  The C preprocessor (which pre-processes C
code before it is compiled) makes up for this by providing _macros_, which are
named fragments of code that can be substituted at other locations.  There are
_object-like_ macros (where the macro is referenced simply by name, and its
definition is simply substituted), and _function-like_ macros (where the macro
is referenced like a C function call (`foo(...)`), where the macro can be passed
_arguments_ (other fragments of code) which are substituted into the definition
of the macro, which is then substituted into the location of the reference.

```c
// Here's an object-like macro called 'BAR'.  After this definition, wherever
// 'BAR' is found in the source code, it will be replaced by this fragment of
// code ('BAZ').
#define BAR BAZ

BAR // The preprocessor will replace this with 'BAZ'.
ABAR // This is not the same as 'BAR', and will not suffer substitution.
"BAR" // C strings are not identifiers, and so cannot be macro references.

// Here's a function-like macro called 'BAZ'.  It takes one argument, called
// 'x', whose definition (provided wherever 'BAZ' is referenced) will be
// substituted into the defined fragment of code.
#define BAZ(x) foo(x)

BAZ // Without parentheses, this is not a function-like macro reference.
BAZ(24) // 'x' is defined as '24' and substituted; the result is 'foo(24)'.

// Since 'BAR' is an object-like macro, this is not considered to be a function-
// like macro reference; hence the preprocessor will initially ignore the '()'
// and just subsitute 'BAR' for 'BAZ'.  After that substitution, it takes a new
// look at the code; seeing 'BAZ(24)', it will interpret this as a reference to
// 'BAZ' and substitute 'foo(24)' accordingly.
BAR(24)
```

After preprocessing, this code looks like:

```
BAZ
ABAR
"BAR"

BAZ
foo(24)

foo(24)
```

There are a tremendous number of edge cases and special considerations with
macros; for example, `BAZ()` is a valid reference, even though it appears as if
no arguments have been provided, because `x` is considered simply empty.  But
ignoring these, macros provide a simple way to reuse code fragments, and to
replace boilerplate code with short and comprehensible macro references.

Before C got inline functions, macros were used instead; for example, one could
define `memcpy()` like this:

```c
// Note: call as 'memcpy(...);', so that a semicolon is provided.
#define memcpy(dst, src, len) \
    for (size_t i = 0; i < len; i++) dst[i] = src[i]
```

Although inline functions have replaced macros for these sorts of use cases, and
provided type safety where there was none, such macros are still necessary in
some cases.  In particular, if the type of a certain argument can vary, it
cannot be defined as a C function.  A generic `max()` function, intended to take
both integers and floating-point values, must be defined as a macro:

```c
#define max(a, b) (a < b) ? b : a
```

Although there are a lot of problems with macros (e.g. both the macros above
re-evaluate one or more of their arguments, which is problematic if they have
side effects; and if any of the arguments are complex expressions, the resulting
code may be parsed incorrectly), they remain useful as a means of sharing code
and reducing boilerplate.

## The Problem

Macros are useful for basic cases of boilerplate.  But as the complexity of the
problem at hand increases, macros get more difficult to work with, and several
limitations come to light.  Most importantly, the C preprocessor does not allow
for recursion: a macro cannot be called within itself, and so iteration and
other complex control flow cannot be expressed.

Consider the following use case: we've defined a custom integer type `int24_t`,
and would like `printf()` to support it.  Rather than adding a new format
specifier for it, which would require us to re-implement `printf()` entirely
(along with all of its format specifiers), we can try to transparently convert
any calls to this custom `printf()` function into a regular `printf()`.

```c
// Our custom integer type.
typedef struct int24 {
    unsigned char data[3];
} int24_t;

// A function convert a regular 'int' into an 'int24_t'.
int24_t int24_from_int(int value);

// A function convert an 'int24_t' into a regular 'int'.
int int24_into_int(int24_t value);

// A 'printf()' that supports our custom integer type.
int my_printf(char const *fmt, ...);

// Ideal usage:
int24_t to_print = int24_from_int(42);
my_printf("Hello %d!\n", to_print);
```

If we can detect instances of `int24_t` in the input argument list, we can
convert them into `int`s and pass them onto `printf()`.  The `%d` format
specifier can then be used as expected.  But variadic functions in C lose type
information, so it would be impossible to distinguish `int`s from `int24_t`s.
However, this type information can be retrieved using C11's `_Generic` operator
combined with macros:

```c
// If the argument is a 'int24_t', it is converted to an 'int'; arguments of any
// other type are passed through as-is.
#define FORMAT_INT24(x) ({ \
    int24_t as_int24 = _Generic((x), int24_t: (x), default: (int24_t) {}); \
    _Generic((x), int24_t: int24_into_int(as_int24), default: (x)); \
})

// It'd be nice if we could just write this:
#define FORMAT_INT24_NICE(x) _Generic(x, int24_t: int24_into_int(x), default: x)
// But we can't have all the nice things.
```

We use GCC expression blocks (in the `({` and `})` enclosure), which introduce a
new scope for variables and allow usage of the macro as a single statement (for
`for` loops and the like).  The `_Generic()` operator allows us to select a code
fragment based on the type of an expression, but it has strict limitations (all
provided code fragments must type-check, regardless of the type of the target
expression), forcing the definition of an additional variable.

Now that we have a transformation macro, we need to apply it to every argument
passed to `my_printf()`.  This is where we hit one of the C preprocessor's most
severe limitations: it is not possible to iterate over the arguments to a
variadic function-like macro, to apply a transformation to each argument.  This
would otherwise be a very nice solution to the problem at hand, for supporting
special new types.

```c
// We have:
my_printf(fmt, a, b, ...)

// We need to turn it into:
printf(fmt, FORMAT_INT24(a), FORMAT_INT24(b), ...)

// The following doesn't work, because recursion is not allowed (and because no
// base case is defined).
#define FORMAT_ALL(x, ...) FORMAT_INT24(x) FORMAT_ALL(__VA_ARGS__)
```

## The Solution

In unrelated work, I was trying to construct a macro iteratively across header
files.  [A StackOverflow answer][cpp-append] by [rtpax][] introduced me to the
`push_macro` and `pop_macro` GCC pragmas, with the following critical note:

[cpp-append]: https://stackoverflow.com/a/45761452
[rtpax]: https://stackoverflow.com/users/4954108/rtpax

> If you pop a macro within it's own definition it will delay it's expansion
> until the macro is expanded for the first time. This allows you to make it's
> previous expansion part of it's own definition. However, since it is popped
> during it's expansion, it can only be used once

I realized that if the previous expansion was the same as the current definition
then recursion was effectively achieved.  rtpax also provided a code example
using the `_Pragma()` operator, which allows pragmas to be used within macro
definition.  Although they mention that the previous expansion can only be used
once, due to the consuming nature of `pop_macro`, I realized that pushing and
popping the same macro within its own definition would work around this.  This
leads us to:

```c
// A recursive macro which prints an infinite stream of 1s.
#define all_ones() \
    _Pragma("push_macro(\"all_ones\")") \
    _Pragma("pop_macro(\"all_ones\")") \
    1 all_ones()
```

In GCC, executing this macro leads to an infinite stream of 1s; the preprocessor
never halts.  In Clang, unfortunately, macro expansion halts immediately, and
only `1 all_ones()` is output, with no further expansion.  Still, at least on
GCC, recursion in macros is possible!

In order to apply this usefully, let's define a few auxilary macros:

```c
// Invoke an inline pragma.
#define cpp_pragma(p) _Pragma(#p)

// Redefine a macro for recursion.
#define cpp_redef(m) cpp_pragma(push_macro(#m)) cpp_pragma(pop_macro(#m))

// Map a function over the given variadic argument list.
#define cpp_va_map(f, ...) __VA_OPT__(cpp_va_map_ne(f, __VA_ARGS__))

// Map a function over the given non-empty variadic argument list.
#define cpp_va_map_ne(f, a, ...) cpp_redef(cpp_va_map_ne) \
        f(a) __VA_OPT__(, cpp_va_map_ne(f, __VA_ARGS__))
```

The latter two macros use the `__VA_OPT__` operator, which expands to its
argument only if some variadic arguments were given (i.e. if `__VA_ARGS__` is
not empty).  The base case (empty argument list) is thus accounted for.  Now to
use these macros for `my_printf()`:

```c
#define my_printf(fmt, ...) printf((fmt), cpp_va_map(FORMAT_INT24, __VA_ARGS__))
```

That was easy!  `cpp_va_map()` can be used in a great deal of situations like
this, eliminating the need for other hacks (e.g. X-macros) and doing so fairly
elegantly.  The nasty core that allows for this recursion remains hidden within
these macros, and is not exposed to the user at all.  Overall, it's a lovely
feature to have, and it makes C macros far more valuable.

## The Future

Recursion is an incredibly powerful tool, so I expect that some truly terrifying
macros will be built around it.  Regardless of whether recursive macros are a
good idea, `cpp_va_map()` provides a great deal of useful functionality in a
manner that is difficult to abuse, and it should be supported in some fashion or
the other even if fully-general recursion is unwanted.

Although the solution here is still fairly hacky, it's much better than the
workarounds used by Boost.Preprocessor and metalang99; they appear to use an
"evaluation engine" to force indirectly self-referential macros to be expanded
by the preprocessor up to a fixed number of times (currently, [16384][ml99-rec]
for metalang99), a solution that is far more verbose and error-prone.

[ml99-rec]: https://github.com/Hirrolot/metalang99/blob/master/include/metalang99/eval/rec.h

It's quite unfortunate that Clang doesn't allow for this hack; LLVM is able to
optimize code much better than GCC in some cases, and I was hoping to use its
optimizer combined with this feature for some more craziness.  I suppose it's
possible to manually pre-process code with GCC before passing it to Clang...
