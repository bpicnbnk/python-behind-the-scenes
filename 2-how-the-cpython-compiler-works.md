# from

[source](https://tenthousandmeters.com/blog/python-behind-the-scenes-2-how-the-cpython-compiler-works/)
# 准备

## 终结符 Terminal symbols

终结符是一个形式语言的基本符号。就是说，它们能在一个形式语法的推导规则的输入或输出字符串存在，而且它们不能被分解成更小的单位。确切地说，一个语法的规则不能改变终结符。例如说，下面的语法有两个规则：
$x \rightarrow xa$
$x \rightarrow ax$
在这种语法之中，a是一个终结符，因为没有规则可以把a变成别的符号。不过，有两个规则可以把x变成别的符号，所以x是非终结符。一个形式语法所推导的形式语言必须完全由终结符构成。

## 非终结符 Nonterminal symbols

非终结符是可以被取代的符号。一个形式文法中必须有一个起始符号；这个起始符号属于非终结符的集合。

在上下文无关文法中，每个推导规则的左边只能有一个非终结符而不能有两个以上的非终结符或终结符。并非所有的语言都可以被上下文无关文法产生。

## 推导规则 Production rules

一种语法的定义由推导规则构成。每个规则规定什么词位（英语：lexeme）可以重写为什么别的词位。这些规则可以用来剖析字符串，也可以用来产生字符串。每个规则有左边和右边。左边有可以被取代的字符串，而右边有可以取代左边的字符串。规则的写法一般为左边 $\rightarrow$ 右边。比如，$z0 \rightarrow z1$  这个规则规定 z0 可以重写为 z1。左边为一个非终结符，但是右边不一定是个终结符。

## 例子

下面的形式文法代表一个整数。整数可能是有符号，就是说，可能是负数。下面使用[巴科斯范式](https://zh.wikipedia.org/wiki/%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F)的变种来表示：
```
<integer> ::= ['-'] <digit> {<digit>}
<digit> ::= '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
```
在这个例子之中，符号 (-,0,1,2,3,4,5,6,7,8,9) 都是终结符，而 `<digit>` 和 `<integer>` 都是非终结符。

# 主题

在系列的第一篇文章中，我们查看了CPython VM。我们了解到，它通过执行一系列称为"字节码"的指令来工作。我们还看到，Python 的字节码不足以完全描述一段代码。这就是为什么存在代码对象的概念。执行代码块（block ，如模块或函数）意味着执行相应的代码对象。代码对象包含 block 的字节码、常数和 block 内使用的变量名称以及 block 的各种属性。

通常，Python 程序员不会编写字节码，也不会创建代码对象，而是编写正常的 Python 代码。因此，CPython 必须能够从源代码创建代码对象。此工作由 CPython 编译器完成。在这部分，我们将探索它是如何工作的。

注意：在这篇文章中，我指的是CPython 3.9。随着 CPython 的发展，一些实施细节肯定会发生变化。我将尝试跟踪重要更改并添加更新说明。

# 什么是CPython编译器

我们了解 CPython 编译器的责任是什么，但在查看如何实施之前，让我们先弄清楚为什么我们首先将其称为编译器。

一般意义上的编译器是将一种语言中的程序转换为另一种语言的等效程序的程序。编译器类型多种多，但大多数情况下，编译器是指静态编译器，它以高级语言将程序转换为机器代码。CPython 编译器是否与此类类型的编译器有共同点？要回答这个问题，让我们来看看静态编译器的传统三阶段设计。

![](pic/2/the%20traditional%20three-stage%20design%20of%20a%20static%20compiler.png)

编译器的前端将源代码转换为某些中间表示 （intermediate representation, IR）。然后，优化器将 IR 进行优化，并将优化的 IR 传递到生成机器代码的后端。如果我们选择不特定于任何源语言和任何目标机器的 IR，那么我们获得三阶段设计的关键优势：对于支持新源语言的编译器，只需要额外的前端，并且要支持新的目标机器，只需要一个额外的后端。

LLVM 工具链是该模型成功的一个很好的例子。C、Rust、Swift 和许多其他编程语言的正面依赖于 LLVM 来提供编译器中更复杂的部分。LLVM的创建者Chris Lattner对其[结构进行了很好的概述](http://aosabook.org/en/llvm.html)。

然而，CPython不需要支持多种源语言和目标计算机，只需要一个Python代码和CPython VM。然而，CPython编译器是三阶段设计的实现。为了了解原因，我们应该更详细地研究三阶段编译器的阶段。
![](pic/2/the%20stages%20of%20a%20three-stage%20compiler.png)

上图代表经典编译器的模型。现在将其与下图中的 CPython 编译器的架构进行比较。

![](pic/2/the%20architecture%20of%20the%20CPython%20compiler.png)

看起来很像，不是吗？这里的要点是，CPython编译器的结构应该为以前研究过编译器的人所熟悉。如果没有，一本著名的[龙书](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)就是对编译器建设理论的极好的介绍。它很长，但即使只读前几章，你也会受益匪浅。

我们所做的比较需要几个注解。首先，由于版本 3.9，CPython 默认使用新的解析器，该解析器可以立即输出 AST（抽象语法树），而无需构建解析树的中间步骤。因此，CPython 编译器的模型进一步简化。其次，与静态编译器的对应部分相比，CPython 编译器的某些呈现阶段做得很少，有些人可能会说，CPython 编译器只不过是一个前端。我们不会对铁杆编译器编写者采取这种观点。

# 编译器架构概述

图表很好，但它们隐藏了许多细节，可能会产生误导，因此让我们花一些时间讨论 CPython 编译器的整体设计。

CPython 编译器的两个主要组件是：

    前端
    后端

前端将 Python 代码生成 AST。后端将 AST 生成代码对象。在整个 CPython 源代码中，术语解析器（ the terms parser ）和编译器分别用于前端和后端。这是单词编译器（the word compiler）的另一个含义。最好称它为代码对象生成器，但我们会坚持使用编译器，因为它似乎不会造成太大的麻烦。

解析器(parser)的工作是检查输入是否是句法上正确的 Python 代码。如果不是，则解析器会报告以下错误：
```
x = y = = 12
        ^
SyntaxError: invalid syntax
```
如果输入正确，则解析器会根据语法规则组织输入。语法定义语言的语法。正式语法的概念对于我们的讨论至关重要，我认为，我们应该稍微偏离一点，去回忆它的正式定义。

根据经典定义，语法由四个条目组成的大块：

    Σ–终结符的有限集，或简单的终结符（通常用小写字母表示）。
    N–非终结符的有限集，或简单的非终结符（通常用大写字母表示）。
    P–推导规则集合。在上下文无关语法（包括 Python 语法）中，推导规则只是从非终端到任何终端和非终端序列的映射，如A→aB.
    S–一个著名的非终结符。

语法grammar 定义了一种语言，这种语言包括所有可以通过应用推导规则生成的终结符序列。要生成某些序列，必须从符号S开始，然后根据推导规则递归地将每个非终端替换为序列，直到整个序列由终端组成。使用既定的约定进行符号，只需列出推导规则来指定语法就足够了。例如，这里有一个简单的语法，生成交替的一和零的序列：

    S→10S | 10 

当我们更详细地研究解析器时，我们将继续讨论语法。

# 抽象语法树 Abstract syntax tree

解析器的最终目标是生成 AST。AST 是一种树数据结构，可作为源代码的高层表示。下面是标准ast模块生成的代码和相应的 AST 转储的示例：
```python
x = 123
f(x)
```
```
$ python -m ast example1.py
Module(
   body=[
      Assign(
         targets=[
            Name(id='x', ctx=Store())],
         value=Constant(value=123)),
      Expr(
         value=Call(
            func=Name(id='f', ctx=Load()),
            args=[
               Name(id='x', ctx=Load())],
            keywords=[]))],
   type_ignores=[])
```
AST 节点的类型使用[the Zephyr Abstract Syntax Definition Language](https://www.cs.princeton.edu/research/techreps/TR-554-97) (ASDL) 正式定义。ASDL 是一种简单的声明性语言，旨在描述树状的 IR，这就是 AST。[Parser/Python.asdl](https://github.com/python/cpython/blob/master/Parser/Python.asdl) 中节点Assign,Expr的定义：
```
stmt = ... | Assign(expr* targets, expr value, string? type_comment) | ...
expr = ... | Call(expr func, expr* args, keyword* keywords) | ...
```
ASDL 规范让我们了解 Python AST 的形式。但是，解析器需要利用 C 代码表示 AST。幸运的是，很容易利用 ASDL 规范生成描述 AST 节点的 C 结构。这就是 CPython 所做的，结果看起来像这样：
```c
struct _stmt {
    enum _stmt_kind kind;
    union {
        // ... other kinds of statements
        struct {
            asdl_seq *targets;
            expr_ty value;
            string type_comment;
        } Assign;
        // ... other kinds of statements
    } v;
    int lineno;
    int col_offset;
    int end_lineno;
    int end_col_offset;
};

struct _expr {
    enum _expr_kind kind;
    union {
        // ... other kinds of expressions
        struct {
            expr_ty func;
            asdl_seq *args;
            asdl_seq *keywords;
        } Call;
        // ... other kinds of expressions
    } v;
    // ... same as in _stmt
};
```
AST 是一个方便的表示方式。它告诉程序做什么，隐藏所有非必要的信息，如缩进，标点符号和其他Python的句法特征。

AST 表示的主要受益者之一是编译器，编译器可以走 AST，并以相对简单的方式生成字节码。许多 Python 工具，除了编译器外，还使用 AST 与 Python 代码配合使用。例如，pytest对 AST 进行更改，以在 `assert` 语句失败时提供有用的信息，而`assert`只能在表达式为 False 时抛出异常 `AssertionError` 。另一个例子是[Bandit](https://github.com/PyCQA/bandit)，通过分析 AST 发现 Python 代码中的常见安全问题。

现在，当我们对 Python AST 进行了一点研究时，我们可以看看解析器是如何从源代码构建它的。

# 从源代码到 AST

事实上，正如我前面提到的，从3.9版开始，CPython有两个解析器。默认情况下使用新解析器。也可以通过传递选项 `-X oldparser` 使用旧解析器。但是，在 CPython 3.10 中，旧解析器将被完全删除。

这两个解析器非常不同。我们将专注于新的，但在此之前，讨论旧的解析器。

# 老解析器

长期以来，Python的语法是由生成式语法正式定义的。这是我们之前讨论过的一种语法。它告诉我们如何生成属于语言的序列。问题是生成式语法并不直接对应于能够解析这些序列的解析算法。幸运的是，人们已经能够区分生成式语法的不同类别，并建立相应的解析器。其中包括 context free, LL(k), LR(k), LALR 和许多其他类型的语法。 python语法为LL(1）。它使用一种 Extended Backus–Naur Form (EBNF) 进行指定。要了解如何使用它来描述 Python 的语法，请查看同时声明的规则。
```
file_input: (NEWLINE | stmt)* ENDMARKER
stmt: simple_stmt | compound_stmt
compound_stmt: ... | while_stmt | ...
while_stmt: 'while' namedexpr_test ':' suite ['else' ':' suite]
suite: simple_stmt | NEWLINE INDENT stmt+ DEDENT
...
```
CPython 扩展了传统符号，其功能包括：

    grouping of alternatives: (a | b)
    optional parts（可选部分）: [a]
    zero or more and one or more repetitions: a* and a+.

我们可以看到为什么吉多·范·罗森选择使用常规表达式（[why Guido van Rossum chose to use regular expressions](https://www.blogger.com/profile/12821714508588242516)）。它们允许以更自然的方式（对于程序员）表达编程语言的语法。Instead of writing A→aA|a , we can just write A→a+.

这一选择随之而来的成本：CPython必须开发一种方法来支持扩展的符号。

分析LL(1)语法是一个已解决的问题。解决方案是下推式自动机（Pushdown Automaton (PDA)），充当自上而下的解析器。PDA 通过使用堆栈（stack）模拟输入字符串的生成来运行。要解析某些输入，它从堆栈上的开始符号开始。然后，它查看输入中的第一个符号，猜测应将哪个规则应用于起始符号，并将其替换为该规则的右侧。如果堆栈上的顶部符号是与输入中的下一个符号匹配的终端，则 PDA 会弹出它并跳过匹配的符号。如果顶部符号是非终端，PDA 会尝试根据输入中的下一个符号来猜测规则以替换它。该过程重复，直到扫描整个输入，或者 PDA 无法将堆栈上的终端与输入中的下一个符号匹配。后一种情况意味着无法解析输入字符串。

由于推导规则的编写方式，CPython无法直接使用此方法，因此必须开发新方法。为了支持扩展的符号，旧解析器代表语法中的每一个规则，带有一个决定性的有限自动机（DFA），它以相当于常规表达而闻名。解析器本身是一个基于堆栈的自动机，如PDA，但它没有推动堆栈上的符号，而是推动DFA的状态。以下是旧解析器使用的关键数据结构：
```c
typedef struct {
    int              s_state;       /* State in current DFA */
    const dfa       *s_dfa;         /* Current DFA */
    struct _node    *s_parent;      /* Where to add next node */
} stackentry;

typedef struct {
    stackentry      *s_top;         /* Top entry */
    stackentry       s_base[MAXSTACK];/* Array of stack entries */
                                    /* NB The stack grows down */
} stack;

typedef struct {
    stack           p_stack;        /* Stack of parser states */
    grammar         *p_grammar;     /* Grammar to use */
                                    // basically, a collection of DFAs
    node            *p_tree;        /* Top of parse tree */
    // ...
} parser_state;
```
[Parser/parser.c](https://github.com/python/cpython/blob/3.9/Parser/parser.c) 中的注释总结了方法：

解析规则表示为决定性有限状态自动机（Deterministic Finite-state Automaton (DFA)）。DFA 中的节点表示解析器的状态（state）：弧线表示过渡（transition）。过渡要么用终结符标记，要么用非终结符标记。当解析器决定follow非终结符标记的弧线时，会反复调用 DFA，将解析规则表示为其初始状态：当DFA递归停止（when that DFA accepts）时，引用它的解析器将继续。由解析器的递归解析构建的子树插入当前解析树中。

解析器在解析输入的同时，构建了一个解析树，也称为具体语法树 （Concrete Syntax Tree，CST）。与 AST 相比，解析树直接对应于导入时应用的规则。解析树中的所有节点都使用相同的`node`结构表示：
```c
typedef struct _node {
    short               n_type;
    char                *n_str;
    int                 n_lineno;
    int                 n_col_offset;
    int                 n_nchildren;
    struct _node        *n_child;
    int                 n_end_lineno;
    int                 n_end_col_offset;
} node;
```
然而，解析树并不是编译器所需要的。它必须转换为 AST。这项工作是在[Python/ast.c](https://github.com/python/cpython/blob/3.9/Python/ast.c)完成的。该算法是递归地走一个解析树，并将其节点转换为 AST 节点。几乎没有人觉得这近 6，000 行代码令人兴奋。

# tokenizer
从语法的角度来看，Python不是一个简单的语言。Python语法，虽然看起来简单，包括评论约200行。这是因为语法符号是token，而不是单个字符。token由类型表示（如NUMBER,NAME,NEWLINE,值和源代码中的位置）。CPython 区分了 63 种token类型，所有都列在[Grammar/Tokens](https://github.com/python/cpython/blob/3.9/Grammar/Tokens)中。我们可以使用标准模块`tokenize`看到token化程序是什么样子的：
```python
def x_plus(x):
    if x >= 0:
        return x
    return 0
```
```
$ python -m tokenize example2.py 
0,0-0,0:            ENCODING       'utf-8'        
1,0-1,3:            NAME           'def'          
1,4-1,10:           NAME           'x_plus'       
1,10-1,11:          OP             '('            
1,11-1,12:          NAME           'x'            
1,12-1,13:          OP             ')'            
1,13-1,14:          OP             ':'            
1,14-1,15:          NEWLINE        '\n'           
2,0-2,4:            INDENT         '    '         
2,4-2,6:            NAME           'if'           
2,7-2,8:            NAME           'x'            
2,9-2,11:           OP             '>='           
2,12-2,13:          NUMBER         '0'            
2,13-2,14:          OP             ':'            
2,14-2,15:          NEWLINE        '\n'           
3,0-3,8:            INDENT         '        '     
3,8-3,14:           NAME           'return'       
3,15-3,16:          NAME           'x'            
3,16-3,17:          NEWLINE        '\n'           
4,4-4,4:            DEDENT         ''             
4,4-4,10:           NAME           'return'       
4,11-4,12:          NUMBER         '0'            
4,12-4,13:          NEWLINE        '\n'           
5,0-5,0:            DEDENT         ''             
5,0-5,0:            ENDMARKER      ''     
```

对于解析器来说程序就是这样的。当 parser 解析器需要一个token时，它会要求tokenizer发出一个token。tokenizer 一次从缓冲区读取一个字符，并尝试将前缀与某种类型的token匹配。在不同的编码的情况下，tokenizer如何工作？它依赖于`io`模块。首先，tokenizer 检测编码方式。如果没有指定编码，则会默认为 UTF-8。然后，tokenizer 利用c函数调用打开一个文件，这相当于Python的`open(fd, mode='r', encoding=enc)`，并通过调用函数`readline()`来读取其内容。此功能返回unicode字符串。tokenizer 读取器读取的字符只是该字节的 UTF-8 表示。

我们可以直接在语法中定义一个数字或一个名字，尽管它会变得更加复杂。在上下文有关语法中(CSG，英语：context-sensitive grammar)缩进很重要，有利于解析。tokenizer通过提供`INDENT` 和 `DEDENT` token使解析器的工作更容易。它们相当于C语言中的大括号。tokenizer是强大的，足以处理缩进，因为它有状态。当前缩进级别保持在堆栈顶部。当级别增加时，压栈。如果级别降低，则从堆栈中弹出所有较高级别。

旧的解析器是 CPython 代码库中一个非平凡的部分。语法规则的 DFA 是自动生成的，但解析器的其他部分是手工编写的。这与新的解析器形成鲜明对比，后者似乎是解析 Python 代码问题的更优雅的解决方案。

# 新解析器

新的解析器附带了新的语法。此语法是解析表达语法（Parsing Expression Grammar，PEG）。重要的是要理解PEG不仅仅是一个语法类别。这是定义语法的另一种方式。2004 年，Bryan Ford 推出了[PEG](https://pdos.csail.mit.edu/~baford/packrat/popl04/)，作为描述编程语言和根据描述生成解析器的工具。PEG 不同于传统的正式语法，因为它的规则将非终结符映射到解析表达式，而不仅仅是符号序列。这是本着CPython的灵魂。解析表达式是感应定义的。如果$e$,$e_1$和$e_2$正在解析表达式，那么是：

1. the empty string
2. any terminal
3. any nonterminal
4. $e_1 e_2$, a sequence
5. $e1/e2$, prioritized choice,优先选择
6. $e∗$, zero-or-more repetitions
7. $!e$, a not-predicate.

PEG 是分析语法，这意味着它们不仅用于生成语言，还用于分析它们。福特定义了解析表达的含义$e$识别输入$x$.基本上，任何试图用一些解析表达来解析输入都可能成功或失败，对应于解析部分输入或未能解析。例如，应用解析表达式$a$解析输入$ab$，成功并解析到$a$.

这种形式化允许将任何PEG转换为递归下降解析器。递归下降解析器将语法中的每一个非终结符与解析函数关联在一起。在 PEG 中，解析函数的主体是相应的解析表达式的实现。如果解析表达式包含非终结符，则其解析函数称为递归。

A nonterminal may have multiple production rules. A recursive descent parser has to decide which one was used to derive the input. If a grammar is LL(k), a parser can look at the next k tokens in the input and predict the correct rule. Such a parser is called a predictive parser. If it's not possible to predict, the backtracking method is used. A parser with backtracking tries one rule, and, if fails, backtracks and tries another. This is exactly what the prioritized choice operator in a PEG does. So, a PEG parser is a recursive descent parser with backtracking.

非终结符可能具有多个推导规则。递归下降解析器必须决定使用哪一个来获取输入。如果语法是LL(k)，解析器可以查看输入中的接下来k个token去预测正确的规则。这种解析器称为预测解析器。如果无法预测，则使用回溯方法。具有回溯的解析器尝试一个规则，如果失败，则回溯并尝试另一个规则。这正是 PEG 中的优先选择运算所做的。因此，PEG 解析器是具有回溯的递归下降解析器。

回溯方法功能强大，但计算成本高昂。考虑一个简单的例子。我们应用该表达式$AB/A$成功解析输入$A$，对于输入$B$则失败.根据优先选择操作员的解释，解析器首先尝试识别$A$，成功，然后尝试识别$B$，$AB/A$中$AB$失败，并试图再次解析$A$。由于这种冗余计算，解析时间可以呈指数级输入的大小。为了解决这个问题，福特建议使用记忆技术，即缓存功能调用的结果。使用此技术的解析器，称为包拉特解析器the packrat parser，以较高的内存消耗保证线性时间复杂度。这就是Cpython的新解析器所做的。这是一个packrat parser！

不管新解析器有多好，都要给出更换旧解析器的理由。这就是 PEP 的工作。[PEP 617 -- New PEG parser for CPython](https://www.python.org/dev/peps/pep-0617/)提供了新旧解析器的背景，并解释了过渡背后的原因。简而言之，新的解析器消除了对语法的 LL(1) 限制，并且应该更容易维护。吉多·范·罗森在PEG解析上写了一个优秀的系列，他详细介绍了如何实施一个简单的PEG解析器。反过来，我们将看看它的CPython实施。

你可能会惊讶地发现，新的[语法文件](https://github.com/python/cpython/blob/3.9/Grammar/python.gram)比旧的大三倍多。这是因为新的语法不仅仅是语法，而是语法导向的翻译方案（Syntax-Directed Translation Scheme (SDTS)）。SDTS 是一种语法，其运算附着在规则上。运算操作是一些代码。在规则解析输入并成功时，解析器执行其相应的运算。在解析时，CPython使用运算构建 AST。要了解如何，让我们看看新语法是什么样子的。我们已经看到了旧语法的规则，所以这里是他们的新语法：
```
file[mod_ty]: a=[statements] ENDMARKER { _PyPegen_make_module(p, a) }
statements[asdl_seq*]: a=statement+ { _PyPegen_seq_flatten(p, a) }
statement[asdl_seq*]: a=compound_stmt { _PyPegen_singleton_seq(p, a) } | simple_stmt
compound_stmt[stmt_ty]:
    | ...
    | &'while' while_stmt
while_stmt[stmt_ty]:
    | 'while' a=named_expression ':' b=block c=[else_block] { _Py_While(a, b, c, EXTRA) }
...
```

每个规则都以非终结符开头，之后是分析函数返回结果的 C 类型。右侧是解析表达。大括号中的代码表示运算。运算是返回 AST 节点或其字段的简单函数调用。

新的解析器是[Parser/pegen/parse.c](https://github.com/python/cpython/blob/3.9/Parser/pegen/parse.c)。它由解析器生成器自动生成。解析器生成器以 Python 书写。这是一个程序，需要语法，并产生一个PEG解析器。语法文件中描述语法，并以`Grammar`类实例表示。要创建这样的实例，必须有语法文件的解析器。[此解析器](https://github.com/python/cpython/blob/3.9/Tools/peg_generator/pegen/grammar_parser.py)也由[元语法metagrammar](https://github.com/python/cpython/blob/3.9/Tools/peg_generator/pegen/metagrammar.gram)的解析生成器自动生成。这就是为什么解析器生成器可以在 Python 中生成解析器的原因。但是是什么解析元语法呢？嗯， 它和语法有相同的符号， 所以生成的语法解析器也能解析元语法。当然，语法解析器必须启动，即第一个版本必须手工编写。完成后，可以自动生成所有解析器。

与旧解析器一样，新解析器从tokenizer中获取token。这是不寻常的PEG解析器，因为它允许统一tokenization 和解析。但是我们看到，tokenizer做一个非平凡的工作，所以CPython开发人员决定使用它。

关于这一点，我们结束对解析的讨论，看看 AST 会发生什么。

# AST 优化

![](pic/2/the%20architecture%20of%20the%20CPython%20compiler.png)
CPython 编译器的结构图向我们展示了 AST 优化器以及解析器和编译器。这可能过分强调了优化器的作用。AST 优化器仅限于常数折叠（Constant folding），仅在 CPython 3.7 中引入。在 CPython 3.7 之前，the peephole optimizer完成了常数折叠（Constant folding）。尽管如此，由于 AST 优化器，我们可以编写类似内容：
```
n = 2 ** 32 # easier to write and to read
```
并期望在编译期间计算结果。

一个不太明显的优化的例子是将常数list和一组常数set分别转换为tuple和frozenset。当在`in` 或 `not in`的右侧使用列表或集合时，将执行此优化。

# 从 AST 到代码对象

到目前为止，我们一直在研究 CPython 如何从源代码创建 AST，但正如我们在第一篇文章中看到的，CPython VM 对 AST 一无所知，只能执行代码对象。将 AST 转换为代码对象是编译器的工作。更具体地说，编译器必须返回包含模块的字节码的模块代码对象，以及模块中其他代码块的代码对象，如定义的函数和类。

有时，理解问题解决方案的最佳方式是考虑自己的问题。让我们思考一下，如果我们是编译器，我们会怎么做。我们从代表模块的 AST 的根节点开始。此节点的子项是语句statements。让我们假设第一个语句是一个简单的任务，比如`x = 1`。它由`Assign`  AST 节点表示：`Assign(targets=[Name(id='x', ctx=Store())], value=Constant(value=1))` .要将此节点转换为代码对象，我们需要创建一个节点，将常数`1`存储在代码对象的常数列表中，将变量的名称`x`存储在代码对象中使用的名称列表中，并发出`LOAD_CONST`和`STORE_NAME`指令。我们可以编写一个函数来做到这一点。但是，即使是简单的任务也可能比较棘手。例如，想象一下，同样的任务是在函数的主体内完成的。如果`x`是局部变量，我们应该发出`STORE_FAST`指令。如果`x`是一个全局变量，我们应该发出`STORE_GLOBAL`指令。最后，如果`x`被嵌套函数引用，我们应该发出`STORE_DEREF`指令。问题是确定变量`x`的类型。CPython 在编译之前构建符号表来解决这个问题。

# 符号表
符号表包含有关代码块及其内使用的符号的信息。它由单个`symtable`结构和一组`_symtable_entry`结构表示，每个代码块在程序中代表一个结构。符号表条目包含代码块的属性，包括其名称、类型（模块、类或函数）和字典，该字典将块内使用的变量名称映射到指示其范围和用途的标记。以下是`_symtable_entry`结构的完整定义：
```c
typedef struct _symtable_entry {
    PyObject_HEAD
    PyObject *ste_id;        /* int: key in ste_table->st_blocks */
    PyObject *ste_symbols;   /* dict: variable names to flags */
    PyObject *ste_name;      /* string: name of current block */
    PyObject *ste_varnames;  /* list of function parameters */
    PyObject *ste_children;  /* list of child blocks */
    PyObject *ste_directives;/* locations of global and nonlocal statements */
    _Py_block_ty ste_type;   /* module, class, or function */
    int ste_nested;      /* true if block is nested */
    unsigned ste_free : 1;        /* true if block has free variables */
    unsigned ste_child_free : 1;  /* true if a child block has free vars,
                                     including free refs to globals */
    unsigned ste_generator : 1;   /* true if namespace is a generator */
    unsigned ste_coroutine : 1;   /* true if namespace is a coroutine */
    unsigned ste_comprehension : 1; /* true if namespace is a list comprehension */
    unsigned ste_varargs : 1;     /* true if block has varargs */
    unsigned ste_varkeywords : 1; /* true if block has varkeywords */
    unsigned ste_returns_value : 1;  /* true if namespace uses return with
                                        an argument */
    unsigned ste_needs_class_closure : 1; /* for class scopes, true if a
                                             closure over __class__
                                             should be created */
    unsigned ste_comp_iter_target : 1; /* true if visiting comprehension target */
    int ste_comp_iter_expr; /* non-zero if visiting a comprehension range expression */
    int ste_lineno;          /* first line of block */
    int ste_col_offset;      /* offset of first line of block */
    int ste_opt_lineno;      /* lineno of last exec or import * */
    int ste_opt_col_offset;  /* offset of last exec or import * */
    struct symtable *ste_table;
} PySTEntryObject;
```
CPython 将名称空间the term namespace一词用作符号表上下文中代码块的同义词。因此，我们可以说符号表条目是名称空间的描述。符号表条目通过`ste_children`字段在程序中形成所有命名空间的层次结构，这是子命名空间列表。我们可以使用标准的可对模块探索此层次结构：
```python
# example3.py
def func(x):
    lc = [x+i for i in range(10)]
    return lc
```
```python
>>> from symtable import symtable
>>> f = open('example3.py')
>>> st = symtable(f.read(), 'example3.py', 'exec') # module's symtable entry
>>> dir(st)
[..., 'get_children', 'get_id', 'get_identifiers', 'get_lineno', 'get_name',
 'get_symbols', 'get_type', 'has_children', 'is_nested', 'is_optimized', 'lookup']
>>> st.get_children()
[<Function SymbolTable for func in example3.py>]
>>> func_st = st.get_children()[0] # func's symtable entry
>>> func_st.get_children()
[<Function SymbolTable for listcomp in example3.py>]
>>> lc_st = func_st.get_children()[0] # list comprehension's symtable entry
>>> lc_st.get_symbols()
[<symbol '.0'>, <symbol 'i'>, <symbol 'x'>]
>>> x_sym = lc_st.get_symbols()[2]
>>> dir(x_sym)
[..., 'get_name', 'get_namespace', 'get_namespaces', 'is_annotated',
 'is_assigned', 'is_declared_global', 'is_free', 'is_global', 'is_imported',
 'is_local', 'is_namespace', 'is_nonlocal', 'is_parameter', 'is_referenced']
>>> x_sym.is_local(), x_sym.is_free()
(False, True)
```

此示例显示每个代码块都有相应的符号表条目。我们意外地在列表理解的命名空间中遇到奇怪的`.0`符号。这个命名空间不包含`range`符号，这也很奇怪。这是因为列表推导式作为匿名函数实现，`range(10)`作为参数传递给它。此参数称为 `.0`。CPython 还对我们隐瞒了什么？

符号表条目以两个通道构建。在第一次通过时，CPython 会走 AST，并为遇到的每个代码块创建一个符号表条目。它还收集可以当场收集的信息，例如符号是否在块中定义或使用。但是，在第一次传递过程中，有些信息是很难推断的。请考虑示例：
```python
def top():
    def nested():
        return x + 1
    x = 10
    ...
```
在为函数`nested()`构建符号表条目时，因为我们尚未看到赋值语句，我们无法判断`x`是全局变量还是自由变量，即函数`top()`中定义的变量。

CPython 通过做第二次传递来解决这个问题。在第二关的开头，它已知道符号的定义和使用位置。缺失的信息通过从顶部开始递归访问所有符号表条目来填充。闭包函数外定义的符号将传递到闭包函数，并且将闭包函数内的自由变量的名称传回。

符号表条目使用`symtable`结构进行管理。它既用于构建符号表条目，也用于在编译过程中访问它们。让我们来看看它的定义：
```c
struct symtable {
    PyObject *st_filename;          /* name of file being compiled,
                                       decoded from the filesystem encoding */
    struct _symtable_entry *st_cur; /* current symbol table entry */
    struct _symtable_entry *st_top; /* symbol table entry for module */
    PyObject *st_blocks;            /* dict: map AST node addresses
                                     *       to symbol table entries */
    PyObject *st_stack;             /* list: stack of namespace info */
    PyObject *st_global;            /* borrowed ref to st_top->ste_symbols */
    int st_nblocks;                 /* number of blocks used. kept for
                                       consistency with the corresponding
                                       compiler structure */
    PyObject *st_private;           /* name of current class or NULL */
    PyFutureFeatures *st_future;    /* module's future features that affect
                                       the symbol table */
    int recursion_depth;            /* current recursion depth */
    int recursion_limit;            /* recursion limit */
};
```
需要注意的最重要的字段是st_stack，st_blocks.st_stack字段是一叠符号表条目。在符号表构造的第一关中，CPython 在进入相应的代码块时将条目压入堆栈，并在退出相应的代码块时弹出堆栈条目。st_blocks字段是编译器用于获取给定 AST 节点的符号表条目的字典。st_cur和st_top字段也很重要，但它们的含义应该是显而易见的。

为了了解更多关于符号表及其结构，我强烈推荐你由[伊莱·本德斯基的文章](https://eli.thegreenplace.net/2010/09/18/python-internals-symbol-tables-part-1)。

# 基本块 basic blocks

符号表帮助我们翻译涉及变量等的语句，例如`x = 1`。但是，如果我们尝试翻译一个更复杂的控制流语句，就会出现一个新的问题。考虑另一个神秘的代码：
```python
if x == 0 or x > 17:
    y = True
else:
    y = False
...
```
相应的 AST 子树具有以下结构：
```
If(
  test=BoolOp(...),
  body=[...],
  orelse=[...]
)
```
编译器将其翻译为以下编解码：
```
1           0 LOAD_NAME                0 (x)
            2 LOAD_CONST               0 (0)
            4 COMPARE_OP               2 (==)
            6 POP_JUMP_IF_TRUE        16
            8 LOAD_NAME                0 (x)
           10 LOAD_CONST               1 (17)
           12 COMPARE_OP               4 (>)
           14 POP_JUMP_IF_FALSE       22

2     >>   16 LOAD_CONST               2 (True)
           18 STORE_NAME               1 (y)
           20 JUMP_FORWARD             4 (to 26)

4     >>   22 LOAD_CONST               3 (False)
           24 STORE_NAME               1 (y)
5     >>   26 ...
```
字节码为线性。`test`节点的说明应排在第一位，`body`块的说明应先于`orelse`块的指令。控制流语句的问题在于它们涉及跳跃，并且跳转通常在它指向的指令之前发出。在我们的示例中，如果第一次测试成功，我们希望立即跳转到第一个`body`指令，但我们还不知道它应该在哪里。如果第二个测试失败，我们到`orelse`块，但只有我们翻译`body`块后，第一个`orelse`指令的位置才会变得已知。

如果我们将每个块的说明移动到单独的数据结构中，可以解决这个问题。然后，我们不是将跳跃目标指定为字节码中的具体位置，而是指向这些数据结构。最后，当所有块被翻译并知道其大小时，我们计算跳转的参数，并将所有块聚合成单指令序列。而编译器就是这样做的。

我们谈论的方块称为基本块。它们并不特定于CPython，尽管CPython的基本块的概念与传统的定义不同。根据龙书，一个基本块是指令的最大序列，如：

1. control may enter only the first instruction of the block; and

2. control will leave the block without halting or branching, except possibly at the last instruction.

CPython 放弃了第二个要求。换句话说，除了第一个块之外，没有基本块的指令可以是跳跃的目标，但基本块本身可以包含跳跃指令。要从我们的例子中翻译 AST，编译器创建四个基本块：

1. instructions 0-14 for test
2. instructions 16-20 for body
3. instructions 22-24 for orelse; and
4. instructions 26-... for whatever comes after the if statement.
   
基本块由定义如下的`basicblock_`结构表示：
```c
typedef struct basicblock_ {
    /* Each basicblock in a compilation unit is linked via b_list in the
       reverse order that the block are allocated.  b_list points to the next
       block, not to be confused with b_next, which is next by control flow. */
    struct basicblock_ *b_list;
    /* number of instructions used */
    int b_iused;
    /* length of instruction array (b_instr) */
    int b_ialloc;
    /* pointer to an array of instructions, initially NULL */
    struct instr *b_instr;
    /* If b_next is non-NULL, it is a pointer to the next
       block reached by normal control flow. */
    struct basicblock_ *b_next;
    /* b_seen is used to perform a DFS of basicblocks. */
    unsigned b_seen : 1;
    /* b_return is true if a RETURN_VALUE opcode is inserted. */
    unsigned b_return : 1;
    /* depth of stack upon entry of block, computed by stackdepth() */
    int b_startdepth;
    /* instruction offset for block, computed by assemble_jump_offsets() */
    int b_offset;
} basicblock;
```
And here's the definition of the instr struct:
```c
struct instr {
    unsigned i_jabs : 1;
    unsigned i_jrel : 1;
    unsigned char i_opcode;
    int i_oparg;
    struct basicblock_ *i_target; /* target block (if jump instruction) */
    int i_lineno;
};
```
我们可以看到，基本块不仅通过跳跃指令连接，而且通过b_list和b_next字段连接。例如，编译器用b_list访问所有分配的块以释放内存。b_next领域现在对我们更感兴趣。正如注释所说，它指向由正常控制流到达的下一个块，这意味着它可以用来以正确的顺序组装块。再次返回到我们的例子，`test`块指向`body`块，`body`块指向`orelse`块，块指向块，`orelse`块指向`if`语句块后。由于基本块指向对方，它们会形成一个称为控制流图（Control Flow Graph,CFG） 的图形。

# 帧块 frame block

还有一个问题需要解决：如何理解在编译类似`continue`和`break`？编译器通过引入另一种类型的称为帧块frame block的块来解决这个问题。有不同类型的帧块。例如，`WHILE_LOOP`帧块指向两个基本块：`body`块和`while statement`之后的块。这些基本块分别在编译`continue`和`break`语句时使用。由于帧块可以嵌套，编译器使用堆栈跟踪它们。帧块在处理`try-except-finally`诸如此类的语句时也很有用，但我们现在不会纠纠于此。让我们来看看`fblockinfo`结构的定义：
```c
enum fblocktype { WHILE_LOOP, FOR_LOOP, EXCEPT, FINALLY_TRY, FINALLY_END,
                  WITH, ASYNC_WITH, HANDLER_CLEANUP, POP_VALUE };

struct fblockinfo {
    enum fblocktype fb_type;
    basicblock *fb_block;
    /* (optional) type-specific exit or cleanup block */
    basicblock *fb_exit;
    /* (optional) additional information required for unwinding */
    void *fb_datum;
};
```
我们已经确定了三个重要问题，并且我们看到了编译器如何解决它们。现在，让我们把所有内容放在一起，看看编译器从头到尾是如何工作的。

# 编译器单元、编译器和组装器 compiler units, compiler and assembler

正如我们已经计算出的，在构建符号表后，编译器执行两个步骤将 AST 转换为代码对象：
1. it creates a CFG of basic blocks; and
2. it assembles a CFG into a code object.
   
此两步过程用于程序中的每个代码块。编译器首先构建模块的 CFG，最后将模块的 CFG 组装到模块的代码对象中。其间，它通过反复调用`compiler_visit_*`和`compiler_*`函数来引导 AST，其中`*`表示访问或编译的内容。例如，`compiler_visit_stmt`将给定语句的编译委托给合适的`compiler_*`函数，`compiler_if`函数知道如何编译 `If` AST 节点。如果节点是新的基本块，编译器将创建它们。如果节点是代码块的开始，编译器将创建一个新的编译单元并输入它。编译单元是捕获代码块的编译状态的数据结构。它充当代码对象的可变原型，并指向新的 CFG。在退出当前代码块的开始节点时，编译器组装此 CFG。组装的代码对象存储在父编译单元中。和往常一样，我鼓励您查看结构定义：
```c
    PySTEntryObject *u_ste;

    PyObject *u_name;
    PyObject *u_qualname;  /* dot-separated qualified name (lazy) */
    int u_scope_type;

    /* The following fields are dicts that map objects to
       the index of them in co_XXX.      The index is used as
       the argument for opcodes that refer to those collections.
    */
    PyObject *u_consts;    /* all constants */
    PyObject *u_names;     /* all names */
    PyObject *u_varnames;  /* local variables */
    PyObject *u_cellvars;  /* cell variables */
    PyObject *u_freevars;  /* free variables */

    PyObject *u_private;        /* for private name mangling */

    Py_ssize_t u_argcount;        /* number of arguments for block */
    Py_ssize_t u_posonlyargcount;        /* number of positional only arguments for block */
    Py_ssize_t u_kwonlyargcount; /* number of keyword only arguments for block */
    /* Pointer to the most recently allocated block.  By following b_list
       members, you can reach all early allocated blocks. */
    basicblock *u_blocks;
    basicblock *u_curblock; /* pointer to current block */

    int u_nfblocks;
    struct fblockinfo u_fblock[CO_MAXBLOCKS];

    int u_firstlineno; /* the first lineno of the block */
    int u_lineno;          /* the lineno for the current stmt */
    int u_col_offset;      /* the offset of the current stmt */
};
```
另一个对编译至关重要的数据结构是`compiler`，代表编译的全局状态。其定义如下：
```c
struct compiler {
    PyObject *c_filename;
    struct symtable *c_st;
    PyFutureFeatures *c_future; /* pointer to module's __future__ */
    PyCompilerFlags *c_flags;

    int c_optimize;              /* optimization level */
    int c_interactive;           /* true if in interactive mode */
    int c_nestlevel;
    int c_do_not_emit_bytecode;  /* The compiler won't emit any bytecode
                                    if this value is different from zero.
                                    This can be used to temporarily visit
                                    nodes without emitting bytecode to
                                    check only errors. */

    PyObject *c_const_cache;     /* Python dict holding all constants,
                                    including names tuple */
    struct compiler_unit *u; /* compiler state for current block */
    PyObject *c_stack;           /* Python list holding compiler_unit ptrs */
    PyArena *c_arena;            /* pointer to memory allocation arena */
};
```
定义前的注释解释了两个最重要的字段：

- The u pointer points to the current compilation unit, while units for enclosing blocks are stored in c_stack. The u and c_stack are managed by compiler_enter_scope() and compiler_exit_scope().
u 指针指向当前编译单元，而封闭块的单元存储在c_stack中。u和c_stack由compiler_enter_scope()和compiler_exit_scope()管理。
  
要将基本块组装成代码对象，编译器首先必须通过将指针替换为字节码中的位置来修复跳跃指令。一方面，这是一个简单的任务，因为所有基本块的大小是众所周知的。另一方面，当我们修复跳跃时，基本块的大小可能会发生变化。当前的解决方案是保持固定跳跃在循环中，而大小的变化。以下是有关此解决方案的源代码的诚实评论：

- This is an awful hack that could hurt performance, but on the bright side it should work until we come up with a better solution.
这是一个可怕的方案，可能会损害性能，但在光明的一面，它应该工作，直到我们想到一个更好的解决方案。

其余的是直截了当的。编译器在基本块上重复并发出指令。进度保留在`assembler`结构中：
```c
struct assembler {
    PyObject *a_bytecode;  /* string containing bytecode */
    int a_offset;              /* offset into bytecode */
    int a_nblocks;             /* number of reachable blocks */
    basicblock **a_postorder; /* list of blocks in dfs postorder */
    PyObject *a_lnotab;    /* string containing lnotab */
    int a_lnotab_off;      /* offset into lnotab */
    int a_lineno;              /* last lineno of emitted instruction */
    int a_lineno_off;      /* bytecode offset of last lineno */
};
```
此时，当前的编译单元和组装包含创建代码对象所需的所有数据。祝贺！我们做到了！几乎。

# peephole optimizer

创建代码对象的最后一步是优化字节码。这是peephole optimizer的工作。以下是它执行的某些类型的优化：

1. The statements like if True: ... and while True: ... generate a sequence of LOAD_CONST trueconst and POP_JUMP_IF_FALSE instructions. The peephole optimizer eliminates（消除） such instructions.
2. The statements like a, = b, lead to the bytecode that builds a tuple and then unpacks it. The peephole optimizer replaces it with a simple assignment.
3. The peephole optimizer removes unreachable instructions after RETURN.

# 总结

这是一个很长的文章，总结一下我们学到了什么。CPython 编译器的架构遵循传统设计。它的两个主要部分是前端和后端。前端也称为解析器。其工作是将源代码转换为 AST。解析器从tokenizer中获取tokens，tokenizer负责从文本中生成一系列有意义的语言单元。解析分为几个步骤，包括解析树的生成和解析树转换为 AST。在CPython 3.9中，引入了新的解析器。它基于解析表达语法，并立即生成 AST。后端（也自相矛盾地称为编译器）采用 AST 并生成代码对象。它首先构建符号表，然后创建称为控制流图的程序的中间表示。CFG 被组装成单个指令序列，然后由peephole optimizer进行优化。最终，代码对象被创建。

在这一点上，我们有足够的知识，以熟悉CPython源代码，并了解它的一些事情。这是我们下次的计划。