# from

[source](https://tenthousandmeters.com/blog/python-behind-the-scenes-7-how-python-attributes-work/)

# pre

当我们获得或设置 Python 对象的属性时会发生什么情况？这个问题并不像一开始看起来那么简单。诚然，任何经验丰富的 Python 程序员都对属性的工作原理有良好的直观理解，文档有助于加强理解。然而，当一个关于属性的非常不平凡的问题出现时，直觉就会失败，文档就再也帮不上忙了。为了获得深刻的理解并能够回答这些问题，我们必须研究属性的实现方式。这就是我们今天要做的。

注意：在这篇文章中，我指的是CPython 3.9。随着 CPython 的发展，一些实现细节肯定会发生变化。我将尝试跟踪重要更改并添加更新说明。

# 快速复习

上次我们研究了Python对象系统的工作原理。我们在这方面学到的一些东西对于我们当前的讨论至关重要，所以让我们简要地回忆一下。

Python 对象是具有至少两个成员的 C 结构的实例：

- 参考计数;
- 对象类型的指针。

每个对象必须有一个类型，因为类型决定对象的行为方式。类型也是 Python 对象，一个PyTypeObject结构实例：
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

类型的成员称为插槽。每个插槽负责对象行为的特定方面。例如，类型的tp_call插槽指定了当我们调用类型的对象时会发生什么。有些插槽被组合在一起的套件。套件的一个例子是"number"套件tp_as_number。上次我们研究了它的nb_add插槽，其中指定了如何添加对象。这和所有其他插槽在[文档](https://docs.python.org/3/c-api/typeobj.html)中描述得很好。

如何设置类型插槽取决于如何定义类型。定义 CPython 中的类型有两种方法：

- 静态：或
- 动态。

静态定义的类型只是静态初始化PyTypeObject实例。所有内置类型均以静态方式定义。例如，以下是float类型的定义：
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

要动态分配一种新类型，我们称之为元类。元类是实例为类型的类型。它决定了类型的行为方式。特别是，它创建了新的类型实例。Python有一个内置的元型，称为type。它是所有内置类型的元类。它也被用作创建类的默认元类。当 CPython 执行class语句时，它通常会调用type()以创建类。我们也可以通过直接调用`type()`创建一个类：
```python
MyClass = type(name, bases, namespace)
```

调用type的tp_new插槽来创建类。此插槽的实现是type_new()函数。此函数分配类型对象并设置它。

静态定义类型的插槽显式指定。类的插槽由元类自动设置。静态和动态定义的类型都可以从其基类中继承一些插槽。

某些插槽映射为特殊方法。如果一个类定义了与某个插槽对应的特殊方法，CPython 会自动将插槽设置为调用特殊方法的默认实现。这就是为什么我们可以加定义了`__add__()`的类对象。CPython 为静态定义的类型做反向工作。如果此类类型实现了与某些特殊方法相对应的插槽，则 CPython 将特殊方法设置为装饰插槽的实现。这就是int类型如何获得其特殊方法__add__()。

所有类型都必须通过调用PyType_Ready()函数进行初始化。此函数可以做很多事情。例如，它进行插槽继承，并根据插槽添加特殊方法。对于一个类，PyType_Ready()由type_new()调用。对于静态定义的类型，PyType_Ready()必须显式调用。当 CPython 启动时，它为每个内置类型调用PyType_Ready()。

考虑到这一点，让我们把注意力转向属性。

# 属性和虚拟机

什么是属性？我们可能会说属性是与对象关联的变量，但它不止于此。很难给出一个能够概括属性所有重要方面的定义。因此，让我们从我们肯定知道的事情开始，而不是从定义开始。

我们确信，在 Python 中，我们可以用属性做三件事：

- 获取属性的值：value = obj.attr
- 设置某些值的属性：obj.attr = value
- 删除属性：del obj.attr

这些操作做什么取决于对象的类型，就像对象行为的任何其他方面一样。类型具有某些插槽，负责获取、设置和删除属性。VM 调用这些插槽来执行类似`value = obj.attr`和`obj.attr = value`。要了解 VM 是如何执行的，以及这些插槽是什么，让我们应用熟悉的方法：

1. 编写获取/设置/删除属性的代码。
2. 使用dis模块将其拆解到字节码。
3. 看看在ceval.c中生成的字节码指令的实现情况。

## 获取属性

让我们先看看VM在获得属性值时会怎么做。编译器生成LOAD_ATTRopcode以加载值：
```shell
$ echo 'obj.attr' | python -m dis
  1           0 LOAD_NAME                0 (obj)
              2 LOAD_ATTR                1 (attr)
...
```

VM 执行此opcode如下：
```c
case TARGET(LOAD_ATTR): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *owner = TOP();
    PyObject *res = PyObject_GetAttr(owner, name);
    Py_DECREF(owner);
    SET_TOP(res);
    if (res == NULL)
        goto error;
    DISPATCH();
}
```

我们可以看到，VM调用PyObject_GetAttr()函数来完成这项工作。此函数的作用如下：
```c
PyObject *
PyObject_GetAttr(PyObject *v, PyObject *name)
{
    PyTypeObject *tp = Py_TYPE(v);

    if (!PyUnicode_Check(name)) {
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     Py_TYPE(name)->tp_name);
        return NULL;
    }
    if (tp->tp_getattro != NULL)
        return (*tp->tp_getattro)(v, name);
    if (tp->tp_getattr != NULL) {
        const char *name_str = PyUnicode_AsUTF8(name);
        if (name_str == NULL)
            return NULL;
        return (*tp->tp_getattr)(v, (char *)name_str);
    }
    PyErr_Format(PyExc_AttributeError,
                 "'%.50s' object has no attribute '%U'",
                 tp->tp_name, name);
    return NULL;
}
```

它首先尝试调用对象类型的tp_getattro插槽。如果此插槽未实现，它将尝试调用插槽tp_getattr。如果未实现tp_getattr，它提出AttributeError。

支持属性访问的类型实现tp_getattro或tp_getattr或两者兼有之。根据[文档](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_getattr)，它们之间的唯一区别是tp_getattro将 Python 字符串作为属性的名称，tp_getattr采用 C 字符串。虽然选择存在，你不会找到在CPython实现tp_getattr的类型，因为为了赞成tp_getattro它已被弃用。

## 设置属性

从 VM 的角度来看，设置属性与获取属性没有太大区别。编译器生成[STORE_ATTR](https://docs.python.org/3/library/dis.html#opcode-STORE_ATTR)opcode，以设置某些值的属性：
```shell
$ echo 'obj.attr = value' | python -m dis
  1           0 LOAD_NAME                0 (value)
              2 LOAD_NAME                1 (obj)
              4 STORE_ATTR               2 (attr)
...
```

VM 执行STORE_ATTR如下：
```c
case TARGET(STORE_ATTR): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *owner = TOP();
    PyObject *v = SECOND();
    int err;
    STACK_SHRINK(2);
    err = PyObject_SetAttr(owner, name, v);
    Py_DECREF(v);
    Py_DECREF(owner);
    if (err != 0)
        goto error;
    DISPATCH();
}
```

我们发现，做这项工作的函数是PyObject_SetAttr()：
```c
int
PyObject_SetAttr(PyObject *v, PyObject *name, PyObject *value)
{
    PyTypeObject *tp = Py_TYPE(v);
    int err;

    if (!PyUnicode_Check(name)) {
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     Py_TYPE(name)->tp_name);
        return -1;
    }
    Py_INCREF(name);

    PyUnicode_InternInPlace(&name);
    if (tp->tp_setattro != NULL) {
        err = (*tp->tp_setattro)(v, name, value);
        Py_DECREF(name);
        return err;
    }
    if (tp->tp_setattr != NULL) {
        const char *name_str = PyUnicode_AsUTF8(name);
        if (name_str == NULL) {
            Py_DECREF(name);
            return -1;
        }
        err = (*tp->tp_setattr)(v, (char *)name_str, value);
        Py_DECREF(name);
        return err;
    }
    Py_DECREF(name);
    _PyObject_ASSERT(name, Py_REFCNT(name) >= 1);
    if (tp->tp_getattr == NULL && tp->tp_getattro == NULL)
        PyErr_Format(PyExc_TypeError,
                     "'%.100s' object has no attributes "
                     "(%s .%U)",
                     tp->tp_name,
                     value==NULL ? "del" : "assign to",
                     name);
    else
        PyErr_Format(PyExc_TypeError,
                     "'%.100s' object has only read-only attributes "
                     "(%s .%U)",
                     tp->tp_name,
                     value==NULL ? "del" : "assign to",
                     name);
    return -1;
}
```

此函数调用tp_setattro和tp_setattr插槽的方式与PyObject_GetAttr()调用tp_getattro和tp_getattr的方式相同。插槽tp_setattro配对tp_getattro，tp_setattr配对tp_getattr。就像tp_getattr，tp_setattr被弃用了。

请注意，PyObject_SetAttr()检查类型是否定义tp_getattro或tp_getattr。类型必须实现支持属性分配的属性访问。

## 删除属性

有趣的是，类型没有用于删除属性的特殊插槽。然后，什么指定如何删除属性？我看看。编译器生成[DELETE_ATTR](https://docs.python.org/3/library/dis.html#opcode-DELETE_ATTR)opcode以删除属性：
```shell
$ echo 'del obj.attr' | python -m dis
  1           0 LOAD_NAME                0 (obj)
              2 DELETE_ATTR              1 (attr)
```              

VM 执行此opcode的方式揭示了答案：
```c
case TARGET(DELETE_ATTR): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *owner = POP();
    int err;
    err = PyObject_SetAttr(owner, name, (PyObject *)NULL);
    Py_DECREF(owner);
    if (err != 0)
        goto error;
    DISPATCH();
}
```

要删除属性，VM 调用其调用与设置属性相同函数PyObject_SetAttr()，因此同一插槽tp_setattro负责删除属性。但是，它如何知道要执行的两个操作中哪一个操作？NULL值表示属性应被删除。

如本节所示，tp_getattro和tp_setattro插槽决定对象属性的工作原理。接下来想到的问题是：这些插槽是如何实现的？

## 插槽实现

适当签名的任何函数都可以是实现tp_getattro和tp_setattro。类型可以以绝对任意的方式实现这些插槽。幸运的是，我们只需要研究几个实现，以了解 Python 属性的工作原理。这是因为大多数类型使用相同的通用实现。

获取和设置属性的通用函数是PyObject_GenericGetAttr()和PyObject_GenericSetAttr()。默认情况下，所有类都使用它们。大多数内置类型指定插槽为显式实现或从使用通用实现的object继承它们。

在这篇文章中，我们将专注于通用的实现，因为它基本上是我们所说的Python属性。在不使用通用实现时，我们还将讨论两个重要案例。第一个案例是type.它以自己的方式实现tp_getattro和tp_setattro插槽，尽管其实现方式与一般实现非常相似。第二种情况是任何通过定义`__getattribute__()`、`__getattr__()`、`__setattr__()`和`__delattr__()`特殊方法自定义属性访问和分配的类。CPython 将此类类的tp_getattro和tp_setattro插槽设置为调用这些方法的函数。

## 通用属性管理

PyObject_GenericGetAttr()和PyObject_GenericSetAttr()函数实现我们都习惯的属性的行为。当我们将对象的属性设置为某些值时，CPython 将值放入对象的字典中：
```shell
$ python -q
>>> class A:
...     pass
... 
>>> a = A()
>>> a.__dict__
{}
>>> a.x = 'instance attribute'
>>> a.__dict__
{'x': 'instance attribute'}
```

当我们尝试获取属性的值时，CPython 会从对象的字典中加载它：
```shell
>>> a.x
'instance attribute'
```

如果对象字典不包含属性，CPython 会加载类型字典中的值：
```shell
>>> A.y = 'class attribute'
>>> a.y
'class attribute'
```

如果类型的字典也不包含属性，CPython 会搜索类型父母的字典中的价值：
```shell
>>> class B(A): # note the inheritance
...     pass
... 
>>> b = B()
>>> b.y
'class attribute'
```

因此，对象的属性是两者之一：

- 实例变量;或
- 类型变量。

实例变量存储在对象的字典中，类型变量存储在类型的字典和类型父母的字典中。要设置某个值的属性，CPython 只需更新对象的字典。为了获得属性的价值，CPython 首先在对象的字典中搜索它，然后在类型的字典和类型父母的字典中搜索它。CPython 在搜索值时对类型进行重复的顺序是方法解决顺序（MRO）。

如果没有描述符，Python 属性将如此简单。

## 描述符

从技术上讲，描述符是 Python 对象，其类型实现某些插槽tp_descr_get或tp_descr_set或两者兼而有之。从本质上讲，描述符是 Python 对象，当用作属性时，它会控制我们获取、设置或删除它所发生的情况。如果PyObject_GenericGetAttr()发现属性值是其类型实现tp_descr_get的描述符，它不仅会按正常方式返回值，还会调用tp_descr_get并返回此调用的结果。tp_descr_get插槽需要三个参数：描述符本身、被查找属性的对象和对象的类型。它由tp_descr_get决定如何处理参数和返回什么。同样，PyObject_GenericSetAttr()查找当前属性值。如果它发现值是其类型实现tp_descr_set的描述符，它调用tp_descr_set代替只更新对象的字典。传递给tp_descr_set的参数是描述符、对象和新的属性值。要删除属性，PyObject_GenericSetAttr()调用tp_descr_set将新属性值设置为NULL。

一方面，描述符使 Python 属性变得有点复杂。另一方面，描述符使 Python 属性变得强大。正如Python的术语表[所说](https://docs.python.org/3/glossary.html#term-descriptor)，

- 理解描述符是深入理解 Python 的关键，因为它们是许多函数的基础，包括函数、方法、属性、类方法、静态方法和超级类的引用。

让我们修改我们在上一部分讨论的描述符的一个重要使用案例：方法。

放入类型字典中的函数不是像普通函数，而是像一种方法。即，当我们调用时，我们不需要显示传递第一个参数：
```shell
>>> A.f = lambda self: self
>>> a.f()
<__main__.A object at 0x108a20d60>
```

属性a.f不仅像一种方法一样工作，而且是一种方法：
```shell
>>> a.f
<bound method <lambda> of <__main__.A object at 0x108a20d60>>
```

但是，如果我们查找类型字典中的价值，我们将获得原始函数：'f'
```shell
>>> A.__dict__['f']
<function <lambda> at 0x108a4ca60> 
```

CPython返回的不是字典中存储的价值，而是其他东西。这是因为函数是描述符。function类型实现插槽`tp_descr_get`，因此`PyObject_GenericGetAttr()`调用此插槽并返回调用结果。调用的结果是存储函数和实例的方法对象。当我们调用一个方法对象时，实例被预先依赖到参数列表中，并且函数被调用。

描述符只有在用作类型变量时才会具有其特殊行为。当它们被用作实例变量时，它们的行为就像普通对象一样。例如，放置在对象字典中的函数不会成为一种方法：
```shell
>>> a.g = lambda self: self
>>> a.g
<function <lambda> at 0x108a4cc10>
```

显然，语言设计人员在没有发现将描述符用作实例变量的好案例。这一决定的一个很好的结果是实例变量非常简单。它们只是数据。

function类型是内置描述符类型的示例。我们还可以定义我们自己的描述符。为此，我们创建了一个实现`__get__()`,`__set__()`,`__delete__()`描述符协议的类：
```python
>>> class DescrClass:
...     def __get__(self, obj, type=None):
...             print('I can do anything')
...             return self
...
>>> A.descr_attr = DescrClass()
>>> a.descr_attr 
I can do anything
<__main__.DescrClass object at 0x108b458e0>
```

如果一个类定义`__get__()`，CPython 将其插槽`tp_descr_get`设置为调用此方法的函数。如果一个类定义`__set__()`或`__delete__()`， CPython 插槽`tp_descr_set`设置:当值是`NULL`调用的函数`__delete__()`，其他调用函数`__set__()`。

如果你想知道为什么有人会想要在首位定义他们的描述，看看[Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html#id1)。

我们的目标是研究获取和设置属性的实际算法。描述符是其中的一个先决条件。另一个是了解对象的字典和类型的字典到底是什么。

# 对象字典和类型字典
对象字典是存储实例变量的字典。每个类型的对象都保留一个指向自己的字典的指针。例如，每个函数对象都有func_dict成员：
```c
typedef struct {
    // ...
    PyObject *func_dict;        /* The __dict__ attribute, a dict or NULL */
    // ...
} PyFunctionObject;
```

要告诉 CPython 对象的哪个成员是对象字典的指针，使用插槽tp_dictoffset的偏移指定对象的类型。function类型如下：
```c
PyTypeObject PyFunction_Type = {
    // ...
    offsetof(PyFunctionObject, func_dict),      /* tp_dictoffset */
    // ... 
};
```

`tp_dictoffset`的正值指从对象结构开始偏移，负值指从结构末端的偏移。零偏移意味着类型的对象没有字典。例如，整数是此类对象：
```python
>>> (12).__dict__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'int' object has no attribute '__dict__'
```

我们可以通过检查`__dictoffset__`属性来确认类型`int`的`tp_dictoffset`设置为0：
```python
>>> int.__dictoffset__
0
```

类通常有一个非零`tp_dictoffset`。唯一的例外是定义`__slots__`属性的类。此属性是一个优化。我们将首先讨论要点，然后再讨论`__slots__`。

类型字典是类型对象的字典。就像函数的`func_dict`成员指向函数的字典一样，类型`tp_dict`插槽指向类型的字典。普通对象字典和类型词典之间的关键区别是CPython知道`tp_dict`，所以它可以避免通过`tp_dictoffset`定位类型字典。以一般方式处理一种类型的字典将引入额外的间接层，不会带来太大的好处。

现在，当我们知道描述符是什么以及属性存储在哪里时，我们准备查看函数`PyObject_GenericGetAttr()`和函数`PyObject_GenericSetAttr()`的作用。

## PyObject_GenericSetAttr()

我们首先从PyObject_GenericSetAttr()函数开始，其工作为设置属性值。此函数原来是关于其他函数的装饰器：
```c
int
PyObject_GenericSetAttr(PyObject *obj, PyObject *name, PyObject *value)
{
    return _PyObject_GenericSetAttrWithDict(obj, name, value, NULL);
}
```

而这个函数实际上做的工作：
```c
int
_PyObject_GenericSetAttrWithDict(PyObject *obj, PyObject *name,
                                 PyObject *value, PyObject *dict)
{
    PyTypeObject *tp = Py_TYPE(obj);
    PyObject *descr;
    descrsetfunc f;
    PyObject **dictptr;
    int res = -1;

    if (!PyUnicode_Check(name)){
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     Py_TYPE(name)->tp_name);
        return -1;
    }

    if (tp->tp_dict == NULL && PyType_Ready(tp) < 0)
        return -1;

    Py_INCREF(name);

    // Look up the current attribute value
    // in the type's dict and in the parent's dicts using the MRO.
    descr = _PyType_Lookup(tp, name);

    // If found a descriptor that implements `tp_descr_set`, call this slot.
    if (descr != NULL) {
        Py_INCREF(descr);
        f = Py_TYPE(descr)->tp_descr_set;
        if (f != NULL) {
            res = f(descr, obj, value);
            goto done;
        }
    }

    // `PyObject_GenericSetAttr()` calls us with `dict` set to `NULL`.
    // So, `if` will be executed.
    if (dict == NULL) {
        // Get the object's dict.
        dictptr = _PyObject_GetDictPtr(obj);
        if (dictptr == NULL) {
            if (descr == NULL) {
                PyErr_Format(PyExc_AttributeError,
                             "'%.100s' object has no attribute '%U'",
                             tp->tp_name, name);
            }
            else {
                PyErr_Format(PyExc_AttributeError,
                             "'%.50s' object attribute '%U' is read-only",
                             tp->tp_name, name);
            }
            goto done;
        }
        // Update the object's dict with the new value.
        // If `value` is `NULL`, delete the attribute from the dict.
        res = _PyObjectDict_SetItem(tp, dictptr, name, value);
    }
    else {
        Py_INCREF(dict);
        if (value == NULL)
            res = PyDict_DelItem(dict, name);
        else
            res = PyDict_SetItem(dict, name, value);
        Py_DECREF(dict);
    }
    if (res < 0 && PyErr_ExceptionMatches(PyExc_KeyError))
        PyErr_SetObject(PyExc_AttributeError, name);

  done:
    Py_XDECREF(descr);
    Py_DECREF(name);
    return res;
}
```

尽管函数很长，函数实现了一个简单的算法：

1. 在类型变量中搜索属性值。搜索的顺序是 MRO。
2. 如果值是其类型实现插槽`tp_descr_set`的描述符，调用插槽。
3. 否则，使用新值更新对象字典。

我们尚未讨论实现插槽`tp_descr_set`的描述符类型，因此您可能想知道为什么我们需要它们。考虑 Python 的[`property()`](https://docs.python.org/3/library/functions.html#property)。文档中的以下示例演示了其创建管理属性的规范用途：
```python
class C:
    def __init__(self):
        self._x = None

    def getx(self):
        return self._x

    def setx(self, value):
        self._x = value

    def delx(self):
        del self._x

    x = property(getx, setx, delx, "I'm the 'x' property.")
```

如果c是C的实例，`c.x`将调用获取器，`c.x = value`将调用设置器和`del c.x`将调用删除器。

`property()`工作原理如何？答案很简单：它是一种描述符类型。它同时实现调用指定函数的插槽tp_descr_get和插槽tp_descr_set。

文档中的示例只是一个框架，没有做太多工作。但是，它可以很容易地扩展，以做一些有用的事情。例如，我们可以编写一个设置器，执行对新属性值的某些验证。

## PyObject_GenericGetAttr()

获取属性的值比设置属性要复杂一些。让我们看看有多少。函数PyObject_GenericGetAttr()还将工作委托给另一个函数：
```c
PyObject *
PyObject_GenericGetAttr(PyObject *obj, PyObject *name)
{
    return _PyObject_GenericGetAttrWithDict(obj, name, NULL, 0);
}
```

函数的作用如下：
```c
PyObject *
_PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name,
                                 PyObject *dict, int suppress)
{
    /* Make sure the logic of _PyObject_GetMethod is in sync with
       this method.

       When suppress=1, this function suppress AttributeError.
    */

    PyTypeObject *tp = Py_TYPE(obj);
    PyObject *descr = NULL;
    PyObject *res = NULL;
    descrgetfunc f;
    Py_ssize_t dictoffset;
    PyObject **dictptr;

    if (!PyUnicode_Check(name)){
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     Py_TYPE(name)->tp_name);
        return NULL;
    }
    Py_INCREF(name);

    if (tp->tp_dict == NULL) {
        if (PyType_Ready(tp) < 0)
            goto done;
    }

    // Look up the attribute value
    // in the type's dict and in the parent's dicts using the MRO.
    descr = _PyType_Lookup(tp, name);

    // Check if the value is a descriptor that implements:
    // * `tp_descr_get`; and
    // * `tp_descr_set` (data descriptor)
    // In this case, call `tp_descr_get`
    f = NULL;
    if (descr != NULL) {
        Py_INCREF(descr);
        f = Py_TYPE(descr)->tp_descr_get;
        if (f != NULL && PyDescr_IsData(descr)) {
            res = f(descr, obj, (PyObject *)Py_TYPE(obj));
            if (res == NULL && suppress &&
                    PyErr_ExceptionMatches(PyExc_AttributeError)) {
                PyErr_Clear();
            }
            goto done;
        }
    }

    // Look up the attribute value in the object's dict
    // Return if found one
    if (dict == NULL) {
        /* Inline _PyObject_GetDictPtr */
        dictoffset = tp->tp_dictoffset;
        if (dictoffset != 0) {
            if (dictoffset < 0) {
                Py_ssize_t tsize = Py_SIZE(obj);
                if (tsize < 0) {
                    tsize = -tsize;
                }
                size_t size = _PyObject_VAR_SIZE(tp, tsize);
                _PyObject_ASSERT(obj, size <= PY_SSIZE_T_MAX);

                dictoffset += (Py_ssize_t)size;
                _PyObject_ASSERT(obj, dictoffset > 0);
                _PyObject_ASSERT(obj, dictoffset % SIZEOF_VOID_P == 0);
            }
            dictptr = (PyObject **) ((char *)obj + dictoffset);
            dict = *dictptr;
        }
    }
    if (dict != NULL) {
        Py_INCREF(dict);
        res = PyDict_GetItemWithError(dict, name);
        if (res != NULL) {
            Py_INCREF(res);
            Py_DECREF(dict);
            goto done;
        }
        else {
            Py_DECREF(dict);
            if (PyErr_Occurred()) {
                if (suppress && PyErr_ExceptionMatches(PyExc_AttributeError)) {
                    PyErr_Clear();
                }
                else {
                    goto done;
                }
            }
        }
    }

    // If _PyType_Lookup found a non-data desciptor,
    // call its `tp_descr_get`
    if (f != NULL) {
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        if (res == NULL && suppress &&
                PyErr_ExceptionMatches(PyExc_AttributeError)) {
            PyErr_Clear();
        }
        goto done;
    }

    // If _PyType_Lookup found some value,
    // return it
    if (descr != NULL) {
        res = descr;
        descr = NULL;
        goto done;
    }

    if (!suppress) {
        PyErr_Format(PyExc_AttributeError,
                     "'%.50s' object has no attribute '%U'",
                     tp->tp_name, name);
    }
  done:
    Py_XDECREF(descr);
    Py_DECREF(name);
    return res;
}
```

此算法的主要步骤是：

1. 在类型变量中搜索属性值。搜索的顺序是 MRO。
2. 如果值是其类型实现tp_descr_get插槽的数据描述符，调用此插槽并返回调用结果。否则，记住值并继续。数据描述符是其类型实现tp_descr_set插槽的描述符。
3. 使用tp_dictoffset定位对象字典。如果字典包含值，则返回它。
4. 如果步骤 2 的值是其类型实现tp_descr_get插槽的描述符，调用此插槽并返回调用结果。
5. 从第 2 步返回值。值可以是`NULL`。

由于属性可以是实例变量和类型变量，CPython 必须决定哪个属性优先于另一个属性。算法所做的本质上是实现某种优先顺序。此顺序是：

1. 类型数据描述符
2. 实例变量
3. 类型非数据描述符和其他类型变量。

自然要问的问题是：为什么它要执行这个特殊的顺序？更具体地说，为什么数据描述符优先于实例变量，而非数据描述符不优先于实例变量？首先，请注意，某些描述符必须优先于实例变量，以便属性能够按预期工作。这样描述符的一个示例是对象的`__dict__`属性。您不会在对象的字典中找到它，因为它是存储在类型字典中的数据描述符：
```python
>>> a.__dict__['__dict__']
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: '__dict__'
>>> A.__dict__['__dict__']
<attribute '__dict__' of 'A' objects>
>>> a.__dict__ is A.__dict__['__dict__'].__get__(a)
True
```

此描述符的`tp_descr_get`插槽返回位于`tp_dictoffset`的对象字典。现在假设数据描述符不优先于实例变量。如果我们把`'__dict__'`放在对象的字典里，并分配给它一些其他字典会发生什么：
```python
>>> a.__dict__['__dict__'] = {}
```

`a.__dict__`属性返回的不是对象的字典，而是我们分配的字典！对于依赖`__dict__`的对象来说，这是完全出乎意料的。幸运的是，数据描述符确实优先于实例变量，因此我们得到了对象的字典：
```python
>>> a.__dict__
{'x': 'instance attribute', 'g': <function <lambda> at 0x108a4cc10>, '__dict__': {}}
```

非数据描述符不优先于实例变量，因此大多数时间实例变量比类型变量具有优先级。当然，现有的优先顺序是众多设计选择之一。吉多·范·罗森在[PEP 252](https://www.python.org/dev/peps/pep-0252/)中解释了其背后的原因：

- 在较复杂的案例中，实例字典中存储的名称与类型字典中存储的名称之间存在冲突。如果两个字典都有相同的条目，我们应返回哪一个？看着经典的 Python 指导， 我发现相互矛盾的规则： 对于类实例， 实例字典覆盖类字典，除了特殊属性 （如`__dict__`和`__class__`）， 它们优先于实例字典。

- 我通过以下一组规则解决了这个问题，其实现于`PyObject_GenericGetAttr()`：...

为什么`__dict__`属性首先作为描述符实现？使其成为实例变量将导致相同的问题。有可能覆盖`__dict__`属性，几乎没有人希望有这种可能性。

我们已经了解了普通对象的属性是如何工作的。现在让我们看看类型属性的工作原理。

## 元类属性管理

基本上，类型属性的工作方式与普通对象的属性类似。当我们将类型属性设置为某个值时，CPython 将值放入类型的字典中：
```python
>>> B.x = 'class attribute'
>>> B.__dict__
mappingproxy({'__module__': '__main__', '__doc__': None, 'x': 'class attribute'})
```

当我们获得属性值时，CPython 会从类型的字典中加载它：
```python
>>> B.x
'class attribute'
```

如果类型的字典不包含属性，CPython 会加载元类字典中的值：
```python
>>> B.__class__
<class 'type'>
>>> B.__class__ is object.__class__
True
```

最后，如果元字典也不包含属性，CPython在元型父母的字典中搜索值。

与通用实现类推是显式的。我们只是用"类型"来代替"对象"、"元类"来代替"类型"。但是，`type`以自己的方式实现`tp_getattro`和`tp_setattro`插槽。为什么？让我们来看看代码。

## type_setattro()
我们从函数type_setattro()开始，插槽tp_setattro的实现：
```c
static int
type_setattro(PyTypeObject *type, PyObject *name, PyObject *value)
{
    int res;
    if (!(type->tp_flags & Py_TPFLAGS_HEAPTYPE)) {
        PyErr_Format(
            PyExc_TypeError,
            "can't set attributes of built-in/extension type '%s'",
            type->tp_name);
        return -1;
    }
    if (PyUnicode_Check(name)) {
        if (PyUnicode_CheckExact(name)) {
            if (PyUnicode_READY(name) == -1)
                return -1;
            Py_INCREF(name);
        }
        else {
            name = _PyUnicode_Copy(name);
            if (name == NULL)
                return -1;
        }
        // ... ifdef
    }
    else {
        /* Will fail in _PyObject_GenericSetAttrWithDict. */
        Py_INCREF(name);
    }

    // Call the generic set function.
    res = _PyObject_GenericSetAttrWithDict((PyObject *)type, name, value, NULL);
    if (res == 0) {
        PyType_Modified(type);

        // If attribute is a special method,
        // add update the corresponding slots.
        if (is_dunder_name(name)) {
            res = update_slot(type, name);
        }
        assert(_PyType_CheckConsistency(type));
    }
    Py_DECREF(name);
    return res;
}
```

此函数调用通用`_PyObject_GenericSetAttrWithDict()`来设置属性值，但它也做其他事情。首先，它确保类型不是静态定义的类型，因为此类类型的设计是不可变的：
```python
>>> int.x = 2
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can't set attributes of built-in/extension type 'int'
```

它还检查属性是否是一种特殊方法。如果属性是一种特殊方法，它会更新与该特殊方法对应的插槽。例如，如果我们在现有类中定义特殊方法`__add__()`，它将将类的`nb_add`插槽设置为调用该方法的默认实现。由于这种机制，一个类的特殊方法和插槽保持同步。

## type_getattro()

该`type_getattro()`函数是插槽`tp_getattro`的实现，不称为通用函数，但类似于它：
```c
/* This is similar to PyObject_GenericGetAttr(),
   but uses _PyType_Lookup() instead of just looking in type->tp_dict. */
static PyObject *
type_getattro(PyTypeObject *type, PyObject *name)
{
    PyTypeObject *metatype = Py_TYPE(type);
    PyObject *meta_attribute, *attribute;
    descrgetfunc meta_get;
    PyObject* res;

    if (!PyUnicode_Check(name)) {
        PyErr_Format(PyExc_TypeError,
                     "attribute name must be string, not '%.200s'",
                     Py_TYPE(name)->tp_name);
        return NULL;
    }

    /* Initialize this type (we'll assume the metatype is initialized) */
    if (type->tp_dict == NULL) {
        if (PyType_Ready(type) < 0)
            return NULL;
    }

    /* No readable descriptor found yet */
    meta_get = NULL;

    /* Look for the attribute in the metatype */
    meta_attribute = _PyType_Lookup(metatype, name);

    if (meta_attribute != NULL) {
        Py_INCREF(meta_attribute);
        meta_get = Py_TYPE(meta_attribute)->tp_descr_get;

        if (meta_get != NULL && PyDescr_IsData(meta_attribute)) {
            /* Data descriptors implement tp_descr_set to intercept
             * writes. Assume the attribute is not overridden in
             * type's tp_dict (and bases): call the descriptor now.
             */
            res = meta_get(meta_attribute, (PyObject *)type,
                           (PyObject *)metatype);
            Py_DECREF(meta_attribute);
            return res;
        }
    }

    /* No data descriptor found on metatype. Look in tp_dict of this
     * type and its bases */
    attribute = _PyType_Lookup(type, name);
    if (attribute != NULL) {
        /* Implement descriptor functionality, if any */
        Py_INCREF(attribute);
        descrgetfunc local_get = Py_TYPE(attribute)->tp_descr_get;

        Py_XDECREF(meta_attribute);

        if (local_get != NULL) {
            /* NULL 2nd argument indicates the descriptor was
             * found on the target object itself (or a base)  */
            res = local_get(attribute, (PyObject *)NULL,
                            (PyObject *)type);
            Py_DECREF(attribute);
            return res;
        }

        return attribute;
    }

    /* No attribute found in local __dict__ (or bases): use the
     * descriptor from the metatype, if any */
    if (meta_get != NULL) {
        PyObject *res;
        res = meta_get(meta_attribute, (PyObject *)type,
                       (PyObject *)metatype);
        Py_DECREF(meta_attribute);
        return res;
    }

    /* If an ordinary attribute was found on the metatype, return it now */
    if (meta_attribute != NULL) {
        return meta_attribute;
    }

    /* Give up */
    PyErr_Format(PyExc_AttributeError,
                 "type object '%.50s' has no attribute '%U'",
                 type->tp_name, name);
    return NULL;
}
```

此算法确实重复了通用实现的逻辑，但有三个重要区别：

- 它通过`tp_dict`获取类型字典。通用实现尝试使用元型的`tp_dictoffset`来定位它。
- 它不仅在类型字典中搜索类型变量，而且在类型父母的字典中搜索类型变量。通用实现将处理一种类型，如没有继承概念的普通对象。
- 它支持类型描述符。通用实现将仅支持元类描述符。

因此，我们有以下优先顺序：

1. 元类数据描述符
2. 类型描述符和其他类型变量
3. 元类非数据描述符和其他元型变量。

这就是`type`实现`tp_getattro`和`tp_setattro`插槽的方式。由于`type`默认情况下是所有内置类型的元类和所有类别的元类，大多数类型的属性根据此实现工作。正如我们已经说过的，类本身默认使用通用实现。如果我们想要更改类实例属性的行为或类属性的行为，我们需要定义使用自定义实现的新类或新元类。Python 提供了一个简单的方法来做到这一点。

## 自定义属性管理

类的`tp_getattro`和`tp_setattro`插槽最初由创建新类的`type_new()`函数设置。通用实现是其默认选择。类可以通过定义`__getattribute__()`、`__getattr__()`、`__setattr__()`和`__delattr__()`特殊方法来自定义属性访问、分配和删除。当一个类定义`__setattr__()`或`__delattr__()`，其`tp_setattro`插槽被设置为`slot_tp_setattro()`函数。当一个类定义`__getattribute__()`或`__getattr__()`，其插槽`tp_getattro`被设置为`slot_tp_getattr_hook()`函数。

`__setattr__()`,`__delattr__()`特殊方法非常简单。基本上，它们允许我们在 Python 中实现`tp_setattro`插槽。函数`slot_tp_setattro()`只是调用`__delattr__(instance, attr_name)`或`__setattr__(instance, attr_name, value)`取决于`value`是否是`NULL`：
```c
static int
slot_tp_setattro(PyObject *self, PyObject *name, PyObject *value)
{
    PyObject *stack[3];
    PyObject *res;
    _Py_IDENTIFIER(__delattr__);
    _Py_IDENTIFIER(__setattr__);

    stack[0] = self;
    stack[1] = name;
    if (value == NULL) {
        res = vectorcall_method(&PyId___delattr__, stack, 2);
    }
    else {
        stack[2] = value;
        res = vectorcall_method(&PyId___setattr__, stack, 3);
    }
    if (res == NULL)
        return -1;
    Py_DECREF(res);
    return 0;
}
```

`__getattribute__()`,`__getattr__()`特殊方法提供了自定义属性访问的方法。两者都以实例和属性名称作为参数并返回属性值。它们之间的区别在于它们被调用时。

获取属性值时，特殊方法`__getattribute__()`是`__setattr__()`和`__delattr__()`的模拟。它代替通用函数被调用。`__getattr__()`特殊方法与`__getattribute__()`或通用函数同时使用。当`__getattribute__()`或通用函数抛出`AttributeError`，它被调用。此逻辑在函数`slot_tp_getattr_hook()`中实现：
```c
static PyObject *
slot_tp_getattr_hook(PyObject *self, PyObject *name)
{
    PyTypeObject *tp = Py_TYPE(self);
    PyObject *getattr, *getattribute, *res;
    _Py_IDENTIFIER(__getattr__);

    getattr = _PyType_LookupId(tp, &PyId___getattr__);
    if (getattr == NULL) {
        /* No __getattr__ hook: use a simpler dispatcher */
        tp->tp_getattro = slot_tp_getattro;
        return slot_tp_getattro(self, name);
    }
    Py_INCREF(getattr);

    getattribute = _PyType_LookupId(tp, &PyId___getattribute__);
    if (getattribute == NULL ||
        (Py_IS_TYPE(getattribute, &PyWrapperDescr_Type) &&
         ((PyWrapperDescrObject *)getattribute)->d_wrapped ==
         (void *)PyObject_GenericGetAttr))
        res = PyObject_GenericGetAttr(self, name);
    else {
        Py_INCREF(getattribute);
        res = call_attribute(self, getattribute, name);
        Py_DECREF(getattribute);
    }
    if (res == NULL && PyErr_ExceptionMatches(PyExc_AttributeError)) {
        PyErr_Clear();
        res = call_attribute(self, getattr, name);
    }
    Py_DECREF(getattr);
    return res;
}
```

让我们将代码翻译成英语：

1. 如果类没有定义`__getattr__()`，请首先将其`tp_getattro`插槽设置为其他函数`slot_tp_getattro()`，然后调用此函数并返回调用的结果。
2. 如果类定义`__getattribute__()`，调用它。否则调用通用`PyObject_GenericGetAttr()`。
3. 如果从前一步的调用抛出AttributeError，调用`__getattr__()`。
4. 返回调用的结果。

函数`slot_tp_getattro()`是 CPython 在类中定义`__getattribute__()`时使用的`tp_getattro`插槽的实现，而不是`__getattr__()`。此函数仅调用`__getattribute__()`：
```c
static PyObject *
slot_tp_getattro(PyObject *self, PyObject *name)
{
    PyObject *stack[2] = {self, name};
    return vectorcall_method(&PyId___getattribute__, stack, 2);
}
```

为什么CPython不将`tp_getattro`插槽设置为函数`slot_tp_getattro()`代替最初的`slot_tp_getattr_hook()`？原因是设计了将特殊方法映射到插槽的机制。机制需要映射到同一插槽的特殊方法，以便为此插槽提供相同的实现。`__getattribute__()`和`__getattr__()`特殊方法映射到同一`tp_getattro`插槽。

即使对`__getattribute__()`,`__getattr__()`特殊方法的工作方式有完美的理解，也无法告诉我们为什么我们需要这两种方法。从理论上讲，`__getattribute__()`应足以完成任何我们想要的属性访问工作。有时，虽然，定义`__getattr__()`是更方便的定义。例如，标准[imaplib](https://docs.python.org/3/library/imaplib.html#module-imaplib)模块提供了可用于与 IMAP4 服务器通话的`IMAP4`类。要发出命令，我们调用类方法。命令的大小写版本都有效：
```python
>>> from imaplib import IMAP4_SSL # subclass of IMAP4
>>> M = IMAP4_SSL("imap.gmail.com", port=993)
>>> M.noop()
('OK', [b'Nothing Accomplished. p11mb154389070lti'])
>>> M.NOOP()
('OK', [b'Nothing Accomplished. p11mb154389070lti'])
```

为了支持此功能，`IMAP4`定义`__getattr__()`：
```python
class IMAP4:
    # ...

    def __getattr__(self, attr):
        #       Allow UPPERCASE variants of IMAP4 command methods.
        if attr in Commands:
            return getattr(self, attr.lower())
        raise AttributeError("Unknown IMAP4 command: '%s'" % attr)

    # ...
```

`__getattribute__()`实现相同的结果首先要求我们显式调用通用函数：`object.__getattribute__(self, attr)` 。这是否对引入另一种特殊方法不方便？也许。为什么`__getattribute__()`,`__getattr__()`两者兼而有之的真正原因是历史问题。`__getattribute__()`特殊方法在[Python 2.2](https://docs.python.org/3/whatsnew/2.2.html#attribute-access)中`__getattr__()`已经存在时被引入。以下是吉多·范·罗森对新函数的需求的[解释](https://www.python.org/download/releases/2.2/descrintro/)：

- `__getattr__()`方法并不是真正实现获取属性操作的方法：它是一个钩子，只有在无法通过正常手段找到属性时才会被调用。这经常被引用为一个缺点——一些类设计需要一个合法的获取属性的方法，此方法被所有属性引用所调用，这个问题现在通过提供`__getattribute__()`解决。

当我们获得或设置 Python 对象的属性时会发生什么情况？我想我们对这个问题给出了详细的答案。然而，答案并不包括Python属性的一些重要方面。我们也来讨论一下。

# 加载方法

我们看到函数对象是将方法对象绑定到实例时返回方法对象的描述符：
```python
>>> a.f
<bound method <lambda> of <__main__.A object at 0x108a20d60>>
```

但是，如果我们需要调用方法，是否真的有必要创建一个方法对象呢？CPython不能将实例中的原始函数作为第一个参数吗？可以。事实上，这正是CPython所做的。

当编译器看到类似`obj.method(arg1,...,argN)`带有位置参数等的方法调用时，它不会生成`LOAD_ATTR` opcode来加载方法和[`CALL_FUNCTION`](https://docs.python.org/3/library/dis.html#opcode-CALL_FUNCTION)opcode来调用方法。相反，它会生成一对[`LOAD_METHOD`](https://docs.python.org/3/library/dis.html#opcode-LOAD_METHOD)和[`CALL_METHOD`](https://docs.python.org/3/library/dis.html#opcode-CALL_METHOD)opcode：
```shell
$ echo 'obj.method()' | python -m dis
  1           0 LOAD_NAME                0 (obj)
              2 LOAD_METHOD              1 (method)
              4 CALL_METHOD              0
...
```

当 VM 执行`LOAD_METHOD`opcode时，它会调用函数`_PyObject_GetMethod()`来搜索属性值。此函数的工作原理与通用函数类似。唯一的区别是，它检查值是否是未绑定的方法，比如一种能够返回与实例绑定的方法对象的描述符。在这种情况下，它不调用描述符类型的插槽`tp_descr_get`，而是返回描述符本身。例如，如果属性值是一个函数，`_PyObject_GetMethod()`则返回函数。function和其他描述符类型，其对象表现为未绑定方法，在他们的tp_flags中指定[Py_TPFLAGS_METHOD_DESCRIPTOR](https://www.python.org/dev/peps/pep-0590/#descriptor-behavior)标志，因此很容易识别它们。

需要注意的是，只有当对象的类型使用`tp_getattro`的通用实现时，`_PyObject_GetMethod()`才能按照上述描述运行。否则，它只是调用自定义实现，不执行任何检查。

如果`_PyObject_GetMethod()`发现未绑定的方法，则调用此方法时其实例必须在参数列表中。如果它发现其他不受限于实例可调用的方法，则参数列表必须保持不变。因此，在VM 执行LOAD_METHOD 后，堆栈上的值可以以两种方式之一排列：

- 未绑定的方法和包括实例在内的参数列表：(method | self | arg1 | ... | argN)
- 其他可调用方法和没有实例的参数列表(NULL | method | arg1 | ... | argN)

CALL_METHOD opcode的存在是为了在每种情况下调用适当方法。

要了解有关此优化的更多信息，请查看源自此优化的[问题](https://bugs.python.org/issue26110)。

# 列出对象的属性

Python 提供内置的[dir()](https://docs.python.org/3/library/functions.html#dir)函数，可用于查看对象具有的属性。您是否想知道此函数是如何找到属性的？它通过调用对象类型的`__dir__()`特殊方法来实现。类型很少定义自己的`__dir__()`，但所有类型都有它。这是因为`object`类型定义`__dir__()`，所有其他类型继承`object`。`object`提供的实现列出对象字典、类型字典和类型父母字典中存储的所有属性。因此，`dir()`有效地返回普通对象的所有属性。但是，当我们在类型上调用`dir()`时，我们并不能得到其所有属性。这是因为`type`提供了自己的`__dir__()`实现。此实现返回存储在类型字典和类型家长的字典中的属性。然而，它忽略了元类字典和元类父母的字典中存储的属性。[文档](https://docs.python.org/3/library/functions.html#dir)解释了为什么会这样：

- 由于`dir()`提供主要是为了方便在交互式提示处使用，它试图提供一组有趣的名称，而不是试图提供严格或一致定义的一组名称，其详细行为可能会因版本而变化。例如，当参数为类时，元类属性不在结果列表中。

# 类型属性来自何处

对于任何内置类型，列出其属性。你会得到：
```python
>>> dir(object)
['__class__', '__delattr__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
>>> dir(int)
['__abs__', '__add__', '__and__', '__bool__', '__ceil__', '__class__', '__delattr__', '__dir__', '__divmod__', '__doc__', '__eq__', '__float__', '__floor__', '__floordiv__', '__format__', '__ge__', '__getattribute__', '__getnewargs__', '__gt__', '__hash__', '__index__', '__init__', '__init_subclass__', '__int__', '__invert__', '__le__', '__lshift__', '__lt__', '__mod__', '__mul__', '__ne__', '__neg__', '__new__', '__or__', '__pos__', '__pow__', '__radd__', '__rand__', '__rdivmod__', '__reduce__', '__reduce_ex__', '__repr__', '__rfloordiv__', '__rlshift__', '__rmod__', '__rmul__', '__ror__', '__round__', '__rpow__', '__rrshift__', '__rshift__', '__rsub__', '__rtruediv__', '__rxor__', '__setattr__', '__sizeof__', '__str__', '__sub__', '__subclasshook__', '__truediv__', '__trunc__', '__xor__', 'as_integer_ratio', 'bit_length', 'conjugate', 'denominator', 'from_bytes', 'imag', 'numerator', 'real', 'to_bytes']
```

我们上次看到，与插槽对应的特殊方法会通过初始化类型的`PyType_Ready()`函数自动添加。但是，其余的属性从何而来呢？它们都必须以某种方式指定，然后在某个时候设置为某些东西。这是一个模糊的声明。让我们说清楚。

指定类型属性的最简单方法是创建一个新的字典，用属性填充它，并将类型的tp_dict设置为此字典。在内置类型被定义之前，我们不能这样做，因此内置类型的tp_dict被初始化为NULL。事实证明，PyType_Ready()函数在运行时创建内置类型的字典，还负责添加所有属性。

首先，PyType_Ready()确保类型具有字典。然后，它将属性添加到字典中。类型通过指定[tp_methods](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_methods)、[tp_members](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_members)和[tp_getset](https://docs.python.org/3/c-api/typeobj.html#c.PyTypeObject.tp_getset)插槽来表示PyType_Ready()要添加的属性。每个插槽都是描述不同属性的结构数组。

## tp_methods
插槽tp_methods是描述方法的[PyMethoddef](https://docs.python.org/3/c-api/structures.html#c.PyMethodDef)结构的数组：
```c
struct PyMethodDef {
    const char  *ml_name;   /* The name of the built-in function/method */
    PyCFunction ml_meth;    /* The C function that implements it */
    int         ml_flags;   /* Combination of METH_xxx flags, which mostly
                               describe the args expected by the C func */
    const char  *ml_doc;    /* The __doc__ attribute, or NULL */
};
typedef struct PyMethodDef PyMethodDef;
```

`ml_meth`成员是实现方法的 C 函数的指针。它的签名可以是众多之一。`ml_flags`位用于告诉 CPython 如何准确地调用函数。

对于`tp_methods`中的每个结构，`PyType_Ready()`在类型的字典中添加一个可调用对象。此对象封装`PyMethodDef` 结构。当我们调用它时，`ml_meth`指向的函数被调用。这是C函数变成为Python类型的方法的原理。

例如，object类型使用此机制定义`__dir__()`和一堆其他方法：
```c
static PyMethodDef object_methods[] = {
    {"__reduce_ex__", (PyCFunction)object___reduce_ex__, METH_O, object___reduce_ex____doc__},
    {"__reduce__", (PyCFunction)object___reduce__, METH_NOARGS, object___reduce____doc__},
    {"__subclasshook__", object_subclasshook, METH_CLASS | METH_VARARGS,
     object_subclasshook_doc},
    {"__init_subclass__", object_init_subclass, METH_CLASS | METH_NOARGS,
     object_init_subclass_doc},
    {"__format__", (PyCFunction)object___format__, METH_O, object___format____doc__},
    {"__sizeof__", (PyCFunction)object___sizeof__, METH_NOARGS, object___sizeof____doc__},
    {"__dir__", (PyCFunction)object___dir__, METH_NOARGS, object___dir____doc__},
    {0}
};
```

添加到字典中的可调用对象通常是一个方法描述符。我们也许应用另一篇文章讨论 Python 可调用物中的方法描述符是什么，其本质上是一个行为像函数对象的对象，即它与实例结合的对象。主要区别在于，与实例绑定的函数返回一个方法对象，一个与实例绑定的方法描述符返回一个内置方法对象。方法对象封装 Python 函数和实例，内置方法对象封装 C 函数和实例。

例如，`object.__dir__`是一个方法描述符：
```python
>>> object.__dir__
<method '__dir__' of 'object' objects>
>>> type(object.__dir__)
<class 'method_descriptor'>
```

如果我们绑定`__dir__`到实例，我们会获得内置方法对象：
```python
>>> object().__dir__
<built-in method __dir__ of object object at 0x1088cc420>
>>> type(object().__dir__)
<class 'builtin_function_or_method'>
```

如果`ml_flags`标志指定方法是静态的，则立即在字典中添加内置方法对象，而不是方法描述符。

任何内置类型的每个方法要么包装一些插槽，要么根据`tp_methods`被添加到字典。

## tp_members

`tp_members`插槽是[PyMemberDef](https://docs.python.org/3/c-api/structures.html#c.PyMemberDef)结构的数组。每个结构描述一个属性，此属性暴露了一个类型对象的 C 成员：
```c
typedef struct PyMemberDef {
    const char *name;
    int type;
    Py_ssize_t offset;
    int flags;
    const char *doc;
} PyMemberDef;
```

成员由 offset指定。其类型由type指定。

对于tp_members的每个结构，PyType_Ready()在类型字典中添加一个成员描述符。成员描述符是一个封装PyMemberDef的数据描述符。其`tp_descr_get`插槽拿到实例，找到位于offset的实例的成员，将其转换为相应的 Python 对象并返回对象。其tp_descr_set插槽拿到实例和值，找到位于offset实例成员，并将其设置为对应值。成员只能通过指定flags设置只读。

例如，通过这一机制，type定义了`__dictoffset__`和其他成员：
```c
static PyMemberDef type_members[] = {
    {"__basicsize__", T_PYSSIZET, offsetof(PyTypeObject,tp_basicsize),READONLY},
    {"__itemsize__", T_PYSSIZET, offsetof(PyTypeObject, tp_itemsize), READONLY},
    {"__flags__", T_ULONG, offsetof(PyTypeObject, tp_flags), READONLY},
    {"__weakrefoffset__", T_PYSSIZET,
     offsetof(PyTypeObject, tp_weaklistoffset), READONLY},
    {"__base__", T_OBJECT, offsetof(PyTypeObject, tp_base), READONLY},
    {"__dictoffset__", T_PYSSIZET,
     offsetof(PyTypeObject, tp_dictoffset), READONLY},
    {"__mro__", T_OBJECT, offsetof(PyTypeObject, tp_mro), READONLY},
    {0}
};
```

## tp_getset

插槽tp_getset是PyGetSetDef结构的数组，此结构可以像property()一样描述任意数据描述符：
```c
typedef struct PyGetSetDef {
    const char *name;
    getter get;
    setter set;
    const char *doc;
    void *closure;
} PyGetSetDef;
```

对于tp_getset的每个结构，PyType_Ready()在类型的字典中添加一个getset描述符。getset描述符的tp_descr_get插槽调用指定的get函数，tp_descr_set插槽调用指定的set函数。

类型使用此机制定义`__dict__`属性。例如，`function`类型是如何处理的：
```c
static PyGetSetDef func_getsetlist[] = {
    {"__code__", (getter)func_get_code, (setter)func_set_code},
    {"__defaults__", (getter)func_get_defaults,
     (setter)func_set_defaults},
    {"__kwdefaults__", (getter)func_get_kwdefaults,
     (setter)func_set_kwdefaults},
    {"__annotations__", (getter)func_get_annotations,
     (setter)func_set_annotations},
    {"__dict__", PyObject_GenericGetDict, PyObject_GenericSetDict},
    {"__name__", (getter)func_get_name, (setter)func_set_name},
    {"__qualname__", (getter)func_get_qualname, (setter)func_set_qualname},
    {NULL} /* Sentinel */
};
```

`__dict__`属性不是作为仅读成员描述符实现的，而是作为 geteset 描述符实现的，因为它不仅仅是返回位于tp_dictoffset的字典。例如，如果字典尚不存在，则描述符会创建字典。

类也通过此机制获得`__dict__`属性。创建类的type_new()函数在调用`PyType_Ready()`之前指定tp_getset。然而，有些类没有得到这个属性，因为他们的实例没有字典。这些是定义`__slots__`的类。

## `__slots__`
类的`__slots__`属性列举了类可以具有的属性：
```python
>>> class D:
...     __slots__ = ('x', 'y')
...
```

如果一个类定义`__slots__`，`__dict__`属性不添加到类的字典和类`tp_dictoffset`设置为0。其主要效果是类实例没有字典：
```python
>>> D.__dictoffset__
0
>>> d = D()
>>> d.__dict__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'D' object has no attribute '__dict__'
```

However, the attributes listed in `__slots__` work fine:
```python
>>> d.x = 4
>>> d.x
4
```

这怎么可能？`__slots__`所列属性成为类实例的成员。对于每个成员，成员描述符将添加到类字典中。`type_new()`函数指定`tp_members`这样做。
```python
>>> D.x
<member 'x' of 'D' objects>
```

由于实例没有字典，`__slots__`属性节省内存。根据[Descriptor HowTo Guide](https://docs.python.org/3/howto/descriptor.html#member-objects-and-slots)，

- 在 64 位 Linux 中，带有`__slots__`的具有两个属性的实例需要 48 字节，而没有需要 152 字节。

指南还列出了使用`__slots__`的其他好处。我建议你去看看。

# 总结

编译器生成`LOAD_ATTR`,`STORE_ATTR`,`DELETE_ATTR`的opcode去获取、设置和删除属性。要执行这些opcode，VM 会调用对象类型的tp_getattro和tp_setattro插槽。一种类型可能会以任意方式实现这些插槽，但大多数情况下，我们必须处理三个实现：

1. 大多数内置类型和类使用的通用实现
2. type所使用的实现
3. 定义`__getattribute__()`,`__getattr__()`,`__setattr__()`、和`__delattr__()`特殊方法的类所使用的实现。

一旦您了解了什么是描述符，通用的实现就简单了。简而言之，描述符是能够控制属性访问、分配和删除的属性。它们允许 CPython 实现许多特性，包括方法和属性。

内置类型使用三种机制定义属性：

- tp_methods
- tp_members;和
- tp_getset.

类还使用这些机制来定义某些属性。例如，`__dict__`定义为getset 描述符，`__slots__`列出的属性被定义为成员描述符。