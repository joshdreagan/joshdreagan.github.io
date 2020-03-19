---
title: Calling Native Code With Camel
tags:
  - fuse
  - camel
banner: post-bg.jpg
permalink: calling_native_code_with_camel
date: 2016-11-21 19:06:03
---


Usually when a customer has some legacy/native code that they need to invoke from [Camel](http://camel.apache.org/) (or any Java program really), I recommend that they expose it via SOAP, REST, or some other standardized remote invocation mechanism. Then they can just call it with the appropriate [Camel Component](http://camel.apache.org/cxf.html). While I still think that this is best option, I recently had a customer ask if Camel had the ability to call native code directly. So I figured what the heck... might as well blog about it...<!--more-->

Native code can be written in many languages (obviously...). And unfortunately, the mechanisms that you would use to invoke the compiled binaries (`.dll`'s or `.so`'s) are not always the same. In this blog post, I'm going to cover what I think are the most common (or at least the most commonly requested) native languages and how to invoke them.

## C

Of the languages that I'll cover in this post, calling C libaries is definitely the easiest. Once I have my compiled library (`.dll` or `.so`), I can just call it using [JNI](http://docs.oracle.com/javase/8/docs/technotes/guides/jni/) or [JNA](https://github.com/java-native-access/jna).

Of the two, I prefer JNA since it doesn't require me to generate any code using `javah` or really have much knowledge of the C code itself. All I need to do is create a Java interface that extends `com.sun.jna.Library` and that matches the signature of the C library (or more specifically its methods). Then I can create a proxy instance automatically using the `com.sun.jna.Native#loadLibrary(java.lang.String, java.lang.Class)` method (passing the name of your library and the interface class).

_Note: You don't give the full path or name of the library. The name is automatically wrapped with the platform-specific parts and the path is looked up via a [variety of mechanisms](https://github.com/java-native-access/jna/blob/master/www/GettingStarted.md) (ie, the `jna.library.path` system property)._

Once I have a proxy object, I can invoke methods on it like I would with any Java Bean in Camel using the [Bean Component](https://camel.apache.org/bean.html).

Fairly straighforward right? Take a look at this project (specifically the "camel-native-c" module) if you'd like to see it all working: [https://github.com/joshdreagan/camel-native](https://github.com/joshdreagan/camel-native).

## C++

Invoking C++ code is slightly more involved than C code, but is still pretty simple. The options (ie, JNI or JNA) are the same, and for the most part the code is the same.

If you choose to use JNA (which I did), you still need to define an interface that matches the method signatures of the C++ library (and implements `com.sun.jna.Library`). And you still need to load the library and create the proxy object using the `com.sun.jna.Native` class. The differences/complications come from this thing called [Name Mangling](https://en.wikipedia.org/wiki/Name_mangling).

Name mangling is something that C++ compilers do to account for namespaces and overloaded methods and such. Most compilers do specify exactly how they perform the mangling, but unfortunately none of them do it the same. That is, the Windows C++ compiler will mangle the names differently than the GNU C++ compiler. At any rate, the end result is that my C++ method will not have the same name in its compiled form as it did in its source form. For example, if I had a method called `add` in my source, the GNU compiler might turn it into something like `_ZN10Calculator3addEii` (depending on the namespace and method signature of course) _(see note below)_. That means that when I try to invoke the method on my `com.sun.jna.Library` interface, I won't have the right name (and will get an exception).

So how do I get around all this name mangling tomfoolery? Well, luckily, JNA allows us to provide a custom `com.sun.jna.FunctionMapper`. That means I can tell JNA that, when I call a function named `add`, it really needs to call a funtion named `_ZN10Calculator3addEii` in the underlying native library. Neat!

One more oddity to handle... For whatever reason, the name mangling also adds an extra argument to the method signature. But once again, JNA has a mechanism to fix it. We can create a custom `com.sun.jna.InvocationMapper` to intercept the invocation and add a `null` argument to the beginning of the arg list.

That's it! Now I can invoke it using the [Bean Component](https://camel.apache.org/bean.html) just like I did in the C example.

Like I said... slightly more involved, but still pretty simple. To see it all in action, take a look at this project (specifically the "camel-native-cpp" module): [https://github.com/joshdreagan/camel-native](https://github.com/joshdreagan/camel-native).

_Note: You can use the `nm` tool on Linux to find the mangled names (ie, `nm -D libc-calculator.so`)._

## C&#35;

C# (or any .NET code) is probably the most complicated case. On Windows, .NET DLLs are not the same format as a truely native DLL. This is because they aren't really native code. They're what Microsoft calls [Managed Code](https://en.wikipedia.org/wiki/Managed_code). Managed code is code that must be run in a virtual machine. Think Java and its JVM, but more Microsofty... So it can't be invoked directly (or at least not without loading a virtual machine). That means that using JNI or JNA directly are out. So what do I do? Well, there are a few options.

If my .NET DLL exposed a [COM](https://en.wikipedia.org/wiki/Component_Object_Model) interface, I could write a C++ wrapper that calls the .NET DLL through COM. Then I could follow the above instructions for invoking C++ code. But what if my library didn't expose a COM interface? In my experience, most don't. And even if it did, that seems like a lot of effort, and I'm quite lazy...

I could perhaps use a paid tool like [javOnet](https://www.javonet.com/) or [JNBridge](http://jnbridge.com/). Both seem simple enough to use and are probaly pretty stable. But they both cost a lot of money, and I'm quite cheap...

So what's left? I went with a project called [jni4net](http://jni4net.com/). It does all the work for me (no writing wrapper code), and the price is right (free!). Plus, it's the only one of the options I found that was open source (and I like open source! :)). Now for the bad news... It's not the most full-featured or simplest tool in the world. That's ok though. My cheapness typically wins out over my laziness. So here goes...

First we need to generate and build the wrapper code. Sadly, the tool has a few steps to its build (making scripting a little complex), but it's not too bad. You have to run the `proxygen.exe` command line tool (located in the `bin` directory of the distribution) and point it at your .NET DLL to generate the Java and C# wrapper code. This process also generates a `build.cmd` file that must then be used to actually build the code into a DLL library and its respective JAR file. This obviously doesn't really mesh well with the way most Java builds are written. But with a bit of cleverness/ugliness, we can make it work.

Now that we have some generated/compiled wrapper code, we need to initialize what jni4net calls the "bridge" and register the assemblies. __This must be done at startup before any of the generated code/methods are used.__ A simple "initializer" class that calls the `net.sf.jni4net.Bridge#init()` and `net.sf.jni4net.Bridge#LoadAndRegisterAssemblyFrom(java.io.File)` methods will do the trick.

Once this is done, we can invoke the generated wrapper class methods using the [Bean Component](https://camel.apache.org/bean.html) just like in the previous examples.

All done right? Not quite... If you tried to this yourself, you probably noticed that there are all kinds of exceptions that occur when you actually try to run your project. This stems from a bit of quirkiness with the way that jni4net finds and loads the DLLs and JARs. As it turns out, the tool is quite opinionated about where its libraries live (making running in production potentially tricky). What do I mean? In order to run, the location of the jni4net JAR files (both generated and the ones that come with the distribution) must live in the same directory as your DLL files (both generated and the .NET one you're trying to make use of). A quick look at [the code](https://github.com/jni4net/jni4net/blob/master/jni4net.j/src/main/java/net/sf/jni4net/CLRLoader.java) showed that this is because it uses a combination of path and naming conventions to figure out the locations of the supporting libs. Unfortunately, this is currently hard-coded. :( Fortunately, this is an open source project! So anyone with a bit of free time could fix this and submit a pull request. :)

All in all it's a bit cumbersome, but works. And that's what really matters right? For an actual running example, take a look at this project (specifically the "camel-native-csharp" module): [https://github.com/joshdreagan/camel-native](https://github.com/joshdreagan/camel-native).
