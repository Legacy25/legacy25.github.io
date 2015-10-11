---
layout: post
title: LLVM
comments: true
shortinfo: Getting to grips with code generation using LLVM
categories: [Compilers]
tags: [LLVM, Tutorial, Code Generation]
---

[LLVM](http://llvm.org/) is an ongoing project at the University of Illinois that provides a modular compiler infrastructure. Anybody familiar with compilers would know that most compilers transform source code into native code through a series of steps. These steps can be broadly broken down into three major chunks -

1. The front-end - This part of the toolchain takes source code and converts it into an intermediate representation or an abstract syntax tree with symbol tables and various metadata about the program.

2. The middle part - This does general optimizations in a series of optimization passes, each pass dealing with a specific kind of inefficiency. During each pass the compiler finds patterns and fixes it.

3. The back-end - The compiler does hardware specific optimizations and produces the final machine code.

The front-end is language-specific.
The back-end is platform-specific.

By designing compilers in a modular fashion, you can reuse much of the toolchain for exsiting languages and platforms to support new languages or platforms, by just writing a new front or back end as the case may be.

The unique thing about the LLVM project however, is that the intermediate representation that it uses is the same for all stages of the process. And it is human readable too. This makes the job of writing new optimization passes much easier. 

You can do many, many things with LLVM, as explained in this [blog](http://adriansampson.net/blog/llvm.html) post by Adrian Sampson. LLVM provides you with the middle and back end portions of the compiler. Hence it is pretty easy to write a compiler for any new language using the LLVM infrastructure. You just have to write a front-end that takes source code and produces LLVM IR, which is basically strongly-typed portable assembly. This involves writing a lexer that parses source code to find tokens, and then generate appropriate LLVM IR for each token. If you follow the Kaleidoscope tutorial on the LLVM website, this is exactly what they do.

### LLVM with C/C++

Most LLVM tutorials start off with a traditional hello world, so I'm not going to reinvent that particular wheel. This particular [article](http://www.ibm.com/developerworks/library/os-createcompilerllvm1/) does a remarkably good job of explaining the baiscs. Please review it before reading on. There is a dearth of more advanced tutorials though so I'm going to do my best to document my own experiences with LLVM. We're going to start somewhere more interesting.

I am currently working with a few students at UB on a project that compiles SQL to native code. The motivation is the [HyperDB](http://hyper-db.com/) system. By moving from a pull-based iterator model, to a push-based producer-consumer model, and removing functional boundaries between operators, HyperDB achieves better data and code cache locality. For in-memory databases, where the data is already in memory, the I/O costs stop dominating. Better cache management makes the data processing pipeline faster[(VLDB 2011)](http://www.vldb.org/pvldb/vol4/p539-neumann.pdf). The upshot of it all is that HyperDB is seriously fast in both OLTP and OLAP workloads.

In HyperDB, code written in C/C++ is combined with code in LLVM IR. This is because writing complex logic in pure LLVM is going to be cumbersome. 

Let's start out by replicating that. This is what we are going to do -


1. We will define an array of integer elements globally in a C file 
2. Then we are going to write a function called ```print()``` in LLVM IR which can print out this array.

To actually write LLVM IR, we can follow two approaches -

1. Output raw llvm instructions to a text file
2. Use the IRBuilder class of the LLVM API which provides a more concise way to generate IR

We are going to use the second approach, because this will let us be more productive. 

### LLVM IR

Before we get to generating LLVM IR, we need to understand a few things about it. The best way to learn IR and how constructs in C map to LLVM code is to write a C program, compile it to IR using clang, the C front-end shipped by the LLVM project, with the ```-O0 -S -emit-llvm``` flags, and checking the resulting IR dump.

You can run this dump using the LLVM bitcode interpreter tool lli.

Note: The code in this article is relevant as of version 3.5 of llvm, with the pre-built ubuntu llvm packages. Since the LLVM APIs, and IR representation changes, often significantly with each release, subsequent versions of llvm might not work exactly this way. 

LLVM has a concept of modules.

-> A module is a single compilation unit, or roughly akin to one C file
-> Each module has a few global functions and variables defined within it
-> Each function is composed of one or more blocks of instructions, called basic blocks
-> Each block is composed of one or more elementary instructions

### Code it up

Let's first define a header file declaring the array and a corresponding C file that initializes it and calls the llvm ```print``` function.

```array.h```
{% highlight c linenos %}
#define SIZE 5
int array[SIZE];
{% endhighlight %}

```array.c```
{% highlight c linenos %}
#include "array.h"

// Prototype for the LLVM IR function
int print();

void initializeArray() {
    for(int i=0; i<SIZE; i++) {
        array[i]=i*i;
    }
}

int main() {
    initializeArray();
    print();
}
{% endhighlight %}

Then we write the code to actually generate IR.

```generate.cpp```
{% highlight c++ linenos %}
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/IRBuilder.h"

#include "array.h"

using namespace llvm;
using namespace std;

int main(){

    // Declaring some variables to make our job easier
    LLVMContext &context = getGlobalContext();
    Module *module = new Module("looper", context);
    IRBuilder<> builder(context);
    Type *intType = Type::getInt32Ty(context);

    // The print function
    FunctionType *printFunctionType = FunctionType::get(intType, false);
    Function *printFunction = Function::Create(printFunctionType, Function::ExternalLinkage, "print", module);

    // Declare C standard library printf so that we can call it
    vector<Type *> printfArgsTypes({Type::getInt8PtrTy(context)});
    FunctionType *printfType = FunctionType::get(intType, printfArgsTypes, true);
    Constant *printfFunc = module->getOrInsertFunction("printf", printfType);

    // Declare array as an external global variable
    Constant *conArray = module->getOrInsertGlobal("array", ArrayType::get(intType, SIZE));

    // entry block - The starting basic block of the print function
    BasicBlock *entry = BasicBlock::Create(context, "entry", printFunction);
    builder.SetInsertPoint(entry);

    // The format string for the printf function, declared as a global literal
    Value *str = builder.CreateGlobalStringPtr("%d\n", "str");

    // loopVar is our counter variable
    Value *loopVar = builder.CreateAlloca(intType);
    builder.CreateStore(ConstantInt::get(intType, 0), loopVar);

    BasicBlock *LoopBB = BasicBlock::Create(getGlobalContext(), "loop", printFunction);
    builder.CreateBr(LoopBB);
    builder.SetInsertPoint(LoopBB);

    // loop block
    Value *i = builder.CreateLoad(loopVar);

    Value* indices[2];
    indices[0] = ConstantInt::get(intType, 0);
    indices[1] = i;
    ArrayRef<Value *> indicesRef(indices);

    Value *v = builder.CreateLoad(builder.CreateGEP(conArray, indicesRef, "arr"));

    vector<Value *> argsV({str, v});
    Value* call = builder.CreateCall(printfFunc, argsV, "calltmp");

    Value *iplusplus = builder.CreateAdd(builder.CreateLoad(loopVar), ConstantInt::get(intType, 1));
    builder.CreateStore(iplusplus, loopVar);
    Value *SltE = builder.CreateICmpULT(builder.CreateLoad(loopVar), ConstantInt::get(intType, SIZE));
    BasicBlock *AfterBB =
         BasicBlock::Create(getGlobalContext(), "afterloop", printFunction);

    builder.CreateCondBr(SltE, LoopBB, AfterBB);


    // after block
    builder.SetInsertPoint(AfterBB);
    builder.CreateRet(ConstantInt::get(intType, 0));

    module->dump();
    return 0;
}
{% endhighlight %}

### Run it!

Before walking through the code, let's first run it to confirm it works. First of all you need to make sure you have llvm, specifically version 3.5 installed. Ubuntu has pre-packaged binaries so you can just ```sudo apt-get install llvm-3.5```.

We will use clang to compile ```generate.cpp```

```
clang++-3.5 -g -O3 generate.cpp `llvm-config-3.5 --cxxflags --ldflags --system-libs --libs core` -o generate
```

Quick notes:
1. The ```-O``` flag, like in gcc is used to indicate the level of optimizations you want the compiler to run on your source.
2. ```-g``` enables debugging symbols for GDB
3. ```llvm-config``` is a tool that generates a lot of the necessary flags so that we do not have to type out a long list of them each time we want to compile our program against the LLVM sources

Next, we run the program. This will generate IR and dump it to the console. Since we need the output for our next steps, we will redirect the output to a dump file.

```
./generate > dump.ll 2>&1
```

Now, ```dump.ll``` has the generate IR. We can us the ```llc``` tool to compile the IR to native assembly.

```
llc-3.5 dump.ll -o dump.s
```

```dump.s``` is now an assembly file.

Finally, we need to compile the assembly file together with ```array.c``` to get the final executable.

```
gcc -std=c99 dump.s array.c -o finalprog
```

Now run ```finalprog```. We will get the following output - 

```
$> ./finalprog
0
1
4
9
16
```

### Great success!

So, what did we do? If you inspect ```dump.ll``` you will get a rough idea of how the code generation works.

```dump.ll```

```
; ModuleID = 'looper'

@array = external global [5 x i32]
@str = private unnamed_addr constant [4 x i8] c"%d\0A\00"

define i32 @print() {
entry:
  %0 = alloca i32
  store i32 0, i32* %0
  br label %loop

loop:                                             ; preds = %loop, %entry
  %1 = load i32* %0
  %arr = getelementptr [5 x i32]* @array, i32 0, i32 %1
  %2 = load i32* %arr
  %calltmp = call i32 (i8*, ...)* @printf(i8* getelementptr inbounds ([4 x i8]* @str, i32 0, i32 0), i32 %2)
  %3 = load i32* %0
  %4 = add i32 %3, 1
  store i32 %4, i32* %0
  %5 = load i32* %0
  %6 = icmp ult i32 %5, 5
  br i1 %6, label %loop, label %afterloop

afterloop:                                        ; preds = %loop
  ret i32 0
}

declare i32 @printf(i8*, ...)
```

### SSA

Looping is a fundamental construct in imperative languages. But its not trivial to implement in IR, at least for a new comer. This is because LLVM uses Single Static Assignment (SSA). This means that once you assign a value to a register, you cannot change it. The register is immutable. LLVM gives you an infinite number of registers, and this formulation makes it easy for the code analyzer to reason about how best to allocate registers to physically available registers. But this also causes a problem with conditional statements like ```if then else``` or loops. Why?

Consider an if statement -

```if x < 5 then y = 0 else y = 1; z = y;```

You'd expect this to translate to something like -

```
entry:
	%0 = icmp ult i32 %x, 5
	br i1 %6, label %then, label %else

then:
	%y = i32 0
	br %after

else:
	%y = i32 1
	br %after

after:
	%z = i32 %y
```

But this is invalid. %y cannot be assigned twice. The way SSA addresses this is through a construct called a PHI node -  

```
entry:
	%0 = icmp ult i32 %x, 5
	br i1 %6, label %then, label %else

then:
	%y1 = i32 0
	br %after

else:
	%y2 = i32 1
	br %after

after:
	%z = phi [%then, i32 %y1] [%else, i32 %y2]
```

You can give multiple inputs to the phi node, each input is a combination of a branch and a value. What value phi resolves to is dependent on which branch was executed before the current branch.

You can see why this kind of restriction can also cause a problem for loops. A ```for``` loop very often depends on an increment or decrement to a counter variable. Since every register value is immutable, we cannot just do ```%i = add i32 %i, i32 0```. Thus we need to allocate a memory space instead where we store the variable's value. This is the reason we do 

```
Value *loopVar = builder.CreateAlloca(intType);			| %0 = alloca i32
builder.CreateStore(ConstantInt::get(intType, 0), loopVar);	| store i32 0, i32* %0
```

Note that ```%0``` is a pointer, not a value. Since the pointer is immutable, we no longer have a problem. We can use store and load instructions to manipulate the counter.


### Value

An important entity in LLVM is the [Value](http://llvm.org/docs/doxygen/html/classllvm_1_1Value.html) class. If you look at the class diagram, you will notice that the Value class is a superclass of a lot of different entities like constants, constant arrays, instructions, arguments, operators, and even basic blocks. Many instructions expect values as input parameters, and are themselves values. So you could potentially chain a lot of values together to generate IR in a concise and readable manner, like here
```
Value *iplusplus = builder.CreateAdd(builder.CreateLoad(loopVar), ConstantInt::get(intType, 1));
```
or
```
Value *SltE = builder.CreateICmpULT(builder.CreateLoad(loopVar), ConstantInt::get(intType, SIZE));
```

### GEP

The [getElementPtr](http://llvm.org/docs/GetElementPtr.html) instruction is important. It is used to dereference memory locations. The LLVM documentation page does a pretty good job of explaining it and its important to know when working with LLVM. 

An interesting observation with the code is that we have to declare the external array as an array type. We tried using an integer pointer and the GEP instruction to walk through the memory location, but this causes a segmentation fault <i>while generating LLVM IR itself</i>. This is because with this approach, we are trying to dereference memory which has not been allocated, even though it would be after linking with ```array.c```. Maybe this is something that can be fixed, but as of this moment, we don't know how.

### Possibilities!

In this post, we basically built a toy program that let us grasp two very important concepts

1. SSA, immutability, and the subtleties of expressing conditionals and loops within the SSA rules
2. How to generate LLVM IR code and link it to C

We could also have made the LLVM function ```main``` and called C functions. The possibilities now are endless. We march on, brandishing the mighty power of IRBuilder.