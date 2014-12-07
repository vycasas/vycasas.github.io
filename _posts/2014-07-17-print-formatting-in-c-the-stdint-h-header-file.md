---
layout: posts
title: Print Formatting in C&#58; The &#60;stdint.h&#62; Header File
category: Programming
---

Printing, whether to standard output, file, network, or any other stream, is a big part of writing programs. After all, programs are supposed to interface with something and interfacing requires some form of I/O (I/O in a sense that data are exchanged). When we print a variable in C, we use the common <stdio.h> functions:

* `printf`
* `fprintf`
* `sprintf`
* `snprintf`
* All the "v" versions of the above for variadic argument.

Using any of these functions to print a variable requires us to know the type and the size of the variable we want to print, as well as how we will be presenting them. The [table provided by cppreference.com](http://en.cppreference.com/w/c/io/fprintf) shows a very good guide which format specifier must be used with the specific type to get the output that we want. In this blog post, we will look at ways to properly use these functions with the `<stdint.h>` header file.

<!--read_more-->

Note: Before we continue, I would like to point out that we will be pedantic and will be following strict semantics here. We will be strict in a sense that specific types must be used with specific print formatting. While it is also sane to say that some interchanges are possible (e.g. using print formatting for wider sizes), we will not be considering them in this post. We want a code that is both warning and error free.

Printing Basics
---------------

Printing values are easy. Piece of cake! Or so we think. In most cases, we have an int. And on top of our heads, we know we use the `"%d"` format specifier we want to display. To print a char, we use `"%c"`. Say we want to print a 64-bit variable of type `unsigned long long int` in hexadecimal format in 16 width padded with `"0"`'s, then we want to use `"%016llx"`. Again, for built-in types, we will typically follow the chart from cppreference.com.

The `<stdint.h>` Header
-------------------------------

This header file provides some constraints on the types. In particular, this header file provides us with types of exact sizes as per specification. The typical fundamental integer types are specified to be at least some size (e.g. `int` type must be at least, but not necessary, 16-bits in size). In `<stdint.h>`, the following exact sized types are provided:

* int**X**_t
* uint**X**_t

Where "X" can be one of the: 8, 16, 32, and 64. These numbers specify the exact size of the integer type. All compilers will have the same size on all platforms when using these types. Interestingly, there is also the `intmax_t`/`uintmax_t` type. Now this type is supposed to represent the biggest width of an integer type.

If we want to print, say `int16_t` type, then we will naively use `"%hi"` because it makes sense. This format specifier is intended to print a `short int` type. And we know that a `short int` type has to be at least 16-bits in size. Well, that's actually a misguided statement. It is true that a `short int` has to be at least 16-bits in size, but `int16_t` has to be **exactly** 16-bits. What if the `short int` type is not 16-bits? Then we cannot use `"%hi"`.

Here's another one. Take for example the `"%p"` format specifier. While `"%p"` is intended to display the addresses of a pointer variable, what happens when we use the same format specifier for the type `uintptr_t`? Well, we will get the following:

![Compile Error](/img/b7b96123cf7cdbea6d621af743f0f581c80ff88f.png)

Ways to solve this involve either: (a) casting the `uintptr_t` type to `(void*)`, or (b) using the right specifier for `uintptr_t`. Solution a is perhaps the easiest way. If we change the code to:

{% highlight c %}
fprintf(stdout, "ptr: %p.\n", (void*) ptr);
{% endhighlight %}

The warning/error goes away. Now, what if one of our requirements is not use explicit type cast? We will have to make solution b to work. Now for solution b, we can just simply change the format specifier to

{% highlight c %}
fprintf(stdout, "ptr: 0x%lx.\n", ptr);
{% endhighlight %}

then we pass the warning/error barrier. But this begs to ask another question, what if the `uintptr_t` is not implemented as `unsigned long int`? Then we will get the warning/error again. Fear not though, as there still a solution. Introducing `<inttypes.h>` header file.

The `<inttypes.h>` Header File
------------------------------

This header file provides the needed format specifier that works for the `<stdint.h>` across all compilers and all platforms. Below will be our solutions to the problems introduced in the previous section.

If we want to print `int16_t`, then we need to use the format constant `PRId16`. The macro is just a string literal, so we can use them with our `printf` function like below:

{% highlight c %}
int16_t myVar = -1;
// note that you still need the "%" sign when using PRId16.
fprintf(stdout, "My int16_t value: %" PRId16 ".\n", myVar);
{% endhighlight %}

For the `uintptr_t` problem, we want to use the `PRIXPTR` macro. This macro will print the pointer in hexadecimal format.

{% highlight c %}
fprintf(stdout, "ptr: %" PRIXPTR ".\n", ptr);
{% endhighlight %}

I want to point out that there is a type from `<stdint.h>` header file where you do not need the `<inttypes.h>` for printing. This is the `intmax_t`/`uintmax_t` type. These types can be formatted using the `"%ji"`/`"%ju"` format specifiers. Using these format specifiers or the macros `PRI_MAX` will be perfectly fine. In a typical implementation, the `PRI_MAX` macro are just defined as `"%ji"`/`"%ju"`.

Conclusion
----------

When we decide to use the types defined in `<stdint.h>` header file, we have to use the macros defined in `<inttypes.h>` for printing. Although, we might be tempted to try to match the format specifier of the fundamental types like `short`, `int`, and `long` with the `<stdint.h>` types based on the size, this is not an ideal solution for a code that is supposed to work across different platforms. Part of the reason is that fundamental types are defined to be at least a specific size and may have different sizes on different CPU architectures (e.g. 32-bit vs 64-bit) whereas the `<stdint.h>` types have strict definitions according to the standard. As such, to make a warning/error free code, we should be using the format specifier as provided by the specification (i.e. use the right format specifiers for fundamental types, and use `<inttypes.h>` macros for `<stdint.h>` types).

