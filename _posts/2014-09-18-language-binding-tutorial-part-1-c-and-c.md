---
layout: posts
title: Language Binding Tutorial Part 1&#58; C and C++ 
category: Programming
---

Over the next few months, I will be posting a series of lessons about binding languages. What is a language binding? A language binding is a way to make an interface of a language communicate to another. For example, if we are provided with a C library, how can we use this in Python? This is where a middle layer must come to play. The role of this middle layer is provide a channel for C and Python to communicate with each other. The goal of this lesson series is to provide us with basic knowledge and starting point on how we would achieve such middle layer. This series is divided into the following parts (grouped by languages):

* C and C++
* Java
* C#
* Objective-C
* Python
* Ruby

In my personal experience, once you get the hang of creating a middle layer between C/C++ and Java, the rest pretty much follows the same patterns and pre-cautions. As mentioned, these lessons are just simply to get the gears oiled. Not every case I will mention here is a silver bullet solution. The design and implementation varies and you will always need to evaluate your best option. I will, however, try to describe as much of the potential issues one commonly goes through when writing these middle layer module.

<!--read_more-->

Normally, when writing a code that will be consumed by different languages, one would start at the lowest level. As such, these lessons will guide you through writing an interface for low level languages. This means that most of the cases outlined in these lessons are about calling a C/C++ function in higher abstracted languages via some interface forwarding. There are some cases, however, where it is necessary to invoke a high level construct in the lower levels (I call these reverse invocation). For example, given a Java object and passed to a C function, how will the C function invoke a method on this object? There will be parts of these lessons where this will be discussed.

SWIG
----

You may have heard of the SWIG utility from before. This is a very neat program which will generate wrapper code for your API in the target language of your choice in a less tedious way. Because the goal of this series is to provide you with knowledge how language communicates, we will not be using SWIG to generate our wrappers and will be writing them manually by hand. For more information about SWIG, please see their website here: [http://www.swig.org/](http://www.swig.org/)

Sample
------

Throughout the lesson we will be building up code as we go along. The sample code is available at this repository: [https://www.github.com/vycasas/language_bindings_lesson](https://www.github.com/vycasas/language_bindings_lesson)

On the Lowest Level: Why C Matters
----------------------------------

If you have not worked lot with C (or C++) to write system level code, then you may want to read this section as the concepts are necessary when crossing one language barrier to another. In particular, if you have never written and worked with the ins and outs of a shared object/library, then I recommend not skipping this section. This section will explain why some things like function names, data types, calling conventions, and error handling should work like they are when making a language binding.

### Key Points ###

Rather than boil too much theory in this blog post, I will outline key points instead. These key points will work as a checklist when writing your native library (note the word "native"). The goal of these key points is to make sure that your low level native library (in C or C++) is designed appropriately so that binding other languages will be less problematic.

**Point 1**: Know your ABI. Perhaps this is the only key point that is actually necessary. ABI describes how your system modules communicate with each other. The following key points mention which specific concepts regarding ABI you must be mostly concerned about when writing a native library.

**Point 2**: Use plain old datatypes (POD) when crossing ABI. Even better, use size-defined plain old datatypes. Doing so helps ensure that you can pass data across ABI without much issues. If your API says this function must take in exactly 4 bytes of integer, then a contract forms that the argument passed to this function will be interpreted as a 4-byte integer.

**Point 3**: Use appropriate naming. Also, avoid C++ name mangling. Naming collisions must be avoided because in most platforms, the first symbol with the name encountered is the one considered for use. For example, if you have a function named "print_this" in both liba.a and libb.a, the following link commands result in different behaviors:

{% highlight bash %}
$LD -o my_program1 my_program.o libb.a liba.a

$LD -o my_program2 my_program.o liba.a libb.a
{% endhighlight %}

Assuming the linker does not error out on multiply defined symbols (which it should), using the above commands will yield to my_program1 using libb.a's print_this, while my_program2 will be using liba.a. To prevent such scenarios, it is necessary to make sure you provide a unique name for each symbol your native library exports.

**Point 4**: Do not throw exceptions across ABI. This is related to key point number 2. Throwing an exception across ABI means you are giving something outside your code that is not a POD. This is especially important, if your core code base is not written in pure C (e.g. C++ code wrapped with C API).

**Point 5**: Compile your native library API in strict C mode. You do not need to compile the whole source code base of the native library in strict C. The only important part is the API. By compiling in strict C mode, you gain an extra guarantee that your code works as close as it would be in the bare metal. And by doing so, you ensure high compatibility not only across different languages, but also across different compilers/platforms. When I compile my native library's interface, I usually prefer to pass the following checks:

{% highlight bash %}
-std=c11 -W -Wall -Wextra -Werrors -pedantic -pedantic-errors
{% endhighlight %}

If you wish to read more about good native library interface design, you can read this paper by Ulrich Drepper: [http://www.akkadia.org/drepper/dsohowto.pdf](http://www.akkadia.org/drepper/dsohowto.pdf). The list of points I laid out here is not complete. Your requirements will definitely vary. At the end of the day, what is important is that you know what your are exporting and you specify which data is passed back and forth to your native library.

Binding C library with C++ code
-------------------------------

Now that we have gone through some key points about writing a native library, let us go through about wrapping this native library with other languages so it can be used by the target language. We will go under the assumption that the native library's interface is in C (we will keep this assumption throughout the lesson series).

### The C API ###

Say we have the following API for a native library (written in C):

[https://github.com/vycasas/language.bindings/blob/master/c/api.h](https://github.com/vycasas/language.bindings/blob/master/c/api.h)

Pretty typical use case API. We can create a "Person" instance with some information attached to it. Let us go through the important points (not in the order above) of this API.

First, notice that the code for the public API (excluding the implementation) is in pure C. All data types passed are just simple POD. You will notice that we return and receive a Person and Address instance via pointer. This actually follows a well-known programming design pattern called PIMPL (pointer to implementation). I recommend using PIMPL as not only it provides an abstraction between API and implementation detail, but it also works very nicely with public API data passing.

Next, we ensure that C++ compilers will not mangle the name when it tries to include this header file. This is done by adding the `extern "C"` directive at the beginning of each function. Without this, the library will run into linkage issues in C++ mode.

Notice that problems with the API are indicated by error codes (all API return types represent error codes). This is done because we do not want exceptions going beyond the API, but we still want to indicate success and failure. In this sample, the error code is implemented as if they are a number representing a state. In other projects, this can be a pointer to a core/internal exception object. Like all other API objects, we want to provide helper functions for the error code so we can obtain more information about it.

Finally, this code compiles in strict C mode (check the Makefile recipe for it).

The sample is implemented in pure C. But as I mentioned before, this may not always be the case. There are many native libraries out there that are written in C++ and provides a C API layer for compatibility. If the core is written in C++, then the PIMPL pattern works by passing a pointer to a class instance out to the API users. For example:

{% highlight c++ %}
/* Assume PersonHandle is an opaque pointer */
extern "C"
int MyLibCreatePerson(const char* name, PersonHandle* person)
{
    CXXPerson* result = new CXXPerson(name);
    person = static_cast<PersonHandle*>(result);
    // some other error checking
    return (0);
}
{% endhighlight %}

As we can see, there is really nothing special. Because the core implementation varies depending on the design, we will not talk too much about it in detail. We are more concerned about the API implementation anyway.

The C++ API
-----------

Now that we have a native library that we know will not have too much compatibility issues, let us move on and try to create the C++ API for it. As you would have thought, creating a C++ layer on top of C code is pretty straight forward. After all, C code is compatible with C++ compilers. This makes it easy enough for us to not worry about type mapping (don't forget that term, we will be encountering huge amount of type mapping topics later).

The following snippet is how I would create a C++ API for the C API above:

[https://github.com/vycasas/language.bindings/blob/master/cxx/api.hxx](https://github.com/vycasas/language.bindings/blob/master/cxx/api.hxx)

Again, pretty simple and straight forward. We create classes that represent our objects and add functions which would operate on the said objects. One thing to mention, though, is the way errors are handled. To avoid too much code duplication, I usually create a macro to check for results, and create a C++ API exception if necessary. In the sample code, this is provided by the following:

{% highlight c++ %}
#define CXXLIB_API_CHECK(result) \
    if (result != 0) \
        throw (CXXLib::Exception(result));
{% endhighlight %}

And you will notice that in several member functions, after a C API is a called, the result gets evaluated as necesary.

Finally, it is very important to consider how this C++ API layer will be provided. Will it be pre-compiled binary or provided as source? I tend to favor the latter because there are currently no ABI specifications for C++ (it may arrive with future specifications). What this means is that when you compile an API layer using GCC, it may/will not work when you attempt to use a different C++ compiler. This is pretty evident with Windows environment. A good example is the Qt Project, if you look at their Windows releases, you can see that they provide multiple builds depending on the compiler. Also, if your API layer is a simple wrapper implementation like the sample above (i.e. no intellectual property must be protected), then there is really no reason to close source this layer. In some libraries, the C++ API layer is provided as a header only and all the implementation detail of class and functions are inlined so that there are barely any, if none, performance loss.

Next Part
---------

We have covered how to properly write a native library that will be highly compatible with different platforms as well as created a simple C++ API layer. In the next part of this series, we will be covering some advance topics about C++ code wrapping. Then after that, we will be moving on and will be covering the Java programming language.
