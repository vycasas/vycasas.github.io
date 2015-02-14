---
layout: posts
title: Language Binding Tutorial Part 4&#58; C&#35;
category: Programming
---
In the previous tutorial, we have covered how to make a thin Java API layer on top of an existing native library that we wrote. This is done by using the Java Native Interface (JNI) feature of the Java language. As we all know, Java uses a virtual machine to perform just in time compilation, so things a little more complex that adding a C++ layer over a native library.

This time we will be going over how to create an API binding library for C# (and other .NET languages as effect) on top of the native library we wrote. Just like the previous tutorials, some concepts here are started from previous topics.

<!--read_more-->

Options for Bindings
--------------------

We have number of ways to make native library function calls from a .NET code (unlike in Java). We have:

* C++/CLI
* P/Invoke
* COM Interfaces

Of the three choices above, we will go with using C++/CLI. This method will allow us to easily wrap an existing native library and create .NET bindings. C++/CLI is especially useful because we already have a C++ API which we can easily work it. Using P/Invoke is also another option, but it will require us to work C (it can be done with C++, but not recommended). We do not want to create a wrapper for C, because it will just make us repeat the work we did when creating C++ bindings. Using COM Interop is also another option we can use to create a .NET language binding, but it is complicated and requires much detail on object management. I have first hand experience with COM Interop (COM OLE Automation in particular), and believe me, it is not something worth exploring unless there are no other options. Don't take that the wrong way though, the COM Interop technology is a very interesting technology albeit being complicated and difficult to work it.

Of course, using C++/CLI also has drawbacks. In particular, because we will be wrapping a native code, we will need to compile in mixed managed and native code for the .NET API library. This means that we cannot target the popular "Any CPU" architecture that pure managed code offers, so if we want to .NET programmers to use our native code, they will need to distribute separate builds per architecture. If this is a big issue, however, then it can be solved by performing "side by side assembly loading" where the assembly (the DLL) is bound during runtime as opposed to during compile time. This post will not cover "side by side assembly loading", but more information can be found [here](https://msdn.microsoft.com/en-us/library/8477k21c%28v=vs.110%29.aspx).

C++/CLI
-------

If you are familiar with both C++ and the .NET framework (at least understanding how the CLR works as well as knowing commonly used libraries), you are likely to find that C++/CLI easy to adapt to. As you can imagine, C++/CLI is the C++ language used to write .NET code. While many people are fast to comment that C++/CLI is a terrible language, I feel that these people have not used C++/CLI a lot, let alone use it extensively to properly judge the language. It does indeed look not C++ at all (which is the top reason of dislike), but this is done to support writing managed code. While I cannot speak for everyone out there, I have personally used and managed C++/CLI code for years and most of the time, the problem lies on how to handle managed memory vs. native memory and not on the language itself. The syntax is much better than its first iteration called "Managed Extensions for C++". Many people tend to call both Managed Extensions for C++ and C++/CLI as "Managed C++". While calling them as managed C++ is correct, they are not really the same language.

As I have mentioned before, using C++/CLI will allow us to easily wrap our existing C++ library into .NET language. C++/CLI is still C++, so the nice thing is that we need not to worry too much about type mapping. We still need to marshal objects, but our native types can be used as is. Because we refer to our core objects via an opaque pointer type (see PIMPL pattern from our previous tutorials), we can use the same pointer type as a member of our .NET API class. For example:

{% highlight c++ %}
public ref class Address sealed
{
// Some code here...
private:
    CXXLib::Address* _impl;
}; // ref class Address
{% endhighlight %}

As long we don't expose the `CXXLib::Address*` in the ABI (it will be a compiler error anyway), we can easily use our existing C++ code within the assembly internally.

I mentioned marshalling of objects, but for the most part, if you are following a PIMPL pattern, the only object you will marshal is the `System::String^` type (or `System.String` in C#) to the C++ `std::string` type. Encoding and locales aside, this can be done using the following code:

{% highlight c++ %}
System::String^ CXXStdStringToSystemString(const std::string& str)
{
    System::String^ result = gcnew System::String(str.c_str());
    return (result);
}

std::string SystemStringToCXXStdString(System::String^ str)
{
    std::string result;
    array<unsigned char>^ byteArray = System::Text::Encoding::UTF8->GetBytes(str);
    for (int i = 0; i < byteArray->Length; i++) {
        result.push_back(static_cast<std::string::value_type>(byteArray[i]));
    }
    return (result);
}
{% endhighlight %}

Pretty simple but it is important to take note of the encoding (as seen from my sample above, it is assumed that UTF-8 encoding is used).

Native Handle to Managed Objects
--------------------------------

Perhaps the real challenge in providing .NET wrappers via C++/CLI is the callback system that must go through the ABI (well, not as frustrating as Java). The issue is that, like all other languages, the core native code (native C++ from our sample's case) cannot understand the API's language. While we can engineer the code to call the callback directly in the core, this makes code tightly coupled which we do not want. To work this around in C++/CLI, we can derive a core base class, add the external .NET instance as a member of this derived class, then use derived class as the core's indirect handle to the external class. Let's see how this works.

In our sample code at the [repository](https://github.com/vycasas/language.bindings/blob/master/dotnet/api_dotnet.cxx), you will see that we have a native C++ class called "DotNetLibGeneratorImpl". This class is what will be used to "pass" the user defined `IGenerator` interface from the .NET (managed) side to the native core code.

{% highlight c++ %}
class DotNetLibGeneratorImpl final : public CXXLib::GeneratorBase
{
public:
    DotNetLibGeneratorImpl(DotNetLib::IGenerator^ generator);
    // We need to implement
    virtual ~DotNetLibGeneratorImpl(void) override;
    virtual int generateInt(int data) const override;
    virtual std::string generateString(int data) const override;

    gcroot<DotNetLib::IGenerator^> _generatorPtr;
};
{% endhighlight %}

The definition for the abstract methods are simply forwarding the same function calls to the reference to .NET implementation (the `_generatorPtr` in the sample's case). Of course, we need to worry about marshalling strings, but in this case, we will use the functions we defined earlier.

{% highlight c++ %}
int DotNetLibGeneratorImpl::generateInt(int data) const
{
    try {
        return (_generatorPtr->GenerateInt(data));
    }
    catch (System::Exception^) {
        throw (DotNetLibException(2));
    }
}

std::string DotNetLibGeneratorImpl::generateString(int data) const
{
    try {
        return (
            DotNetLibCore::Utils::SystemStringToCXXStdString(_generatorPtr->GenerateString(data))
        );
    }
    catch (System::Exception^) {
        throw (DotNetLibException(2));
    }
}
{% endhighlight %}

The important piece of this class will be the instance member `_generatorPtr`. You will notice that its type is `gcroot<DotNetLib::IGenerator^>`. As we mentioned earlier, we want to have a reference to the user implemented instance which is what we did here. But what is this `gcroot<>` template class?

The `gcroot<>` template class's purpose is to ensure that the managed object can be referred to by the native code without any issues. When a managed instance is wrapped in a `gcroot<>` template class, the CLR (or the .NET virtual machine) ensures that the managed object is never garbage collected. Not only that, but when the garbage collector moves objects around in the managed heap (to defragment memory), the pointed address of a `gcroot<>` instance gets updated to the new location. In other words, the `gcroot<>` template class allows us to not worry about .NET heap management details in our native code. You can read more about the `gcroot<>` template class here: [https://msdn.microsoft.com/en-us/library/481fa11f.aspx](https://msdn.microsoft.com/en-us/library/481fa11f.aspx)

There is an alternative to prevent the garbage collector from messing with a managed object's memory which is known as "pinning pointers". This mechanism pins the memory of the managed object so that it is not moved by the garbage collector. The [`pin_ptr<>`](https://msdn.microsoft.com/en-us/library/1dz8byfh.aspx) template class provides this mechanism. I do not recommend using `pin_ptr<>` over `gcroot<>` because pinning an object in the managed memory causes fragmentation and prevents the GC from optimizing the heap. There also little to no benefit in using `pin_ptr<>`, so using `gcroot<>` is what I recommend.

Build Stuff
-----------

Building the whole C++/CLI API should be no problem in Visual Studio. Just keep in mind that because we are combining both native and managed code, we need to make sure we are not using using `/clr:pure` flag (or any other flags similar to it). The [sample project](https://github.com/vycasas/language.bindings/tree/master/dotnet) I provided uses CMake to generate the Visual Studio project files so you can also used it as a base framework.

When built, the generated assembly (the DLL) can be used with any supported .NET languages. The sample project contains examples written in [C++/CLI, C#, and VB.NET](https://github.com/vycasas/language.bindings/tree/master/dotnet/test) which consumes the assembly built.

Conclusion
----------

We have now covered C, C++, Java, and .NET. If you have read and understood all the tutorials so far, you will see that most of the patterns are pretty much similar with each other. That is, we always write a middle layer library then forward the calls to the library that we are making an API of. I have already [committed a while back](https://github.com/vycasas/language.bindings) several examples for different languages which achieves our goal. For the next part of this series, I am thinking of covering some other language that I find challenging to make an API layer for.
