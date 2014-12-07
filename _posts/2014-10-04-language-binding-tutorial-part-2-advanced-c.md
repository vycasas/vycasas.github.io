---
layout: posts
title: Language Binding Tutorial Part 2&#58; Advanced C++
category: Programming
---
In the [previous part](/2014/09/18/language-binding-tutorial-part-1-c-and-c/) of this series, we have covered basics on how to properly design a native library so that we can create a middle layer to allow it to be interfaced in other languages. Of course, the solution provided in the previous lesson (and any future lessons from here on in) are not silver bullet solutions. We just covered (and will continue to do so) necessary points that allows us to write a clean interface.

We have also created a simple native library in C and have written a C++ interface for this. On this part of the lesson, we will covering advanced topics about C++. In particular, we will be going over callback system.

<!--read_more-->

Callbacks in C
--------------

A common pattern in providing callbacks in C is to pass in the callback function to be invoked. Many APIs and libraries do this pattern. For example:

{% highlight c %}
typedef int (*IntCallback)(int data);
void DoSomethingAsync(IntCallback callback);
{% endhighlight %}

Then a user will simply define a function that matches IntCallback's signature.

{% highlight c %}
int MyIntCallback(int data)
{
    return (data * data);
}
 
/* somewhere in the code... */
DoSomethingAsync(&MyIntCallback);
{% endhighlight %}

It is quite normal to provide such interface for your native library. C does not have as much features as C++, so we are limited (in a positive way) to use such patterns for callback mechanisms. The Windows API is an example of such pattern user. We didn't cover this topic on the previous section because callback functions are simple enough. The next section will cover about C++ callbacks.

Callbacks in C++
----------------

Callbacks in C++ is not normally done like in C. In particular, C++ functions can be a free or a member function. Even if both a free function and a member function have the same signatures (e.g. same parameters and return types), they are of different function types. Static class functions are just free functions (their names are just mangled accordingly) so these are also of different type from a member function of the same signature. If you try the following code snippet, you will run into compiler error:

{% highlight c++ %}
typedef int (*IntCallback)(int data);
class MyClass
{
    int someFunction(int data);
}
 
// somewhere...
MyClass cls;
DoSomethingAsync(&(cls.someFunction)); // error
DoSomethingAsync(&(MyClass::someFunction)); // error
{% endhighlight %}

One of the easiest solutions is to create a free function to act as your callback (just like in C). But in some cases, this does not look good if one wishes to have an object oriented pattern. One good way to demonstrate an object oriented pattern in callback system is to let users implement an abstract base class (or an interface in other languages). A typical scenario would be:

{% highlight c++ %}
// Provided by API:
class CallbackClass
{
    virtual void callbackFunction(int data) = 0;
};
 
// Function that uses the callback class will be declared like:
void DoSomethingAsync(std::unique_ptr<CallbackClass> callback);
{% endhighlight %}

At the top of the hood, this seems simple to implement. If you are using [PIMPL pattern](http://en.wikipedia.org/wiki/Opaque_pointer) to abstract C++ code with C++ interface (which may pass through a C API like our example), then the translation is not too hard to do. But in order to have such mechanism to work, the core C++ abstract class must know the type of external C++. Doing so, however, breaks encapsulation patterns and couples the API and the core implementation unnecessarily. Perhaps that is quite unclear, so before we go about the solution, let us go through a use case scenario.

Say in our core C++ code (e.g. the implementation detail), we have the following:

{% highlight c++ %}
namespace Internal
{
    class Generator
    {
    public:
        virtual int generateInt(int data) const = 0;
    };
}
{% endhighlight %}

As well as classes and functions that consume them (assuming). Now suppose we want to have a client API that mirrors the above:

{% highlight c++ %}
namespace API
{
    class Generator
    {
    public:
        virtual int generateInt(int data) const = 0;
    };
}
{% endhighlight %}

In a normal PIMPL pattern, it is easy to map these two classes with each other if `API::Generator` holds an opaque pointer to `Internal::Generator`. But because the referencing must be done the other way around, this will not work. Keep in mind that behind the API walls, the `API::Generator::generateInt` member function is needed to be called because this is the one provided by the client.

So if you have an API function like below:

{% highlight c++ %}
namespace API
{
    void CallGeneratorFunctions(Generator& generator);
}
{% endhighlight %}

If the implementation does not go deeper into the core of the code, then we can easily make the callback like:

{% highlight c++ %}
namespace API
{
    void CallGeneratorFunctions(Generator& generator)
    {
        int someData;
        // set someData to something...
        auto userInt = generator.generateInt(someData);
        // pass results to internal implementation
        Internal::SomeOtherFunction(userInt);
        return;
    }
}
{% endhighlight %}

While a valid solution, it works only on a specific case/situation. Now, what if `API::CallGeneratorFunctions` spawns a new thread and requires the `Generator` functions to be called later (e.g. when a situation is satisfied). Quick thinking will lead us to storing an owned/strong reference somewhere in the API side which will be consumed at a later point. There are multiple ways of achieving this, but in this post, I will describe the solution that makes the most the sense.

To solve the problem above, we will need to make use of the free function callbacks as discussed at the beginning of this post. The purpose of these free functions is to bridge the type identification so that the core (or implementation detail) can invoke code in the API even if it does not know the type. This is especially useful in C++ because two classes are considered to be a different type if they are on a different namespace even if they both have the same layout and size (e.g. `API::Generator` and `Internal::Generator`).

To achieve the solution, we create a static class functions for the `API::Generator` class:

{% highlight c++ %}
namespace API
{
    class Generator
    {
    public:
        virtual int generateInt(int data) const = 0;
 
        static int StaticGenerateInt(
            int* result, int data, void* userData
        );
    };
}
{% endhighlight %}

The static functions themselves are pretty much similar to the instance functions. The difference is that I modified it slightly to use C plain old datatypes instead of the C++ classes. This not only solves the "internal calling external functions" dilemma in C++, but also allows us to bridge callback classes through a C API. Keep in mind that C++ static class functions are just free functions with its name mangled to be related to the C++ class name. You will also notice that there is an additional parameter called "`userData`". What is this "`userData`"? Well, because the internal code does not know anything about the API code, the only way to reference them is to use a generic pointer (`void*` in this case). On top of that, if we are going through the C API, then we can only use the generic pointer as a handle to the class instance. But why pass it? We will see below.

The implementation of these static functions should be like below:

{% highlight c++ %}
namespace API
{
    int Generator::StaticGenerateInt(
        int* result, int data, void* userData
    )
    {
        // validate arguments...
        auto* genPtr = static_cast<API::Generator*>(userData);
        *result = genPtr->generateInt(data);
        return (0); // no error
    }
}
{% endhighlight %}

This will let the core/implementation detail call the API callback functions without knowing anything about `API::Generator` class. All the core needs to keep track of are the callback functions and the `void*` instance. So, why pass the `userData`? As you can see from above, we need to reference it and tie it to the callback function, so that the callback function can call `API::Generator` functions (by appropriately casting it). Of course, this changes the API function `API::CallGeneratorFunctions`.

{% highlight c++ %}
namespace API
{
    typedef int (*GenerateIntCallback)(
        int* result, int data, void* userData
    );
    void CallGeneratorFunctions(
        GenerateIntCallback callback, void* userData
    );
}
{% endhighlight %}

Again, we want to change it this way so that we can pass through the C API. As you can see, free functions and plain old datatypes are used. Now all that is left is how to tie things together. We ultimately want `Internal::Generator` function calls to be calling `API::Generator` functions. `API::CallGeneratorFunctions` will typically be implemented like below:

{% highlight c++ %}
namespace Internal
{
    typedef int (*GenerateIntFunction)(
        int* result, int data, void* userData
    );
    // We extend the Internal::Generator to be able to call
    // API::Generator functions...
    struct GeneratorImpl : public Generator
    {
        GeneratorImpl(
            GenerateIntFunction generateIntFunction, void* apiData
        );
        virtual int generateInt(int data) const override
        {
            int result;
            _generateIntFunction(&result, data, _apiData);
            return (result;)
        }
        GenerateIntFunction* _generateIntFunction;
        void* _apiData;
    };
}
 
namespace API
{
    // client code calls this...
    void CallGeneratorFunctions(
        GenerateIntCallback callback, void* userData
    )
    {
        Internal::GeneratorImpl generator(callback, userData);
        // now pass generator deeper into the core...
        // do something else...
        // then somewhere in the core, calling
        // Internal::GeneratorImpl::generateInt will eventually
        // call API::Generator::generateInt...
        return;
    }
}
{% endhighlight %}

And obviously, the other parts of the code will be calling this function like:

{% highlight c++ %}
// assume MyGenerator derives from API::Generator
MyGenerator gen;
API::CallGeneratorFunctions(
    &API::Generator::StaticGenerateInt,
    static_cast<void*>(gen)
);
{% endhighlight %}

If you follow the whole sample, you will notice that nothing in the `Internal` namespace references anything in the `API` namespace, but we are still able to invoke `API` functions within `Internal`. Also, we can easily change the code in `API` without affecting `Internal`. Although the leg work to achieve the solution is huge, the benefits are worth it. It allows flexibility by not coupling modules unnecessarily.

Next
----

In the [Github repository](https://github.com/vycasas/language.bindings/tree/master/cxx), I have provided a sample of such callback mechanism. This callback system has been used in many other famous libraries (especially the ones written in C). In the next part of this series, we will be covering a slightly complicated language: Java. If you have not heard of Java native interface, or JNI, I recommend reading a little bit about it as it will be the main course for the next topic.
