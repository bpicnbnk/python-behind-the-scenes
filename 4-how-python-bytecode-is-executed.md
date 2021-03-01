本次讨论的目的是了解 CPython 如何执行我们告诉它要执行的字节代码，即它如何执行我们编写的代码编译的字节代码。

注意：在这篇文章中，我指的是CPython 3.9。随着 CPython 的发展，一些实施细节肯定会发生变化。我将尝试跟踪重要更改并添加更新说明。

# 起点

让我们简要地回顾一下我们在前几部分学到的东西。我们通过编写Python代码让Cpython执行。然而，CPython VM只懂Python的字节码。将 Python 代码转换为字节码是编译器的工作。编译器将字节码存储在代码对象中，代码对象是一个完全描述代码块（如模块或函数）的结构。要执行代码对象，CPython 首先为它创建一个的执行状态，称为帧对象。然后将帧对象传递到帧评估函数以执行实际计算。默认帧评估函数`_PyEval_EvalFrameDefault()`定义在Python/ceval.c。此函数实现 CPython VM 的核心。也就是说，它实现了执行Python字节码的逻辑。所以，这个函数是我们今天要学习的。

要了解`_PyEval_EvalFrameDefault()`工作原理，了解其输入（帧对象）是什么至关重要。框架对象是由以下 C 结构定义的 Python 对象：
```c
// typedef struct _frame PyFrameObject; in other place
struct _frame {
    PyObject_VAR_HEAD
    struct _frame *f_back;      /* previous frame, or NULL */
    PyCodeObject *f_code;       /* code segment */
    PyObject *f_builtins;       /* builtin symbol table (PyDictObject) */
    PyObject *f_globals;        /* global symbol table (PyDictObject) */
    PyObject *f_locals;         /* local symbol table (any mapping) */
    PyObject **f_valuestack;    /* points after the last local */
    /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
       Frame evaluation usually NULLs it, but a frame that yields sets it
       to the current stack top. */
    PyObject **f_stacktop;
    PyObject *f_trace;          /* Trace function */
    char f_trace_lines;         /* Emit per-line trace events? */
    char f_trace_opcodes;       /* Emit per-opcode trace events? */

    /* Borrowed reference to a generator, or NULL */
    PyObject *f_gen;

    int f_lasti;                /* Last instruction if called */
    int f_lineno;               /* Current line number */
    int f_iblock;               /* index in f_blockstack */
    char f_executing;           /* whether the frame is still executing */
    PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
    PyObject *f_localsplus[1];  /* locals+stack, dynamically sized */
};
```

帧对象的`f_code`字段指向代码对象。代码对象也是 Python 对象,其定义如下：
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
    /* The rest aren't used in either hash or comparisons, except for co_name,
       used in both. This is done to preserve the name and line number
       for tracebacks and debuggers; otherwise, constant de-duplication
       would collapse identical functions/lambdas defined on different lines.
    */
    Py_ssize_t *co_cell2arg;    /* Maps cell vars which are arguments. */
    PyObject *co_filename;      /* unicode (where it was loaded from) */
    PyObject *co_name;          /* unicode (name, for reference) */
    PyObject *co_lnotab;        /* string (encoding addr<->lineno mapping) See
                                   Objects/lnotab_notes.txt for details. */
    void *co_zombieframe;       /* for optimization only (see frameobject.c) */
    PyObject *co_weakreflist;   /* to support weakrefs to code objects */
    /* Scratch space for extra data relating to the code object.
       Type is a void* to keep the format private in codeobject.c to force
       people to go through the proper APIs. */
    void *co_extra;

    /* Per opcodes just-in-time cache
     *
     * To reduce cache size, we use indirect mapping from opcode index to
     * cache object:
     *   cache = co_opcache[co_opcache_map[next_instr - first_instr] - 1]
     */

    // co_opcache_map is indexed by (next_instr - first_instr).
    //  * 0 means there is no cache for this opcode.
    //  * n > 0 means there is cache in co_opcache[n-1].
    unsigned char *co_opcache_map;
    _PyOpcache *co_opcache;
    int co_opcache_flag;  // used to determine when create a cache.
    unsigned char co_opcache_size;  // length of co_opcache.
};
```

代码对象最重要的字段是`co_code`。它是指向代表字节码的Python字节对象的指针。字节码是一系列两比特指令：一个用于opcode，一个用于参数。

不要担心，如果上述结构的一些成员对你仍然是一个谜。我们将在尝试了解 CPython VM 如何执行字节码时查看它们的用途。

# 评估循环概述 Overview of the evaluation loop

执行 Python 字节码的问题对你来说可能并不麻烦。事实上，VM 所要做的就是迭代指令并按指令操作。而这正是_PyEval_EvalFrameDefault()本质上所做的。它包含一个无限for (;;)循环，我们称之为评估循环。在该循环中，所有可能的opcode都包含在一个巨大的switch语句。每个opcode都有一个相应的case块，包含执行该opcode的代码。字节码由 an array of 16-bit unsigned integers 表示，每个指令一个integer。VM 使用next_instr变量跟踪下一个要执行的指令，next_instr变量是指令数组的指针。在评估循环的每个迭代开始时，VM 分别以下一个指令中最低有效字节和最高有效字节来计算下一个opcode及其参数，然后增加next_instr（指向下一个指令）。_PyEval_EvalFrameDefault()函数有近 3000 行长，但其本质可以通过以下简化版本显示：
```c
_PyEval_EvalFrameDefault(PyThreadState *tstate, PyFrameObject *f, int throwflag)
{
    // ... declarations and initialization of local variables
    // ... macros definitions
    // ... call depth handling
    // ... code for tracing and profiling

    for (;;) {
        // ... check if the bytecode execution must be suspended,
        // e.g. other thread requested the GIL

        // NEXTOPARG() macro
        _Py_CODEUNIT word = *next_instr; // _Py_CODEUNIT is a typedef for uint16_t
        opcode = _Py_OPCODE(word);
        oparg = _Py_OPARG(word);
        next_instr++;

        switch (opcode) {
            case TARGET(NOP) {
                FAST_DISPATCH(); // more on this later
            }

            case TARGET(LOAD_FAST) {
                // ... code for loading local variable
            }

            // ... 117 more cases for every possible opcode
        }

        // ... error handling
    }

    // ... termination
}
```

为了获得详细内容，让我们更详细地讨论一些省略的部分。

# 暂停循环的原因

当前运行的线程停止执行字节码以执行其他事情或无所事事。有四个原因：

1. 有信号要处理。当您使用[`signal.signal()`](https://docs.python.org/3/library/signal.html#signal.signal)将一个函数注册为信号处理程序 signal handler，CPython 将此函数存储在处理程序handler 数组中。当线程接收到信号时，实际调用的函数是signal_handler()（类 Unix 的系统上,它传递到 [`sigaction()`](https://www.man7.org/linux/man-pages/man2/sigaction.2.html)库函数）。调用时，`signal.signal()`设置一个布尔变量，告诉与接收信号对应的处理程序数组中的函数必须调用。解释器的主线程定期调用处理程序。
2. 有pending calls的调用。pending calls是一种机制，它允许安排函数去主线程执行。该机制通过Py_AddPendingCall()函数由Python/C API暴露。
3. 提出了异步异常。异步异常是另一个线程中设置的异常。这可以使用 Python/C API 提供的PyThreadState_SetAsyncExc()功能完成。
4. 要求当前运行的线程释放 GIL。当它看到这样的请求时，它会丢弃 GIL 并等待，直到它再次获得 GIL。

CPython 具有每个事件的指示器indicators。表示有处理程序调用的变量(The variable indicating that there are handlers to call )是runtime->ceval的一个成员，runtime->ceval是一个_ceval_runtime_state结构体：
```c
struct _ceval_runtime_state {
    /* Request for checking signals. It is shared by all interpreters (see
       bpo-40513). Any thread of any interpreter can receive a signal, but only
       the main thread of the main interpreter can handle signals: see
       _Py_ThreadCanHandleSignals(). */
    _Py_atomic_int signals_pending;
    struct _gil_runtime_state gil;
};
```
其他indicators是interp->ceval的成员，interp->ceval是一个_ceval_state结构体：
```c
struct _ceval_state {
    int recursion_limit;
    /* Records whether tracing is on for any thread.  Counts the number
       of threads for which tstate->c_tracefunc is non-NULL, so if the
       value is 0, we know we don't have to check this thread's
       c_tracefunc.  This speeds up the if statement in
       _PyEval_EvalFrameDefault() after fast_next_opcode. */
    int tracing_possible;
    /* This single variable consolidates all requests to break out of
       the fast path in the eval loop. */
    _Py_atomic_int eval_breaker;
    /* Request for dropping the GIL */
    _Py_atomic_int gil_drop_request;
    struct _pending_calls pending;
};
```

将所有indicators放在一起存储在eval_breaker变量中。它告诉当前运行的线程是否有理由停止其正常的字节码执行。评估循环的每个迭代都从检查eval_breaker是否为true开始。如果是true，线程会检查指示器indicators，以确定它究竟被要求做什么，执行该indicator然后继续执行字节码。

# computed GOTOs

评估循环的代码充满了宏，如TARGET()和DISPATCH()。这些不是使代码更紧凑的手段。它们扩展到不同的代码，具体取决于是否使用某些优化（称为"computed GOTOs"（又名"线程代码，threaded code"）。优化的目标是加快字节码执行，以便 CPU 能够使用其[分支预测机制](https://en.wikipedia.org/wiki/Branch_predictor)来预测下一个opcodeopcode。

执行任何给定指令后，VM 会执行三件事之一：

1. 从评估函数返回。当 VM 执行RETURN_VALUE，YIELD_VALUE或YIELD_FROM指令时，就会发生这种情况。
2. 处理错误或继续执行或设置异常从评估函数返回。例如，当 VM 执行BINARY_ADD指令并且加运算的对象不实现__add__和__radd__方法时，可能会出现错误。
3. 继续执行。如何使 VM 执行下一个指令？最简单的解决方案是用每个不返回的case块末尾添加continue语句。然而，真正的解决方案要复杂一些。

要看到简单的continue语句的问题，我们需要了解switch编译成什么。opcode是 0 和 255 之间的整数。由于范围密集，编译器可以创建一个跳转表，存储case块的地址，并使用 opcode 作为索引添加到该表中。现代编译器确实这样做，因此，case调度实现为一个单一的间接跳表as a single indirect jump。这是一个有效的switch实现方式。但是，在循环内放置switch并添加continue语句会产生两个低效：

1. case块末尾的continue语句又增加了一个跳跃。因此，要执行opcode，VM 必须跳两次：首先到循环的开始，然后跳到下一个case块。

2. 由于所有opcode都是通过一个跳表a single jump分发的，CPU很难预测预测下一个opcode。它能做的最好的是选择最近一个opcode，或者，最频繁的一个。

优化的理念是在每个非返回case块的末尾放置一个单独的调度跳转表。首先，它减少了跳转。其次，CPU可以预测下一个opcode是当前opcode之后最可能的opcode。

这种优化可以启用或禁用。这取决于编译器是否支持称为"[labels as values](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)"的GCC C 扩展。开启优化的效果是某些宏以不同的方式扩展，如TARGET()，DISPATCH()，FAST_DISPATCH()。这些宏在整个评估循环的代码中被广泛使用。每个case表达语句都有一个TARGET(op)形式，其中op是一个表示opcode的整数数字的宏。每个未返回的case块以DISPATCH()或FAST_DISPATCH()宏结尾。让我们先看看当优化被禁用时，这些宏扩展到：
```c
for (;;) {
    // ... check if the bytecode execution must be suspended

fast_next_opcode:
    // NEXTOPARG() macro
    _Py_CODEUNIT word = *next_instr;
    opcode = _Py_OPCODE(word);
    oparg = _Py_OPARG(word);
    next_instr++;

    switch (opcode) {
        // TARGET(NOP) expands to NOP
        case NOP: {
            goto fast_next_opcode; // FAST_DISPATCH() macro
        }

        // ...

        case BINARY_MULTIPLY: {
            // ... code for binary multiplication
            continue; // DISPATCH() macro
        }

        // ...
    }

    // ... error handling
}
```

`FAST_DISPATCH()`宏用于某些执行该`opcode`后不暂停评估循环的`opcode`。另外，实现非常简单。

如果编译器支持"labels as values"扩展，我们可以在label上使用`&&`一元操作符来获取其地址。它是`void *`类型，因此我们可以将其存储在指针中：
```c
void *ptr = &&my_label;
```

然后，我们可以通过解引用指针得到label：
```c
goto *ptr;
```

此扩展允许将 跳转表实现为C语言中的label指针数组。这就是CPython所做的：
```c
static void *opcode_targets[256] = {
    &&_unknown_opcode,
    &&TARGET_POP_TOP,
    &&TARGET_ROT_TWO,
    &&TARGET_ROT_THREE,
    &&TARGET_DUP_TOP,
    &&TARGET_DUP_TOP_TWO,
    &&TARGET_ROT_FOUR,
    &&_unknown_opcode,
    &&_unknown_opcode,
    &&TARGET_NOP,
    &&TARGET_UNARY_POSITIVE,
    &&TARGET_UNARY_NEGATIVE,
    &&TARGET_UNARY_NOT,
    // ... quite a few more
};
```

以下是评估循环的优化版本：
```c
for (;;) {
    // ... check if the bytecode execution must be suspended

fast_next_opcode:
    // NEXTOPARG() macro
    _Py_CODEUNIT word = *next_instr;
    opcode = _Py_OPCODE(word);
    oparg = _Py_OPARG(word);
    next_instr++;

    switch (opcode) {
        // TARGET(NOP) expands to NOP: TARGET_NOP:
        // TARGET_NOP is a label
        case NOP: TARGET_NOP: {
            // FAST_DISPATCH() macro
            // when tracing is disabled
            f->f_lasti = INSTR_OFFSET();
            NEXTOPARG();
            goto *opcode_targets[opcode];
        }

        // ...

        case BINARY_MULTIPLY: TARGET_BINARY_MULTIPLY: {
            // ... code for binary multiplication
            // DISPATCH() macro
            if (!_Py_atomic_load_relaxed(eval_breaker)) {
              FAST_DISPATCH();
            }
            continue;
        }

        // ...
    }

    // ... error handling
}
```

扩展由GCC和Clang编译器支持。因此，当您运行python时，您可能启用了这个优化。当然，问题在于它如何影响性能。在这里，我将依靠源代码中的注释：

在编写本文时，"threaded code"版本比正常"switch"版本快 15-20%，具体取决于编译器和 CPU 架构。

本节应该让我们了解 CPython VM 如何从一个指令转到下一个指令，以及在它中间可能做什么。下一个合乎逻辑的步骤是更深入地研究 VM 如何执行单个指令。CPython 3.9 有119 种不同的[opcode](https://docs.python.org/3/library/dis.html#python-bytecode-instructions)。当然，我们不会研究此帖子中每个opcode的实施情况。相反，我们将专注于 VM 用于执行它们的一般原则。

# 值堆栈 Value stack

CPython VM是基于堆栈的。这意味着要计算时，VM 会从堆栈中pop弹出（或peek：查看值，不pop）值，执行对它们的计算并将结果压栈。下面是一些示例：

- opcode UNARY_NEGATIVE从堆栈中弹出值，做负运算并将结果压栈。
- opcode GET_ITER从堆栈中弹出值，调用iter()并将结果压栈。
- opcode BINARY_ADD从堆栈中弹出值，从堆栈顶部peek另一个值，将第一个值加到第二个值，然后将结果替换到堆栈顶部。

值堆栈位于帧对象中。它是作为称为"数组"f_localsplus的一部分实现的。数组被分成几个部分来存储不同的东西，但只有最后一部分用于值堆栈。此部分的开头是堆栈的底部。帧对象的字段f_valuestack指向它。要定位堆栈顶部，CPython 保留本地变量stack_pointer，该变量指向堆栈顶部之后的下一个插槽。f_localsplus数组的元素是指向 Python 对象的指针，指向 Python 对象的指针是 CPython VM 实际工作的内容。

# 错误处理和阻塞堆栈 Error handling and block stack

并非 VM 执行的所有计算都成功。假设我们尝试将数字添加到字符串中，如1 + '41'。编译器生成BINARY_ADD opcode以加两个对象。当 VM 执行此opcode时，它会调用PyNumber_Add()来计算结果：
```c
case TARGET(BINARY_ADD): {
    PyObject *right = POP();
    PyObject *left = TOP();
    PyObject *sum;
    // ... special case of string addition
    sum = PyNumber_Add(left, right);
    Py_DECREF(left);
    Py_DECREF(right);
    SET_TOP(sum);
    if (sum == NULL)
        goto error;
    DISPATCH();
}
```

现在对我们来说重要的不是如何实现PyNumber_Add()，而是调用它会导致错误。错误意味着两件事：

1. PyNumber_Add()返回NULL
2. PyNumber_Add()将当前异常设置为TypeError异常。这涉及到设置tstate->curexc_type，tstate->curexc_value以及tstate->curexc_traceback。

NULL是错误指示器。VM 看到它，在评估循环的末尾跳到标签error。接下来会发生什么取决于我们是否设置了任何异常处理程序。如果我们没有，VM 会到达break语句，评估函数返回NULL，并在线程状态上设置的异常with the exception set on the thread state。CPython 打印异常的详细信息，然后退出。我们得到预期的结果：

```shell
$ python -c "1 + '42'"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

但是，假设我们在try-finally语句的try字句中放置了相同的代码。在这种情况下，finally字句内的代码也会执行：
```shell
$ python -q
>>> try:
...     1 + '41'
... finally:
...     print('Hey!')
... 
Hey!
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

发生错误后，VM 如何继续执行？让我们来看看编译器为try-finally语句编制的字节码：
```shell
$ python -m dis try-finally.py

  1           0 SETUP_FINALLY           20 (to 22)

  2           2 LOAD_CONST               0 (1)
              4 LOAD_CONST               1 ('41')
              6 BINARY_ADD
              8 POP_TOP
             10 POP_BLOCK

  4          12 LOAD_NAME                0 (print)
             14 LOAD_CONST               2 ('Hey!')
             16 CALL_FUNCTION            1
             18 POP_TOP
             20 JUMP_FORWARD            10 (to 32)
        >>   22 LOAD_NAME                0 (print)
             24 LOAD_CONST               2 ('Hey!')
             26 CALL_FUNCTION            1
             28 POP_TOP
             30 RERAISE
        >>   32 LOAD_CONST               3 (None)
             34 RETURN_VALUE
```

注意操作代码SETUP_FINALLY和操作代码POP_BLOCK。第一个设置异常处理程序，第二个删除它。如果 VM 执行它们之间的指令时出现错误，则执行将继续执行偏移 22 的指令，这是finally字句的开始。否则，该finally字句在try字句之后执行。在这两种情况下，该finally字句的字节码几乎相同。唯一的区别是处理程序重新提出了try字句中设置的异常。

异常处理程序实现为记作block的简单 C 结构体：
```c
typedef struct {
    int b_type;                 /* what kind of block this is */
    int b_handler;              /* where to jump to find handler */
    int b_level;                /* value stack level to pop to */
} PyTryBlock;
```

VM 将块保留在块堆栈the block stack中。设置异常处理程序意味着将新块推入块堆栈。这就是操作代码SETUP_FINALLY做的事情。error标签指向一个代码，该代码尝试使用块堆栈上的块处理错误。VM 查找块堆栈，直到它找到中最顶部类型为SETUP_FINALLY的块。它将值堆栈级别恢复到块字段b_level指定的水平，并继续执行偏移b_handler处的字节码。这基本上是CPython实现try-except，try-finally以及with等语句的方法。

关于异常处理，还有一件事要说。想想当 VM 处理异常时发生错误时会发生什么情况：
```shell
$ python -q
>>> try:
...     1 + '41'
... except:
...     1/0
... 
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
TypeError: unsupported operand type(s) for +: 'int' and 'str'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "<stdin>", line 4, in <module>
ZeroDivisionError: division by zero
```

不出所料，CPython 打印了原始异常。要实现此类行为，当 CPython 使用SETUP_FINALLY块处理异常时，它会设置另一个类型的块。如果在块堆栈上有EXCEPT_HANDLER类型的块，程序出现错误时，VM 将从值堆栈中获取原始异常，并将它设置为当前异常。CPython曾经有不同的块，但现在它只有SETUP_FINALLY和EXCEPT_HANDLER。

块堆栈作为帧对象中的f_blockstack数组实现。数组的大小静态定义为 20。所以，如果你堆叠超过20个try语句，你会得到SyntaxError: too many statically nested blocks。

# 总结

今天，我们了解到，CPython VM 在无限循环中逐一执行代号指令。该循环包含所有可能的操作代码的switch语句。每个操作代码都在相应的case块中执行。评估函数在线程中运行，有时该线程会暂停循环以执行其他工作。例如，线程可能需要释放 GIL，以便其他线程可以取下它并继续执行其字节码。为了加快字节码的执行速度，CPython 采用了允许使用 CPU 的分支预测机制的一种优化。一条注释说，它使CPython快了15-20%。

我们还研究了两个对字节码执行至关重要的数据结构：

1. VM 用于计算事物的值堆栈value stack：和
2. VM 用于处理异常的块堆栈block stack。

帖子中最重要的结论是：如果你想研究Python某些方面的实现，评价循环是一个完美的起点。想知道你写下x + y时会发生什么吗？查看操作代码BINARY_ADD的代码。想知道该with语句是如何实现的吗？看SETUP_WITH。对函数调用的确切语义感兴趣？操作代码CALL_FUNCTION是您要查找的内容。下次研究CPython中变量的实现方式时，我们将应用此方法。