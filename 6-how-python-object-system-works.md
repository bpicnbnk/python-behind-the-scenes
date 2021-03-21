# from

[source](https://tenthousandmeters.com/blog/python-behind-the-scenes-6-how-python-object-system-works/)

# pre

正如我们从本系列的前几个部分了解到的，Python 程序的执行包括两个主要步骤：

- CPython 编译器将 Python 代码翻译为字节码。
- CPython VM执行字节码。

我们已经把注意力集中在第二步相当长一段时间了。在第4 部分，我们查看了评估循环，这是执行 Python 字节码的地方。在第5 部分，我们研究了 VM 如何执行用于实现变量的指令。我们尚未涵盖的是VM如何实际计算某些内容。我们推迟了这个问题，因为要回答这个问题，我们首先需要了解语言最基本的部分是如何运作的。今天，我们将研究Python对象系统。

注意：在这篇文章中，我指的是CPython 3.9。随着 CPython 的发展，一些实施细节肯定会发生变化。我将尝试跟踪重要更改并添加更新说明。

# Motivation 动机
考虑一个非常简单的 Python 代码：
```python
def f(x):
    return x + 7
```

要计算函数`f`，CPython 必须计算表达式`x + 7`。我想问的问题是：CPython是如何做到的？特殊的方法，如`__add__()`和`__radd__()`可能来到你的脑海。当我们在类中定义这些方法时，可以使用`+`操作符对该类的实例做加法。所以，你可能会认为CPython做这样的事情：

1. 它调用`x.__add__(7)`或`type(x).__add__(x, 7)`.
2. 如果`x`没有`__add__()`，或者如果这种方法失败，它调用`(7).__radd__(x)`或`int.__radd__(7, x)`。

现实是有点复杂。真正发生的事情取决于是`x`什么。例如，如果`x`是用户定义的类的实例，则上面描述的算法接近真实。但是，如果`x`是内置类型的实例，例如`int`,`float`, CPython 根本不调用任何特殊方法。

要了解一些 Python 代码是如何执行的，我们可以执行以下工作：

1. 将代码反汇编到字节码中。
2. 研究 VM 如何执行反汇编的字节码指令。

让我们将此算法应用到函数`f`中。编译器将此函数的主体转换为以下字节码：
```shell
$ python -m dis f.py
...
  2           0 LOAD_FAST                0 (x)
              2 LOAD_CONST               1 (7)
              4 BINARY_ADD
              6 RETURN_VALUE
```

以下是这些代名码说明的操作：

1. LOAD_FAST将参数值x加载到堆栈中。
1. LOAD_CONST将常数7加载到堆栈上。
1. BINARY_ADD从堆栈中弹出两个值，添加它们并将结果推回堆栈。
1. RETURN_VALUE从堆栈中弹出值并返回它。

VM 如何加两个值？要回答这个问题，我们需要了解这些值是什么。对我们来说，7是一个`int`实例，`x`是任何东西。不过，对于 VM 来说，一切都是 Python 对象。VM 推到堆栈上的所有值和从堆栈中弹出的所有值都是指向PyObject结构的指针（因此短语"Everything in Python is an object ，Python 中的一切都是对象"）。

VM 不需要知道如何加整数或字符串，即如何进行算术或串联序列。所有它需要知道的是，每个Python对象都有一个类型。反过来，一个类型，知道关于它的对象的一切。例如，`int`类型知道如何加整数，`float`类型知道如何加浮点数。因此，VM 要求该类型执行操作。

这种简化的解释抓住了解决方案的精髓，但它也省略了很多重要的细节。为了获得更逼真的信息，我们需要了解 Python 对象和类型的真实情况以及它们的工作原理。

# Python 对象和类型

我们已经在第3部分讨论了一点Python对象。这个讨论值得这里重复一遍。

我们从PyObject结构的定义开始：
```c
typedef struct _object {
    _PyObject_HEAD_EXTRA // macro, for debugging purposes only
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
```

它有两个成员：

- CPython 用于垃圾收集的引用计数ob_refcnt;
- 对象类型的指针ob_type。

我们说，VM把任何Python对象当作PyObject。这怎么可能？C 编程语言没有类和继承的概念。然而，有可能在 C 中实现一些可以称为单一继承（a single inheritance）的东西。C 标准指出，指向任何结构的指针可以转换为指向其第一个成员的指针，反之亦然。因此，我们可以通过定义其第一个成员的新结构为PyObject来"扩展"PyObject。

例如，如何定义`float`对象：
```c
typedef struct {
    PyObject ob_base; // expansion of PyObject_HEAD macro
    double ob_fval;
} PyFloatObject;
```

float对象存储PyObject存储的所有东西和浮点值ob_fval。C 标准只是指出，我们可以将PyFloatObject指针转换为PyObject指针，反之亦然：
```c
PyFloatObject float_object;
// ...
PyObject *obj_ptr = (PyObject *)&float_object;
PyFloatObject *float_obj_ptr = (PyFloatObject *)obj_ptr;
```

VM 之所以像对待每个 Python 对象为PyObject，是因为它需要访问的只是对象的类型。类型也是 Python 对象，PyTypeObject结构示例：
```c
// PyTypeObject is a typedef for "struct _typeobject"

struct _typeobject {
    PyVarObject ob_base; // expansion of PyObject_VAR_HEAD macro
    const char *tp_name; /* For printing, in format "<module>.<name>" */
    Py_ssize_t tp_basicsize, tp_itemsize; /* For allocation */

    /* Methods to implement standard operations */

    destructor tp_dealloc;
    Py_ssize_t tp_vectorcall_offset;
    getattrfunc tp_getattr;
    setattrfunc tp_setattr;
    PyAsyncMethods *tp_as_async; /* formerly known as tp_compare (Python 2)
                                    or tp_reserved (Python 3) */
    reprfunc tp_repr;

    /* Method suites for standard classes */

    PyNumberMethods *tp_as_number;
    PySequenceMethods *tp_as_sequence;
    PyMappingMethods *tp_as_mapping;

    /* More standard operations (here for binary compatibility) */

    hashfunc tp_hash;
    ternaryfunc tp_call;
    reprfunc tp_str;
    getattrofunc tp_getattro;
    setattrofunc tp_setattro;

    /* Functions to access object as input/output buffer */
    PyBufferProcs *tp_as_buffer;

    /* Flags to define presence of optional/expanded features */
    unsigned long tp_flags;

    const char *tp_doc; /* Documentation string */

    /* Assigned meaning in release 2.0 */
    /* call function for all accessible objects */
    traverseproc tp_traverse;

    /* delete references to contained objects */
    inquiry tp_clear;

    /* Assigned meaning in release 2.1 */
    /* rich comparisons */
    richcmpfunc tp_richcompare;

    /* weak reference enabler */
    Py_ssize_t tp_weaklistoffset;

    /* Iterators */
    getiterfunc tp_iter;
    iternextfunc tp_iternext;

    /* Attribute descriptor and subclassing stuff */
    struct PyMethodDef *tp_methods;
    struct PyMemberDef *tp_members;
    struct PyGetSetDef *tp_getset;
    struct _typeobject *tp_base;
    PyObject *tp_dict;
    descrgetfunc tp_descr_get;
    descrsetfunc tp_descr_set;
    Py_ssize_t tp_dictoffset;
    initproc tp_init;
    allocfunc tp_alloc;
    newfunc tp_new;
    freefunc tp_free; /* Low-level free-memory routine */
    inquiry tp_is_gc; /* For PyObject_IS_GC */
    PyObject *tp_bases;
    PyObject *tp_mro; /* method resolution order */
    PyObject *tp_cache;
    PyObject *tp_subclasses;
    PyObject *tp_weaklist;
    destructor tp_del;

    /* Type attribute cache version tag. Added in version 2.6 */
    unsigned int tp_version_tag;

    destructor tp_finalize;
    vectorcallfunc tp_vectorcall;
};
```

顺便说一下，请注意，类型的第一个成员不是`PyObject`，而是`PyVarObject`，其定义如下：
```c
typedef struct {
    PyObject ob_base;
    Py_ssize_t ob_size; /* Number of items in variable part */
} PyVarObject;
```

然而，由于`PyVarObject`第一个成员是`PyObject`，类型指针仍然可以转换为`PyObject`指针。

那么，类型是什么，为什么它有这么多的成员？类型决定该类型的对象的行为方式。一种类型的每个成员（称为插槽slot）都负责对象行为的特定方面。例如：

- tp_new是创建类型新对象的函数的指针。
- tp_str是用于实现该类型对象的函数str()的指针。
- tp_hash是用于实现该类型对象的函数hash()的指针。

某些插槽（称为子插槽 sub-slots）被分组在一个套件（suite ）中。套件只是包含相关插槽的结构。例如，`PySequenceMethods`结构是实现序列协议的子插槽套件：
```c
typedef struct {
    lenfunc sq_length;
    binaryfunc sq_concat;
    ssizeargfunc sq_repeat;
    ssizeargfunc sq_item;
    void *was_sq_slice;
    ssizeobjargproc sq_ass_item;
    void *was_sq_ass_slice;
    objobjproc sq_contains;

    binaryfunc sq_inplace_concat;
    ssizeargfunc sq_inplace_repeat;
} PySequenceMethods;
```

如果你计数所有的插槽和子插槽，你会得到一个可怕的数字。幸运的是，每个插槽在 Python/C API 参考手册中都有很好的[记录](https://docs.python.org/3/c-api/typeobj.html)（我强烈建议您为此链接添加书签）。今天，我们将只覆盖几个插槽。然而，它将给我们一个如何使用插槽的通用想法。

由于我们对 CPython 如何加对象感兴趣，让我们找到负责加法的插槽。必须至少有一个这样的插槽。仔细检查`PyTypeObject`结构后，我们发现它有"编号"套件`PyNumberMethods`，该套件的第一个插槽是称为二进制函数`nd_add`：
```c
typedef struct {
    binaryfunc nb_add; // typedef PyObject * (*binaryfunc)(PyObject *, PyObject *)
    binaryfunc nb_subtract;
    binaryfunc nb_multiply;
    binaryfunc nb_remainder;
    binaryfunc nb_divmod;
    // ... more sub-slots
} PyNumberMethods;
```

看来`nb_add`插槽是我们要找的。关于此插槽，自然会出现两个问题：

- 它设定了什么？

- 它是如何使用的？

我觉得最好从第二个开始。我们应该期待VM调用`nb_add`执行`BINARY_ADD`操作代码。因此，让我们暂时暂停关于类型的讨论，并看看`BINARY_ADD`操作代码是如何实现的。

# BINARY_ADD

与任何其他操作代码一样，`BINARY_ADD`在[Python/ceval.c](https://github.com/python/cpython/blob/3.9/Python/ceval.c#L1684)中的评估循环中实现：
```c
case TARGET(BINARY_ADD): {
    PyObject *right = POP();
    PyObject *left = TOP();
    PyObject *sum;
    /* NOTE(haypo): Please don't try to micro-optimize int+int on
        CPython using bytecode, it is simply worthless.
        See http://bugs.python.org/issue21955 and
        http://bugs.python.org/issue10044 for the discussion. In short,
        no patch shown any impact on a realistic benchmark, only a minor
        speedup on microbenchmarks. */
    if (PyUnicode_CheckExact(left) &&
                PyUnicode_CheckExact(right)) {
        sum = unicode_concatenate(tstate, left, right, f, next_instr);
        /* unicode_concatenate consumed the ref to left */
    }
    else {
        sum = PyNumber_Add(left, right);
        Py_DECREF(left);
    }
    Py_DECREF(right);
    SET_TOP(sum);
    if (sum == NULL)
        goto error;
    DISPATCH();
}
```

此代码需要一些注释。我们可以看到，它调用`PyNumber_Add()`添加两个对象，但如果对象是字符串，它调用`unicode_concatenate()`代替。为什么会这样？这是一个优化。Python 字符串似乎不可变immutable，但有时 CPython 会变异mutates 字符串，从而避免创建新的字符串。考虑将一个字符串附加到另一个字符串：
```python
output += some_string
```

如果变量`output`指向没有其他引用的字符串，则可以安全地变异该字符串。这正是`unicode_concatenate()`实现的逻辑。

在评估循环中处理其他特殊案例并优化整数和浮点数可能很诱人。该评论明确警告不要这样做。问题是，有附带一个额外的检查新的特殊情况，这个检查只有在它成功时才有用。否则，它可能会对性能产生负面影响。

在这个小小的偏离之后，让我们来看看`PyNumber_Add()`：
```c
PyObject *
PyNumber_Add(PyObject *v, PyObject *w)
{
    // NB_SLOT(nb_add) expands to "offsetof(PyNumberMethods, nb_add)"
    PyObject *result = binary_op1(v, w, NB_SLOT(nb_add));
    if (result == Py_NotImplemented) {
        PySequenceMethods *m = Py_TYPE(v)->tp_as_sequence;
        Py_DECREF(result);
        if (m && m->sq_concat) {
            return (*m->sq_concat)(v, w);
        }
        result = binop_type_error(v, w, "+");
    }
    return result;
}
```
我建议立即介入binary_op1()，并找出其余的PyNumber_Add()
以后做什么：
```c
static PyObject *
binary_op1(PyObject *v, PyObject *w, const int op_slot)
{
    PyObject *x;
    binaryfunc slotv = NULL;
    binaryfunc slotw = NULL;

    if (Py_TYPE(v)->tp_as_number != NULL)
        slotv = NB_BINOP(Py_TYPE(v)->tp_as_number, op_slot);
    if (!Py_IS_TYPE(w, Py_TYPE(v)) &&
        Py_TYPE(w)->tp_as_number != NULL) {
        slotw = NB_BINOP(Py_TYPE(w)->tp_as_number, op_slot);
        if (slotw == slotv)
            slotw = NULL;
    }
    if (slotv) {
        if (slotw && PyType_IsSubtype(Py_TYPE(w), Py_TYPE(v))) {
            x = slotw(v, w);
            if (x != Py_NotImplemented)
                return x;
            Py_DECREF(x); /* can't do it */
            slotw = NULL;
        }
        x = slotv(v, w);
        if (x != Py_NotImplemented)
            return x;
        Py_DECREF(x); /* can't do it */
    }
    if (slotw) {
        x = slotw(v, w);
        if (x != Py_NotImplemented)
            return x;
        Py_DECREF(x); /* can't do it */
    }
    Py_RETURN_NOTIMPLEMENTED;
}
```

该函数`binary_op1()`需要三个参数：左操作、右操作和识别插槽的偏移。两个操作的类型可以实现插槽。因此，`binary_op1()`查找这两个实现。要计算结果，它根据以下逻辑调用一个或另一个实现：

1. 如果一个操作的类型是另一个操作的子类型，请调用子类型的插槽。

1. 如果左操作没有插槽，请调用右操作的插槽。

1. 否则，请调用左操作的插槽。

优先考虑子插槽的原因是允许子类型覆盖其祖先的行为：
```shell
$ python -q
>>> class HungryInt(int):
...     def __add__(self, o):
...             return self
...
>>> x = HungryInt(5)
>>> x + 2
5
>>> 2 + x
7
>>> HungryInt.__radd__ = lambda self, o: self
>>> 2 + x
5
```

让我们回到`PyNumber_Add()`。如果`binary_op1()`成功，`PyNumber_Add()`只需返回`binary_op1()`结果。但是，如果`binary_op1()`返回[NotImplemented](https://docs.python.org/3/library/constants.html#NotImplemented)常量，这意味着无法执行对给定类型组合的操作，则`PyNumber_Add()`调用第一个操作的`sq_concat`"序列"插槽并返回此调用的结果：
```c
PySequenceMethods *m = Py_TYPE(v)->tp_as_sequence;
if (m && m->sq_concat) {
    return (*m->sq_concat)(v, w);
}
```

一种类型可以通过实现`nb_add`或`sq_concat`支持+操作。这些插槽有不同的含义：

- nb_add意味着代数添加与属性，如`a + b = b + a`。
- sq_concat表示序列的串联。

内置类型，如`int`和`float`实施nb_add，和内置类型，如str和list实施`sq_concat`。从技术上讲，没有太大区别。选择一个插槽而不是另一个插槽的主要原因是指示适当的含义。事实上，该`sq_concat`插槽是不必要，在所有用户定义的类型（即类）它设置为NULL。

我们看到了`nb_add`插槽的使用方式：它是由函数`binary_op1()`调用的。下一步是看看它设定了什么。

# nb_add是什么

由于加法是针对不同类型的不同操作，则一个类型的nb_add插槽必须是两种操作之一：

- 它要么是加该类型的对象的类型特定函数：或
- 它是一种类型不可知函数，来调用某些类型特定函数，例如类型的特殊方法__add__()。

这确实是这两个之一，用哪一个取决于类型。例如，内置类型，如int和float有自己的nb_add实现。相比之下，其他所有类都共享相同的实现。从根本上说，内置类型和类是相同的-PyTypeObject实例。它们之间的重要区别是如何创建它们的。此差异会影响插槽的设置方式，因此我们应该讨论它。

# 创建类型的方法

创建类型对象的方法有两种：

- 通过静态定义它;或
- 通过动态分配它。

## 静态定义类型

静态定义类型的示例是任何内置类型。例如，CPython 如何定义float类型：
```c
PyTypeObject PyFloat_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "float",
    sizeof(PyFloatObject),
    0,
    (destructor)float_dealloc,                  /* tp_dealloc */
    0,                                          /* tp_vectorcall_offset */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_as_async */
    (reprfunc)float_repr,                       /* tp_repr */
    &float_as_number,                           /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    (hashfunc)float_hash,                       /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,   /* tp_flags */
    float_new__doc__,                           /* tp_doc */
    0,                                          /* tp_traverse */
    0,                                          /* tp_clear */
    float_richcompare,                          /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    float_methods,                              /* tp_methods */
    0,                                          /* tp_members */
    float_getset,                               /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    0,                                          /* tp_init */
    0,                                          /* tp_alloc */
    float_new,                                  /* tp_new */
};
```

静态定义类型的插槽被明确指定。通过查看"number"套件，我们可以轻松地了解该float类型是如何实现nb_add的：
```c
static PyNumberMethods float_as_number = {
    float_add,          /* nb_add */
    float_sub,          /* nb_subtract */
    float_mul,          /* nb_multiply */
    // ... more number slots
};
```

float_add()函数是nb_add的直接实现：
```c
static PyObject *
float_add(PyObject *v, PyObject *w)
{
    double a,b;
    CONVERT_TO_DOUBLE(v, a);
    CONVERT_TO_DOUBLE(w, b);
    a = a + b;
    return PyFloat_FromDouble(a);
}
```

浮点算术对于我们的讨论来说并不重要。此示例演示了如何指定静态定义类型的行为。事实证明，这很容易：只需将插槽的实现写入相应的实施点，然后将每个插槽指向相应的实现。

如果你想学习如何静态地定义自己的类型，请查看[Python's tutorial for C/C++ programmers](https://docs.python.org/3/extending/newtypes_tutorial.html)。

## 动态分配类型

动态分配的类型是我们使用`class`语句定义的类型。正如我们已经说过的，它们是`PyTypeObject`实例，就像静态定义的类型一样。传统上，我们称它们为类，但我们也可以称它们为用户定义的类型。

从程序员的角度来看，在 Python 中定义一个类比在 C 中的类型更容易。这是因为 CPython 在创建类时在幕后做了很多事情。让我们看看这个过程涉及什么。

如果我们不知道从哪里开始，我们可以应用熟悉的方法：

1. 定义一个简单的类
```python
class A:
    pass
```

2. 运行反编译器：
```shell
$ python -m dis class_A.py
```

3. 研究 VM 如何执行生成的编解码指令。

如果您找到时间，请随时阅读，或阅读伊莱·本德斯基[关于类的文章](https://eli.thegreenplace.net/2012/06/15/under-the-hood-of-python-class-definitions)。我们走捷径。

对象是由类型的调用创建的，例如`list()` 或`MyClass()`。类是由对元类metatype的调用创建的。元类只是一种实例为类型的类型。Python有一个内置的元类`PyType_Type`，这对我们来说就是所谓的`type`。以下是它的定义：
```c
PyTypeObject PyType_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "type",                                     /* tp_name */
    sizeof(PyHeapTypeObject),                   /* tp_basicsize */
    sizeof(PyMemberDef),                        /* tp_itemsize */
    (destructor)type_dealloc,                   /* tp_dealloc */
    offsetof(PyTypeObject, tp_vectorcall),      /* tp_vectorcall_offset */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_as_async */
    (reprfunc)type_repr,                        /* tp_repr */
    0,                                          /* tp_as_number */
    0,                                          /* tp_as_sequence */
    0,                                          /* tp_as_mapping */
    0,                                          /* tp_hash */
    (ternaryfunc)type_call,                     /* tp_call */
    0,                                          /* tp_str */
    (getattrofunc)type_getattro,                /* tp_getattro */
    (setattrofunc)type_setattro,                /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
    Py_TPFLAGS_BASETYPE | Py_TPFLAGS_TYPE_SUBCLASS |
    Py_TPFLAGS_HAVE_VECTORCALL,                 /* tp_flags */
    type_doc,                                   /* tp_doc */
    (traverseproc)type_traverse,                /* tp_traverse */
    (inquiry)type_clear,                        /* tp_clear */
    0,                                          /* tp_richcompare */
    offsetof(PyTypeObject, tp_weaklist),        /* tp_weaklistoffset */
    0,                                          /* tp_iter */
    0,                                          /* tp_iternext */
    type_methods,                               /* tp_methods */
    type_members,                               /* tp_members */
    type_getsets,                               /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    offsetof(PyTypeObject, tp_dict),            /* tp_dictoffset */
    type_init,                                  /* tp_init */
    0,                                          /* tp_alloc */
    type_new,                                   /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
    (inquiry)type_is_gc,                        /* tp_is_gc */
};
```

所有内置类型的类型是`type`，所有类的类型默认为`type`。因此，`type`确定类型的行为方式。例如，当我们调用指定tp_call插槽的类型时，例如`list()`或`MyClass()`,会发生什么情况。type的插槽tp_call实现是函数type_call()。它的工作是创建新的对象。它调用另外两个插槽来做到这一点：

- 它调用类型的tp_new来创建对象。
- 它调用类型的tp_init来初始化创建的对象。

type的类型是type本身。因此，当我们调用type()时，函数type_call()被调用。它检查特殊情况——传递单一参数给type()。在这种情况下，type_call()只需返回传递对象的类型：
```shell
$ python -q
>>> type(3)
<class 'int'>
>>> type(int)
<class 'type'>
>>> type(type)
<class 'type'>
```

但是，当我们传递三个参数给type()，type_call()创建一个新的类型通过调用tp_new和tp_init。以下示例演示了如何使用type()创建类：
```shell
$ python -q
>>> MyClass = type('MyClass', (), {'__str__': lambda self: 'Hey!'})
>>> instance_of_my_class = MyClass()
>>> str(instance_of_my_class)
Hey!
```

我们传递给type()的参数是：

1. 类的名称
1. 其bases（基本元素）的元组：和
1. 一个命名空间。

其他元类也是这种形式。

我们看到，我们可以通过调用type()创建一个类，但这不是我们通常做的。通常，我们使用class语句来定义一个类。事实证明，在这种情况下，VM最终也调用一些元类，而且最常见的是它调用type()。

要执行该class语句，VM 会从builtins模块调用该__build_class__()函数。此函数的作用可以概括为：

1. 决定调用哪个元类来创建该类。
1. [准备命名空间](https://docs.python.org/3/reference/datamodel.html#preparing-the-class-namespace)。命名空间将用作类的字典。
1. 在命名空间中执行类的主体，从而填充名称空间。
1. 调用元类。

我们可以__build_class__()，其使用metaclass关键字指示它应该调用的元类。如果metaclass未指定，则__build_class__()默认呼叫type()。它还考虑了base的元类。在文档中很好地描述了[选择元类的确切逻辑](https://docs.python.org/3/reference/datamodel.html#determining-the-appropriate-metaclass)。

假设我们定义了一个新类，但未指定metaclass。该类实际上是在哪里创建的？在这种情况下，`__build_class__()`调用`type()`，调用type_call()函数，调用`type`的`tp_new`和`tp_init`插槽。`type`的tp_new插槽指向函数`type_new()`。这是创建类的函数。`type`的插槽`tp_init`指向无所事事的函数，因此所有工作都由`type_new()`完成。

该type_new()函数有近 500 行长，可能值得单独发布。其本质，虽然，可以简要总结如下：

1. 分配Allocate新类型的对象。
2. 设置分配的类型对象。

要完成第一步，`type_new()`必须分配一个`PyTypeObject`实例以及suites。`PyTypeObject`的suites必须各自分配，因为suites本身只包含指针，而不是suites本身。要处理此不便，type_new()分配扩展PyTypeObject并包含套件的PyHeapTypeObject结构的实例：

```c
/* The *real* layout of a type object when allocated on the heap */
typedef struct _heaptypeobject {
    PyTypeObject ht_type;
    PyAsyncMethods as_async;
    PyNumberMethods as_number;
    PyMappingMethods as_mapping;
    PySequenceMethods as_sequence;
    PyBufferProcs as_buffer;
    PyObject *ht_name, *ht_slots, *ht_qualname;
    struct _dictkeysobject *ht_cached_keys;
    PyObject *ht_module;
    /* here are optional user slots, followed by the members. */
} PyHeapTypeObject;
```

设置类型对象意味着设置其插槽。这是type_new()在大多数情况下所做的事情。

## 类型初始化

在使用任何类型之前，应使用该PyType_Ready()函数进行初始化。对于一个类，由`type_new()`调用`PyType_Ready()`。对于静态定义的类型，必须明确调用PyType_Ready()。当 CPython 启动时，它为每个内置类型调用PyType_Ready()。

该函数可以做很多事情。例如，它确实会进行插槽继承。PyType_Ready()

## 插槽继承

当我们定义从其他类型继承的类时，我们期望该类继承该类型的某些行为。例如，当我们定义一个继承int的类时，我们希望它支持添加：
```shell
$ python -q
>>> class MyInt(int):
...     pass
... 
>>> x = MyInt(2)
>>> y = MyInt(4)
>>> x + y
6
```

MyInt是否继承了int的nb_add？是的，确实如此。从单个祖先那里继承插槽非常简单：只需复制该类没有的插槽。当一个class有多个bases时，它就复杂了一点。由于base也可以继承其他类型，所有这些祖先类型合并形成一个层次结构。等级制度的问题在于它没有指定继承顺序。要解决此问题，PyType_Ready()将此层次结构转换为列表。方法解决顺序（The Method Resolution Order，[MRO](https://www.python.org/download/releases/2.3/mro/)） 决定如何执行此转换。一旦计算了 MRO，在一般情况下就很容易实现继承。根据 MRO，该PyType_Ready()函数在祖先身上重复。从每个祖先，它复制那些没有设置在类型上的插槽。有些插槽支持继承，有些插槽不支持继承。您可以在[文档](https://docs.python.org/3/c-api/typeobj.html#type-objects)中检查是否可以继承特定插槽。

与类相比，静态定义的类型最多只能指定一个基类base。这是通过实现tp_base插槽完成的。

如果没有指定基类，则PyType_Ready()假定object类型是唯一的基类base。每种类型直接或间接继承object。为什么？因为它实现了每个类型预期拥有的插槽。例如，它实现tp_alloc，tp_init和tp_repr插槽。

# 最终问题

到目前为止，我们已经看到可以设置插槽的两种方式：

- 它可以被明确指定（如果类型是静态定义的类型）。
- 它可以从祖先那里继承。

目前还不清楚一个类的插槽是如何连接到其特殊方法的。此外，对于内置类型，我们还有一个反向问题。他们如何实施特殊方法？他们当然有：
```shell
$ python -q
>>> (3).__add__(4)
7
```

我们来这个post的最终问题：特殊方法和插槽之间的联系是什么？

## 特殊方法和插槽 special methods and slots

答案在于，CPython 在特殊方法和插槽之间保持映射。此映射由slotdefs数组表示。看起来像这样：
```c
#define TPSLOT(NAME, SLOT, FUNCTION, WRAPPER, DOC) \
    {NAME, offsetof(PyTypeObject, SLOT), (void *)(FUNCTION), WRAPPER, \
     PyDoc_STR(DOC)}

static slotdef slotdefs[] = {
    TPSLOT("__getattribute__", tp_getattr, NULL, NULL, ""),
    TPSLOT("__getattr__", tp_getattr, NULL, NULL, ""),
    TPSLOT("__setattr__", tp_setattr, NULL, NULL, ""),
    TPSLOT("__delattr__", tp_setattr, NULL, NULL, ""),
    TPSLOT("__repr__", tp_repr, slot_tp_repr, wrap_unaryfunc,
           "__repr__($self, /)\n--\n\nReturn repr(self)."),
    TPSLOT("__hash__", tp_hash, slot_tp_hash, wrap_hashfunc,
           "__hash__($self, /)\n--\n\nReturn hash(self)."),
    // ... more slotdefs
}
```

此数组的每个条目都是一个slotdef结构：
```c
// typedef struct wrapperbase slotdef;

struct wrapperbase {
    const char *name;
    int offset;
    void *function;
    wrapperfunc wrapper;
    const char *doc;
    int flags;
    PyObject *name_strobj;
};
```

此结构的四个成员对于我们的讨论很重要：

- name是一种特殊方法的名称。
- offset是PyHeapTypeObject结构中插槽的偏移。它指定了与特殊方法对应的插槽。
- function是一个插槽的实现。当定义特殊方法时，相应的插槽设置到function。通常，function调用特殊方法来完成工作。
- wrapper是一个围绕插槽的装饰函数。当定义插槽时，wrapper为相应的特殊方法提供实施。它调用插槽来完成工作。

例如，下面是一个条目，该条目将特殊方法__add__()映射到插槽nb_add：

- name是"__add__"。
- offset是offsetof(PyHeapTypeObject, as_number.nb_add)。
- function是slot_nb_add()。
- wrapper是wrap_binaryfunc_l()。

该slotdefs数组是一个多到多的映射。例如，我们将看到，__add__()和__radd__()特殊方法都映射到同一个nb_add插槽。相反，mp_subscript"映射"插槽和sq_item"序列"插槽映射都采用相同的__getitem__()特殊方法。

CPython 以两种方式使用slotdefs数组：

- 根据特殊方法设置插槽;和
- 根据插槽设置特殊方法。

## 基于特殊方法的插槽
基于特殊方法，type_new()函数调用fixup_slot_dispatchers()设置插槽。该fixup_slot_dispatchers()函数为slotdefs数组中的每个插槽调用update_one_slot()，如果类具有相应的特殊方法，update_one_slot()将插槽设置为function。

让我们以插槽nb_add为例。该slotdefs数组有两个与该插槽对应的条目：
```c
static slotdef slotdefs[] = {
    // ...
    BINSLOT("__add__", nb_add, slot_nb_add, "+"),
    RBINSLOT("__radd__", nb_add, slot_nb_add,"+"),
    // ...
}
```

BINSLOT(),RBINSLOT()是宏。让我们扩展它们：
```c
static slotdef slotdefs[] = {
    // ...
    // {name, offset, function,
    //     wrapper, doc}
    // 
    {"__add__", offsetof(PyHeapTypeObject, as_number.nb_add), (void *)(slot_nb_add),
        wrap_binaryfunc_l, PyDoc_STR("__add__" "($self, value, /)\n--\n\nReturn self" "+" "value.")},

    {"__radd__", offsetof(PyHeapTypeObject, as_number.nb_add), (void *)(slot_nb_add),
        wrap_binaryfunc_r, PyDoc_STR("__radd__" "($self, value, /)\n--\n\nReturn value" "+" "self.")},
    // ...
}
```

`update_one_slot()`查找`class.__add__()`和`class.__radd__()`。如果其中任何一个被定义，它设置类的nb_add到slot_nb_add()。请注意，两个条目都同意slot_nb_add()为function 。否则，当两者都被定义时，我们就会发生冲突。

现在，什么是slot_nb_add()？此函数定义为宏，如下：
```c
static PyObject *
slot_nb_add(PyObject *self, PyObject *other) {
    PyObject* stack[2];
    PyThreadState *tstate = _PyThreadState_GET();
    _Py_static_string(op_id, "__add__");
    _Py_static_string(rop_id, "__radd__");
    int do_other = !Py_IS_TYPE(self, Py_TYPE(other)) && \
        Py_TYPE(other)->tp_as_number != NULL && \
        Py_TYPE(other)->tp_as_number->nb_add == slot_nb_add;
    if (Py_TYPE(self)->tp_as_number != NULL && \
        Py_TYPE(self)->tp_as_number->nb_add == slot_nb_add) {
        PyObject *r;
        if (do_other && PyType_IsSubtype(Py_TYPE(other), Py_TYPE(self))) {
            int ok = method_is_overloaded(self, other, &rop_id);
            if (ok < 0) {
                return NULL;
            }
            if (ok) {
                stack[0] = other;
                stack[1] = self;
                r = vectorcall_maybe(tstate, &rop_id, stack, 2);
                if (r != Py_NotImplemented)
                    return r;
                Py_DECREF(r); do_other = 0;
            }
        }
        stack[0] = self;
        stack[1] = other;
        r = vectorcall_maybe(tstate, &op_id, stack, 2);
        if (r != Py_NotImplemented || Py_IS_TYPE(other, Py_TYPE(self)))
            return r;
        Py_DECREF(r);
    }
    if (do_other) {
        stack[0] = other;
        stack[1] = self;
        return vectorcall_maybe(tstate, &rop_id, stack, 2);
    }
    Py_RETURN_NOTIMPLEMENTED;
}
```

您不需要仔细研究此代码。回想一下`binary_op1()`调用插槽`nb_add`。该函数`slot_nb_add()`基本上重复了binary_op1()。主要的区别是，`slot_nb_add()`最终调用`__add__()`或`__radd__()`。

## 在现有类上设置特殊方法

假设我们创建一个没有特殊方法`__add__()`,`__radd__()`的类。在这种情况下，类的插槽`nb_add`设置为`NULL`。不出所料，我们不能加该类的实例。但是，如果在创建类后我们设置`__add__()`或`__radd__()`，加法就好像该方法是类定义的一部分。我的意思是：
```shell
$ python -q
>>> class A:
...     pass
... 
>>> x = A()
>>> x + 2
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unsupported operand type(s) for +: 'A' and 'int'
>>> A.__add__ = lambda self, o: 5
>>> x + 2
5
>>> 
```

这是如何工作的？要在对象上设置属性，VM 调用对象类型的插槽tp_setattro。type的插槽tp_setattro指向type_setattro()函数，因此，当我们在类上设置属性时，此函数会被调用。要设置属性，它在类字典中存储属性的值。在此之后，它会检查属性是否是一种特殊方法。如果是特殊方法，则设置相应的插槽。同样的update_one_slot()函数也被要求这样做。

在了解 CPython 如何反向工作之前，即它如何为内置类型添加特殊方法，我们需要了解什么是方法。

## 方法
方法是一种属性，但却是一种奇特的属性。当我们从实例调用方法时，该方法会隐性地接收实例作为其第一个参数，我们通常用self表示：
```shell
$ python -q
>>> class A:
...     def method(self, x):
...             return self, x
...
>>> a = A()
>>> a.method(1)
(<__main__.A object at 0x10d10bfd0>, 1)
```

但是，当我们从class上调用相同的方法时，我们必须显式传递所有参数：
```python
>>> A.method(a, 1)
(<__main__.A object at 0x10d10bfd0>, 1)
```

在我们的示例中，该方法在一个案例中采用一个参数，在另一个案例中采用两个参数。根据我们访问方式的不同，同一属性怎么可能是不同的？

首先，要认识到我们在class上定义的方法只是一种函数。通过实例访问的函数与通过实例类型访问的同一函数不同，因为该function类型实现了[描述符协议，the descriptor protocol](https://docs.python.org/3/howto/descriptor.html#descriptor-protocol)。如果您不熟悉描述符，我强烈建议您阅读雷蒙德·海廷格的[Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html)。简言之，描述符是一个对象，当用作属性时，它自行决定您如何获取、设置和删除它。从技术上讲，描述符是实现`__get__()`，`__set__()`或`__delete__()`特殊方法的对象。 

function类型实现`__get__()`。当我们查找某些方法时，我们得到的是调用`__get__()`的结果。三个参数传递给它：

- 属性，比如函数
- 实例
- 实例的类型。

如果我们在类型上查找一种方法，则实例是`NULL`，并且`__get__()`只是返回函数。如果我们在实例上查找方法，`__get__()`返回方法对象：
```shell
>>> type(A.method)
<class 'function'>
>>> type(a.method)
<class 'method'>
```

方法对象存储函数和实例。调用时，它会将实例预先列在参数列表中，并调用函数。

现在，我们准备解决最后一个问题。

## 基于插槽的特殊方法

回顾初始化类型并进行插槽继承的函数`PyType_Ready()`。它还根据已实现的插槽向类型添加特殊方法。 `PyType_Ready()`调用`add_operators()`做到这一点。该`add_operators()`函数在slotdefs数组中的条目上迭代。对于每个条目，它检查是否应将条目指定的特殊方法添加到该类型的字典中。如果尚未定义特殊方法并且类型实现了条目指定的插槽，则添加特殊方法。例如，如果特殊方法`__add__()`不是在类型上定义的，而是该类型实现插槽`nb_add`，则`add_operators()`将`__add__()`放入该类型的字典中。

`__add__()`设置什么？与任何其他方法一样，必须将其设置为某种描述，才能像方法一样行事。虽然程序员定义的方法是函数，但由add_operators()设置的方法是装饰描述符。装饰描述符是存储两个东西的描述符：

- 它存储一个装饰插槽。装饰插槽为特殊方法设置装饰描述符。例如，将`float`类型的`__add__()`特殊方法的装饰描述符存储装饰槽`float_add()`。
- 它存储装饰函数。装饰函数"知道"如何调用装饰插槽。它是一个slotdef条目的wrapper。

当我们调用`add_operators()`添加的特殊方法时，我们调用装饰描述符。当我们调用装饰描述符时，它调用装饰函数。装饰描述符传递给装饰函数的参数为我们传递给特殊方法的参数加装饰插槽。最后，装饰函数调用装饰槽。

让我们看看实现插槽`nb_add`的内置类型如何获得其`__add__()`,`__radd__()`特殊方法。回忆`nb_add`相应的`slotdef`条目：
```c
static slotdef slotdefs[] = {
    // ...
    // {name, offset, function,
    //     wrapper, doc}
    // 
    {"__add__", offsetof(PyHeapTypeObject, as_number.nb_add), (void *)(slot_nb_add),
        wrap_binaryfunc_l, PyDoc_STR("__add__" "($self, value, /)\n--\n\nReturn self" "+" "value.")},

    {"__radd__", offsetof(PyHeapTypeObject, as_number.nb_add), (void *)(slot_nb_add),
        wrap_binaryfunc_r, PyDoc_STR("__radd__" "($self, value, /)\n--\n\nReturn value" "+" "self.")},
    // ...
}
```

如果类型实现插槽nb_add，则add_operators()将类型的__add__()设置为带有装饰函数wrap_binaryfunc_l()和装饰插槽nb_add的装饰说明符。它同样地设置类型的__radd__()，只有一个例外：装饰函数是wrap_binaryfunc_r()。

wrap_binaryfunc_l()和wrap_binaryfunc_r()两者都以两个参数加上一个装饰插槽作为参数。唯一的区别是他们如何调用插槽：

- wrap_binaryfunc_l(x, y, slot_func)调用slot_func(x, y)
- wrap_binaryfunc_r(x, y, slot_func)调用slot_func(y, x)。

此调用的结果是我们调用特殊方法时获得的结果。

# 总结

今天，我们已经揭开了Python最神奇的一面。我们了解到，Python 对象的行为是由对象类型的插槽决定的。静态定义类型的插槽可以明确指定，任何类型都可以从其祖先那里继承某些插槽。真正的见解是，一个类的插槽是由CPython根据定义的特殊方法自动设置的。CPython也做了相反的事——如果类型实现相应的插槽，则向类型字典添加特殊方法。

我们学到了很多东西。然而，Python对象系统是一个如此庞大的主题，至少还有同样多的问题有待覆盖。例如，我们还没有真正讨论属性的工作原理。这就是我们下次要做的。