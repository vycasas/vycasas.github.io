---
layout: posts
title: Language Binding Tutorial Part 5&#58; Objective-C
category: Programming
---
We are continuing the next part of this series about language bindings. On this post's topic is about creating a thin API layer for Objective-C. A good use case for this scenario is when one wants to use a native library (in C or C++) in an iOS project. Of course, one can simply use the C/C++ API directly within his/her Objective-C code, but I highly discourage this as it introduces high coupling. This is also a big issue when the C/C++ API gets updated (or some API gets deprecated).

This topic will be short as most of the concepts from the 1st and 2nd parts of this series will carry over here. So if you haven't had the chance to read my article about [C++ language bindings](/2014/09/18/language-binding-tutorial-part-1-c-and-c), please do so before continuing.

<!--read_more-->

What's Involved
---------------

As with all the other language bindings I have presented so far, for Objective-C, we will still follow the PIMPL pattern. To start with, we create the interface for the API like we did with C++. For example, with the Address class.

{% highlight objective-c %}
@interface OLAddress : NSObject

- (id) initWithStreetNum:(int)streetNum street:(NSString*)street
    city:(NSString*)city province:(NSString*)province
    country:(NSString*)country zipCode:(NSString*)zipCode;
- (void) dealloc;
- (NSString*) toString;

@property (readonly) int streetNum;
@property (readonly) NSString* street;
@property (readonly) NSString* city;
@property (readonly) NSString* province;
@property (readonly) NSString* country;
@property (readonly) NSString* zipCode;

@end /* interface OLAddress */
{% endhighlight %}

The nice thing (or maybe not) about Objective-C is that we can choose to hide the core/implementation pointer from the interface. This can be done because of Objective-C's message passing mechanism which is performed during runtime. You will notice that in the sample I provided, the following code resides in the .mm file instead of the header file.

{% highlight objective-c %}
@interface OLAddress()
{
    CXXLib::Address* _impl;
}
- (CXXLib::Address*) getImpl;
@end // interface OLAddress
{% endhighlight %}

Keeping this part in the .mm file allows us to have proper data separation (or information hiding) because clients will never see this file.

On the same .mm file, the implementation details are as follows:

{% highlight objective-c %}
@implementation OLAddress

- (CXXLib::Address*) getImpl
{
    return (_impl);
}

- (id) initWithStreetNum:(int)streetNum street:(NSString*)street
    city:(NSString*)city province:(NSString*)province
    country:(NSString*)country zipCode:(NSString*)zipCode
{
    self = [super init];
    if (self != nil) {
        _impl = new CXXLib::Address(
            streetNum,
            OBJCLibCore::Utils::NSStringToCXXStdString(street),
            OBJCLibCore::Utils::NSStringToCXXStdString(city),
            OBJCLibCore::Utils::NSStringToCXXStdString(province),
            OBJCLibCore::Utils::NSStringToCXXStdString(country),
            OBJCLibCore::Utils::NSStringToCXXStdString(zipCode)
        );
    }
    else {
        OBJCLibCore::OBJCLibException ex(2);
        @throw ([OLException createFromCXXLibException: &ex]);
    }
    return (self);
}

// some other code here...

- (int) streetNum
{
    BEGIN_EX_GUARD;
    return (_impl->getStreetNum());
    END_EX_GUARD;
}

- (NSString*) street
{
    BEGIN_EX_GUARD;
    return (OBJCLibCore::Utils::CXXStdStringToNSString(_impl->getStreet()));
    END_EX_GUARD;
}

// some other code here...

@end // implementation OLAddress
{% endhighlight %}

I cut out some parts from it because as you can see the pattern is pretty much forward the Objective-C method call to the appropriate C++ function. A lot of the concepts we have seen from the previous tutorials apply to Objective-C.

What to Watch Out For
---------------------

Perhaps this will be the most important part of this blog post. In this section, we will focus on object passing/invocation across language barriers (as with all other languages we have gone through). This is important because while we can easily do some C/C++ code execution within an Objective-C source, passing an Objective-C object to a C/C++ code and passing message to it requires careful management. This issue is indeed evident when implementing a callback system for our example (i.e. having a C/C++ core code call Objective-C methods).

In the example, I have implemented the `CXXLib::GeneratorBase` class as below:

{% highlight c++ %}
class OBJCLibGeneratorImpl final : public CXXLib::GeneratorBase
{
public:
    OBJCLibGeneratorImpl(OLGenerator* impl);

    virtual ~OBJCLibGeneratorImpl(void) override;

    virtual int generateInt(int data) const override;
    virtual std::string generateString(int data) const override;
private:
    void* _impl;
}; // class OBJCLibGeneratorImpl
{% endhighlight %}

As you can see, we use a transparent pointer (`void*` type) to refer or maintain a handle to the Objective-C instance. Now, how do we cast it back and forth? At first, we might be tempted to perform C/C++ casting because Objective-C can run these codes after all. But if you go ahead and try, you will encounter myriad of compiler error messages (at least for me). If not, then you will definitely get runtime crashes. So what do we do?

To get around this issue, the developers of Objective-C seemed to have thought about this. The concept is called "bridging". This concept allows us to get a hold of the Objective-C instance without its garbage collector system interfering. To do so, we use the following pattern in creating and destroying the `Generator` instance:

{% highlight c++ %}
OBJCLibGeneratorImpl::OBJCLibGeneratorImpl(OLGenerator* impl) :
    _impl(nullptr)
{
    _impl = (__bridge_retained void*) impl;
    return;
}

OBJCLibGeneratorImpl::~OBJCLibGeneratorImpl(void)
{
    // give _impl back to ARC so that it can be collected
    (void)((__bridge_transfer OLGenerator*) _impl);
    return;
}
{% endhighlight %}

That allows the C++ code to hold on to the `OLGenerator*` instance without the garbage collector interfering. The next part is how we can invoke operations (or send messages) to this Objective-C instance that we are holding (the `void*` type data member in the sample's case). This is done by using the `(__bridge OLGenerator*)` cast like below:

{% highlight objective-c %}
int OBJCLibGeneratorImpl::generateInt(int data) const
{
    if (_impl == nil) {
        throw (OBJCLibCore::OBJCLibException(2));
    }
    @try {
        return ([(__bridge OLGenerator*) _impl generateIntWithData: data]);
    }
    @catch (NSException*) {
        throw (OBJCLibCore::OBJCLibException(2));
    }
}

std::string OBJCLibGeneratorImpl::generateString(int data) const
{
    if (_impl == nil) {
        throw (OBJCLibCore::OBJCLibException(2));
    }
    @try {
        // note: perhaps it is a good idea to make this part synchronized
        NSString* callbackResult = [(__bridge OLGenerator*) _impl generateStringWithData: data];
        return (OBJCLibCore::Utils::NSStringToCXXStdString(callbackResult));
    }
    @catch (NSException*) {
        throw (OBJCLibCore::OBJCLibException(2));
    }
}
{% endhighlight %}

While there are more solutions to bridging Objective-C and C++ code space, this is the solution I found that worked easily and looked straightforward.

Conclusion
----------

As you have read, the only difficult (or perhaps tricky) part in creating your Objective-C bindings for a native library is the bridging part. Of course, we haven't discussed about garbage collection system used by Objective-C (the language used to not have ARC system before). There is also the compilation issues, but for these topics, it is quite challenging to provide a general soltion (if there is one). For most cases, people writing code using Objective-C are targeting Apple's iOS platform so they most likely have their build systems appropriately set up.
