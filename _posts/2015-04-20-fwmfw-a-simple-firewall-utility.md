---
layout: posts
title: FWMFW&#58; A Simple Firewall Utility
category: Programming
---
When I was trying to prevent an application from connecting to the internet, I found out that in many cases, adding the main executable is not enough -especially if this executable attempts to launch another executable and/or this certain executable is communicating with a different executable to perform network operations. I had to either add all of the related executables to the firewall rules or completely disconnect the machine from the internet. While the second solution is probably the most effective, I rather want to be able to still use the internet for other purposes while isolating other executables from connecting outside. Adding each and every of the related executable is very tedious so it becomes frustrating to keep doing it.

To solve this problem, I decided to write a simple utility called [FWMFW](https://github.com/vycasas/FWMFW). The name stands for "FireWall Manager For Windows", but that is actually a little bit misleading at the moment. The current state of the utility does not provide a full working firewall manager for Windows (perhaps in the future). It is, however, enough to solve my problem as I mentioned earlier. I can simply provide a list of files and folders to this utility, and it will go through the list and add a firewall block rules for each of the items.

On this blog posting, I will discuss the internals of using COM Interop to achieve such functionality.

<!--read_more-->

COM Interop
-----------

Majority of the work involved in writing the utility was in finding out how to programmatically add the firewall rules. I have achieved this by using both COM Interop and the NetFW API (found in Windows' netfw.h). The COM Interop is necessary to instantiate the NetFW classes. If you look at the code here: https://github.com/vycasas/FWMFW/blob/master/Source/WinNetFW.cxx, you will notice that as soon as we have obtained the `IDispatch` instance for the required class, we can just simply perform a cast on it to obtain a handle that provides an easy access to the APIs. The following snippet shows how to obtain an `INetFwRule` instance:

{% highlight c++ %}
IUnknown* fwRuleUnknown = nullptr;
HRESULT hr = CoCreateInstance(
    CLSID_NetFwRule, NULL, CLSCTX_INPROC_SERVER, IID_IUnknown, (LPVOID*) &fwRuleUnknown
);
if (FAILED(hr)) {
    // error
}

IDispatch* fwRuleDispatch = nullptr;
hr = fwRuleUnknown->QueryInterface(IID_IDispatch, (void**) &fwRuleDispatch);
if (FAILED(hr)) {
    // error
}

INetFwRule* fwRule = static_cast<INetFwRule*>(fwRuleDispatch);
{% endhighlight %}

This pattern pretty much applies to all the objects you wish to obtain a handle of. Once you obtain the handle, you can
treat them as object instances and they have member functions to invoke API calls. For example:

{% highlight c++ %}
// Continuing from above:
fwRule->put_Action(NET_FW_ACTION_BLOCK);
fwRule->put_ApplicationName(OLESTR("MyApp.exe"));
fwRule->put_Description(OLESTR("Blocked using FMWFW."));
fwRule->put_Direction(NET_FW_RULE_DIR_OUT);
fwRule->put_Enabled(VARIANT_TRUE);
// and so on...
{% endhighlight %}

This may seem slightly different from the actual code I committed, but the only difference is that I used smart pointers and created appropriate deleters so that I don't have to write and use `goto` labels for cleaning up. And speaking of cleaning up, always make sure that you have a matching call to `Release` for all the COM objects you created. This is why I use (and recommend) to do the following pattern if you can use C++:

{% highlight c++ %}
// A simple deleter for COM pointers stored within the std::unique_ptr class
template<typename COMPtrType>
struct ReleaseDeleter
{
    void operator()(COMPtrType ptr) const
    {
        ptr->Release();
        return;
    }
}; // struct ReleaseDeleter

// then somewhere later in code...
IUnknown* fwRuleUnknownTmp = nullptr;
CoCreateInstance(
    CLSID_NetFwRule, NULL, CLSCTX_INPROC_SERVER, IID_IUnknown, (LPVOID*) &fwRuleUnknownTmp
);

typedef ReleaseDeleter<IUnknown*> UnknownDeleter;
std::unique_ptr<IUnknown, UnknownDeleter> fwRuleUnkown(fwRuleUnknownTmp, UnknownDeleter());

// at this point, if program abruptly goes out of scope (e.g. an exception occurred),
// the fwRuleUnknown will still be cleaned up.
{% endhighlight %}

With the above pattern, it is important to keep in mind that the `fwRuleUnknownTmp` handle must not be touched after it
is managed by the `fwRuleUnkown` smart pointer. You can do something like below to help:

{% highlight c++ %}
typedef ReleaseDeleter<IUnknown*> UnknownDeleter;
std::unique_ptr<IUnknown, UnknownDeleter> fwRuleUnkown(nullptr, UnknownDeleter());
{
    IUnknown* fwRuleUnknownTmp = nullptr;
    CoCreateInstance(
        CLSID_NetFwRule, NULL, CLSCTX_INPROC_SERVER, IID_IUnknown, (LPVOID*) &fwRuleUnknownTmp
    );
    fwRuleUnknown.reset(fwRuleUnknownTmp);
}
{% endhighlight %}

C&#35; Version
--------------

After I wrote the C++ version of this utility, I decided to port it into C#, and obviously, the code became much more readable. Because of the COM technology, the API can be easily used in a .NET environment. For example, the code above can be done in C# by:

{% highlight c# %}
var fireWallRule = Activator.CreateInstance(Type.GetTypeFromProgID("HNetCfg.FWRule")) as INetFwRule;
fireWallRule.Action = NET_FW_ACTION_.NET_FW_ACTION_BLOCK;
fireWallRule.ApplicationName = "MyApp.exe";
fireWallRule.Description = "Blocked using FWMFW.";
fireWallRule.Direction = NET_FW_RULE_DIRECTION_.NET_FW_RULE_DIR_IN;
fireWallRule.Enabled = true;
{% endhighlight %}

Maybe I should have started writing this utility in C#? But if I did so, then I will not able to have a refresher for using COM Interop.

Conclusion
----------

Overall, the code is not too complex to follow. Nothing was really invented and all the code does is simply calling API
using COM Interop. I do believe though that this project will be a good learning material for developers wanting to get
into COM Interop programming. There are still (I believe definitely) better ways to write this, but the fundamentals of
COM Interop programming is there (mixed with some good C++ techniques).
