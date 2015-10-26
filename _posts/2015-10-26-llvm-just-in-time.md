---
layout: post
title: LLVM JIT Compilation
comments: true
shortinfo: Dynamic compilation of generated code
categories: [Compilers]
tags: [LLVM, Tutorial, JIT, Compile]
---

Generating LLVM IR code is pretty easy. However, since it is a very low level language, it may not be feasible to write all parts of your program in LLVM. In fact, for our query runtime project, we are aiming to only generate the "hot-path" of our program, that is the query piepline itself in pure LLVM. Operators like selection and projection will operate on possibly millions of tuples. The logic for these operators is simple too, making it ideal to avail of the perfromance benefits. However, complex logic like sorting and building hash-tables for joins would be simpler to implement in a higher level language. Fortunately, functions defined in C land and LLVM can be called from each other very simply. C++ functions, with its mangled names, is more of a challenge, but it can be worked around by wrapping these functions in a C style function.

In this post, we will revisit how to compile the code we generate at runtime, so that the generated llvm code can be called. In the first llvm post, we took a roundabout way to actually call our llvm function from C. We dumped the module to stdout, redirected the output to a file, and then called ```llc``` on it. We then linked the assembly file llc produces with the rest of our C program using gcc. The C parts of the program may be precompiled to object files to speed up the process.

This however, is not very elegant. It means we have to split our program into two completely separate fragments, one which generates the LLVM code, and the second which actually calls the compiled code and has the functions and data structures needed by that code loaded into memory. 

For example, we represent a table as an instance of class Schema. All tuples of a table are loaded into memory at a location pointed to by Schema. Now this loading has to be done in the second fragment of the code, so that compiled llvm has access to it. However, we also need details about the table, like attribute information, also wrapped within the Schema class, during plan generation time. Offcourse, we can work around this problem by splitting the Schema class itself into two classes, one with information required at plan generation time and the second with information required at execution time, but that represents a rather poor design.

The more elegant approach is to compile the llvm code and have it immediately available in the current program. No need to dump it to a file, call llc on it and then build a second executable. To do this, we use Just-in-Time compilation provided by llvm.

For some reason, the llvm tutorial does a rather poor job of explaining how to JIT compile a module, which is why we had to look around for other resources and figure it out after consulting several examples. There are two ways to do JIT compilation in llvm. One is to use the older JIT, which was an "ad-hoc" approach. The second is to use the newer MCJIT. The difference as far as we could gather is that MCJIT is a little slower in performance, because it actually isn't really just-in-time, it compiles all the functions in a module at one go. But for our purposes, where we generate only one function, intentionally, to keep the overhead of function calls over millions of tuples to a bare minimum, this doesn't really affect us. So we decided to use MCJIT.

To do this, we first need to do some initialization. These three lines need to be added before we can do any of this stuff.

{% highlight c++ %}
InitializeNativeTarget();
InitializeNativeTargetAsmPrinter();
InitializeNativeTargetAsmParser();
{% endhighlight %}

All three functions are defined in the header file ```llvm/Support/TargetSelect.h``` so make sure to include it. Then we need to create an instance of ```ExecutionEngine```.

{% highlight c++ %}
std::string error;
ExecutionEngine *executionEngine = EngineBuilder(module).	
			// module is the pointer to the Module you want to compile
            setErrorStr(&error).
            setUseMCJIT(true).
			// if you want to use the older JIT, leave this off
            create();

executionEngine->finalizeObject();
{% endhighlight %}

Include the file ```llvm/ExecutionEngine/MCJIT.h```. ```finalizeObject()``` will compile all the functions in Module ```module```. Note that you cannot change module after you call finalize object or it will crash, most probably causing a segmentation fault. Now, to call this function, we can get a pointer to it using the ```getPointerToNamedFunction``` function of ```exceutionEngine```. Cast it to the proper type, and call it. For example, if we had a function ```int add(int, int*)``` in ```module```, to call it, we would do...

{% highlight c++ %}
int a = 5;
int (*add)(int, int*) = 
		(int (*)(int, int*))(excecutionEngine->getPointerToNamedFunction("add", true));
add(1, &a);
{% endhighlight %}

