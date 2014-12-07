---
layout: posts
title: Understanding Character/String Literals in C/C++ 
category: Programming
---

In the latest published standard of C ([ISO/IEC 9899:2011](http://www.open-std.org/jtc1/sc22/wg14/)) and C++ ([ISO/IEC 14882:2011](http://www.open-std.org/jtc1/sc22/wg14/)), there are new character literal types introduced, these standards now give us 5 different character literal types. These are the following:

1. `char` literal type (surrounded by single quotes, e.g. `'A'`).
2. `wchar_t` literal type (surrounded by single quotes with L prefix, e.g. `L'A'`).
3. multi-character literal type (like `char` literal type, but has two characters instead, e.g. `'ABCD'`).
4. `char16_t` literal type (surrounded by single quotes with u prefix, e.g. `u'A'`).
5. `char32_t` literal type (surrounded by single quotes with U prefix, e.g. `U'A'`).

In addition to that, the standard has also added new string literal types. The following are ways to create string literal types in C11/C++11:

1. `char` string literal type (surrounded by double quotes, e.g. `"ABC"`).
2. `wchar_t` string literal type (surrounded by double quotes with L prefix, e.g. `L"ABC"`).
3. UTF-8 string literal type (surrounded by double quotes with u8 prefix, e.g. `u8"ABC"`). This string literal's character type is `char`, so to reference this literal, the variable must be of type `const char*`.
4. `char16_t` string literal type (surrounded by double quotes with u prefix, e.g. `u"ABC"`).
5. `char32_t` string literal type (surrounded by double quotes with U prefix, e.g, `U"ABC"`).

The goal of this blog post is to cover correct values/characters that can be used inside the character and/or string literal types. In particular, using Unicode characters will be given emphasis. All code snippets provided in this post have been verified with GCC 4.8.3 for C11 and clang 3.4 for C++11.

<!--read_more-->

Before continuing, please do not be confused between a character type vs a character literal type. This article focuses more on the literal types.

The Arbitrary Escape Sequences
------------------------------

Arbitrary escape sequences are the following:

* `\NNN` : specifies a value in octal mode for character and string literals. e.g. `'\101'` or `"\101"`.
* `\xNN...` : specifies a value in hexadecimal mode for character and string literals. e.g. `'\x41'` or `"\x41"`.
* `\uNNNN` : specifies a value in UTF-16 encoding for character and string literals. e.g. `'\u3042'` or `"\u3042"`.
* `\UNNNNNNNN` : specifies a value in UTF-32 encoding for character and string literals. e.g. `'\U0010FFFF'` or `"\U0010FFFF"`.

The are called arbitrary escape sequences because arbitrary values are used to represent a character. There is a special characteristic for the hexadecimal arbitrary escape sequence. It has no specific length unlike the other escape sequence patterns. The octal, UTF-16, and UTF-32 escape sequence patterns require specific number of digits in order to be well formed. The octal escape sequence pattern, however, does not require to exactly 3 digits (it can be 1, 2, or 3 digits in length).

Let’s clarify the above notes with some samples below.

{% highlight c %}
char c1 = '\101'; // c1 = 'A'
char c2 = '\x41'; // c2 = 'A'

// Notice that \41 is a 2 digit octal escape sequence.
const char* cstr1 = "\101\41"; // cstr1 = "A!"
const char* cstr2 = "\x41\41"; // cstr2 = "A!"

// c16str1 = "A!"
const char16_t* c16str1 = u"\u0041\u0021";
// c16str2 = "A!";
const char16_t* c16str2 = u"\x0041\x0021";
{% endhighlight %}

We will continue on later with more examples for `\u` and `\U` escape sequence patterns. For now, just keep in mind that `\u` and `\U` are not restricted to `char16_t` and `char32_t` types. They can be used in other types, but there are rules surrounding the use of these escape sequence patterns for these types. Also, these two escape sequence patterns require exact amount of digits. The rationale for this can be seen with the following snippet:

{% highlight c %}
const char16_t* c16str = u"\u4142";
{% endhighlight %}

If we allow any number of digits for the `\u`, then how should we translate the code above? Should it be `"A42"`? or should it be `"䅂"` (U+4142). This ambiguity is the reason why `\u` and `\U` must be exactly N amount of digits. If we want to make the above snippet to translate to `"A42"`, we will have to do the following:

{% highlight c %}
const char16_t* c16str = u"\u004142";
{% endhighlight %}

We will cover more of this in the following sections.

The `char` and `wchar_t` Literal Types
----------------------------------

We will not go that too deep with these types. We have learned this throughout our programming classes. If you have not read my previous articles, however, please take a quick look at [some notes I made](/2014/05/21/the-interesting-state-of-unicode-in-c/) regarding `wchar_t` type (The `wchar_t` section should also apply to C++).

The `char` literal type, requires that each character in the literal must be at most `CHAR_BIT` bits in size (typically, compilers define `CHAR_BIT` to be 8). What this means is that as long as each character is `CHAR_BIT` bits in size, then we are crafting a valid char literal. For example:

{% highlight c %}
char
    c1 = 'A', // OK
    c2 = '\101', // OK
    c3 = '\x41', // OK
    c4 = '\u0041', // OK
    c5 = '\U00000041' // OK
;

char
    c6 = '\x3042', // ERROR
    c7 = '\u0144', // ERROR
    c8 = '\U0010FFFF' // ERROR
;
{% endhighlight %}

----
{: .dszline}

NOTE: Before proceeding, please keep in mind that the character inside a literal type will be evaluated using the literal's type not the destination's type. A common misunderstanding is that the destination's type represents the parsed literal's type. This is not a very accurate statement. A literal type can be implicitly converted to the destination type and warnings will be raised by smart compilers if there will be a possible data slicing.

As with the example above,  it is possible to do the following:

{% highlight c %} 
char c = '\u0041';
{% endhighlight %}

This is OK because the literal operator's type evaluates to char and the literal value `\u0041` can fit in a single `char` type.

----
{: .dszline}

The `wchar_t` literal type follows the same rules (like with any other literals) as the `char` type. The characters in a `wchar_t` literal type must be at most `(sizeof (wchar_t) * CHAR_BIT)` bits in size. For example:

{% highlight c %}
// Assuming (sizeof (wchar_t) * CHAR_BIT) evaluates to 32
// like in Linux and Mac OS.
 
wchar_t wc1 = L'\u3042'; // OK
wchar_t wc2 = L'\U0010FFFF'; // OK
{% endhighlight %}

Finally, before moving on, let us clarify the multi-character literal type a bit. A multi-character literal type, e.g. `'1234'`, will always have an `int` type. The literal is not restricted to 4 characters, but the result of its translation will be stored in an `int` type. So for example, if we use `'12345678'`, those 8 characters will be translated by a compiler into an `int` type (provided the compiler allows this much in a characters in the literal). While a valid C/C++ literal type, it is not commonly seen in the wild. Personally, I saw them used in some legacy code that wants to create an int type and initialize its value using a character literal. This goes with the assumption that each character in the literal will get a value as they are in ASCII. For example:

{% highlight c %}
// Assuming compiler will treat each character in ASCII mode,
// the value of myInt will be 0x41424344.
int myInt = 'ABCD';
{% endhighlight %}

It is recommended to avoid this literal if possible. As with any good programming practices, seeing the intention through code must be given priority over "wise" tricks that lead to confusion.

The `char16_t` and `char32_t` Literal Types
-------------------------------------------

The `char16_t` and `char32_t` literal values must be exactly 16 and 32 bits respectively. The following code snippet explains this concept.

{% highlight c %}
// u'' literal operator produces a char16_t.
// Values must be at most 16-bits.
char16_t c16 = u'A'; // OK
c16 = u'\u3042'; // OK
c16 = u'\x3042'; // OK
c16 = u'\U00003042'; // OK
 
c16 = u'\x10FFFF'; // ERROR
c16 = u'\U0010FFFF'; // ERROR
 
// U'' literal operator produces a char32_t.
// Values must be at most 32-bits.
char32_t c32 = U'A'; // OK
c32 = U'\u3042'; // OK
c32 = U'\U0010FFFF'; // OK
c32 = U'\x10FFFF'; // OK
{% endhighlight %}

Keep in mind that in C, the `char16_t` and `char32_t` types themselves may not be stored in memory as UTF-16 and UTF-32 encoding respectively (this seems to be implementation specific). They are just guaranteed to have specific sizes in memory. Having said that, do not be confused nor interchange the purpose of `u''` and `\u` (or `U''` and `\U`). The former (`u''`) is a literal operator to create `char16_t` (which may not be UTF-16), the latter (`\u`) is a escape sequence where its face value in hex must be of UTF-16 encoding. In C++, however, the standard specifically says that these two types must have the code point values of the Unicode characters.

You may think that if `char16_t` (or `char32_t`) is stored as UTF-16 (or UTF-32 for `char32_t`) in memory, then the `\x` and `\u` escape sequences have not differences. This is incorrect. While we can say that in this scenario, both escape sequences will produce the same value in memory, the `\u`, however, will need to make sure that it is a valid UTF-16 encoding. Take the following snippet for example:

{% highlight c %}
// Assume that char16_t is stored as UTF-16 in memory.
char16_t c16a = u'\x3041';
char16_t c16b = u'\u3041';
char16_t c16c = u'\xDC00'; // OK
char16_t c16d = u'\uDC00'; // ERROR
{% endhighlight %}

In the sample above, `c16a` and `c16b` will be equivalent with each other. `c16c` will be fine as the `\x` escape sequence assigns a value of `0xDC00`. `c16d`, however, is not valid because it is an invalid UTF-16 encoding (it is missing a surrogate pair). Of course, interpreting the valid `c16c` in I/O is questionable.

Throughout the rest of this article, we will work under the assumption that `char16_t` and `char32_t` types are encoded as UTF-16 and UTF-32 in memory (i.e. C `char16_t`/`char32_t` types are implemented like C++). It will be insane for a compiler not to use these encoding schemes anyway since (1) `char16_t` is limited to Unicode characters in the basic multilingual plane (which are code points of no more than 16-bits) and (2) `char32_t` is wide enough to cover all valid code points in Unicode. Using other schemes to store these types in memory just makes things complicated and is unnecessary.

The String Literal Types
------------------------

The string literal operators are surrounded by double quotes with a prefix depending on the type one wishes to create. These should also be straightforward since C++ programmers are taught about this at the early stages. There is, however, a small distinction between a character and string literal type. While in a character literal type, the values of the literal must fit in the literal type, the string literal types relaxes (but not completely eliminates) this restriction. On the previous examples, we see that the following code will result in a compilation error because the resulting literal type's value will not fit in the literal type.

{% highlight c %}
char16_t c16 = u'\U0010FFFF'; // ERROR
{% endhighlight %}

The literal value `\U0010FFFF` requires 32-bits but the literal type, `u''` only allows for 16-bits. The string literal types, however, allow this

{% highlight c %}
const char16_t* c16str = u"\U0010FFFF";
// The literal evaluates to type of const char16[3].
{% endhighlight %}

Pretty easy. Keep in mind, however, that `c16str` from above will not have the following:

{% highlight c %}
c16str[0] == 0x0010; // false
c16str[1] == 0xFFFF; // false
{% endhighlight %}

The actual values will be:

{% highlight c %}
c16str[0] == 0xDBFF; // true
c16str[1] == 0xDFFF; // true
{% endhighlight %}

This is because the `\U0010FFFF` literal value requires surrogate pairs to be encoded as UTF-16. As mentioned, the restriction is relaxed, not eliminated. The following will still produce a compiler error:

{% highlight c %}
const char16_t* c16str = u"\u3042\x0010FFFF"; // ERROR
{% endhighlight %}

The literal value of `\x0010FFFF` is asking for a 32-bit character. The `\U0010FFFF` is allowed because it is a valid Unicode code point (and can be encoded into UTF-16 using surrogate pairs).

Now, here is small quiz, what is the resulting length of the following literal?

{% highlight c %}
const char16_t* cstr = u"AB\U0010FFFF\u3042\x43";
{% endhighlight %}

Following the pattern from the previous example, it is easy to guess that the literal will of a type of an array of 7 `const char16_t` values (i.e. `const char16_t[7]`).

What about the following?

{% highlight c %}
const char* cstr = "AB\U0010FFFF\u3042\x43";
{% endhighlight %}

The answer to the question is actually implementation defined. If a compiler chooses to store this literal in memory by using UTF-8 encoding, then you will get a length of 10 (plus the `'\0'` character). But some compilers may choose to use a different MBCS (multi-byte character string) method so you may not get a length of 10.

In a `char` string literal type, the resulting value produced has a type of an array of `const char`. That being the requirement, a `char` string (referred to as narrow string from here on out) literal is stored on memory usually based on the current character set used by platform. This is where the `u8""` literal operator comes handy. The `u8""` literal operator will make the string be encoded as UTF-8 in memory.

For example, if the system and C encoding is Shift-JIS, then the following snippet shows how narrow string literals will be handled:

{% highlight c %}
const char* cstr = "必要";
// cstr will have a value of:
// 0    1    3    4    5
// 0x95 0x4B 0x97 0x76 0x00
const char* c8str = u8"必要";
// c8str will have a value of:
// 0    1    2    3    4    5    6
// 0xE5 0xBF 0x85 0xE8 0xA6 0x81 0x00
{% endhighlight %}

If the system and C encoding is UTF-8, then both of these strings will have the same value (equal to that of `c8str`).

Conclusion: Fitting the Bits
----------------------------

Creating literals are easy enough, but rules must still be followed. When you try to craft your own character literal, always think about the resulting type. **The resulting character or string type must always be able to hold enough information about the literal characters**.  While we can use any of the arbitrary escape sequences to create a character literal, each of the characters must fit on the resulting type. We have covered them all over this article, in particular:

* `char` character literal types : the character in the literal must be at most `CHAR_BIT` bits in size.
* `wchar_t` character literal types: the character in the literal must be at most `(sizeof (wchar_t) * CHAR_BIT)` bits in size.
* `char16_t` character literal types: the character in the literal must be at most 16 bits in size.
* `char32_t` character literal types: the character in the literal must be at most 32 bits in size.

String literal types relaxes these requirements, but the resulting string of characters must be representable in one or more instances of the underlying character types.

Another important thing to remember is that the enclosing type is evaluated first before the destination type, as I have noted before. Take for example the following snippet:

{% highlight c %}
char16_t c16 = '\u3041';
{% endhighlight %}

A quick glance make it seem like OK, because we judge this statement as creating a `char16_t` type. This is, however, an invalid literal. It is indeed OK to assign a char literal to `char16_t` type, but the rule "a char literal type cannot have more than `CHAR_BIT` bits per character" still applies. The `\u` escape sequence pattern above requires 16 bits of storage, but the enclosing literal is a `char` literal type. So be careful of this.

These points are very straight forward and easy to keep in mind. Hopefully, by having this information equipped, you are now getting less stressed when crafting character and string literals in C and C++.
