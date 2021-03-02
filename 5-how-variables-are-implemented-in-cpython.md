# from

[source](https://tenthousandmeters.com/blog/python-behind-the-scenes-5-how-variables-are-implemented-in-cpython/)

# pre
闭包：enclosing function
内嵌函数（nested function）
闭包必须包含：
- We must have a nested function (function inside a function).　　     一个嵌套函数  
- The nested function must refer to a value defined in the enclosing function.      嵌套函数必须引用一个外部定义的变量
- The enclosing function must return the nested function.      enclosing函数返回嵌套函数

# begin

考虑 Python 中的简单作业说明：
```python
a = b
```
这个语句的含义似乎微不足道。我们在这里做的是将b的值赋值给变量名a，但我们真的吗？这是一个模棱两可的解释，引起了很多问题：

- 变量名与值关联是什么意思？什么是值？
- CPython 如何为变量名赋值？
- 所有变量都是以同样的方式实现的吗？

今天，我们将回答这些问题，并了解如何在 CPython 中实现变量（变量是编程语言中如此重要的方面）。

注意：在这篇文章中，我指的是CPython 3.9。随着 CPython 的发展，一些实现细节肯定会发生变化。我将尝试跟踪重要更改并添加更新说明。

# 开始调查
我们应该从哪里开始调查？我们从前面的部分知道，要运行 Python 代码，CPython 将其编译为副代码，因此让我们从查看编译的副词典开始：a = b
```shell
$ echo 'a = b' | python -m dis

  1           0 `LOAD_NAME`                0 (b)
              2 `STORE_NAME`               1 (a)
...
```

上一篇文章我们得知 CPython VM 使用值堆栈运行。典型的字节码指令会弹出堆栈中的值，用它们计算，并将计算结果推回堆栈。`LOAD_NAME`指令和`STORE_NAME`指令是可完成上述功能。以下是他们在我们的示例中所做的：

- `LOAD_NAME`获取变量名b的值并将其推入堆栈。
- `STORE_NAME`从堆栈中弹出值，并将变量名a与该值关联在一起。

上次我们也了解到，所有的操作代码都实现在Python/ceval.c的一个巨大的switch语句，所以通过研究switch中相应的case，我们可以看到`LOAD_NAME`和`STORE_NAME`操作代码的工作原理。让我们从操作代码`STORE_NAME`开始，因为在获得该变量名的值之前，我们需要将变量名与某些值关联在一起。下面是执行操作代码`STORE_NAME`的case块：
```c
case TARGET(`STORE_NAME`): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *v = POP();
    PyObject *ns = f->`f_locals`;
    int err;
    if (ns == NULL) {
        _PyErr_Format(tstate, PyExc_SystemError,
                      "no locals found when storing %R", name);
        Py_DECREF(v);
        goto error;
    }
    if (PyDict_CheckExact(ns))
        err = PyDict_SetItem(ns, name, v);
    else
        err = PyObject_SetItem(ns, name, v);
    Py_DECREF(v);
    if (err != 0)
        goto error;
    DISPATCH();
}
```

让我们来分析一下它做什么：

1. 变量名为字符串。它们存储在一个称为co_names的元组的代码对象中。变量names只是co_names的一个速记。指令`STORE_NAME`的参数不是变量名，而是用于在co_names中查找变量名的索引。VM 做的第一件事就是从co_names获取变量名，并将为它赋值一个值.
2. VM 从堆栈中弹出值。
3. 变量值存储在帧对象中。帧对象的`f_locals`字段是从局部变量变量名到其值的映射。VM 通过设置``f_locals`[name] = v`将变量名name与值v关联。

我们从这两个关键事实中得到：

- Python 变量是映射到值的变量名。
- 变量名的值是 Python 对象（值v）的引用。

执行操作代码`LOAD_NAME`的逻辑要复杂一些，因为 VM 不仅在`f_locals`也在其他几个地方查找变量名的值：
```c
case TARGET(`LOAD_NAME`): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *locals = f->`f_locals`;
    PyObject *v;

    if (locals == NULL) {
        _PyErr_Format(tstate, PyExc_SystemError,
                        "no locals when loading %R", name);
        goto error;
    }

    // look up the value in `f->`f_locals``
    if (PyDict_CheckExact(locals)) {
        v = PyDict_GetItemWithError(locals, name);
        if (v != NULL) {
            Py_INCREF(v);
        }
        else if (_PyErr_Occurred(tstate)) {
            goto error;
        }
    }
    else {
        v = PyObject_GetItem(locals, name);
        if (v == NULL) {
            if (!_PyErr_ExceptionMatches(tstate, PyExc_KeyError))
                goto error;
            _PyErr_Clear(tstate);
        }
    }

    // look up the value in `f->`f_globals`` and `f->f_builtins`
    if (v == NULL) {
        v = PyDict_GetItemWithError(f->`f_globals`, name);
        if (v != NULL) {
            Py_INCREF(v);
        }
        else if (_PyErr_Occurred(tstate)) {
            goto error;
        }
        else {
            if (PyDict_CheckExact(f->f_builtins)) {
                v = PyDict_GetItemWithError(f->f_builtins, name);
                if (v == NULL) {
                    if (!_PyErr_Occurred(tstate)) {
                        format_exc_check_arg(
                                tstate, PyExc_NameError,
                                NAME_ERROR_MSG, name);
                    }
                    goto error;
                }
                Py_INCREF(v);
            }
            else {
                v = PyObject_GetItem(f->f_builtins, name);
                if (v == NULL) {
                    if (_PyErr_ExceptionMatches(tstate, PyExc_KeyError)) {
                        format_exc_check_arg(
                                    tstate, PyExc_NameError,
                                    NAME_ERROR_MSG, name);
                    }
                    goto error;
                }
            }
        }
    }
    PUSH(v);
    DISPATCH();
}
```

此代码翻译如下：

1. 对于操作代码`STORE_NAME`，VM首先获得变量的名称。
2. VM 在本地变量映射中查找名称值：v = `f_locals`[name]。
3. 如果名称不在`f_locals`，VM 会查找全局变量词典`f_globals`中的值。如果名称不在`f_globals`其中，VM 会查找f_builtins其中的价值。帧对象的f_builtins字段是指向builtins模块字典的指针，其中包含内置类型、函数、异常和常数。如果名称不存在，VM 将放弃并设置NameError异常。
4. 如果 VM 找到值，它会将值推入堆栈。

VM 搜索值的方式具有以下效果：

- 我们总是有builtin字典里的名字，比如int，next，ValueError和None。
- 如果我们使用内置名称来表示本地变量或全局变量，则新变量将覆盖内置变量。
- 局部变量覆盖同名全局变量。

由于我们只需要将变量与值关联并获取其值，您可能会认为`STORE_NAME`和`LOAD_NAME`操作代码足以实现 Python 中的所有变量。事实并非如此。请考虑示例：
```python
x = 1

def f(y, z):
    def _():
        return z

    return x + y + z
```

函数f必须加载变量值x,y,z，加它们并返回结果。请注意编译器为做到这一点而生成的操作代码：

```shell
$ python -m dis global_fast_deref.py
...
  7          12 LOAD_GLOBAL`              0 (x)
             14 LOAD_FAST                0 (y)
             16 BINARY_ADD
             18 LOAD_DEREF               0 (z)
             20 BINARY_ADD
             22 RETURN_VALUE
...
```

没有一个操作代码是`LOAD_NAME`。编译器生成LOAD_GLOBAL`操作代码来加载x值，操作代码LOAD_FAST加载值y和操作代码LOAD_DEREF加载值z。要了解编译器为什么产生不同的操作代码，我们需要讨论两个重要的概念：命名空间和作用域。

# 命名空间和作用域 Namespaces and scopes

Python 程序由代码块组成。代码块是 VM 作为单个单元执行的代码。CPython 区分了三种类型的代码块：
1. 模块
2. 函数（推导式和lambda（comprehensions and lambdas）也是函数）
3. 类定义。

编译器为程序中的每个代码块创建代码对象。代码对象是描述代码块工作的内容的结构。特别是，它包含一个块的字节码。要执行代码对象，CPython 为它创建一个称为帧对象的执行状态。除其他内容外，帧对象还包含名称和值的映射，例如`f_locals`，`f_globals`，f_builtins。这些映射称为命名空间。每个代码块都会引入一个命名空间：本地命名空间。程序中的同一名称可能指不同命名空间中的不同变量：
```python
x = y = "I'm a variable in a global namespace"

def f():
    x = "I'm a local variable"
    print(x)
    print(y)

print(x)
print(y)
f()
```
```shell
$ python namespaces.py 
I'm a variable in a global namespace
I'm a variable in a global namespace
I'm a local variable
I'm a variable in a global namespace
```

另一个重要的概念是作用域的概念。以下是 Python 文档对此的[看法](https://docs.python.org/3/tutorial/classes.html#python-scopes-and-namespaces)：

- 作用域是 Python 程序的文本区域，区域内可直接访问命名空间。此处的"直接访问Directly accessible"意味着对名称的无条件引用会尝试在命名空间中查找名称。

我们可以将作用域视为名称的属性，该属性可告知该名称的值存储在哪里。示例是本地作用域。名称的作用域与代码块相对。以下示例说明了以下要点：
```python
a = 1

def f():
    b = 3
    return a + b
```

在这里，名称a在这两种情况下都指相同的变量。从函数的角度来看，它是一个全局变量，但从模块的角度来看，它既是全局性的，也是本地的。该变量b是函数f的局部变量，它根本不存在于模块级别。

如果该变量受该代码块中的约束，则该变量将被视为代码块的本地代码。如赋值语句a = 1，绑定名称a与值1。但是，赋值语句并不是绑定名称的唯一方法。Python 文档列出了[更多](https://docs.python.org/3/reference/executionmodel.html#naming-and-binding)：

- 以下构造原理绑定名称：函数参数、import语句、类和函数定义（在定义块defining block中绑定了类或函数的名称），以及出现在赋值、for循环头或在with语句或except语句后是标识符的变量。from ... import *形式的import语句将被导入模块定义的所有名称，但以下划线开头的名称除外。此赋值形式只能在模块级别上使用。

由于名称的任何赋值都使编译器认为名称是本地的，因此以下代码会提出一个异常：
```python
a = 1

def f():
    a += 1
    return a

print(f())
$ python unbound_local.py
...
    a += 1
UnboundLocalError: local variable 'a' referenced before assignment
```

a += 1语句是一种赋值形式，因此编译器认为a是本地的。要执行计算操作，VM 会尝试加载a，失败后设置异常。要告诉编译器a是全局的，我们可以使用global语句：
```python
a = 1

def f():
    global a
    a += 1
    print(a)

f()
$ python global_stmt.py 
2
```

同样，我们可以使用nonlocal语句告诉编译器，在嵌套函数中绑定的名称是指闭包中的变量：
```python
a = "I'm not used"

def f():
    def g():
        nonlocal a
        a += 1
        print(a)
    a = 2
    g()

f()
```
```shell
$ python nonlocal_stmt.py
3
```

这是编译器的工作，以分析代码块内名称的使用情况，考虑类似global和nonlocal的语句，并生成正确的操作代码来加载和存储值。一般来说，编译器为名称生成的操作编码取决于该名称的作用域和当前编译的代码块的类型。VM 以不同的方式执行不同的操作代码。所有这一切都是为了使 Python 变量正常工作。

CPython 总共使用四对加载/存储操作代码和一个加载操作代码：

- LOAD_FAST和STORE_FAST
- LOAD_DEREF和STORE_DEREF
- LOAD_GLOBAL`和`STORE_GLOBAL`
- `LOAD_NAME`和`STORE_NAME`
- LOAD_CLASSDEREF.

让我们弄清楚他们做什么，为什么CPython需要所有这些。

# LOAD_FAST和STORE_FAST

编译器通过操作代码LOAD_FAST和操作代码STORE_FAST管理函数本地的变量。下面是一个例子：
```python
def f(x):
    y = x
    return y
```
```shell
$ python -m dis fast_variables.py
...
  2           0 LOAD_FAST                0 (x)
              2 STORE_FAST               1 (y)

  3           4 LOAD_FAST                1 (y)
              6 RETURN_VALUE
```

变量y是f本地的，因为它被赋值约束在f中。该变量x是f本地的，因为它被绑定为其f参数。

让我们来看看执行操作代码STORE_FAST的代码：
```c
case TARGET(STORE_FAST): {
    PREDICTED(STORE_FAST);
    PyObject *value = POP();
    SETLOCAL(oparg, value);
    FAST_DISPATCH();
}
```

SETLOCAL()是一个宏，本质是扩展到fastlocals[oparg] = value。变量fastlocals只是帧对象`f_locals`plus字段的速记。此字段是指向 Python 对象的指针数组。它存储本地变量、cell variables、free variable和值堆栈的值。上次我们得知`f_locals`plus数组用于存储值堆栈。在这篇文章的下一部分，我们将看到它如何被用来存储 cell and free variables。目前，我们对用于局部变量的数组的第一部分感兴趣。

我们已经看过在操作代码`STORE_NAME`的情况下，VM 首先从co_names获取名称，然后将该名称映射到堆栈顶部的值。`f_locals`用作名称-值映射，通常是字典。在操作代码STORE_FAST的情况下，VM不需要获取名称。本地变量的数量可以通过编译器静态计算，因此 VM 可以使用数组来存储其值。每个局部变量都可以与该数组的索引关联。要将名称映射到值，VM 只需将值存储在相应的索引中。

VM 不需要将变量的名称本地化到函数来加载和存储其值。然而，它将这些名称存储在co_varnames元组的函数的代码对象中。为什么？调试和错误消息需要名称。它们还被dis等工具使用，读取co_varnames并在括号中显示名称：
```c
              2 STORE_FAST               1 (y)
```

CPython 提供[locals()](https://docs.python.org/3/library/functions.html#locals)内置函数，该函数以字典的形式返回当前代码块的本地命名空间。VM不会为函数保存这样的字典，但它可以通过映射键从co_varnames到`f_locals`plus值为他们建立一个联系。

操作代码LOAD_FAST只需将`f_locals`plus[oparg]推入堆栈：
```c
case TARGET(LOAD_FAST): {
    PyObject *value = GETLOCAL(oparg);
    if (value == NULL) {
        format_exc_check_arg(tstate, PyExc_UnboundLocalError,
                             UNBOUNDLOCAL_ERROR_MSG,
                             PyTuple_GetItem(co->co_varnames, oparg));
        goto error;
    }
    Py_INCREF(value);
    PUSH(value);
    FAST_DISPATCH();
}
```

LOAD_FAST和STORE_FAST操作代码的存在仅出于性能原因。它们被*_FAST调用，是因为 VM 使用数组进行映射，其工作速度比字典快。速度增益是怎样的？让我们来衡量一下STORE_FAST和`STORE_NAME`.以下代码存储变量i的价值 1 亿次：
```python
for i in range(10**8):
    pass
```

如果我们将上述代码放置在模块中，编译器将生成`STORE_NAME`操作代码。如果我们将其置于函数中，编译器将生成STORE_FAST操作代码。让我们两者兼得，并比较运行时间：
```python
import time


# measure `STORE_NAME`
times = []
for _ in range(5):
    start = time.time()
    for i in range(10**8):
        pass
    times.append(time.time() - start)

print('`STORE_NAME`: ' + ' '.join(f'{elapsed:.3f}s' for elapsed in sorted(times)))


# measure STORE_FAST
def f():
    times = []
    for _ in range(5):
        start = time.time()
        for i in range(10**8):
            pass
        times.append(time.time() - start)

    print('STORE_FAST: ' + ' '.join(f'{elapsed:.3f}s' for elapsed in sorted(times)))
```
```shell
f()
$ python fast_vs_name.py
`STORE_NAME`: 4.536s 4.572s 4.650s 4.742s 4.855s
STORE_FAST: 2.597s 2.608s 2.625s 2.628s 2.645s
```

理论上`STORE_NAME`和STORE_FAST实现的另一个区别可能影响这些结果。操作代码STORE_FAST的case块以宏FAST_DISPATCH()结尾，这意味着 VM 在执行指令STORE_FAST后立即转到下一个指令。操作代码`STORE_NAME`的case块以宏DISPATCH()结尾，这意味着 VM 可能进入评估循环的开头。在评估循环的开头，VM 检查它是否必须暂停按代码执行，例如，释放 GIL 或处理信号。我在`STORE_NAME`块，用FAST_DISPATCH()取代宏DISPATCH()的情况下，重新编译CPython，并得到了类似的结果。因此，时间的差异确实应该解释为：

- 获得名称的额外步骤：
- 字典比数组慢的事实。

# LOAD_DEREF和STORE_DEREF

有一种情况，编译器不为函数本地化的变量生成LOAD_FAST和STORE_FAST操作代码：当在嵌套函数中使用变量时，就会发生这种情况。
```python
def f():
    b = 1
    def g():
        return b
```
```shell
$ python -m dis nested.py
...
Disassembly of <code object f at 0x1027c72f0, file "nested.py", line 1>:
  2           0 LOAD_CONST               1 (1)
              2 STORE_DEREF              0 (b)

  3           4 LOAD_CLOSURE             0 (b)
              6 BUILD_TUPLE              1
              8 LOAD_CONST               2 (<code object g at 0x1027c7240, file "nested.py", line 3>)
             10 LOAD_CONST               3 ('f.<locals>.g')
             12 MAKE_FUNCTION            8 (closure)
             14 STORE_FAST               0 (g)
             16 LOAD_CONST               0 (None)
             18 RETURN_VALUE

Disassembly of <code object g at 0x1027c7240, file "nested.py", line 3>:
  4           0 LOAD_DEREF               0 (b)
              2 RETURN_VALUE
```

编译器为cell and free variables生成操作代码LOAD_DEREF和操作代码STORE_DEREF。cell variable是嵌套函数中引用的局部变量。在我们的例子中，b是f函数的cell variable，因为它被g引用。从嵌套函数的角度来看，free variable是一个cell variable,是一个不绑定在嵌套函数中的变量，而是在封闭函数或已声明nonlocal的变量中绑定。在我们的例子中，b是g函数的free variable，因为它不是被g束缚的，而是被束缚在f。

在正常局部变量值之后，cell和free variable的值存储在`f_locals`plus数组中。唯一的区别是，`f_locals`plus[index_of_cell_or_free_variable]不是直接指向值，而是指向包含值的cell object：
```c
typedef struct {
    PyObject_HEAD
    PyObject *ob_ref;       /* Content of the cell or NULL when empty */
} PyCellObject;
```

操作代码STORE_DEREF从堆栈中弹出值，获取oparg指定的cell variable，并将该cell的ob_ref赋值到弹出值：
```c
case TARGET(STORE_DEREF): {
    PyObject *v = POP();
    PyObject *cell = freevars[oparg]; // freevars = f->`f_locals`plus + co->co_nlocals
    PyObject *oldobj = PyCell_GET(cell);
    PyCell_SET(cell, v); // expands to ((PyCellObject *)(cell))->ob_ref = v
    Py_XDECREF(oldobj);
    DISPATCH();
}
```

操作代码LOAD_DEREF的工作原理是将cell的内容推入堆栈：
```c
case TARGET(LOAD_DEREF): {
    PyObject *cell = freevars[oparg];
    PyObject *value = PyCell_GET(cell);
    if (value == NULL) {
      format_exc_unbound(tstate, co, oparg);
      goto error;
    }
    Py_INCREF(value);
    PUSH(value);
    DISPATCH();
}
```

将值存储在cell中的原因是什么？这样做是为了将一个free variable与相应的cell variable连接起来。它们的值存储在不同帧对象的不同命名空间中，但存储在同一cell中。当 VM 创建嵌套函数enclosed function时，将闭包的cell传递给enclosed function。操作代码LOAD_CLOSURE将cell推入堆栈，opcode MAKE_FUNCTION创建一个函数对象，带有该cell，用于相应的free variable。由于cell机制，当闭包enclosing function重新赋值cell variable时，enclosed function会看到重新赋值：
```python
def f():
    def g():
        print(a)
    a = 'assigned'
    g()
    a = 'reassigned'
    g()

f()
```
```shell
$ python cell_reassign.py 
assigned
reassigned
```

反之亦然：
```python
def f():
    def g():
        nonlocal a
        a = 'reassigned'
    a = 'assigned'
    print(a)
    g()
    print(a)

f()
```
```shell
$ python free_reassign.py 
assigned
reassigned
```

我们真的需要cell机制来实现这种行为吗？难道我们不能只使用嵌套函数的命名空间来加载和存储free variable的值吗？是的，我们可以，但考虑以下示例：
```python
def get_counter(start=0):
    def count():
        nonlocal c
        c += 1
        return c

    c = start - 1
    return count

count = get_counter() # count是count()函数,不是get_counter()
print(count())
print(count())
```
```shell
$ python counter.py 
0
1
```

请回想一下，当我们调用函数时，CPython 会创建一个帧对象来执行它。此示例显示，嵌套函数比闭包的帧对象的生命更久。cell机制的好处是，它允许避免将闭包的帧对象及其所有引用保留在内存中。

# LOAD_GLOBAL`和`STORE_GLOBAL`

编译器在函数中生成全局变量的操作代码LOAD_GLOBAL`和操作代码`STORE_GLOBAL`。如果该变量被声明global，或者它不受函数和任何闭包的约束（即它既不是本地的，也不是free），则该变量在函数中被视为全局性的。下面是一个例子：
```python
a = 1
d = 1

def f():
    b = 1
    def g():
        global d
        c = 1
        d = 1
        return a + b + c + d
```

对于g，变量c不是全局性的，因为它是g本地的；变量b对g不是全局性的，因为它是free；变量a对g是全局性的，因为它既不是本地的，也不是free variable；变量d是全局性的，因为它被global声明。

操作代码`STORE_GLOBAL`的实现情况如下：
```c
case TARGET(`STORE_GLOBAL`): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *v = POP();
    int err;
    err = PyDict_SetItem(f->`f_globals`, name, v);
    Py_DECREF(v);
    if (err != 0)
        goto error;
    DISPATCH();
}
```

帧对象的`f_globals`字段是将全局名称映射到其值的字典。当 CPython 为模块创建帧对象时，它会将模块的字典赋值到`f_globals`中。我们可以轻松地检查：
```shell
$ python -q
>>> import sys
>>> globals() is sys.modules['__main__'].__dict__
True
```

当 VM 执行操作代码MAKE_FUNCTION以创建新函数对象时，它会将该对象的func_globals字段=当前帧对象的`f_globals`。当函数被调用时，VM 会为其创建一个新的帧对象，并将`f_globals`设置为func_globals。

LOAD_GLOBAL`的实现类似于带有两个异常的`LOAD_NAME`：

- 它不在`f_locals`查找值。
- 它使用缓存来减少查找时间。

CPython 将结果缓存到co_opcache数组中的代码对象中。此数组存储指向_PyOpcache结构体的指针：
```c
typedef struct {
    PyObject *ptr;  /* Cached pointer (borrowed reference) */
    uint64_t globals_ver;  /* ma_version of global dict */
    uint64_t builtins_ver; /* ma_version of builtin dict */
} _PyOpcache_LoadGlobal;

struct _PyOpcache {
    union {
        _PyOpcache_LoadGlobal lg;
    } u;
    char optimized;
};
```

_PyOpcache_LoadGlobal结构ptr字段指向LOAD_GLOBAL`的实际结果。缓存按指令编号维护。代码对象中的另一个co_opcache_map数组将字节码中的每个指令映射为co_opcache的索引减去一。如果指令不是LOAD_GLOBAL`，它会将指令映射到0，这意味着指令永远不会缓存。缓存的大小不超过 254。如果字节码包含超过 254 个LOAD_GLOBAL`指令，则co_opcache_map也会映射额外的说明为0。

如果 VM 在执行LOAD_GLOBAL`时在缓存中找到一个值，则意味着自上次查找这个值以来，f_global和f_builtins字典没有被修改过。这是通过分别比较globals_ver、builtins_ver与字典ma_version_tag的异同。每次修改字典时，字典的ma_version_tag字段都会更改。有关详细信息，请参阅[PEP 509](https://www.python.org/dev/peps/pep-0509/)。

如果 VM 在缓存中找不到值，则先在`f_globals`中查找，然后再查找f_builtins。如果它最终找到一个值，将值推入堆栈，并记当前ma_version_tag为globals_ver、builtins_ver。

# `LOAD_NAME`和`STORE_NAME`（和LOAD_CLASSDEREF）

在这一点上，你可能想知道为什么CPython不使用`LOAD_NAME`和`STORE_NAME`操作代码。编译器在编译函数时确实不会生成这些操作代码。但是，除了函数之外，CPython 还有另外两种类型的代码块：模块和类定义。我们根本没有讨论过类定义，所以让我们来解决它。

首先，了解当我们定义一个类时，VM 执行其主体至关重要。我的意思是：
```python
class A:
    print('This code is executed')
```
```shell
$ python create_class.py 
This code is executed
```

编译器为类定义创建代码对象，就像为模块和函数创建代码对象一样。有趣的是，编译器几乎总是为类主体内的变量生成`LOAD_NAME`和`STORE_NAME`操作代码。此规则有两个罕见的异常：free variable和明确声明global的变量。

VM 以不同的方式执行操作代码*_NAME和操作代码*_FAST。因此，变量在类主体中的工作方式与在函数中的工作方式不同：

```python
x = 'global'

class C:
    print(x)
    x = 'local'
    print(x)
```
```
$ python class_local.py
global
local
```

在第一次加载时，VM 会从`f_globals`加载变量x的值。然后，它将新值存储进`f_locals`来，并在第二次加载时从`f_locals`加载它。如果C是一个函数，当我们调用它，我们会得到UnboundLocalError: local variable 'x' referenced before assignment，因为编译器会认为变量x是C本地的。

类和函数的命名空间如何相互作用？当我们将函数放置在类中（这是实现方法的常见做法）时，函数看不到在类命名空间中绑定的名称：
```python
class D:
    x = 1
    def method(self):
        print(x)

D().method()
```
```shell
$ python func_in_class.py
...
NameError: name 'x' is not defined
```

这是因为 VM 在执行类定义时`STORE_NAME`存储其x值，并在执行函数时尝试加载LOAD_GLOBAL`。但是，当我们在函数内放置类定义时，cell机制的工作方式就像将函数放置在函数内一样：
```python
def f():
    x = "I'm a cell variable"
    class B:
        print(x)

f()
```
```
$ python class_in_func.py 
I'm a cell variable
```

不过，这是有区别的。编译器生成操作代码LOAD_CLASSDEREF代替LOAD_DEREF加载x 。dis模块的文档[解释了LOAD_CLASSDEREF做了什么](https://docs.python.org/3/library/dis.html#opcode-LOAD_CLASSDEREF)：

- 很像[LOAD_DEREF](https://docs.python.org/3/library/dis.html#opcode-LOAD_DEREF)，但先检查locals 的字典，然后再考虑cell。这用于在类主体中加载free variable。

为什么它首先检查locals 的字典？在函数的情况下，编译器可以确定变量是否为本地。在类中，编译器无法确定。这是因为 CPython 具有元类metaclasses，元类可以通过实现[`__prepare__`](https://docs.python.org/3/reference/datamodel.html#preparing-the-class-namespace)方法为类准备一个非空的locals 字典。

我们现在可以看到为什么编译器为类定义生成`LOAD_NAME`和`STORE_NAME`操作代码，但我们也看到它为模块命名空间内的变量生成这些操作代码，例如在示例`a = b`中。它们的工作如预期的那样，因为模块的`f_locals`和模块的`f_globals`是一样的：
```shell
$ python -q
>>> locals() is globals()
True
```

您可能想知道为什么Cpthon在这种情况下不使用`LOAD_GLOBAL`和`STORE_GLOBAL`操作代码。老实说，我不知道确切的原因，如果有的话，但我有一个猜测。CPython 提供内置[compile()](https://docs.python.org/3/library/functions.html#compile), [eval()](https://docs.python.org/3/library/functions.html#eval)和[exec()](https://docs.python.org/3/library/functions.html#exec)函数，可用于动态编译和执行 Python 代码。 这些函数使用顶层命名空间内的操作代码`LOAD_NAME`和操作代码`STORE_NAME`。它非常有意义，因为它允许在类主体中动态执行代码，并获得与该代码写在那里相同的效果：

```python
a = 1

class A:
    b = 2
    exec('print(a + b)', globals(), locals())
```
```shell
$ python exec.py
3
```

CPython选择始终使用模块`LOAD_NAME`和`STORE_NAME`操作代码。这样，编译器以正常方式运行模块时产生的字节码与执行exec()模块时相同的。

# 编译器如何决定要生成哪个操作代码

我们在本系列的第 2 部分中了解到，在编译器为代码块创建代码对象之前，它为该块构建了一个符号表。符号表包含有关代码块中使用的符号（即名称）的信息，包括其作用域。编译器根据其作用域和当前正在编译的代码块的类型，决定为给定名称生成哪个负载/存储操作代码。该算法可以总结如下：

1. 确定变量的作用域：
    1. 如果变量被宣布global，它是一个明确的全局变量。
    2. 如果变量被宣布nonlocal，它是一个free variable。
    3. 如果变量被绑定在当前代码块内，则它是一个本地变量。
    4. 如果变量被绑定在不属于类定义的闭包中，则该变量为free variable。
    5. 否则，这是一个隐含的全局变量。
2. 更新作用域：
    如果变量是本地的，并且它在嵌套函数中是free variable，则它是一个cell variable。
3. 决定要生成哪个操作代码：
    1. 如果变量是cell variable或free variable，生成*_DEREF操作代码：如果当前代码块是类定义，则生成LOAD_CLASSDEREF操作代码以加载值。
    1. 如果变量是局部变量，而当前代码块是函数，则生成*_FAST opcode。
    1. 如果变量是一个明确的全局变量，或者如果它是一个隐含的全局变量，且当前代码块是一个函数，则生成*_GLOBAL opcode。
    1. 否则，生成*_NAME操作代码。

你不需要记住这些规则。您可以随时阅读源代码。查看[Python/symtable.c](https://github.com/python/cpython/blob/3.9/Python/symtable.c#L497)，了解编译器如何确定变量的作用域，以及 [Python/compile.c](https://github.com/python/cpython/blob/3.9/Python/compile.c#L3550)查看它如何决定要生成哪个操作代码。

# 结论

Python 变量的话题比最初看起来要复杂得多。Python 文档的很大一部分与变量有关，包括关于[命名和绑定](https://docs.python.org/3/reference/executionmodel.html#naming-and-binding)的部分以及关于[作用域和命名空间](https://docs.python.org/3/tutorial/classes.html#python-scopes-and-namespaces)的部分。Python 常见问题解答的首要问题是关于变量的。我对堆栈溢出的问题只字不提。虽然官方资源给出了一些关于Python变量工作方式的想法，但仍然很难理解和记住所有的规则。幸运的是，通过研究 Python 实现的源代码，更容易理解 Python 变量的工作原理。这就是我们今天所做的。

我们研究了 CPython 用来加载和存储变量值的一组操作代码。要了解VM如何执行其他实际计算某些操作代码，我们需要讨论Python-Python对象系统的核心。这是我们下次的计划。