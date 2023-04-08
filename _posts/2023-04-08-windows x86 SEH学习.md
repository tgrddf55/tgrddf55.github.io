# 前言

参考师傅博客：

> https://www.yunzh1jun.com/2022/05/27/WindowsSEH/
>
> http://s0rry.cn/archives/yi-chang-chu-li-yuan-li-ji-zhan-zhan-kai-xiang-guan-cao-zuo

参考其他文章：

> https://zhuanlan.zhihu.com/p/573449712

参考视频链接：

> https://youtu.be/COEv2kq_Ht8

这是youtube上的，b站有人搬运了

> https://www.bilibili.com/video/BV1UU4y1K7et?spm_id_from=333.337.search-card.all.click

这两个视频是一样的，但是如果有条件最好还是看youtube上的，也有中文翻译。

这个视频非常推荐，建议仔细观看！

# 一些需要扫盲的知识

读者们可能在系统学习SEH之前了解过相关名词，什么异常处理，什么SEH链之类的，这里讲一下我在学习过程中感到疑惑的知识。

1. SEH是***Windows操作系统***提供的一种异常处理方式，也就是说，***它跟语言的选择没有关系***，你当然可以使用C语言，也可以使用其他语言，甚至还可以使用汇编语言！ 但是，要使用SEH必要的条件是你要在Windows操作系统上使用！这意味着你不能在linux下使用。

2. 我们这里讲解的是windows x86平台级别的SEH异常处理，关键词是```__try / __except``` ，要注意与try/catch的区别。

下面是二者的区别（来自chatgpt)

```try/catch```和```__try/__except```都是异常处理机制，但是它们的使用方式和适用范围略有不同。

try/catch是C++和Java等语言中的异常处理机制，用于捕获和处理程序中的异常。try块中放置可能会抛出异常的代码，catch块中放置对异常的处理代码。当try块中的代码抛出异常时，程序会跳转到最近的匹配异常类型的catch块中进行处理。

__try/__except是Windows平台下的异常处理机制，用于处理系统级别的异常，比如访问非法内存、除零等。__try块中放置可能会抛出异常的代码，__except块中放置对异常的处理代码。当__try块中的代码抛出异常时，程序会跳转到__except块中进行处理。

因此，try/catch适用于C++和Java等语言中的程序级别异常处理，而__try/__except适用于Windows平台下的系统级别异常处理。

3.  即使是SEH,在x86和x64平台上也有很大区别！微软在设计x64的时候考虑到了x86平台中SEH的缺陷，于是在x64平台内就将SEH重新进行了设计。本章主要以x86平台的SEH为主，也会讲解x64平台的机制。如果读者在自己编写测试程序时与学到的内容不同，不妨看一下是不是平台不同导致的原因。
3.  except handler3和except handler4都是编译器级别提供的异常处理函数，这两个版本的异常处理函数是由于编译器不同而导致的。目前最新版本的visual studio 提供的是except handler4

# 0x01:底层系统的相关知识

大家在学习反调试的时候可能会接触一个叫做线程环境块（TEB)的内容，本文将从TEB开始阐述。

TEB部分定义：

```C++
typedef struct _TEB{
	NT_TIB Tib;
	PVOID EnvironmentPointer;
    //····
}TEB,*PTEB;
```

在这里我们主要关注TIB的部分。

TIB（线程信息块）的部分定义：

```C++
typedef struct _NT_TIB{
	struct _EXCEPTION_REGISTRATION_RECORD *ExceptionList;
    //···
}NT_TIB;
```

上面的部分是每个程序都有的，具体情况在这里不多作阐释，只解释和本文相关的知识。

其中我们要注意TIB中的第一个成员，这个成员是一个链表结构，这就是我们所说的SEH链了。

当程序发生异常时，会通过一定的方法在这个SEH链中寻找异常的解决办法，并沿着这个链一直找，没错，SEH链中存储着异常的解决办法。

接下来，我们就讲一下这个结构体的相关知识：

```C++
typedef struct _EXCEPTION_REGISTRATION_RECORD {
    struct _EXCEPTION_REGISTRATION_RECORD *Next; //ext成员指向下一个_EXCEPTION_REGISTRATION_RECORD结构体指针
    PEXCEPTION_ROUTINE Handler;  //handler成员是异常处理函数
} EXCEPTION_REGISTRATION_RECORD;
//若Next成员的值为FFFFFFFF，则表示它是链表最后一个结点

```

这个结构体就是一个简单的链表，Handler成员就是具体的异常处理函数，Next成员指向下一个```_EXCEPTION_REGISTRATION_RECORD```.

然而这个版本的SEH链太简单了，这么简单肯定不符合我们的认知。（太简单的版本肯定就不安全）

于是编译器就提供了更复杂的版本，当然，在系统级别还是上面这个简单的版本，编译器提供的只是在这个简单版本的扩充。

```c++
struct C_EXCEPTION_REGISTARATION_RECORD
{
	void* StackPointer;// 记录栈帧，便于在发生异常时定位栈位置
	EXCEPTION_POINTERS* Exception;//  包含线程上下文以及操作系统记录异常信息的结构体
	EXCEPTION_REGISTRATION_RECORD HandlerRegistration;//异常处理结构体SHE链上的那个
	SCOPETABLE_ENTRY* ScopeTable;
	int TryLevel;	// 当前状态等级
}
```

可以看到第三个成员就是原先的版本。

这里千万要注意，这个复杂版本的结构体是***编译器***级别的！

可以理解为，编译器作为一个程序员，它自己定义了一个结构体，操作系统是不会知道这个结构体的存在的！它只认得它原来的简单链表！

# 0x02: 异常处理函数注册

OK，现在我们已经知道，操作系统提供了一个简单链表，我们可以通过在这个简单链表上添加自己的异常处理函数，从而实现我们想要的异常处理。

这里以汇编语言来解释如何注册异常处理函数。

```assembly
PUSH @MyHandler	; 要添加的异常处理器
PUSH DWORD PTR FS:[0] ; 原先的异常列表
MOV DWORD PTR FS:[0], ESP ; 将添加后的链表设置到链表
```

其中FS[0]就是SEH链的头节点。

从这里也可以看到，x86平台上是以栈来存储这个链表的，注册异常处理函数的时候，只需要压入我们异常处理函数，再压入原先的SEH头结点，再把当前的节点设置为新的头结点。

很好，这很符合我们对链表的认知。

当然，在高级语言中也可以使用这样的注册方法。

但是，在VS的C语言中，我们还可以使用另一种更方便的方法来进行异常处理函数的注册。

```C
__try {
        // 受保护的代码
}
__except ( /*异常过滤器exception filter*/ ) {
        // 异常处理程序exception handler
}
```

在这里，要注意，我们要更换我们的思维了，现在我们不必使用原先的简陋的SEH链表，我们可以使用编译器提供的更高级的链表。

这里再提一嘴 ***这里是编译器级别的！***，但是底层系统级别的仍然是简陋的SEH链表。

以下内容引用自***云之君***师傅的博客。

```C++
__try
__try块中包含可能触发异常的代码。如果代码抛出异常，则交由__except块处理。

__except
__except块中是用户定义的处理异常的代码。

exception filter
exception filter称为异常过滤器。顾名思义，它的作用是对异常进行过滤。

异常过滤器只有三个值（定义在Windows的Excpt.h中）：

EXCEPTION_CONTINUE_EXECUTION（-1）
在发生异常的地方继续执行。
EXCEPTION_CONTINUE_SEARCH （0）
异常无法识别。继续搜索下一个处理程序。
EXCEPTION_EXECUTE_HANDLER （1）
异常被识别。通过执行控制转移到异常处理程序__except复合语句，然后继续执行__except块。
异常过滤器决定了是否处理当前异常，即是否执行__except块中的代码（异常处理程序exception handler）。

异常过滤器的使用：

__try {
   ……
}
__except ( MyFilter( GetExceptionCode() ) )
{
   ……
}

LONG MyFilter(DWORD dwExceptionCode )
{
        if ( dwExceptionCode == EXCEPTION_ACCESS_VIOLATION )
                return EXCEPTION_EXECUTE_HANDLER ;
        else
                return EXCEPTION_CONTINUE_SEARCH ;
}
对于这段代码而言，在异常过滤器中自定义了一个函数MyFilter，以GetExceptionCode()的返回值作为参数，返回值是一个异常过滤器的值，所以也可以直接在__except块的参数中写入异常过滤器的值，如__except (EXCEPTION_EXECUTE_HANDLER) 。
具体而言，GetExceptionCode()函数返回__try块中产生的异常值（也就是产生异常的原因），据此我们可以实现对异常的过滤。
```

OK ,现在我们也已经简单了解了这几个关键词的使用。

接下来又是一个非常关键的内容：

***不要认为__try/__except关键词使用的异常处理机制与我们之前讲的SEH机制不同！，在编译器层次中，会将关键词中的异常处理转化成SEH节点并插入SEH链表！***

这里就要涉及到新的结构体：

```C++
struct C_EXCEPTION_REGISTARATION_RECORD
{
	void* StackPointer;// 记录栈帧，便于在发生异常时定位栈位置
	EXCEPTION_POINTERS* Exception;//  包含线程上下文以及操作系统记录异常信息的结构体
	EXCEPTION_REGISTRATION_RECORD HandlerRegistration;//异常处理结构体SHE链上的那个
	SCOPETABLE_ENTRY* ScopeTable;
	int TryLevel;	// 当前状态等级
}
```

其中每个成员都很重要。

其中ScopeTable也是个结构体，这个结构体中包含了我们之前讲解关键词中给出的过滤器函数和异常处理函数。

```C++
struct SCOPETABLE_ENTRY
{
	int32_t EnclosingLevel; // 状态等级
	FILTER_CALLBACK* Filter; // 指向异常过滤器的地址，如果是finally没有位nullptr
	HANDLER_CALLBACK* Handler; // 指向except/finally的块地址 
}

```

**OK,这里可能有的同学就会有问题了，既然在scopetable中有了异常处理函数，那么原来的EXCEPTION_REGISTRATION_RECORD中的异常处理函数是干嘛的？**

这个问题非常好，因为这也正是我学习的过程中产生疑问的地方。

具体来说，EXCEPTION_REGISTRATION_RECORD中的异常处理函数会调用scopetable中的异常处理函数。



# #0x03:异常处理机制的实现过程

发生异常时，我们的异常处理机制是如何工作的？

这里涉及到栈展开等一系列用文字很难表达清楚的事情。

即使看了这么多大佬的博客，我也没有看到一篇能更通俗易懂的讲解这个过程的文章。

所以，我也没有自信能够在很短的文字内就可以将这个过程讲解给你。

因此，这里我推荐观看视频讲解：

>https://youtu.be/COEv2kq_Ht8

简单来说，我们的异常处理函数是在函数头的位置注册的，当发生异常时，EXCEPTION_REGISTRATION_RECORD中的异常处理函数会一层一层调用scopetable中的函数，直到找到一个能处理这个异常的函数。找到这个能够处理这个异常的异常处理函数肯定需要一个标志，标志是什么呢？

在于Handler的返回值。

Handler的返回值是一个枚举类型_EXCEPTION_DISPOSITION。

```c++
typedef enum _EXCEPTION_DISPOSITION
{
    ExceptionContinueExecution = 0, //已经处理了异常，回到异常触发点继续执行
    ExceptionContinueSearch = 1,    //没有处理异常，继续遍历异常链表
    ExceptionNestedException = 2,   //OS内部使用
    ExceptionCollidedUnwind = 3     //OS内部使用
}EXCEPTION_DISPOSITION;
```

这里再讲解一个我在学习的时候遇到的困惑点：

_EXCEPTION_DISPOSITION和  PEXCEPTION_ROUTINE。

***前者是一个枚举类型，是异常处理函数的返回值，后者是函数指针类型，是一个指向异常处理函数的指针。***

OK，这里，还有一个点：***这个返回值和前面提到的过滤器的返回值不一样！！！！***



找到这个异常处理函数还涉及一个栈展开，详细讲解请移步视频。

相信看完视频，读者就已经能够理解大部分内容了。

# 0x04: 通过异常处理机制实现反调试

通过上面的学习，我们已经大致了解了异常处理机制的工作原理。

那么很容易就可以知道，在异常处理函数或者过滤器函数中我们可以进行一些猥琐的操作。

这里就要详细讲解一下过滤器函数和异常处理函数。

过滤器函数中的参数不仅仅有GetExceptionCode()，还有其他的可能。

```
GetExceptionCode
返回 (一个32位整数) 的代码，也就是异常原因相对应的异常值。
GetExceptionInformation
返回一个指向 EXCEPTION_POINTERS 结构的指针，该结构包含有关异常的其他信息。
```

上面的内容来自云之君师傅的博客。

其中 EXCEPTION_POINTERS 定义如下：

```C++
typedef struct _EXCEPTION_POINTERS {
  PEXCEPTION_RECORD ExceptionRecord;
  PCONTEXT          ContextRecord;
} EXCEPTION_POINTERS, *PEXCEPTION_POINTERS;
```

ExceptionRecord定义如下：

```C++
typedef struct _EXCEPTION_RECORD {
  DWORD                    ExceptionCode;//异常代码
  DWORD                    ExceptionFlags;
  struct _EXCEPTION_RECORD *ExceptionRecord;
  PVOID                    ExceptionAddress;//异常发生地址
  DWORD                    NumberParameters;
  ULONG_PTR                ExceptionInformation[EXCEPTION_MAXIMUM_PARAMETERS];
} EXCEPTION_RECORD;
```

Context:

```C++
typedef struct _CONTEXT {
    DWORD ContextFlags;
    DWORD   Dr0;                //0x04
    DWORD   Dr1;                //0x08
    DWORD   Dr2;                //0x0c
    DWORD   Dr3;                //0x10
    DWORD   Dr6;                //0x14
    DWORD   Dr7;                //0x18

    FLOATING_SAVE_AREA FloatSave;

    DWORD   SegGs;              //0x88
    DWORD   SegFs;              //0x90
    DWORD   SegEs;              //0x94
    DWORD   SegDs;              //0x98

    DWORD   Edi;                //0x9c
    DWORD   Esi;                //0xa0
    DWORD   Ebx;                //0xa4
    DWORD   Edx;                //0xa8
    DWORD   Ecx;                //0xac
    DWORD   Eax;                //0xb0
    DWORD   Ebp;                //0xb4
    DWORD   Eip;                //0xb8

    DWORD   SegCs;              //0xbc MUST BE SANITIZED
    DWORD   EFlags;             //0xc0 MUST BE SANITIZED
    DWORD   Esp;                //0xc4
    DWORD   SegSs;              //0xc8

    BYTE    ExtendedRegisters[MAXIMUM_SUPPORTED_EXTENSION];
} CONTEXT;
```

也就是说，在过滤器函数中是可以获得包含异常代码、异常地址、异常发生时的上下文信息等等。

因此在过滤器函数中可以对context进行更改，并返回``` EXCEPTION_CONTINUE_EXECUTION ```

这样就可以实现反调试功能了。

而在except块中的内容，也就是异常处理函数，实际上并不是函数，只是一段代码而已，也就是说它是无法F5的。



还记得那个简陋SEH链表中的异常处理函数吗？

这个函数才是一个真正的函数。

定义如下：

```C++
EXCEPTION_DISPOSITION __cdecl _except_handler
(
  EXCEPTION_RECORD              *pRecord,
  EXCEPTION_REGISTRATION_RECORD *pFrame,
  CONTEXT                       *pContext,
  PVOID                          pValue
){
//···
}
```

其中的前三个成员我们都已经在前面介绍过了。

很显然，在这个函数中也可以进行猥琐操作，但是如果要在这个函数中进行猥琐操作的话，就不能使用``` __try/____except```关键词了，而应该使用最朴素的链表来进行操作。



