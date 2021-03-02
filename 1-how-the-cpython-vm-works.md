# from

[source](https://tenthousandmeters.com/blog/python-behind-the-scenes-1-how-the-cpython-vm-works/)
# 1. 引入

你是否曾好奇，运行一个程序时，python 做了哪些事情？

    $ python script.py

本文以及由此开启的系列文章，就是回答这个问题的。我们将深入 python 的主流实现，即 CPython，从更深层次理解这门语言。

如果你熟悉 python，不抗拒阅读 C 代码，并且对 CPython 的源码有些兴趣，相信你会喜欢这个系列的文章。

## 1.1 什么是 CPython，为什么要了解 CPython
CPython 是一个由 C 语言实现的 python 解释器，与 PyPy、Jython、IronPython 和许多其它实现一样，它只是 python 的实现之一。区别在于，CPython 是最原始的、重点维护的以及最主流的实现。

CPython 实现了 python。那么，什么是 python？有人可能会说—— python 是一门编程语言。那么，我们把问题提得更具体一点——是什么定义了 python？

与 C 语言不同，python 没有官方标准，最接近官方标准的文档是所谓的 python 语言参考手册（Python Language Reference），而 python 语言参考手册的开头是这么说的：

    我希望尽可能地保证内容精确无误，但还是选择使用自然语言而不是正式的标准说明进行描述，正式的标准说明仅用于句法和词法解析部分。这样应该能使这份文档对于普通人来说更容易理解，但也可能导致一些歧义。

    因此，如果你来自火星并且想凭借这份文档把 Python 重新实现一遍，有时也许需要自行猜测，实际上最终大概会得到一个十分不同的语言。而在另一方面，如果你正在使用 Python 并且想了解有关该语言特定领域的精确规则，你应该能够在这里找到它们。

    【译者注：来自手册中文翻译，字句略有调整。】

也就是说，python 并不完全由参考手册定义。但我们也不能说它是由 CPython 定义的，因为 CPython 的很多实现细节并不属于 python 的一部分，比如说，基于引用计数的垃圾回收机制就是一个例子。因此，我们或许可以说，python 的定义包括两部分，其一是 python 语言参考手册，其二是其主要实现 CPython。

这种考究或许显得无聊，但我想，搞清楚我们要研究的对象到底扮演着什么角色还是很重要的。你或许会问，即便如此，我们为什么要研究它呢？除了单纯的满足好奇心，还可以有以下理由：

    全局视野可以帮助我们更深入地理解这门语言。掌握实现细节让我们更容易理解 python 的一些特性。

    在实践中，实现细节很重要。当我们试图了解 python 的优势与限制、估算性能表现、优化关键代码的时候，理解对象是如何存储的、垃圾回收是如何运作的、多线程是如何协调的等问题就会变得愈发重要。

    CPython 提供了 Python/C API，允许我们用 C 语言扩展 python，或者在 C 语言中调用 python 代码。为了高效使用这个 API，我们应该充分理解 CPython 的运行机制。

## 1.2 如何学习 CPython

CPython 从一开始就被设计得易于维护，任何一位新人都可以阅读并理解它都源代码。当然，这个过程可能要花一点时间，希望我写的这个系列可以帮助大家缩短这个时间。

## 1.3 系列文章的安排

我采用一种自顶向下的方式来安排这个系列。本篇中，我们将探索 CPython 虚拟机的一些核心概念；随后，我们将看到 CPython 如何将一段代码编译成可供虚拟机执行的对象；再之后，通过学习解释器的源码和执行步骤，我们会知道程序具体是怎么执行的；最后，我们将一一剖析 Python 语言的不同特性及其实现方式。不过，这只是我的大概想法，而不是什么严格的写作计划。

注意：文章中默认使用 CPython 3.9，随着 CPython 的演化，一些实现细节可能会有所调整，我会尽力留意一些重要变化并更新文章内容。

# 2. 整体介绍

Python 代码的执行可以大概分成三个阶段：

    初始化
    编译
    解释

在初始化阶段，CPython 会初始化 Python 运行所需的各种数据结构，准备内建类型、配置信息，加载内建模块，建立依赖系统，并完成很多其它必要的准备工作。这个过程很重要，却往往被忽视。

之后是编译阶段。CPython 是一个解释器而不是编译器，意思是说，它并不生成机器码。但和其它解释器一样，CPython 会在执行前把源代码转换成一种中间形式，这个转换过程其实和编译器做的事情是一样的：解析源码，建立 AST（Abstract Syntax Tree 抽象语法树），根据 AST 生成字节码，并对字节码做一些可能的优化。

在讨论下一个阶段之前，我们需要理解什么是字节码。字节码是一系列操作指令，每个指令包括两个字节，其中一个表示操作码，另一个表示操作参数。考虑下面这个例子：
```python
def g(x):
    return x + 3
```
CPython 会把函数体 g() 转换成以下字节系列：[124, 0, 100, 1, 23, 0, 83, 0] 。通过标准库中的 dis 模块，我们可以将它解析为下面这些操作指令：
```
$ python -m dis example1.py ... 
2           0 LOAD_FAST            0 (x)
            2 LOAD_CONST           1 (3)
            4 BINARY_ADD 
            6 RETURN_VALUE
```
LOAD_FAST 指令对应于操作码 124 ，使用参数 0 ， LOAD_CONST 指令对应于操作码 100 ，使用参数 1 ，操作码 BINARY_ADD 和 RETURN_VALUE 的编码总是 (23, 0) 和 (83, 0) ，因为它们不需要任何参数。

CPython 的核心是一个执行字节码的虚拟机。通过前面的例子，你可能已经猜到它是如何工作的了。CPython 的运行是基于栈的，也就是说，它通过栈来存取数据。LOAD_FAST 指令往栈中推入一个局部变量， LOAD_CONST 推入一个常数， BINARU_ADD 则从栈中取出两个对象，对它们求和，并将结果推回栈中。最后， RETURN_VALUE 取出栈中数据并将其返回给调用者。

整个字节码的执行过程被包含在一个巨大的求值循环（evaluation loop）中，只要还有可执行指令，就会一直执行，直到最终返回一个值，或者遇到某种错误。

以上这个简单介绍，事实上引出了很多问题：LOAD_FAST 和 LOAD_CONST 指令的操作码是什么意思？它们是代表一种索引吗？是对什么的索引？虚拟机会把对象的值或者索引放入栈中吗CPython 怎么知道 x 是一个局部变量？万一参数太大，不能用一个字节表示怎么办？对两个数字求和的指令和结合两个字符串的指令一样吗？如果是的话，虚拟机怎么区分这两种操作呢？为回答这些问题，以及许多其它可能的问题，我们需要了解 CPython 虚拟机的一些核心概念。

## 2.1 代码对象（Code objects）、函数对象（function object）、帧（frames）

### 2.1.1 代码对象（Code objects）

我们已经看到一个简单函数的字节码是什么样的了，但一个典型的 Python 程序会复杂得多。虚拟机是如何执行一个包含函数定义与函数调用的模块的呢？

考虑下面这个程序：
```python
def f(x):
    return x + 1

print(f(1))
```
这段程序的字节码是怎么样的？

为回答这个问题，让我们一起分析下这段代码的功能，首先，它定义了一个函数 f() ，然后传入参数 1 ，调用这个函数，打印结果。不论函数 f() 的功能是什么，它显然不是模块字节码的一部分，这一点可以通过解析字节码来确认：
```
$ python -m dis example2.py
1           0 LOAD_CONST               0 (<code object f at 0x10bffd1e0, file "example.py", line 1>)
            2 LOAD_CONST               1 ('f')
            4 MAKE_FUNCTION            0
            6 STORE_NAME               0 (f)
4           8 LOAD_NAME                1 (print)
           10 LOAD_NAME                0 (f)
           12 LOAD_CONST               2 (1)
           14 CALL_FUNCTION            1
           16 CALL_FUNCTION            1
           18 POP_TOP
           20 LOAD_CONST               3 (None)
           22 RETURN_VALUE
...
```
从第一行开始，我们定义了函数 f() ，具体步骤是引入一个叫代码对象的东西，将它和绑定到函数名称 f 上以构建函数。我们没有看到函数体 f() 中返回参数加 1 的字节码。

模块、函数体等被当作一个执行单元的代码被称做代码块。CPython 通过一个被称为代码对象（Code objects）的结构体来存储代码块，其中包含了字节码，也包含了代码块中使用的参数名称列表。运行一个模块或调用一个函数体也就意味着对一个代码对象（Code objects）求值。

### 2.1.2 函数对象（function object）

当然，一个函数并不只是一个代码对象（Code objects），它还包含很多其它信息，如函数名称、文档字符串、参数默认值以及闭包中定义的变量等。函数的这些信息与代码对象（Code objects）一起，被保存在一个函数对象（function object）中， MAKE_FUNCTION 指令的作用就是创建一个函数对象（function object）。

CPython 源码中对函数对象（function object）结构体的定义之前，有以下说明：

不要混淆**函数对象（function object）与代码对象（Code objects）：函数对象（function object）由 def 语句创建，并在其 `__code__` 属性中引用一个代码对象（Code objects），而代码对象（Code objects）只是单纯的句法对象（syntactic object）而已**，也就是说，只是几行源代码的编译结果。

每个源代码“块（fragment）”对应于一个代码对象（Code objects），但**每个代码对象（Code objects）可以被零个或多个函数对象（function object）引用，具体数量取决于源代码中 def 语句的执行次数。**

多个函数对象（function object）是如何引用同一个代码对象（Code objects）的呢？我们可以看一个例子：
```python
def make_add_x(x):
    def add_x(y):
        return x + y
    return add_x

add_4 = make_add_x(4)
add_5 = make_add_x(5)
```
make_add_x() 函数的字节码包含了 MAKE_FUNCTION 指令，函数 add_4() 和 add_5() 是把同一个代码对象（Code objects）作为参数调用这条指令的结果。当然，它们的 x 参数是不一样的，通过 cell 变量（cell variables）机制，这两次调用创建了两个闭包。

在讨论下一个概念前，可以看一下代码对象（Code objects）和函数对象（function object）的结构体定义：
```c
struct PyCodeObject {
    PyObject_HEAD
    int co_argcount;            /* #arguments, except *args */
    int co_posonlyargcount;     /* #positional only arguments */
    int co_kwonlyargcount;      /* #keyword only arguments */
    int co_nlocals;             /* #local variables */
    int co_stacksize;           /* #entries needed for evaluation stack */
    int co_flags;               /* CO_..., see below */
    int co_firstlineno;         /* first source line number */
    PyObject *co_code;          /* instruction opcodes */
    PyObject *co_consts;        /* list (constants used) */
    PyObject *co_names;         /* list of strings (names used) */
    PyObject *co_varnames;      /* tuple of strings (local variable names) */
    PyObject *co_freevars;      /* tuple of strings (free variable names) */
    PyObject *co_cellvars;      /* tuple of strings (cell variable names) */
    Py_ssize_t *co_cell2arg;    /* Maps cell vars which are arguments. */
    PyObject *co_filename;      /* unicode (where it was loaded from) */
    PyObject *co_name;          /* unicode (name, for reference) */
        /* ... more members ... */
};

typedef struct {
    PyObject_HEAD
    PyObject *func_code;        /* A code object, the __code__ attribute */
    PyObject *func_globals;     /* A dictionary (other mappings won't do) */
    PyObject *func_defaults;    /* NULL or a tuple */
    PyObject *func_kwdefaults;  /* NULL or a dict */
    PyObject *func_closure;     /* NULL or a tuple of cell objects */
    PyObject *func_doc;         /* The __doc__ attribute, can be anything */
    PyObject *func_name;        /* The __name__ attribute, a string object */
    PyObject *func_dict;        /* The __dict__ attribute, a dict or NULL */
    PyObject *func_weakreflist; /* List of weak references */
    PyObject *func_module;      /* The __module__ attribute, can be anything */
    PyObject *func_annotations; /* Annotations, a dict or NULL */
    PyObject *func_qualname;    /* The qualified name */
    vectorcallfunc vectorcall;
} PyFunctionObject;
```
### 2.1.3 帧对象(frame)

虚拟机在执行代码对象（Code objects）时，需要记录不同变量的值，追踪持续变化的栈中的内容，也需要记住在一个代码对象（Code objects）中的当前位置，以便在跳转执行另一个代码对象（Code objects）后，可以返回继续执行。CPython 将这些信息保存在一个帧对象(frame)中。**帧保存的是执行代码对象（Code objects）所需的状态信息**。

我想我们已经越来越习惯于看源代码了，所以也提供帧对象(frame)的源代码如下：
```c
struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    PyObject **f_stacktop;          /* Next free slot in f_valuestack.  ... */
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */
    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;
    int f_lasti;                /* Last instruction if called */
    /* ... */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
};
```
第一个帧执行模块的代码对象（Code objects），随后，每执行一个新的代码对象（Code objects），CPython 就会创建一些新的帧，**每个帧都包含了对前一个帧的引用**。于是，我们有了一个存储不同帧的栈，或者说调用栈，**位于调用栈顶部的，就是当前帧**。

调用一个函数时，CPython 会往帧栈推入一个新的帧；当前帧返回时，则会从记录的最后指令开始继续执行上一个帧。某种意义上说，CPython 虚拟机所做的工作无非就是构建并执行帧而已。当然，我们马上会看到，这么简洁的概括，实际上忽略了很多细节。

## 2.2 线程、解释器、运行时

我们已经了解了三个重要概念：

    代码对象（Code objects）
    函数对象（function object）
    帧对象(frame)

CPython 还有另外三个概念：

    线程状态
    解释器状态
    运行时状态

### 2.2.1 线程状态

线程状态是一个记录线程状态信息的数据结构，包括调用栈、异常状态、debug设置等。它和系统线程紧密相关，但并不是一回事。当我们调用 threading 模组在另一个线程中运行一个函数时：
```python
from threading import Thread

def f():
    """Perform an I/O-bound task"""
    pass

t = Thread(target=f)
t.start()
t.join()
```
t.start() 通过调用系统函数（在类 UNIX 系统中是 pthread_create() ，Windows 系统中是 _beginthreadex() ）创建了一个系统线程。新的线程引用了 `_thread` 模块提供的一个函数以调用目标函数，这个函数接收的参数包括目标函数、目标函数的参数，同时也包括一个在新的系统线程中使用的新的线程状态。系统线程进入求值循环时使用的是新的线程状态。

大家可能记得著名的，阻止多个线程同时运行的 GIL（全局解释器锁）。GIL 的主要用途是在不引入更多锁的前提下，防止破坏 CPython 的状态。Python/C API 参考手册对 GIL 提供了清楚的解释：

Python 解释器并不是线程安全的。为了支持多线程，需要引入一个全局锁，称为全局解释器锁，或者 GIL，当前线程必须持有这个锁，才能安全使用 Python 对象。如果没有这个锁，多线程程序中哪怕最简单的操作都可能产生问题：例如，两个线程同时增加同一个对象的引用计数，这个对象的引用计数很可能只增加了一次。

为了管理多线程，还要用到比线程状态更高层级的数据结构。

### 2.2.2 解释器状态与运行时状态

事实上，有两种比线程状态更高层级的数据结构：解释器状态与运行时状态。之所以要两种而不是一种数据结构，原因并不是一望而知的。不过，**执行任何程序都会带有至少一个解释器状态和一个运行时状态**，而且这么做是很有道理的。

解释器状态的内容是一组线程和与这组线程相关的数据，包括线程之间共享的各种内容，如加载的模块（sys.modules）、内建对象（`builtins.__dict__`）、依赖系统（importlib）等。

而运行时状态是一个全局变量，它存储着进程相关数据，包括 CPython 状态（如是否已经初始化）、GIL 机制等。

通常，一个进程中的所有线程都属于同一个解释器，不过，偶尔还是会有例外。有时我们希望将一组线程独立出来，使用一个子解释器（subinterpreter）。其中一个例子是 mod_wsgi ，它使用一个独立的解释器来运行 WSGI 应用。

把一组线程独立出来的一个直接效果是，所有模块它都有自己的版本，包括全局命名空间 `__main__` 。CPython 并没有像提供 threading 模块一样提供创建新解释器的方法，用户只能通过 Python/C API 实现这个功能，不过未来可能会有所调整。

# 3. 架构总结

我们可以简单总结下 CPython 的架构，看看各个概念之间是如何互相配合的。解释器可以被看作一种层级结构，包括以下各个层级：

运行时：进程的全局状态，包括 GIL 和内存分配机制；

解释器：一组线程，以及线程间共享的数据，如引入的模块等；

线程：一个系统线程的相关数据，包括调用栈等；

帧：调用栈中存储的元素，包括一个代码对象（Code objects）和执行这个代码对象（Code objects）所需的状态信息；

求值循环：执行帧的地方；

如我们前面讨论的，这些层级由不同数据结构所表示，但有时层级与对应数据结构之间并不是完全等同的。例如，依赖于全局变量实现的内存分配机制，并不在运行时状态中，但确实属于运行时这个层级。

# 4. 结论

本文概括了运行一个程序时 Python 所做的事情，我们看到，它的工作可以分为三个阶段：

    初始化 CPython

    将源代码编译为模块的代码对象（Code objects）

    执行代码对象（Code objects）的字节码

解释器中负责执行字节码的部分叫做虚拟机。CPython 虚拟机依赖于几个重要概念：代码对象（Code objects）、帧对象(frame)、线程状态、解释器状态与运行时状态。这些数据结构组成了 CPython 架构的核心。

当然，很多事情我们都还没谈到，我们没有深入挖掘源代码，初始化过程与编译过程也完全没有讨论。不过，通过概览虚拟机，我们已经可以更好地理解不同阶段所负责的内容。

现在，我们已经知道，源代码会被编译为代码对象（Code objects），下一篇文章，就让我们一起来看看编译过程是如何实现的。