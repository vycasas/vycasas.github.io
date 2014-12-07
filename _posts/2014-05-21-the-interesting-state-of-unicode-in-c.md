---
layout: posts
title: The Interesting State of Unicode in C
category: Programming
---

My [previous technical article](/2014/05/17/unicode-a-small-walkthrough/) discussed about Unicode in general. This time, I want to briefly cover the C (and C++) aspects of it. When I wrote project [UniCString](https://github.com/vycasas/unicstring), I was looking for a way to include support for the new C11 character types, namely char16_t and char32_t. If you browse the source tree of project UniCString, you will notice that the [core implementation does support](https://github.com/vycasas/unicstring/blob/master/Core/String.hpp) these types, but the support is limited to how the project itself interprets the bits of char16_t and/or char32_t. In this blog post, I will quickly summarize my findings about these data types.

<!--read_more-->

The ever fancy wchar_t data type.
---------------------------------

Let me quickly go over the wchar_t type. To be clear, the wchar_t type is not supposed to represent any Unicode encodings. The purpose of wchar_t is to support bigger character sets than what a plain old char data type can. A char data type can hold the maximum value of byte (a byte is not necessarily 8-bits). The wchar_t's definition is as follows:

> an integer type whose range of values can represent distinct codes for all members of the largest extended character set speciﬁed among the supported locales

So what is the issue here? Well, I wouldn't really call it an issue, but more of a nuisance. The implementation of wchar_t depends on the compiler's stddef.h (the libc in most cases). Most of you should already know the following statements:

* wchar_t in Linux and Mac OS is 4 bytes wide.
* wchar_t in Windows is 2 bytes wide.

I decided to do a bit of investigation, I ran the following code on Linux, Mac OS, and Windows to test their wchar_t implementations:

{% highlight c %}
// 𧋊: U+272CA
// UTF-8:0xF0A78B8A
// UTF-16:0xD85C 0xDECA
// UTF-32: 0x000272CA
const wchar_t* wstr = L"𧋊";
uint32_t hexValue = 0x00;
 
fprintf(stdout, "sizeof (wchar_t): %lu.\n", sizeof (wchar_t));
fprintf(stdout, "wcslen(wstr): %lu.\n", wcslen(wstr));
 
if (sizeof (wchar_t) == 2) {
    hexValue = hexValue | (((uint32_t) wstr[0]) << 0x10);
    hexValue = hexValue | (((uint32_t) wstr[1]) << 0x00);
}
else if (sizeof (wchar_t) == 4) {
    hexValue = ((uint32_t) wstr[0]);
}
 
fprintf(stdout, "hexValue: 0x%08X.\n", hexValue);
{% endhighlight %}



And, the following are the results:

* Linux and Mac OS have the same results

    `sizeof (wchar_t): 4.`\\
    `wcslen(wstr): 1.`\\
    `hexValue: 0x000272CA.`

* Windows:

    `sizeof (wchar_t): 2.`\\
    `wcslen(wstr): 2.`\\
    `hexValue: 0xD85CDECA.`

If you work in the business of writing code that must compile across platforms, you will need to add some conditionals (#if directives or run time checks) so that your wchar_t data will work. This is why I say, it's more of a nuisance. By the way, don't worry about the result of wcslen from the sample above. The disparity happened because my Windows test platform did not use the right locale for the character in question (Chinese).

Again, wchar_t is not supposed to be Unicode, that is just the way some platforms chose to implement wchar_t. It's just that Linux and Mac OS chose to implement wchar_t with 4 bytes so they can store a character's Unicode code point in it. And Windows chose to implement with UTF-16 encoding. These are just 3 platforms, in other platforms, they may be using a different character set as a base implementation for wchar_t.

To make room for consistent implementation (especially for developers who like to fiddle with bits a lot), there are now two additional wide character types, the char16_t and char32_t.

The new char16_t and char32_t data types as well as the uchar.h header.
-----------------------------------------------------------------------

C11 introduced some consistency to deal with wide character strings. As of C11, there are 5 ways to create a string literal:

* `"string" /* basic character string */`
* `u8"string" /* UTF-8 encoded character string */`
* `L"string" /* wide character string */`
* `u"string" /* wide character string where each character is 16-bit wide */`
* `U"string" /* wide character string where each character is 32-bit wide */`

The new types char16_t and char32_t are supposed to hold the smallest character units in u"" and U"" string literals respectively. In particular, char16_t is for storing 16-bit wide characters, and char32_t for storing 32-bit wide characters.

----
{: .dszline}

HALT! OK. Before we continue, I need to make it clear that according to the standards, at least on the final draft, char16_t and char32_t does not necessarily represent UTF-16 and UTF-32 encodings. They are only UTF-16 and UTF-32 if and only if the following conditions are set:

* If the predefined macro __STDC_UTF_16__ is 1, then the encoding scheme used for char16_t is UTF-16.
* If the predefined macro __STDC_UTF_32__ is 1, then the encoding scheme used for char32_t is UTF-32.

The only thing that is guaranteed to be Unicode is the u8"" literal, which is supposed to be a UTF-8 encoded string.

----
{: .dszline}

While the standards seem to be going to the right direction here, it is lacking. Sure, it is nice that I can indicate the way a character will be stored in memory (by the number of bytes), but how do I pass it in I/O through streams? We have printf and wprintf, which works (hopefully). But how do I go about printing a char32_t* literal? There is no Uprintf, u32printf, or something. The only way to do it at the moment is to print out the bit values of char16_t/char32_t and make sure that the output file or stream's encoding is set to the right encoding. Unfortunately, in order to do this correctly, we will need to normalize manually.

Compiler wise, GCC and Clang support the new string literals, but MSVC has the following [statement](http://msdn.microsoft.com/en-us/library/69ze775t.aspx) (as of MSVC 12.0):

> The U, u, and u8 prefixes are not supported.

So what's the point of the new string literals? Well, it is not really a futile effort by the committee. In C11, we have a new header file called uchar.h. The uchar.h is supposed to provide the following interface (taken from en.cppreference.com):

{% highlight c %}
size_t mbrtoc16(char16_t* pc16, const char* s, size_t n, mbstate_t* ps);

size_t c16rtomb(char* s, char16_t c16, mbstate_t* ps);

size_t mbrtoc32(char32_t* pc32, const char* s, size_t n, mbstate_t* ps);

size_t c32rtomb(char* s, char32_t c32, mbstate_t* ps);
{% endhighlight %}

I see, so we use the new string literal support as a means to convert MBCS to the requested  encoding. This means that if we want to use the new string literals with I/O streams, we will need to use the utility functions as provided above (at least for now).

The only thing that bothered me in the interface above is that in the final draft of C11, nowhere within that document a possibility of UTF-16 encoding with surrogate pairs is mentioned. I am not sure if this is an oversight or the committee just decided not to bother with it and leave it to implementation (but this defeats the standard's purpose). The [glibc project](https://sourceware.org/git/?p=glibc.git;a=blob_plain;f=wcsmbs/c16rtomb.c;hb=HEAD) also put out the following comment:

`// XXX The ISO C 11 spec I have does not say anything about handling`\\
`// XXX surrogates in this interface.`

This means that processing any UTF-16 encoded character that requires surrogate pair (like in my sample snippet above) may or may not work in C11. Either way, it is not entirely wrong because as mentioned in my big note above, these APIs will only be in UTF-16 and UTF-32 encoding if the predefined macros are set to 1. Perhaps it is just incomplete, maybe another API is necessary, e.g. mbrtom16 and m16rtomb which will allow multi-16-bit characters.

Well, fortunately, as of this writing, I saw codes and several implementation of the uchar.h header file, but none of them are in working state. I attempted to use the C libraries on Linux, Mac OS, and Windows, and none of them provide a working interface for the uchar.h functions. At the moment, Unicode in C/C++ is not yet ready. I am not sure what is its direction, but perhaps, we should stick to using 3rd party libraries to deal with Unicode encoding.

PS: All of my findings and tests from this post are based on systems where en_CA.UTF-8 is the locale and console code pages set to UTF-8.
