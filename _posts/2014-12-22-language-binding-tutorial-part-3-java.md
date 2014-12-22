---
layout: posts
title: Language Binding Tutorial Part 3&#58; Java
category: Programming
---
We have covered the basics (and some advanced topics) in making a language binding for a native library. In our previous discussions, we wrote a simple native library in C (for higher compatibility) and provided the necessary middle layer so that the C objects can be used in C++ in an object oriented manner. In this article, we will go over adding an API for Java. We will be taking what we have worked on the previous topics and will be adding on top of it.

<!--read_more-->

The Java Native Interface
-------------------------

In order to add a middle layer to interface with Java, we will need to use the Java Native Interface, or JNI, so that our native library can communicate with the JVM and vice versa. As the name states, Java Native Interface is a mechanism that allows native code to be consumed in the Java side. 

Technically, there is nothing advanced about the JNI - it is all just following specific conventions while taking type mapping into consideration. I said "nothing advanced", not "nothing complicated". The mechanism itself is somewhat complicated due to lack of detailed documentation. Add the fact that not too many choose to develop with JNI (thanks to Android, JNI has been significantly adopted), all the solutions you may need will most likely come from trial and error approach.

Another thing to keep in mind that, when you use JNI, the compiled native library must match the JVM's architecture. Most of the errors in loading is due to the incompatibility between the native code and JVM. When you are writing Java code, you need not to think about whether your code will work on x86 or x64, but when you add JNI, this becomes a very important aspect.

Creating a Middle Layer
-----------------------

OK, let's work on our existing code. In the previous topic, we have a working C++ library that wraps a C library. Now that we want to add Java language support for our small library, we can go about it in two ways: (A) wrap the C++ library, or (B) wrap the C library. Both methods are valid solutions. Both have pros and cons and it all really boils down to preference. Wrapping the C++ library means when we need to use the Java API, we have to depend on both C++ and C libraries. But the layer provided by C++ is so thin so it does not have a very huge effect. Wrapping the C library seems more straightforward because the JNI is essentially a C API code to prevent ABI incompatibilities (see my previous posts about ABI). For this project, we will be wrapping the C++ library. I prefer to take advantage of some C++ features thus the decision.

First thing we do is to create the API itself. We want a close 1:1 equivalence of types and consistency, so we have define the following class in Java:

{% highlight java %}
public class JavaLibException extends Exception
{
    // ... JavaLibException members
}

public class Library
{
    // ... Library members
}

public class Address
{
    // ... Address members
}

public class Person
{
    // ... Person memebers
}

public interface IGenerator
{
    // ... IGenerator definition
}

public class Printer
{
    // ... Printer members
}
{% endhighlight %}

You can the individual implementation details here: [https://github.com/vycasas/language.bindings/tree/master/java/net/dotslashzero/javalib](https://github.com/vycasas/language.bindings/tree/master/java/net/dotslashzero/javalib). We will discuss the implementation below.

As with any project, I prefer creating a utility module. For this project, we put them in the Core.java file. This file simply provides abstract wrapping for the native objects. So how we map the native objects with the Java classes we have written?

The pattern that I follow is similar to what we did in C++ - I use the PIMPL pattern. In PIMPL pattern, the API classes only holds a pointer to the implementation. API methods acts on the implementation itself. To do this pattern in Java, the API classes holds the address of the native type's instance. Having said that, we have to understand that there are no "pointer types" in Java, thus we have to refer to an instance via its address as an integral value. In the sample's case, we store this value in a Java long type. This is why when inspecting the Core.java file, you can see that `NativeType`'s definition is "`class NativeType extends WrappedType<Long>`". This class is our reference (or a Java pointer if you prefer) to the C++ native type.

Object Livelihood
-----------------

Before we move on with ABI communications, we need to understand object management. In particular, we need to stress a bit about ownership. In C++, we manage object livelihood (in memory) manually. Whereas in Java, the JVM's garbage collector checks for unused/un-referenced objects and deletes them. Because we are creating an API for the Java language, it is necessary to adhere to Java's rules of object management. So, how do we do this? It is really that bad. Essentially, our Java API classes must override Object's finalize method. This is because it will only be during this time that we can tell that this current Java object is being garbage collected. Having said that, our pattern for creating and destroying objects with our library will be as follows:

{% highlight java %}
public class MyJavaAPIClass
{
    private NativeType _nativePtr;
    public MyJavaAPIClass()
    {
        _nativePtr = createCXXObject();
    }

    @Override
    public void finalize()
    {
        destroyCXXObject(_nativePtr);
        return;
    }
}
{% endhighlight %}

Keep in mind this pattern as we will be pretty much using the same with other language bindings in the future. As you can see from the snippet, we create a native type on the constructor (obviously), and destory the native type when the JVM's garbage collector cleans up the object.

ABI
---

In order for the Java language to call native functions, it has to inform that JVM that a native function must be called. The JVM will then find this native function from loaded modules then call them. To achieve such cohesiveness, the JVM and the native module are binded via the system's ABI. ABI's are technically defined by the system (the operating system to be precise) and varies from one another. Luckily for us developers, the JNI specification exists so that we can write consistent code which can compile/work across different systems.

Going back to our snippet above (re: creating and destroying objects), there are two methods that provide actions for the `NativeType` data. The "`createCXXObject`" and "`destroyCXXObject`" methods are called "`native`" methods in JNI terms. When defined, the JVM is informed that these methods must be called from one of the loaded native modules. To complete the snippet above:

{% highlight java %}
public class MyJavaAPIClass
{
    private NativeType _nativePtr;
    public MyJavaAPIClass()
    {
        _nativePtr = createCXXObject();
    }

    @Override
    public void finalize()
    {
        destroyCXXObject(_nativePtr);
        return;
    }

    private static native NativeType createCXXObject();
    private static native void destroyCXXObject(NativeType nativeType);
}
{% endhighlight %}

There is no need to provide implementations for these methods because their definitions will reside on the native module.

Now, how do we write the native module? This is where the meat of JNI comes to play. The Java development kit (JDK) comes with a tool called javah. This tool will generate the header file that you can use to write your native module. You don't need to exactly use what javah outputs but doing so leads to better compatibility. The javah tool will basically scan your .java files for native methods and will generate a header file that contains function declarations to match the native methods. For example, we have a complied Java code from above in mylibrary.mypackage.MyJavaAPIClass, we need to run the javah tool like:

{% highlight bash %}
> javah mylibrary.mypackage.MyJavaAPIClass
{% endhighlight %}

Note that MyJavaAPIClass must be compiled first using javac. The command above will create a C header file which will have contents like below:

{% highlight c %}
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class mylibrary_mypackage_MyJavaAPIClass */

#ifndef _Included_mylibrary_mypackage_MyJavaAPIClass
#define _Included_mylibrary_mypackage_MyJavaAPIClass
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     mylibrary_mypackage_MyJavaAPIClass
 * Method:    createCXXObject
 * Signature: ()Ljava/lang/Object;
 */
JNIEXPORT jobject JNICALL Java_mylibrary_mypackage_MyJavaAPIClass_createCXXObject
  (JNIEnv *, jclass);

/*
 * Class:     mylibrary_mypackage_MyJavaAPIClass
 * Method:    destroyCXXObject
 * Signature: (Ljava/lang/Object;)V
 */
JNIEXPORT void JNICALL Java_mylibrary_mypackage_MyJavaAPIClass_destroyCXXObject
  (JNIEnv *, jclass, jobject);

#ifdef __cplusplus
}
#endif
#endif
{% endhighlight %}

After having this file, what we need to do next is to provide the implementation for the header file. Because our java native module is just a thin layer, we do not need to write complex code -we need just enough to forward and interface Java with our existing C++ native library. For example:

{% highlight c++ %}
extern "C"
{

JNIEXPORT jobject JNICALL Java_mylibrary_mypackage_MyJavaAPIClass_createCXXObject
(JNIEnv *, jclass)
{
    // add the implementation here...
}

JNIEXPORT void JNICALL Java_mylibrary_mypackage_MyJavaAPIClass_destroyCXXObject
(JNIEnv *, jclass, jobject)
{
    // add the implementation here...
}

} // extern "C"
{% endhighlight %}

We will not cover the specifics of actually writing the implementation detail because this involves several information regarding the JNI technology. The important thing to keep in mind though is that you are wrapping a C/C++ code in a Java code so that the Java code can invoke this C/C++ code. This means that all your types must be properly converted to Java types. For an actual example of writing an implementation, please consult the repository here: [https://github.com/vycasas/language.bindings/tree/master/java/jni](https://github.com/vycasas/language.bindings/tree/master/java/jni). Also, the documentation for JNI here: [http://docs.oracle.com/javase/8/docs/technotes/guides/jni/](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/) will come in handy (especially when matching function signatures).

Initializing/Loading the Native Module
--------------------------------------

This section is nothing new but it is necessary to add here so you don't get discouraged in continuing. All the information in this section will be documented properly (and in more detail) in the JNI documentation by Oracle.

To initialze a native module, you need to keep in mind a couple of things:

1. Add a code in the Java library to load the native module, and
2. When starting the JVM, make sure to indicate the path where to load the native module.

The first part can be done by the following code:

{% highlight java %}
public class Library
{
    public static void initialize() throws UnsatisfiedLinkError
    {
        System.loadLibrary("javalib");
        return;
    }
}
{% endhighlight %}

Then somewhere at your entrypoint, you will need to call `Library.initialize(0)`. Other libraries prefer to add the library loading code in a static block. The benefit of this is remove the explicit call to initialize. For example:

{% highlight java %}
public class Library
{
    static
    {
        System.loadLibrary("javalib");
    }
}
{% endhighlight %}

The second part must be done when launching the JVM. If you launch JVM using the java executable (e.g. /usr/bin/java), then you need to pass -Djava.library.path argument pointing to where the native module is located. For example, if you put the native module in /home/dotslashzero/javalibs, then you need to do:

{% highlight bash %}
/usr/bin/java -Djava.library.path=/home/dotslashzero/javalibs ...
{% endhighlight %}

when starting the JVM. This also assumes that your native module can locate all of its necessary dependencies.

Another option you can do if you do not want to use the -Djava.library.path option is to place all native modules in the library search paths. Please consult your OS documentation for locations of these directories.

Next
----

As usual, please consult the [Github repository](https://github.com/vycasas/language.bindings/tree/master/java) for more details. I made the Java part a bit compact and only added necessary information because otherwise, we might end up with 100 pages of information. I will most likely update this post from time to time to reflect new relevant information. The next part of this series will cover .NET (C# in particular, but also works with VB.NET).
