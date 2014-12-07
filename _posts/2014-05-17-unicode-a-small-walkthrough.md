---
layout: posts
title: Unicode&#58; A Small Walkthrough
category: Programming
---

I am not going to justify why you should learn Unicode or not. [Joel Spolsky already provided more than enough rationale for you to do so](http://www.joelonsoftware.com/articles/Unicode.html). Here I will attempt to quickly explain how Unicode character sets are encoded in UTF-8, UTF-16, and UTF-32 encoding schemes. Of course, one can easily go to the [following](http://en.wikipedia.org/wiki/UTF-8) [Wikipedia](http://en.wikipedia.org/wiki/UTF-16) [pages](http://en.wikipedia.org/wiki/UTF-32), but this post will put the three together for an easy reference.

So why cover those three encoding schemes? What about the others? I decided to cover the three as they are the most commonly used Unicode encoding schemes. UTF-8, which I am sure you are aware, are used in many websites. There are still non-Unicode pages in circulation, but UTF-8 is dominant in this area. UTF-16 is the most popular choice of storing Unicode character in memory (more to this later). And UTF-32 is a simple encoding scheme to represent a Unicode character which is gaining traction in usage.

<!--read_more-->

A Bit of a Review About Unicode
-------------------------------

Before I dabble in the how character sets are encoded with the three encoding schemes, let's quickly go over some important stuff.

There are some terms, when used in the right context, will make your understanding of Unicode give a whole lot of sense.

These are the following:

* Code Point: You see this as U+XXXX where the U+ is the prefix to indicate that this number literal is a Unicode value, and the XXXX is a hexadecimal number, which, as of this writing, goes from 0000 to 10FFFF. Note that this number is not exactly the same as the encoded data (this is a common misconception about Unicode). The U+XXXX is an abstract representation of a single character.

* Octet: It used to be that a byte may not necessarily mean a group of 8 bits. It is just a convention to call it so. The Unicode consortium originally referred to a group of 8 bits as octets. Recent updates, trends, and usage of the term "byte" led the consortium to use it as well.

* Byte Order Mark: This is a sequence of bytes added at the begging of an encoded Unicode data. This serves two purposes:
    1. Indicate that the data coming after the BOM is to be treated as encoded Unicode data.
    2. Specify the byte ordering (endianness) of the data.

So, really, what is Unicode? Unicode is UTF-8, no? Unicode is a way to represent characters. I will not go on about the history, but think of Unicode as the "supposed" standard to make all characters/glyphs from all languages converge into a one big group to make information interchange easier (speaking of standards, see this relevant [xkcd](https://xkcd.com/927/)). Unicode is not UTF-8. When you define a "Unicode value" of a character, you would normally refer to the character's Unicode code point. For example, the character "ᄈ" (it's a Hangul), is represented in Unicode with code point U+1108. This U+1108 will have to be stored in computer's memory in some binary data. Sure, we can just use the hexadecimal value 0x1108 (and we indeed do so in some cases), but that is just one way of doing it. This is where the encoding schemes come to play like UTF-8, UTF-16, UTF-32, etc. Why have so many encoding schemes? Well, because different situations and use cases calls for different requirements. Key reasons include backward compatibility, storage concerns, and ease of conversions.

UTF-8 can be Seen in Many Places
--------------------------------

We all know that UTF-8 is used heavily in the world wide web. It is also the most flexible encoding because it is fully compatible with ASCII character set.

When I was starting out my computer science studies/work, I first thought that UTF-8 encoding is strictly 8-bit per character (and that UTF-16 and UTF-32 are both strictly 16-bit and 32-bit per character respectively). I was so taken aback when I started dissecting fonts from my work at PDFTron. Within a font program (or font file), glyphs are indexed by OS code pages or Unicode. Font files usually store UTF-16 encoded indexes (more on this later), but I encountered a font file that indexed glyphs using UTF-8. It threw me off a bit because I was having issues getting the necessary outline or strokes of a character's glyph that is beyond U+007F. It was then I realized that my knowledge about Unicode is incomplete and mostly incorrect. The point is that UTF-8 and UTF-16 are not fixed 8-bit and 16-bit. UTF-8 and UTF-16 can represent all the characters in the Unicode table as specified by the standard (U+0000 up to U+10FFFF).

Going back to UTF-8. As I just mentioned UTF-8 can represent all Unicode code points from U+0000 to U+10FFFF. In UTF-8 encoding, the most significant bit is used as a marker indicating how many octet sequences are part of this data encoding. If this bit is 0, then it only requires a single octet and Unicode code point encoded must be from U+0000 to U+007F. This single octet UTF-8 encoding is fully compatible with the ASCII character set. If the MSB is 1, then further processing of subsequent octets are needed. The head sequence's most significant bits will tell how many octets to process and the trailing sequence's MSB will be 10b. UTF-8 can be as small as 8-bits or as big as 32-bits.

UTF-16 is not Unicode
---------------------

Earlier, I mentioned that font files usually index glyphs in Unicode using UTF-16 encoding. The key reason for this is that most used characters in the Unicode scheme can be represented in no more than 16-bits. Many programs rarely use anything beyond U+FFFF. A key point is that UTF-16 requires less storage to encode the range U+0800 – U+FFFF than UTF-8 does (UTF-16 requires 16-bits, while UTF-8 requires 24-bits). Beyond U+FFFF, UTF-8 and UTF-16 both require 32-bits. You can only store so much in a single octet of UTF-8, so many platforms decided to go with UTF-16 as the main Unicode encoding. This seemed to somewhat made many beginners into thinking that UTF-16 is the way Unicode represents characters (I have no source, just my opinion).

In UTF-16 encoding scheme, code points from U+0000 to U+FFFF (except for the range U+D800 to U+DFFF) are represented in a single 16-bit data. The U+D800 to U+DFFF are used to represent surrogate pairs. Surrogate pairs are used to represent anything beyond U+FFFF in UTF-16. Surrogate pairs, as the name implies, is a pair of 16-bit data. To encode anything beyond U+FFFF, you take the hexadecimal face value of the code point and subtract 0x10000 from it. Add 0-bits at the beginning until you have a 20-bit data. Split the result into two 10-bit data. Add the first half to 0xD800 and the other half to 0xDC00, and you have your UTF-16 encoded character.

UTF-32 is Simple
----------------

Kind of. In this encoding scheme. Code points are represented in fixed 32-bit wide data. There is no bit fiddling involved in this scheme except for octet ordering. Each 32-bit data represent the exact Unicode code point as they are.

Some articles talk about UTF-32 is rarely used because no one wants to waste storage. I think that is partially incorrect. These days we have more storage than we did decades ago. Many platforms are switching towards UTF-32 encoding. The key advantages are starting to outweigh the storage concern. UTF-32 simply makes it easy to index characters in a string. It also makes it faster to encode to another code page since each UTF-32 character is already representing its Unicode code point as it is.

Byte Order Mark
---------------

Byte order marks are used in the following context based on the encoding schemes:

* UTF-8: The BOM is just an indicator that the data sequence must be understood as UTF-8 encoding. It is really more of a UTF-8 indicator more than a byte order indicator.

* UTF-16: The BOM indicates the byte ordering of the UTF-16.

* UTF-32: The same purpose as UTF-16.

Project UniCString
------------------

I wrote a simple Unicode library [here](https://github.com/vycasas/unicstring). It contains the code implementations that I have briefly discussed in this post. I do not really recommend using that library for production applications. I wrote it for edification purposes only. In particular, you will want to look at [Core/Codecs](https://github.com/vycasas/unicstring/tree/master/Core/Codecs) directory for UTF-8, UTF-16, and UTF-32 implementations. I wrote it in C++, but the encoding and decoding logic should be a useful reference.

If you want to use a well-written Unicode library, you should look into something more industry quality like [ICU](http://site.icu-project.org/).
