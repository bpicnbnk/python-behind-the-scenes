# from

[source](https://tenthousandmeters.com/blog/python-behind-the-scenes-3-stepping-through-the-cpython-source-code/)

在这个系列的第一部分和第二部分，我们探索了执行和编译Python程序背后的思想。我们将继续关注下一部分的想法，但这次我们将破例，看看将这些想法带入生活的实际代码。

# 今天的计划

CPython 代码库约为 350，000 行 C 代码（不包括header files）和近 600，000 行 Python 代码。毫无疑问，一次理解这一切将是一项艰巨的任务。所以，今天我们将把我们的研究局限于**每次运行python时执行的源代码**的一部分。我们将从main()函数开始，深入源代码，直到我们到达the evaluation loop，这里是Python 字节码被执行的地方。

我们的目标不是了解我们将遇到的每一个代码，而是突出最有趣的部分，研究它们，并最终大致了解 Python 程序执行开始时会发生什么。

我还要再做两个通知。首先，我们不会进入每一个函数。我们将只对某些部分进行高层概述，然后深入到其他部分。不过，我保证按函数执行顺序描述过程。其次，除了一些结构定义外，我将保留代码。我唯一允许自己的是添加一些注释和重新措辞现有的。在整个帖子中，所有多行注释`/**/`都是原创的，所有单行注释`//`都是我的。说完，让我们开始我们的旅程通过CPython源代码。

# 获取CPython源码

在探索源代码之前，我们需要获得它。让我们克隆 CPython 存储库：

`$ git clone https://github.com/python/cpython/ && cd cpython`

目前的master分支是未来的CPython 3.10。我们对最新的稳定版本（CPython 3.9）感兴趣，因此让我们切换到分支：3.9

`$ git checkout 3.9`

在根目录内，我们找到以下内容：
```
$ ls -p
CODE_OF_CONDUCT.md      Objects/                config.sub
Doc/                    PC/                     configure
Grammar/                PCbuild/                configure.ac
Include/                Parser/                 install-sh
LICENSE                 Programs/               m4/
Lib/                    Python/                 netlify.toml
Mac/                    README.rst              pyconfig.h.in
Makefile.pre.in         Tools/                  setup.py
Misc/                   aclocal.m4
Modules/                config.guess
```
在此系列过程中，列出的一些子目录对我们特别重要：

- Grammar/ 包含我们上次讨论的语法文件。
- Include/ 包含 header files。这些 header files 由CPython和[Python/C API](https://docs.python.org/3/c-api/index.html)的用户使用。
- Lib/ 包含用Python编写的标准库模块。虽然有些模块，如`argparse`和`wave`，完全由Python写成，其他许多模块wrap C代码。例如，Python `io`模块wraps C语言的 `_io`模块。
- Modules/ 包含用C编写的标准库模块。虽然某些模块（如itertools）打算直接导入，但其他模块则由 Python 模块wrap。
- Objects/ 包含内置类型的实现。如果你想了解如何实现int或list，这是最终的地方。
- Parser/ 包含旧解析器、旧解析器生成器、新解析器和tokenizer。
- Programs/ 包含汇编为可执行文件的源文件。
- Python/ 包含解释器本身的源文件。这包括编译器、the evaluation loop、builtins内建模块和许多其他有趣的内容。
- Tools/ 包含可用于构建和管理 CPython 的工具。例如，新的解析器生成器在这里。

如果你没有看到一个测试目录，你的心脏开始跳动更快，放松。它在`Lib/test/`。测试不仅对 CPython 的发展有用，而且对了解 CPython 的工作原理也很有用。例如，要了解窥视孔优化器预期要进行哪些类型的优化，您可以去看看`/Lib/test/test_peepholer.py`。为了了解窥视孔优化器的某些代码做什么，您可以删除该代码，重新编译CPython，运行

`$ ./python.exe -m test test_peepholer`
并查看哪些测试失败。

在理想世界中，我们要做的就是运行`./configure`和`make`：
```shell
$ ./configure
$ make -j -s
```
make将产生一个可执行文件python，但是不要惊讶在Mac操作系统看到python.exe。在大小写不敏感文件系统上，`.exe`扩展用于区分可执行文件与`Python/`目录。查看 Python 开发人员指南，了解有关编译的更[多信息](https://devguide.python.org/setup/#compiling)。

在这一点上，我们可以自豪地说，我们已经建立了我们自己的CPython：
```
$ ./python.exe
Python 3.9.0+ (heads/3.9-dirty:20bdeedfb4, Oct 10 2020, 16:55:24)
[Clang 10.0.0 (clang-1000.10.44.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> 2 ** 16
65536
```

# 源代码

CPython 的执行，就像执行任何其他 C 程序一样，从Python/python.c中的main()函数开始：
```c
/* Minimal main program -- everything is loaded from the library */

#include "Python.h"

#ifdef MS_WINDOWS
int
wmain(int argc, wchar_t **argv)
{
    return Py_Main(argc, argv);
}
#else
int
main(int argc, char **argv)
{
    return Py_BytesMain(argc, argv);
}
#endif
```
那里没有太多事发生。唯一值得一提的是，在Windows CPython上使用`wmain()`而不是`main()`作为入口点接收以`UTF-16`编码的字符串参数`argv`。其影响在于，在其他平台上，CPython 执行将`char`字符串转换为`wchar_t`字符串的额外步骤。`char`字符串的编码取决于本地设置，`wchar_t`字符串的编码取决于`wchar_t`的字节大小。例如，如果`sizeof(wchar_t) == 4`使用编码`UCS-4`。[PEP 383](https://www.python.org/dev/peps/pep-0383/)对此有更多的话要说。

我们在`Modules/main.c`找到`Py_Main()`,`Py_BytesMain()`。他们所做的基本上是用略有不同的参数来调用`pymain_main()`
```c
int
Py_Main(int argc, wchar_t **argv)
{
    _PyArgv args = {
        .argc = argc,
        .use_bytes_argv = 0,
        .bytes_argv = NULL,
        .wchar_argv = argv};
    return pymain_main(&args);
}


int
Py_BytesMain(int argc, char **argv)
{
    _PyArgv args = {
        .argc = argc,
        .use_bytes_argv = 1,
        .bytes_argv = argv,
        .wchar_argv = NULL};
    return pymain_main(&args);
}
```
我们应该`pymain_main()`多停留一会儿，不过，乍一看，它似乎也没有多大用：
```c
static int
pymain_main(_PyArgv *args)
{
    PyStatus status = pymain_init(args);
    if (_PyStatus_IS_EXIT(status)) {
        pymain_free();
        return status.exitcode;
    }
    if (_PyStatus_EXCEPTION(status)) {
        pymain_exit_error(status);
    }

    return Py_RunMain();
}
```
上次我们了解到，在 Python 程序开始执行之前，CPython 会做很多事情来编译它。事实证明，CPython甚至在开始编译程序之前就做了很多事情。这些东西构成了CPython的初始化。正如我们在第一部分已经说过的，CPython 分三个阶段工作：
1. initialization 初始化
2. compilation; and 编译
3. interpretation. 解释

因此，pymain_main()调用pymain_init()执行初始化，然后调用Py_RunMain()以继续下一个阶段。问题仍然存在：CPython 在初始化过程中会怎么做？让我们考虑一下这个问题。至少CPython必须：
- 查找与操作系统的相同语言，以正确处理参数编码、环境变量、标准流和文件系统的编码
- 解析命令行参数并读取环境变量，以确定要运行的选项
- 初始化运行时状态、主解释器状态和主线程状态
- 初始化内置类型和`builtins`模块
- 初始化模块`sys`
- set up the import system
- create the `__main__` module.
  
在进入pymain_init()之前，先讨论初始化过程。

# initialization

CPython 3.8开始，初始化分为3个阶段：
1. preinitialization
2. core initialization; and
3. main initialization.

初始化阶段逐渐引入新的功能。`preinitialization`阶段初始化运行时状态`runtime state`，设置默认内存分配器并执行非常基本的配置。还没有Python的迹象。`core initialization`阶段初始化了主解释器状态和主线程状态、内置类型和异常、`builtins` 模块、`sys`模块和导入系统the import system。此时，您可以使用 Python 的"core"。但是，有些东西还不可用。例如，`sys`模块仅部分初始化，并且仅支持built-in and frozen modules的导入。在`main initialization`阶段之后，CPython 已完全初始化，并准备编译和执行 Python 程序。

具有不同的初始化阶段有什么好处？简言之，它允许更轻松地调整CPython。例如，可以在`preinitialized`状态中设置自定义内存分配器，或覆盖`core_initialized`状态中的路径配置。当然，CPython本身不需要调整任何东西。这些功能对于扩展和嵌入 Python 的 Python/C API 用户非常重要。[PEP 432](https://www.python.org/dev/peps/pep-0432/)和[PEP 587](https://www.python.org/dev/peps/pep-0587/)更详细地解释了为什么进行多阶段初始化是个好主意。

pymain_init()主要涉及`preinitialization`和在此阶段结束时调用`Py_InitializeFromConfig()`，以执行初始化的核心和主要阶段：
```c
pymain_init(const _PyArgv *args)
{
    PyStatus status;

    // Initialize the runtime state
    status = _PyRuntime_Initialize();
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }

    // Initialize default preconfig
    PyPreConfig preconfig;
    PyPreConfig_InitPythonConfig(&preconfig);

    // Perfrom preinitialization
    status = _Py_PreInitializeFromPyArgv(&preconfig, args);
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }
    // Preinitialized. Prepare config for the next initialization phases

    // Initialize default config
    PyConfig config;
    PyConfig_InitPythonConfig(&config);

    // Store the command line arguments in `config->argv`
    if (args->use_bytes_argv) {
        status = PyConfig_SetBytesArgv(&config, args->argc, args->bytes_argv);
    }
    else {
        status = PyConfig_SetArgv(&config, args->argc, args->wchar_argv);
    }
    if (_PyStatus_EXCEPTION(status)) {
        goto done;
    }

    // Perform core and main initialization
    status = Py_InitializeFromConfig(&config);
        if (_PyStatus_EXCEPTION(status)) {
        goto done;
    }
    status = _PyStatus_OK();

done:
    PyConfig_Clear(&config);
    return status;
}
```

`_PyRuntime_Initialize()`初始化运行时状态。运行时状态存储在称为`_PyRuntimeState`类型的全局变量`_PyRuntime`中，定义如下：
```c
/* Full Python runtime state */

typedef struct pyruntimestate {
    /* Is running Py_PreInitialize()? */
    int preinitializing;

    /* Is Python preinitialized? Set to 1 by Py_PreInitialize() */
    int preinitialized;

    /* Is Python core initialized? Set to 1 by _Py_InitializeCore() */
    int core_initialized;

    /* Is Python fully initialized? Set to 1 by Py_Initialize() */
    int initialized;

    /* Set by Py_FinalizeEx(). Only reset to NULL if Py_Initialize() is called again. */
    _Py_atomic_address _finalizing;

    struct pyinterpreters {
        PyThread_type_lock mutex;
        PyInterpreterState *head;
        PyInterpreterState *main;
        int64_t next_id;
    } interpreters;

    unsigned long main_thread;

    struct _ceval_runtime_state ceval;
    struct _gilstate_runtime_state gilstate;

    PyPreConfig preconfig;

    // ... less interesting stuff for now
} _PyRuntimeState;
```
`_PyRuntimeState`最后一个变量`preconfig`保存用于`preinitialize`  CPython 的配置。下一阶段还使用它来完成配置。以下是`PyPreConfig`的定义：
```c
typedef struct {
    int _config_init;     /* _PyConfigInitEnum value */

    /* Parse Py_PreInitializeFromBytesArgs() arguments?
       See PyConfig.parse_argv */
    int parse_argv;

    /* If greater than 0, enable isolated mode: sys.path contains
       neither the script's directory nor the user's site-packages directory.

       Set to 1 by the -I command line option. If set to -1 (default), inherit
       Py_IsolatedFlag value. */
    int isolated;

    /* If greater than 0: use environment variables.
       Set to 0 by -E command line option. If set to -1 (default), it is
       set to !Py_IgnoreEnvironmentFlag. */
    int use_environment;

    /* Set the LC_CTYPE locale to the user preferred locale? If equals to 0,
       set coerce_c_locale and coerce_c_locale_warn to 0. */
    int configure_locale;

    /* Coerce the LC_CTYPE locale if it's equal to "C"? (PEP 538)

       Set to 0 by PYTHONCOERCECLOCALE=0. Set to 1 by PYTHONCOERCECLOCALE=1.
       Set to 2 if the user preferred LC_CTYPE locale is "C".

       If it is equal to 1, LC_CTYPE locale is read to decide if it should be
       coerced or not (ex: PYTHONCOERCECLOCALE=1). Internally, it is set to 2
       if the LC_CTYPE locale must be coerced.

       Disable by default (set to 0). Set it to -1 to let Python decide if it
       should be enabled or not. */
    int coerce_c_locale;

    /* Emit a warning if the LC_CTYPE locale is coerced?

       Set to 1 by PYTHONCOERCECLOCALE=warn.

       Disable by default (set to 0). Set it to -1 to let Python decide if it
       should be enabled or not. */
    int coerce_c_locale_warn;

#ifdef MS_WINDOWS
    /* If greater than 1, use the "mbcs" encoding instead of the UTF-8
       encoding for the filesystem encoding.

       Set to 1 if the PYTHONLEGACYWINDOWSFSENCODING environment variable is
       set to a non-empty string. If set to -1 (default), inherit
       Py_LegacyWindowsFSEncodingFlag value.

       See PEP 529 for more details. */
    int legacy_windows_fs_encoding;
#endif

    /* Enable UTF-8 mode? (PEP 540)

       Disabled by default (equals to 0).

       Set to 1 by "-X utf8" and "-X utf8=1" command line options.
       Set to 1 by PYTHONUTF8=1 environment variable.

       Set to 0 by "-X utf8=0" and PYTHONUTF8=0.

       If equals to -1, it is set to 1 if the LC_CTYPE locale is "C" or
       "POSIX", otherwise it is set to 0. Inherit Py_UTF8Mode value value. */
    int utf8_mode;

    /* If non-zero, enable the Python Development Mode.

       Set to 1 by the -X dev command line option. Set by the PYTHONDEVMODE
       environment variable. */
    int dev_mode;

    /* Memory allocator: PYTHONMALLOC env var.
       See PyMemAllocatorName for valid values. */
    int allocator;
} PyPreConfig;
```

`_PyRuntime_Initialize()`调用后，`_PyRuntime`全局变量初始化为默认值。接下来，`PyPreConfig_InitPythonConfig()`初始化新的默认值`preconfig`，然后`_Py_PreInitializeFromPyArgv()`执行实际的预启动。已经有一个`_PyRuntime`，为什么初始化另一个`preconfig`？请记住，CPython 的许多函数也通过 Python/C API 暴露。因此，CPython 只是按照它本身的设计使用此 API 。另一个后果是，当您阅读 CPython 源代码时，就像我们今天所做的一样，您经常会遇到一些似乎比您期望的做的多的函数。例如，在初始化过程中多次调用`_PyRuntime_Initialize()`。当然，它在随后的调用中没有任何作用。

`_Py_PreInitializeFromPyArgv()`读取命令行参数、环境变量和全局配置变量、当前定位和内存分配器去配置`_PyRuntime.preconfig`。它不读取所有配置，但仅读取与`preinitialization` 阶段相关的参数。例如，它仅解析`-E -I -X`参数。

此时，运行时已预启动。其余的`pymain_init()`是准备下一个初始化阶段。你不应该混淆`config`和`preconfig`。前者是持有大部分 Python 配置的结构。它在初始化阶段大量使用，然后在 Python 程序执行期间也大量使用。要了解`config`用途，我建议您查看其冗长的定义：
```c
/* --- PyConfig ---------------------------------------------- */

typedef struct {
    int _config_init;     /* _PyConfigInitEnum value */

    int isolated;         /* Isolated mode? see PyPreConfig.isolated */
    int use_environment;  /* Use environment variables? see PyPreConfig.use_environment */
    int dev_mode;         /* Python Development Mode? See PyPreConfig.dev_mode */

    /* Install signal handlers? Yes by default. */
    int install_signal_handlers;

    int use_hash_seed;      /* PYTHONHASHSEED=x */
    unsigned long hash_seed;

    /* Enable faulthandler?
       Set to 1 by -X faulthandler and PYTHONFAULTHANDLER. -1 means unset. */
    int faulthandler;

    /* Enable PEG parser?
       1 by default, set to 0 by -X oldparser and PYTHONOLDPARSER */
    int _use_peg_parser;

    /* Enable tracemalloc?
       Set by -X tracemalloc=N and PYTHONTRACEMALLOC. -1 means unset */
    int tracemalloc;

    int import_time;        /* PYTHONPROFILEIMPORTTIME, -X importtime */
    int show_ref_count;     /* -X showrefcount */
    int dump_refs;          /* PYTHONDUMPREFS */
    int malloc_stats;       /* PYTHONMALLOCSTATS */

    /* Python filesystem encoding and error handler:
       sys.getfilesystemencoding() and sys.getfilesystemencodeerrors().

       Default encoding and error handler:

       * if Py_SetStandardStreamEncoding() has been called: they have the
         highest priority;
       * PYTHONIOENCODING environment variable;
       * The UTF-8 Mode uses UTF-8/surrogateescape;
       * If Python forces the usage of the ASCII encoding (ex: C locale
         or POSIX locale on FreeBSD or HP-UX), use ASCII/surrogateescape;
       * locale encoding: ANSI code page on Windows, UTF-8 on Android and
         VxWorks, LC_CTYPE locale encoding on other platforms;
       * On Windows, "surrogateescape" error handler;
       * "surrogateescape" error handler if the LC_CTYPE locale is "C" or "POSIX";
       * "surrogateescape" error handler if the LC_CTYPE locale has been coerced
         (PEP 538);
       * "strict" error handler.

       Supported error handlers: "strict", "surrogateescape" and
       "surrogatepass". The surrogatepass error handler is only supported
       if Py_DecodeLocale() and Py_EncodeLocale() use directly the UTF-8 codec;
       it's only used on Windows.

       initfsencoding() updates the encoding to the Python codec name.
       For example, "ANSI_X3.4-1968" is replaced with "ascii".

       On Windows, sys._enablelegacywindowsfsencoding() sets the
       encoding/errors to mbcs/replace at runtime.


       See Py_FileSystemDefaultEncoding and Py_FileSystemDefaultEncodeErrors.
       */
    wchar_t *filesystem_encoding;
    wchar_t *filesystem_errors;

    wchar_t *pycache_prefix;  /* PYTHONPYCACHEPREFIX, -X pycache_prefix=PATH */
    int parse_argv;           /* Parse argv command line arguments? */

    /* Command line arguments (sys.argv).

       Set parse_argv to 1 to parse argv as Python command line arguments
       and then strip Python arguments from argv.

       If argv is empty, an empty string is added to ensure that sys.argv
       always exists and is never empty. */
    PyWideStringList argv;

    /* Program name:

       - If Py_SetProgramName() was called, use its value.
       - On macOS, use PYTHONEXECUTABLE environment variable if set.
       - If WITH_NEXT_FRAMEWORK macro is defined, use __PYVENV_LAUNCHER__
         environment variable is set.
       - Use argv[0] if available and non-empty.
       - Use "python" on Windows, or "python3 on other platforms. */
    wchar_t *program_name;

    PyWideStringList xoptions;     /* Command line -X options */

    /* Warnings options: lowest to highest priority. warnings.filters
       is built in the reverse order (highest to lowest priority). */
    PyWideStringList warnoptions;

    /* If equal to zero, disable the import of the module site and the
       site-dependent manipulations of sys.path that it entails. Also disable
       these manipulations if site is explicitly imported later (call
       site.main() if you want them to be triggered).

       Set to 0 by the -S command line option. If set to -1 (default), it is
       set to !Py_NoSiteFlag. */
    int site_import;

    /* Bytes warnings:

       * If equal to 1, issue a warning when comparing bytes or bytearray with
         str or bytes with int.
       * If equal or greater to 2, issue an error.

       Incremented by the -b command line option. If set to -1 (default), inherit
       Py_BytesWarningFlag value. */
    int bytes_warning;

    /* If greater than 0, enable inspect: when a script is passed as first
       argument or the -c option is used, enter interactive mode after
       executing the script or the command, even when sys.stdin does not appear
       to be a terminal.

       Incremented by the -i command line option. Set to 1 if the PYTHONINSPECT
       environment variable is non-empty. If set to -1 (default), inherit
       Py_InspectFlag value. */
    int inspect;

    /* If greater than 0: enable the interactive mode (REPL).

       Incremented by the -i command line option. If set to -1 (default),
       inherit Py_InteractiveFlag value. */
    int interactive;

    /* Optimization level.

       Incremented by the -O command line option. Set by the PYTHONOPTIMIZE
       environment variable. If set to -1 (default), inherit Py_OptimizeFlag
       value. */
    int optimization_level;

    /* If greater than 0, enable the debug mode: turn on parser debugging
       output (for expert only, depending on compilation options).

       Incremented by the -d command line option. Set by the PYTHONDEBUG
       environment variable. If set to -1 (default), inherit Py_DebugFlag
       value. */
    int parser_debug;

    /* If equal to 0, Python won't try to write ``.pyc`` files on the
       import of source modules.

       Set to 0 by the -B command line option and the PYTHONDONTWRITEBYTECODE
       environment variable. If set to -1 (default), it is set to
       !Py_DontWriteBytecodeFlag. */
    int write_bytecode;

    /* If greater than 0, enable the verbose mode: print a message each time a
       module is initialized, showing the place (filename or built-in module)
       from which it is loaded.

       If greater or equal to 2, print a message for each file that is checked
       for when searching for a module. Also provides information on module
       cleanup at exit.

       Incremented by the -v option. Set by the PYTHONVERBOSE environment
       variable. If set to -1 (default), inherit Py_VerboseFlag value. */
    int verbose;

    /* If greater than 0, enable the quiet mode: Don't display the copyright
       and version messages even in interactive mode.

       Incremented by the -q option. If set to -1 (default), inherit
       Py_QuietFlag value. */
    int quiet;

   /* If greater than 0, don't add the user site-packages directory to
      sys.path.

      Set to 0 by the -s and -I command line options , and the PYTHONNOUSERSITE
      environment variable. If set to -1 (default), it is set to
      !Py_NoUserSiteDirectory. */
    int user_site_directory;

    /* If non-zero, configure C standard steams (stdio, stdout,
       stderr):

       - Set O_BINARY mode on Windows.
       - If buffered_stdio is equal to zero, make streams unbuffered.
         Otherwise, enable streams buffering if interactive is non-zero. */
    int configure_c_stdio;

    /* If equal to 0, enable unbuffered mode: force the stdout and stderr
       streams to be unbuffered.

       Set to 0 by the -u option. Set by the PYTHONUNBUFFERED environment
       variable.
       If set to -1 (default), it is set to !Py_UnbufferedStdioFlag. */
    int buffered_stdio;

    /* Encoding of sys.stdin, sys.stdout and sys.stderr.
       Value set from PYTHONIOENCODING environment variable and
       Py_SetStandardStreamEncoding() function.
       See also 'stdio_errors' attribute. */
    wchar_t *stdio_encoding;

    /* Error handler of sys.stdin and sys.stdout.
       Value set from PYTHONIOENCODING environment variable and
       Py_SetStandardStreamEncoding() function.
       See also 'stdio_encoding' attribute. */
    wchar_t *stdio_errors;

#ifdef MS_WINDOWS
    /* If greater than zero, use io.FileIO instead of WindowsConsoleIO for sys
       standard streams.

       Set to 1 if the PYTHONLEGACYWINDOWSSTDIO environment variable is set to
       a non-empty string. If set to -1 (default), inherit
       Py_LegacyWindowsStdioFlag value.

       See PEP 528 for more details. */
    int legacy_windows_stdio;
#endif

    /* Value of the --check-hash-based-pycs command line option:

       - "default" means the 'check_source' flag in hash-based pycs
         determines invalidation
       - "always" causes the interpreter to hash the source file for
         invalidation regardless of value of 'check_source' bit
       - "never" causes the interpreter to always assume hash-based pycs are
         valid

       The default value is "default".

       See PEP 552 "Deterministic pycs" for more details. */
    wchar_t *check_hash_pycs_mode;

    /* --- Path configuration inputs ------------ */

    /* If greater than 0, suppress _PyPathConfig_Calculate() warnings on Unix.
       The parameter has no effect on Windows.

       If set to -1 (default), inherit !Py_FrozenFlag value. */
    int pathconfig_warnings;

    wchar_t *pythonpath_env; /* PYTHONPATH environment variable */
    wchar_t *home;          /* PYTHONHOME environment variable,
                               see also Py_SetPythonHome(). */

    /* --- Path configuration outputs ----------- */

    int module_search_paths_set;  /* If non-zero, use module_search_paths */
    PyWideStringList module_search_paths;  /* sys.path paths. Computed if
                                       module_search_paths_set is equal
                                       to zero. */

    wchar_t *executable;        /* sys.executable */
    wchar_t *base_executable;   /* sys._base_executable */
    wchar_t *prefix;            /* sys.prefix */
    wchar_t *base_prefix;       /* sys.base_prefix */
    wchar_t *exec_prefix;       /* sys.exec_prefix */
    wchar_t *base_exec_prefix;  /* sys.base_exec_prefix */
    wchar_t *platlibdir;        /* sys.platlibdir */

    /* --- Parameter only used by Py_Main() ---------- */

    /* Skip the first line of the source ('run_filename' parameter), allowing use of non-Unix forms of
       "#!cmd".  This is intended for a DOS specific hack only.

       Set by the -x command line option. */
    int skip_source_first_line;

    wchar_t *run_command;   /* -c command line argument */
    wchar_t *run_module;    /* -m command line argument */
    wchar_t *run_filename;  /* Trailing command line argument without -c or -m */

    /* --- Private fields ---------------------------- */

    /* Install importlib? If set to 0, importlib is not initialized at all.
       Needed by freeze_importlib. */
    int _install_importlib;

    /* If equal to 0, stop Python initialization before the "main" phase */
    int _init_main;

    /* If non-zero, disallow threads, subprocesses, and fork.
       Default: 0. */
    int _isolated_interpreter;

    /* Original command line arguments. If _orig_argv is empty and _argv is
       not equal to [''], PyConfig_Read() copies the configuration 'argv' list
       into '_orig_argv' list before modifying 'argv' list (if parse_argv
       is non-zero).

       _PyConfig_Write() initializes Py_GetArgcArgv() to this list. */
    PyWideStringList _orig_argv;
} PyConfig;
```

与`pymain_init()`调用`PyPreConfig_InitPythonConfig()`创建默认值`preconfig`的方式相同，它现在调用`PyConfig_InitPythonConfig()`创建默认值`config`。然后，它调用`PyConfig_SetBytesArgv()`存储命令行参数`config.argv`，调用`Py_InitializeFromConfig()`执行核心和主要初始化阶段。我们从`pymain_init()`进一步到`Py_InitializeFromConfig()`：
```c
PyStatus
Py_InitializeFromConfig(const PyConfig *config)
{
    if (config == NULL) {
        return _PyStatus_ERR("initialization config is NULL");
    }

    PyStatus status;

    // Yeah, call once again
    status = _PyRuntime_Initialize();
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }
    _PyRuntimeState *runtime = &_PyRuntime;

    PyThreadState *tstate = NULL;
    // The core initialization phase
    status = pyinit_core(runtime, config, &tstate);
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }
    config = _PyInterpreterState_GetConfig(tstate->interp);

    if (config->_init_main) {
        // The main initialization phase
        status = pyinit_main(tstate);
        if (_PyStatus_EXCEPTION(status)) {
            return status;
        }
    }

    return _PyStatus_OK();
}
```
我们可以清楚地看到初始化阶段之间的分离。核心阶段由`pyinit_core()`完成，主阶段`pyinit_main()`完成。`pyinit_core()`函数初始化了 Python 的"核心"。更具体地说，

1. 准备配置：解析命令行参数，读取环境变量，计算路径配置，选择标准流和文件系统的编码，并将所有这些内容写入`config`适当的位置。
2. 应用配置：配置标准流、生成哈希的私钥、创建主解释器状态和主线程状态、初始化 GIL 并采用 GIL、启用 GC、初始化内置类型和异常、初始化`sys`模块和`builtins`模块以及设置built-in and frozen modules的导入系统。
   

在第一步，CPython计算config.module_search_paths，这将稍后复制到sys.path。否则，这一步不是很有趣，所以让我们看看pyinit_core()调用的pyinit_config()，执行第二步：
```c
static PyStatus
pyinit_config(_PyRuntimeState *runtime,
              PyThreadState **tstate_p,
              const PyConfig *config)
{
    // Set Py_* global variables from config.
    // Initialize C standard streams (stdin, stdout, stderr).
    // Set secret key for hashing.
    PyStatus status = pycore_init_runtime(runtime, config);
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }

    PyThreadState *tstate;
    // Create the main interpreter state and the main thread state.
    // Take the GIL.
    status = pycore_create_interpreter(runtime, config, &tstate);
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }
    *tstate_p = tstate;

    // Init types, exception, sys, builtins, importlib, etc.
    status = pycore_interp_init(tstate);
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }

    /* Only when we get here is the runtime core fully initialized */
    runtime->core_initialized = 1;
    return _PyStatus_OK();
}
```

首先，pycore_init_runtime()将config的一些字段复制到相应的[全局配置变量](https://docs.python.org/3/c-api/init.html#global-configuration-variables)中。这些全局变量在PyConfig之前用于配置 CPython，并是 Python/C API 的一部分。

接下来，`pycore_init_runtime()`为`stdio`，`stdout`和`stderr`[文件句柄](http://www.cplusplus.com/reference/cstdio/FILE/)"设置缓冲模式。在类 Unix 的系统上，这是通过调用[setvbuf()](https://linux.die.net/man/3/setvbuf)库函数来完成的。

最后，`pycore_init_runtime()`生成用于哈希的私钥，该密钥存储在全局变量`_Py_HashSecret`中。私钥与[SipHash24](https://en.wikipedia.org/wiki/SipHash)散列函数的输入一起取，CPython 使用该函数来计算哈希。每次 CPython 启动时，都会随机生成私钥。随机化的目的是保护 Python 应用程序免受哈希碰撞 DoS 攻击。Python 和许多其他语言，包括 PHP、Ruby、JavaScript 和 C# 曾经容易受到此类攻击。攻击者可以向应用程序发送一组具有相同哈希的字符串，并显著增加将这些字符串放入字典所需的 CPU 时间，因为它们都恰巧位于同一存储桶中。解决方案是提供一个特别的哈希函数，其带有攻击者不知道的、随机生成的密钥。Python 还允许通过将环境变量`PYTHONHASHSEED`设置为某些固定值来确定生成密钥。要了解有关攻击的更多内容，请查看此[演示文稿](https://fahrplan.events.ccc.de/congress/2011/Fahrplan/attachments/2007_28C3_Effective_DoS_on_web_application_platforms.pdf)。要了解更多有关CPython的哈希算法，请查看[PEP 456](https://www.python.org/dev/peps/pep-0456/)。

在第一部分中，我们了解到，CPython 使用线程状态存储线程特定数据，如调用堆栈和异常状态，以及存储特定于解释器的数据（如已加载的模块和导入设置）的解释器状态。pycore_create_interpreter()函数为主操作系统线程创建解释器状态和线程状态。我们还没有看到这些结构是什么样子，所以下面是解释器状态结构的定义：
```c 
// The PyInterpreterState typedef is in Include/pystate.h.
struct _is {

    // _PyRuntime.interpreters.head stores the most recently created interpreter
    // `next` allows to access all interpreters.
    struct _is *next;
    // `tstate_head` points to the most recently created thread state.
    // Thread states of the same interpreter are linked together.
    struct _ts *tstate_head;

    /* Reference to the _PyRuntime global variable. This field exists
       to not have to pass runtime in addition to tstate to a function.
       Get runtime from tstate: tstate->interp->runtime. */
    struct pyruntimestate *runtime;

    int64_t id;
    // For tracking references to the interpreter
    int64_t id_refcount;
    int requires_idref;
    PyThread_type_lock id_mutex;

    int finalizing;

    struct _ceval_state ceval;
    struct _gc_runtime_state gc;

    PyObject *modules;  // sys.modules points to it
    PyObject *modules_by_index;
    PyObject *sysdict;  // points to sys.__dict__
    PyObject *builtins; // points to builtins.__dict__
    PyObject *importlib;

    // A list of codec search functions
    PyObject *codec_search_path;
    PyObject *codec_search_cache;
    PyObject *codec_error_registry;
    int codecs_initialized;

    struct _Py_unicode_state unicode;

    PyConfig config;

    PyObject *dict;  /* Stores per-interpreter state */

    PyObject *builtins_copy;
    PyObject *import_func;
    /* Initialized to PyEval_EvalFrameDefault(). */
    _PyFrameEvalFunction eval_frame;

    // See `atexit` module
    void (*pyexitfunc)(PyObject *);
    PyObject *pyexitmodule;

    uint64_t tstate_next_unique_id;

    // See `warnings` module
    struct _warnings_runtime_state warnings;

    // A list of audit hooks, see sys.addaudithook
    PyObject *audit_hooks;

#if _PY_NSMALLNEGINTS + _PY_NSMALLPOSINTS > 0
    // Small integers are preallocated in this array so that they can be shared.
    // The default range is [-5, 256].
    PyLongObject* small_ints[_PY_NSMALLNEGINTS + _PY_NSMALLPOSINTS];
#endif

  // ... less interesting stuff for now
};
``` 
这里需要注意的是，config属于解释器状态。以前读取的配置存储在新创建的解释器状态的config中。线程状态结构定义如下：
```c
// The PyThreadState typedef is in Include/pystate.h.
struct _ts {

    // Double-linked list is used to access all thread states belonging to the same interpreter
    struct _ts *prev;
    struct _ts *next;
    PyInterpreterState *interp;

    // Reference to the current frame (it can be NULL).
    // The call stack is accesible via frame->f_back.
    PyFrameObject *frame;

    // ... checking if recursion level is too deep

    // ... tracing/profiling

    /* The exception currently being raised */
    PyObject *curexc_type;
    PyObject *curexc_value;
    PyObject *curexc_traceback;

    /* The exception currently being handled, if no coroutines/generators
     * are present. Always last element on the stack referred to be exc_info.
     */
    _PyErr_StackItem exc_state;

    /* Pointer to the top of the stack of the exceptions currently
     * being handled */
    _PyErr_StackItem *exc_info;

    PyObject *dict;  /* Stores per-thread state */

    int gilstate_counter;

    PyObject *async_exc; /* Asynchronous exception to raise */
    unsigned long thread_id; /* Thread id where this tstate was created */

    /* Unique thread state id. */
    uint64_t id;

    // ... less interesting stuff for now
};
``` 

在为主操作系统线程创建线程状态后，`pycore_create_interpreter()`初始化 GIL，防止多个线程同时与 Python 对象工作。如果您使用`threading`模块生成新线程，它将开始在评估循环中执行给定目标。在这种情况下，线程等待 GIL 并在评估循环的每次迭代开始时将其取走。线程可以访问其线程状态，因为线程状态作为参数被传递到评估函数。但是，如果您编写 C 扩展并调用 Python/C API 来获取 GIL，CPython 不仅需要获取 GIL，还需要将当前线程与相应的线程状态关联。这是通过在线程状态创建时将线程状态存储在线程特定存储（类 Unix 的系统上的pthread_setspecific()库函数）中。这是允许任何线程访问其线程状态的机制。GIL需要单独的介绍。Python 对象系统和导入机制也是如此，我们在此帖子中将简要提及。

创建第一个口译状态和第一个线程状态后，pyinit_config()调用pycore_interp_init()以完成核心初始化阶段。代码pycore_interp_init()是不言自明的：
```c
static PyStatus
pycore_interp_init(PyThreadState *tstate)
{
    PyStatus status;
    PyObject *sysmod = NULL;

    status = pycore_init_types(tstate);
    if (_PyStatus_EXCEPTION(status)) {
        goto done;
    }

    status = _PySys_Create(tstate, &sysmod);
    if (_PyStatus_EXCEPTION(status)) {
        goto done;
    }

    status = pycore_init_builtins(tstate);
    if (_PyStatus_EXCEPTION(status)) {
        goto done;
    }

    status = pycore_init_import_warnings(tstate, sysmod);

done:
    // Py_XDECREF() decreases the reference count of an object.
    // If the reference count becomes 0, the object is deallocated.
    Py_XDECREF(sysmod);
    return status;
}
``` 

`pycore_init_types()`函数初始化内置类型。但这是什么意思呢？究竟是什么类型？你可能知道，你在Python工作的所有东西都是一个对象。数字、字符串、列表、函数、模块、帧对象、用户定义的类和内置类型都是 Python 对象。Python 对象是PyObject struct的实例或其第一个字段为PyObject类型的任意 C struct 的实例。PyObject struct 有两个字段,第一个字段`Py_ssize_t`存储引用计数。第二个字段`PyTypeObject`指向对象的 Python 类型。以下是PyObject的定义：
```c
typedef struct _object {
    _PyObject_HEAD_EXTRA // for debugging only
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
``` 
下面是一个更熟悉的 Python 对象示例：float
```c
typedef struct {
    PyObject_HEAD // macro that expands to PyObject ob_base;
    double ob_fval;
} PyFloatObject;
```
C 标准指出，指向任何结构（struct）的指针可以转换为指向其第一个成员的指针，反之亦然。因为任何 Python 对象都将PyObject作为其第一个成员，CPython 可以将任何 Python 对象视为PyObject。你可以把它看作是C进行子分类的方式。这种伎俩的好处是，它允许实现多态性。例如，它允许编写一个函数，可以通过PyObject，采取任何Python对象作为参数。

CPython之所以能够利用PyObject做一些有用的事情，是因为Python对象的行为是由它的类型决定的，并且PyObject总是有一个类型。一个类型知道如何创建该类型的对象，如何计算它们的哈希，如何添加它们，如何调用它们，如何访问它们的属性，如何处理释放它们等等。类型也是由PyTypeObject 结构表示的 Python 对象。所有类型都有相同的类型，即PyType_Type。PyType_Type的类型指向本身。如果此解释似乎很复杂，则此示例很简单：
```shell
$ ./python.exe -q
>>> type([])
<class 'list'>
>>> type(type([]))
<class 'type'>
>>> type(type(type([])))
<class 'type'>
``` 
Python/C API 参考手册中很好地记录了PyTypeObject的字段。我只留下PyTypeObject的基本结构的定义，以获得Python类型存储的信息量的方法：```c
// PyTypeObject is a typedef for struct _typeobject
struct _typeobject {
    PyObject_VAR_HEAD // expands to 
                      // PyObject ob_base;
                      // Py_ssize_t ob_size;
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

内置类型（如int和list）通过静态定义PyTypeObject的实例实现，例如：
```c
PyTypeObject PyList_Type = {
    PyVarObject_HEAD_INIT(&PyType_Type, 0)
    "list",
    sizeof(PyListObject),
    0,
    (destructor)list_dealloc,                   /* tp_dealloc */
    0,                                          /* tp_vectorcall_offset */
    0,                                          /* tp_getattr */
    0,                                          /* tp_setattr */
    0,                                          /* tp_as_async */
    (reprfunc)list_repr,                        /* tp_repr */
    0,                                          /* tp_as_number */
    &list_as_sequence,                          /* tp_as_sequence */
    &list_as_mapping,                           /* tp_as_mapping */
    PyObject_HashNotImplemented,                /* tp_hash */
    0,                                          /* tp_call */
    0,                                          /* tp_str */
    PyObject_GenericGetAttr,                    /* tp_getattro */
    0,                                          /* tp_setattro */
    0,                                          /* tp_as_buffer */
    Py_TPFLAGS_DEFAULT | Py_TPFLAGS_HAVE_GC |
        Py_TPFLAGS_BASETYPE | Py_TPFLAGS_LIST_SUBCLASS, /* tp_flags */
    list___init____doc__,                       /* tp_doc */
    (traverseproc)list_traverse,                /* tp_traverse */
    (inquiry)_list_clear,                       /* tp_clear */
    list_richcompare,                           /* tp_richcompare */
    0,                                          /* tp_weaklistoffset */
    list_iter,                                  /* tp_iter */
    0,                                          /* tp_iternext */
    list_methods,                               /* tp_methods */
    0,                                          /* tp_members */
    0,                                          /* tp_getset */
    0,                                          /* tp_base */
    0,                                          /* tp_dict */
    0,                                          /* tp_descr_get */
    0,                                          /* tp_descr_set */
    0,                                          /* tp_dictoffset */
    (initproc)list___init__,                    /* tp_init */
    PyType_GenericAlloc,                        /* tp_alloc */
    PyType_GenericNew,                          /* tp_new */
    PyObject_GC_Del,                            /* tp_free */
    .tp_vectorcall = list_vectorcall,
};
```
CPython 还需要初始化内置类型。这就是我们开始讨论类型的原因。例如，所有类型都需要一些初始化，以添加特殊方法，如__call__和__eq__，到类型的字典，并指向相应的tp_*函数。这种常见的初始化是通过调用PyType_Ready()完成的：
```c
PyStatus
_PyTypes_Init(void)
{
    // The names of the special methods "__hash__", "__call_", etc. are interned by this call
    PyStatus status = _PyTypes_InitSlotDefs();
    if (_PyStatus_EXCEPTION(status)) {
        return status;
    }

#define INIT_TYPE(TYPE, NAME) \
    do { \
        if (PyType_Ready(TYPE) < 0) { \
            return _PyStatus_ERR("Can't initialize " NAME " type"); \
        } \
    } while (0)

    INIT_TYPE(&PyBaseObject_Type, "object");
    INIT_TYPE(&PyType_Type, "type");
    INIT_TYPE(&_PyWeakref_RefType, "weakref");
    INIT_TYPE(&_PyWeakref_CallableProxyType, "callable weakref proxy");
    INIT_TYPE(&_PyWeakref_ProxyType, "weakref proxy");
    INIT_TYPE(&PyLong_Type, "int");
    INIT_TYPE(&PyBool_Type, "bool");
    INIT_TYPE(&PyByteArray_Type, "bytearray");
    INIT_TYPE(&PyBytes_Type, "str");
    INIT_TYPE(&PyList_Type, "list");
    INIT_TYPE(&_PyNone_Type, "None");
    INIT_TYPE(&_PyNotImplemented_Type, "NotImplemented");
    INIT_TYPE(&PyTraceBack_Type, "traceback");
    INIT_TYPE(&PySuper_Type, "super");
    INIT_TYPE(&PyRange_Type, "range");
    INIT_TYPE(&PyDict_Type, "dict");
    INIT_TYPE(&PyDictKeys_Type, "dict keys");
    // ... 50 more types
    return _PyStatus_OK();

#undef INIT_TYPE
}
``` 

某些内置类型需要额外的特定类型初始化。例如，int需要对`interp->small_ints`数组中预分配小整数，以便重新利用。float初始化需要确定当前机器如何表示浮点数。

当内置类型初始化时，`pycore_interp_init()`调用`_PySys_Create()`以创建`sys`模块。为什么要创建模块的第一个模块？当然，这是非常重要的，因为它包含诸如传递到程序的命令行参数（sys.argv）、搜索模块的路径条目列表（sys.path）、大量针对系统和实现特定数据（sys.version、sys.implementation、sys.thread_info等）以及允许与解释器交互的各种函数（sys.addaudithook()、sys.settrace()等）。不过，这么早就创建sys模块的主要原因是初始化`sys.modules`。`sys.modules` 由`_PySys_Create()`创建，指向interp->modules字典，并作为所有导入模块的缓存。这是查找模块的第一个地方，也是保存所有加载模块的地方。导入系统严重依赖sys.modules。

`_PySys_Create()`调用后，sys模块仅**部分初始化**，这时，函数和大多数变量是可用的，但一些特定的数据将在主要的初始化阶段（the main initialization phase）设置。如`sys.argv`和`sys._xoptions`，和路径相关的配置，如`sys.path`和`sys.exec_prefix`，

创建`sys`模块时，`pycore_interp_init()`调用`pycore_init_builtins()`以初始化`builtins`模块。内置函数，如`abs()`，`dir()`和`print()`，内置类型，如`dict`，`int`和`str`，内置的异常，如`Exception`和`ValueError`，和内置常数，比如`False`，`Ellipsis`和`None`，都是builtins模块的成员。内置函数是模块定义的一部分，但其他成员必须明确放置在模块字典中。函数`pycore_init_builtins()`完成这个功能。稍后，将`frame->f_builtins`设置为字典用来查找名称。这就是为什么我们不需要直接导入builtins。

核心初始化阶段的最后一步由函数pycore_init_import_warnings()执行。您可能知道 Python 有发出[警告的机制](https://docs.python.org/3/library/warnings.html)，如：
```shell
$ ./python.exe -q
>>> import imp
<stdin>:1: DeprecationWarning: the imp module is deprecated in favour of importlib; ...
``` 

警告可以忽略，或转变成异常，或以各种方式显示。CPython 有过滤器来做到这一点。默认情况下，函数`pycore_init_import_warnings()`打开默认过滤器。然而，最关键的是`pycore_init_import_warnings()`建立built-in and frozen modules的导入系统。

built-in and frozen modules是两种特殊类型的模块,它们被直接编译到python可执行文件。不同的是，内置模块以 C 书写，而frozen模块以 Python 编写。如何将用 Python 编写的模块编译成可执行模块？通过将模块代码对象的二进制表示合并到 C 源代码中，可以巧妙地做到这一点。要生成二进制表示，使用[Freeze utility ](https://github.com/python/cpython/tree/master/Tools/freeze)。

frozen modules的一个示例是`_frozen_importlib`。这是导入系统的核心。Python 的import声明最终都会调用`_frozen_importlib._find_and_load()`函数。为了支持内置和冻结模块的导入，`pycore_init_import_warnings()`调用`init_importlib()`，init_importlib()的第一件事就是导入`_frozen_importlib`。看来，CPython必须导入`_frozen_importlib`才能导入`_frozen_importlib`，但事实并非如此。在导入模块时,`_frozen_importlib`模块是通用 API 的一部分。然而，如果CPython知道它需要导入一个frozen modules，它可以无需依赖`_frozen_importlib`，直接导入。

`_frozen_importlib`模块取决于另外两个模块。首先，它需要`sys`模块才能访问`sys.modules`。其次，它需要实现底层导入函数的`_imp`模块，`_imp`模块包括创建内置和冻结模块的函数。问题是`_frozen_importlib`不能导入任何模块，因为import语句取决于`_frozen_importlib`自身。解决方案是在init_importlib()中创建`_imp`模块并通过在`_frozen_importlib`中调用`_frozen_importlib._install(sys, _imp)`注入`_imp`模块和`sys`模块。导入系统的自引导结束了核心初始化阶段。

我们离开pyinit_core(),进入pyinit_main()，执行主要的初始化阶段。我们发现，它会做一些检查，然后调用init_interp_main()做实际的工作。工作总结如下：

1. 获取system's realtime and monotonic clocks，确保time.time()，time.monotonic()time.perf_counter()正常工作。

    - Clock_realtime 
        代表机器上可以理解为当前的我们所常看的时间，其当time-of-day 被修改的时候而改变，这包括NTP对它的修改（NTP:Network Time Protocol（NTP）是用来使计算机时间同步化的一种协议，它可以使计算机对其服务器或时钟源（如石英钟，GPS等等）做同步化，它可以提供高精准度的时间校正（LAN上与标准间差小于1毫秒，WAN上几十毫秒），且可介由加密确认的方式来防止恶毒的协议攻击。）
    - CLOCK_MONOTONIC  
        代表从过去某个固定的时间点开始的绝对的逝去时间，它不受任何系统time-of-day时钟修改的影响，如果你想计算出在一台计算机上不受重启的影响，两个事件发生的间隔时间的话，那么它将是最好的选择。

2. 完成`sys`模块的初始化。这包括设置路径配置变量，例如，sys.path和sys.executable，和调用特定的变量（invocation-specific variables），如sys.exec_prefix和sys.argvsys._xoptions.

3. 为导入基于路径的（外部）模块添加支持。这是通过导入另一个名为importlib._bootstrap_external的"frozen module"来完成的。它使模块的导入依赖于sys.path。此外，导入[zipimport ](https://docs.python.org/3/library/zipimport.html)冷冻模块frozen module。它允许从 ZIP 档案中导入模块，即sys.path中列出的路径可以是 ZIP 档案路径。
    - 此模块添加了从 ZIP 格式档案中导入 Python 模块（ *.py ， *.pyc ）和包的能力。通常不需要明确地使用 zipimport 模块，内置的 import 机制会自动将此模块用于 ZIP 档案路径的 sys.path 项目上。
    - 通常， sys.path 是字符串的目录名称列表。此模块同样允许 sys.path 的一项成为命名 ZIP 文件档案的字符串。 ZIP 档案可以容纳子目录结构去支持包的导入，并且可以将归档文件中的路径指定为仅从子目录导入。比如说，路径 **example.zip/lib/** 将只会从档案中的 lib/ 子目录导入。
    - 任何文件都可以存在于 ZIP档案之中，但是只有 .py 和 .pyc 文件是能够导入的。不允许导入 ZIP 中的动态模组（ .pyd ， .so ）。请注意，如果档案中只包含 .py 文件， Python不会尝试通过添加对应的 .pyc 文件修改档案，意思是如果 ZIP 档案不包含 .pyc 文件，导入或许会变慢。

4. 将文件系统和标准流的编码名称规范化。为处理文件系统时的编码和解码设置错误处理程序。

5. 加载默认信号处理器signal handlers。当一个进程收到类似SIGINT的信号时，信号处理程序会被执行。可以使用[signal](https://docs.python.org/3/library/signal.html)模块设置自定义信号处理器。

6. 导入io模块并初始化sys.stdin，sys.stdout，sys.stderr.这基本上是通过调用io.open()完成的。

7. 设置`builtins.open`为`io.OpenWrapper`，以便`open()`作为内置函数提供。

8. 创建`__main__`模块，设置`__main__.__builtins__`为`builtins`,设置`__main__.__loader__`为 `_frozen_importlib.BuiltinImporter`。此时，`__main__`模块不包含任何其他内容。

9. 导入[warnings](https://docs.python.org/3/library/warnings.html)和[site](https://docs.python.org/3/library/site.html)模块。`site`模块将特定的目录添加到 `sys.path`。这就是为什么`sys.path`通常包含一个已安装模块目录，比如`/usr/local/lib/python3.9/site-packages/`。

10. 设置`interp->runtime->initialized = 1`

CPython 的初始化已完成。pymain_init()函数返回，我们进入Py_RunMain()查看 CPython 在进入评估循环之前还能做什么。

# 运行python程序
Py_RunMain()函数似乎不是发生操作的地方：
```c
Py_RunMain(void)
{
    int exitcode = 0;

    pymain_run_python(&exitcode);

    if (Py_FinalizeEx() < 0) {
        /* Value unlikely to be confused with a non-error exit status or
           other special meaning */
        exitcode = 120;
    }

    // Free the memory that is not freed by Py_FinalizeEx()
    pymain_free();

    if (_Py_UnhandledKeyboardInterrupt) {
        exitcode = exit_sigint();
    }

    return exitcode;
}
``` 
首先，Py_RunMain()调用pymain_run_python()运行Python。其次，调用Py_FinalizeEx()撤消初始化undo the initialization。Py_FinalizeEx()释放了CPython能够释放的大部分内存，其余的则由pymain_free()释放。结束 CPython 的另一个重要步骤是调用退出函数，包括在 [atexit](https://docs.python.org/3/library/atexit.html)模块注册的函数。

如您所知，python有许多运行方式，即：

- interactively 交互：
```shell
$ ./cpython/python.exe
>>> import sys
>>> sys.path[:1]
['']
``` 
- from stdin：
```shell
$ echo "import sys; print(sys.path[:1])" | ./cpython/python.exe
['']
```
- as a command：
```shell
$ ./cpython/python.exe -c "import sys; print(sys.path[:1])"
['']
``` 
- as a script:
```shell
$ ./cpython/python.exe 03/print_path0.py
['/Users/Victor/Projects/tenthousandmeters/python_behind_the_scenes/03']
``` 
- as a module 作为模块：
```shell
$ ./cpython/python.exe -m 03.print_path0
['/Users/Victor/Projects/tenthousandmeters/python_behind_the_scenes']
``` 
- 和，不太明显的，包作为一个脚本（print_path0_package是一个带有__main__.py的目录）：
```shell
$ ./cpython/python.exe 03/print_path0_package
['/Users/Victor/Projects/tenthousandmeters/python_behind_the_scenes/03/print_path0_package']
``` 
我从`cpython/`目录向上移动了一个级别，以显示不同的调用模式导致不同的`sys.path[0]`值。下一个函数`pymain_run_python()`计算sys.path[0]值，并添加到sys.path，并根据`config`以适当的模式运行Python：
```c
static void
pymain_run_python(int *exitcode)
{
    PyInterpreterState *interp = _PyInterpreterState_GET();
    PyConfig *config = (PyConfig*)_PyInterpreterState_GetConfig(interp);

    // Prepend the search path to `sys.path`
    PyObject *main_importer_path = NULL;
    if (config->run_filename != NULL) {
        // Calculate the search path for the case when the filename is a package
        // (ex: directory or ZIP file) which contains __main__.py, store it in `main_importer_path`.
        // Otherwise, left `main_importer_path` unchanged.
        // Handle other cases later.
        if (pymain_get_importer(config->run_filename, &main_importer_path,
                                exitcode)) {
            return;
        }
    }

    if (main_importer_path != NULL) {
        if (pymain_sys_path_add_path0(interp, main_importer_path) < 0) {
            goto error;
        }
    }
    else if (!config->isolated) {
        PyObject *path0 = NULL;
        // Compute the search path that will be prepended to `sys.path` for other cases.
        // If running as script, then it's the directory where the script is located.
        // If running as module (-m), then it's the current working directory.
        // Otherwise, it's an empty string.
        int res = _PyPathConfig_ComputeSysPath0(&config->argv, &path0);
        if (res < 0) {
            goto error;
        }

        if (res > 0) {
            if (pymain_sys_path_add_path0(interp, path0) < 0) {
                Py_DECREF(path0);
                goto error;
            }
            Py_DECREF(path0);
        }
    }

    PyCompilerFlags cf = _PyCompilerFlags_INIT;

    // Print version and platform in the interactive mode
    pymain_header(config);
    // Import `readline` module to provide completion,
    // line editing and history capabilities in the interactive mode
    pymain_import_readline(config);

    // Run Python depending on the mode of invocation (script, -m, -c, etc.)
    if (config->run_command) {
        *exitcode = pymain_run_command(config->run_command, &cf);
    }
    else if (config->run_module) {
        *exitcode = pymain_run_module(config->run_module, 1);
    }
    else if (main_importer_path != NULL) {
        *exitcode = pymain_run_module(L"__main__", 0);
    }
    else if (config->run_filename != NULL) {
        *exitcode = pymain_run_file(config, &cf);
    }
    else {
        *exitcode = pymain_run_stdin(config, &cf);
    }

    // Enter the interactive mode after executing a program.
    // Enabled by `-i` and `PYTHONINSPECT`.
    pymain_repl(config, &cf, exitcode);
    goto done;

error:
    *exitcode = pymain_exit_err_print();

done:
    Py_XDECREF(main_importer_path);
}
``` 
我们不会讨论所有运行方式，但假设我们运行 Python 脚本程序。这会调用pymain_run_file()来检查指定的文件是否可以打开，确保它不是一个目录和调用PyRun_AnyFileExFlags()。当文件是终端（([isatty(fd)](https://man7.org/linux/man-pages/man3/isatty.3.html) returns 1)）时，该PyRun_AnyFileExFlags()处理特殊情况。如果是这样的话，它进入交互模式：
```shell
$ ./python.exe /dev/ttys000
>>> 1 + 1
2
``` 
否则，它调用PyRun_SimpleFileExFlags()。您应该熟悉在__pycache__目录中不断增加的.pyc文件。.pyc文件包含 Python 程序的已编译的源代码，即模块的编组代码对象。由于.pyc文件，我们不需要每次导入模块时都重新编译模块。我想你知道，但你知道有可能直接运行一个.pyc文件吗？.
```shell
$ ./cpython/python.exe 03/__pycache__/print_path0.cpython-39.pyc
['/Users/Victor/Projects/tenthousandmeters/python_behind_the_scenes/03/__pycache__']
``` 
`PyRun_SimpleFileExFlags()`函数实现此逻辑。它检查文件是否是一个`.pyc`文件，它是否为当前的CPython版本编译，如果是，调用`run_pyc_file()`。如果文件不是`.pyc`文件，它会调用`PyRun_FileExFlags()`。然而，最重要的是，`PyRun_SimpleFileExFlags()`将`__main__`模块导入并传递`__main__`的字典到`PyRun_FileExFlags()`，作为执行文件的全局和本地命名空间：
```c
int
PyRun_SimpleFileExFlags(FILE *fp, const char *filename, int closeit,
                        PyCompilerFlags *flags)
{
    PyObject *m, *d, *v;
    const char *ext;
    int set_file_name = 0, ret = -1;
    size_t len;

    m = PyImport_AddModule("__main__");
    if (m == NULL)
        return -1;
    Py_INCREF(m);
    d = PyModule_GetDict(m);

    if (PyDict_GetItemString(d, "__file__") == NULL) {
        PyObject *f;
        f = PyUnicode_DecodeFSDefault(filename);
        if (f == NULL)
            goto done;
        if (PyDict_SetItemString(d, "__file__", f) < 0) {
            Py_DECREF(f);
            goto done;
        }
        if (PyDict_SetItemString(d, "__cached__", Py_None) < 0) {
            Py_DECREF(f);
            goto done;
        }
        set_file_name = 1;
        Py_DECREF(f);
    }

    // Check if a .pyc file is passed
    len = strlen(filename);
    ext = filename + len - (len > 4 ? 4 : 0);
    if (maybe_pyc_file(fp, filename, ext, closeit)) {
        FILE *pyc_fp;
        /* Try to run a pyc file. First, re-open in binary */
        if (closeit)
            fclose(fp);
        if ((pyc_fp = _Py_fopen(filename, "rb")) == NULL) {
            fprintf(stderr, "python: Can't reopen .pyc file\n");
            goto done;
        }

        if (set_main_loader(d, filename, "SourcelessFileLoader") < 0) {
            fprintf(stderr, "python: failed to set __main__.__loader__\n");
            ret = -1;
            fclose(pyc_fp);
            goto done;
        }
        v = run_pyc_file(pyc_fp, filename, d, d, flags);
    } else {
        /* When running from stdin, leave __main__.__loader__ alone */
        if (strcmp(filename, "<stdin>") != 0 &&
            set_main_loader(d, filename, "SourceFileLoader") < 0) {
            fprintf(stderr, "python: failed to set __main__.__loader__\n");
            ret = -1;
            goto done;
        }
        v = PyRun_FileExFlags(fp, filename, Py_file_input, d, d,
                              closeit, flags);
    }
    flush_io();
    if (v == NULL) {
        Py_CLEAR(m);
        PyErr_Print();
        goto done;
    }
    Py_DECREF(v);
    ret = 0;
  done:
    if (set_file_name) {
        if (PyDict_DelItemString(d, "__file__")) {
            PyErr_Clear();
        }
        if (PyDict_DelItemString(d, "__cached__")) {
            PyErr_Clear();
        }
    }
    Py_XDECREF(m);
    return ret;
}
``` 
`PyRun_FileExFlags()`函数开始编译过程。它运行解析器parser，返回模块的 AST，并调用`run_mod()`以运行 AST。它还创建一个`PyArena`对象，[CPython 用它来分配小对象](https://www.evanjones.ca/memoryallocator/)（小于等于 512 字节）：
```c
PyObject *
PyRun_FileExFlags(FILE *fp, const char *filename_str, int start, PyObject *globals,
                  PyObject *locals, int closeit, PyCompilerFlags *flags)
{
    PyObject *ret = NULL;
    mod_ty mod;
    PyArena *arena = NULL;
    PyObject *filename;
    int use_peg = _PyInterpreterState_GET()->config._use_peg_parser;

    filename = PyUnicode_DecodeFSDefault(filename_str);
    if (filename == NULL)
        goto exit;

    arena = PyArena_New();
    if (arena == NULL)
        goto exit;

    // Run the parser.
    // By default the new PEG parser is used.
    // Pass `-X oldparser` to use the old parser.
    // `mod` stands for module. It's the root node of the AST.
    if (use_peg) {
        mod = PyPegen_ASTFromFileObject(fp, filename, start, NULL, NULL, NULL,
                                        flags, NULL, arena);
    }
    else {
        mod = PyParser_ASTFromFileObject(fp, filename, NULL, start, 0, 0,
                                         flags, NULL, arena);
    }

    if (closeit)
        fclose(fp);
    if (mod == NULL) {
        goto exit;
    }
    // Compile the AST and run.
    ret = run_mod(mod, filename, globals, locals, flags, arena);

exit:
    Py_XDECREF(filename);
    if (arena != NULL)
        PyArena_Free(arena);
    return ret;
}
``` 
`run_mod()`通过调用`PyAST_CompileObject()`运行编译器，返回模块的代码对象，并调用`run_eval_code_obj()`来执行代码对象。在此期间，它提出了`exec`事件，这是一个CPython的通知审计工具的方式，当重要的事情发生在Python运行时中时。[PEP 578](https://www.python.org/dev/peps/pep-0578/)解释了此机制。
```c
static PyObject *
run_mod(mod_ty mod, PyObject *filename, PyObject *globals, PyObject *locals,
            PyCompilerFlags *flags, PyArena *arena)
{
    PyThreadState *tstate = _PyThreadState_GET();
    PyCodeObject *co = PyAST_CompileObject(mod, filename, flags, -1, arena);
    if (co == NULL)
        return NULL;

    if (_PySys_Audit(tstate, "exec", "O", co) < 0) {
        Py_DECREF(co);
        return NULL;
    }

    PyObject *v = run_eval_code_obj(tstate, co, globals, locals);
    Py_DECREF(co);
    return v;
}
```

我们已经从[第二篇文章](https://tenthousandmeters.com/blog/python-behind-the-scenes-2-how-the-cpython-compiler-works/)知道，编译器的工作原理是：
1. 构建符号表
2. 创建基本块的CFG;和
3. 将CFG组装成代码对象。

这正是PyAST_CompileObject()做的，所以我们现在不会专注于它。

`run_eval_code_obj()`开始一系列琐碎的函数调用，最终会到`_PyEval_EvalCode()`。我在这里粘贴所有这些函数，只是为了让您看到`_PyEval_EvalCode()`参数的来源：
```c
static PyObject *
run_eval_code_obj(PyThreadState *tstate, PyCodeObject *co, PyObject *globals, PyObject *locals)
{
    PyObject *v;
    // The special case when CPython is embeddded. We can safely ignore it.
    /*
     * We explicitly re-initialize _Py_UnhandledKeyboardInterrupt every eval
     * _just in case_ someone is calling into an embedded Python where they
     * don't care about an uncaught KeyboardInterrupt exception (why didn't they
     * leave config.install_signal_handlers set to 0?!?) but then later call
     * Py_Main() itself (which _checks_ this flag and dies with a signal after
     * its interpreter exits).  We don't want a previous embedded interpreter's
     * uncaught exception to trigger an unexplained signal exit from a future
     * Py_Main() based one.
     */
    _Py_UnhandledKeyboardInterrupt = 0;

    /* Set globals['__builtins__'] if it doesn't exist */
    // In our case, it's been already set to the `builtins` module during the main initialization.
    if (globals != NULL && PyDict_GetItemString(globals, "__builtins__") == NULL) {
        if (PyDict_SetItemString(globals, "__builtins__",
                                 tstate->interp->builtins) < 0) {
            return NULL;
        }
    }

    v = PyEval_EvalCode((PyObject*)co, globals, locals);
    if (!v && _PyErr_Occurred(tstate) == PyExc_KeyboardInterrupt) {
        _Py_UnhandledKeyboardInterrupt = 1;
    }
    return v;
}
```
```c
PyObject *
PyEval_EvalCode(PyObject *co, PyObject *globals, PyObject *locals)
{
    return PyEval_EvalCodeEx(co,
                      globals, locals,
                      (PyObject **)NULL, 0,
                      (PyObject **)NULL, 0,
                      (PyObject **)NULL, 0,
                      NULL, NULL);
}
```
```c
PyEval_EvalCodeEx(PyObject *_co, PyObject *globals, PyObject *locals,
                  PyObject *const *args, int argcount,
                  PyObject *const *kws, int kwcount,
                  PyObject *const *defs, int defcount,
                  PyObject *kwdefs, PyObject *closure)
{
    return _PyEval_EvalCodeWithName(_co, globals, locals,
                                    args, argcount,
                                    kws, kws != NULL ? kws + 1 : NULL,
                                    kwcount, 2,
                                    defs, defcount,
                                    kwdefs, closure,
                                    NULL, NULL);
}
```
```c
PyObject *
_PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals,
           PyObject *const *args, Py_ssize_t argcount,
           PyObject *const *kwnames, PyObject *const *kwargs,
           Py_ssize_t kwcount, int kwstep,
           PyObject *const *defs, Py_ssize_t defcount,
           PyObject *kwdefs, PyObject *closure,
           PyObject *name, PyObject *qualname)
{
    PyThreadState *tstate = _PyThreadState_GET();
    return _PyEval_EvalCode(tstate, _co, globals, locals,
               args, argcount,
               kwnames, kwargs,
               kwcount, kwstep,
               defs, defcount,
               kwdefs, closure,
               name, qualname);
}
```

回想一下，代码对象描述代码，但要执行代码对象，CPython 需要为它创建状态，这就是帧对象。 _PyEval_EvalCode()为具有指定参数的给定代码对象创建帧对象。在我们的例子中，大多数参数是NULL，所以需要做的很少。例如，当 CPython 执行带有多个不同类型的参数的函数代码对象时，需要做更多的工作。因此，_PyEval_EvalCode()是近300行长。在接下来的部分，我们将看到其中大多数是做什么的。现在，您可以跳过_PyEval_EvalCode()去确定最终它调用_PyEval_EvalFrame()来评估创建的帧对象：
```c
PyObject *
_PyEval_EvalCode(PyThreadState *tstate,
           PyObject *_co, PyObject *globals, PyObject *locals,
           PyObject *const *args, Py_ssize_t argcount,
           PyObject *const *kwnames, PyObject *const *kwargs,
           Py_ssize_t kwcount, int kwstep,
           PyObject *const *defs, Py_ssize_t defcount,
           PyObject *kwdefs, PyObject *closure,
           PyObject *name, PyObject *qualname)
{
    assert(is_tstate_valid(tstate));

    PyCodeObject* co = (PyCodeObject*)_co;
    PyFrameObject *f;
    PyObject *retval = NULL;
    PyObject **fastlocals, **freevars;
    PyObject *x, *u;
    const Py_ssize_t total_args = co->co_argcount + co->co_kwonlyargcount;
    Py_ssize_t i, j, n;
    PyObject *kwdict;

    if (globals == NULL) {
        _PyErr_SetString(tstate, PyExc_SystemError,
                         "PyEval_EvalCodeEx: NULL globals");
        return NULL;
    }

    /* Create the frame */
    f = _PyFrame_New_NoTrack(tstate, co, globals, locals);
    if (f == NULL) {
        return NULL;
    }
    fastlocals = f->f_localsplus;
    freevars = f->f_localsplus + co->co_nlocals;

    /* Create a dictionary for keyword parameters (**kwags) */
    if (co->co_flags & CO_VARKEYWORDS) {
        kwdict = PyDict_New();
        if (kwdict == NULL)
            goto fail;
        i = total_args;
        if (co->co_flags & CO_VARARGS) {
            i++;
        }
        SETLOCAL(i, kwdict);
    }
    else {
        kwdict = NULL;
    }

    /* Copy all positional arguments into local variables */
    if (argcount > co->co_argcount) {
        n = co->co_argcount;
    }
    else {
        n = argcount;
    }
    for (j = 0; j < n; j++) {
        x = args[j];
        Py_INCREF(x);
        SETLOCAL(j, x);
    }

    /* Pack other positional arguments into the *args argument */
    if (co->co_flags & CO_VARARGS) {
        u = _PyTuple_FromArray(args + n, argcount - n);
        if (u == NULL) {
            goto fail;
        }
        SETLOCAL(total_args, u);
    }

    /* Handle keyword arguments passed as two strided arrays */
    kwcount *= kwstep;
    for (i = 0; i < kwcount; i += kwstep) {
        PyObject **co_varnames;
        PyObject *keyword = kwnames[i];
        PyObject *value = kwargs[i];
        Py_ssize_t j;

        if (keyword == NULL || !PyUnicode_Check(keyword)) {
            _PyErr_Format(tstate, PyExc_TypeError,
                          "%U() keywords must be strings",
                          co->co_name);
            goto fail;
        }

        /* Speed hack: do raw pointer compares. As names are
           normally interned this should almost always hit. */
        co_varnames = ((PyTupleObject *)(co->co_varnames))->ob_item;
        for (j = co->co_posonlyargcount; j < total_args; j++) {
            PyObject *name = co_varnames[j];
            if (name == keyword) {
                goto kw_found;
            }
        }

        /* Slow fallback, just in case */
        for (j = co->co_posonlyargcount; j < total_args; j++) {
            PyObject *name = co_varnames[j];
            int cmp = PyObject_RichCompareBool( keyword, name, Py_EQ);
            if (cmp > 0) {
                goto kw_found;
            }
            else if (cmp < 0) {
                goto fail;
            }
        }

        assert(j >= total_args);
        if (kwdict == NULL) {

            if (co->co_posonlyargcount
                && positional_only_passed_as_keyword(tstate, co,
                                                     kwcount, kwnames))
            {
                goto fail;
            }

            _PyErr_Format(tstate, PyExc_TypeError,
                          "%U() got an unexpected keyword argument '%S'",
                          co->co_name, keyword);
            goto fail;
        }

        if (PyDict_SetItem(kwdict, keyword, value) == -1) {
            goto fail;
        }
        continue;

      kw_found:
        if (GETLOCAL(j) != NULL) {
            _PyErr_Format(tstate, PyExc_TypeError,
                          "%U() got multiple values for argument '%S'",
                          co->co_name, keyword);
            goto fail;
        }
        Py_INCREF(value);
        SETLOCAL(j, value);
    }

    /* Check the number of positional arguments */
    if ((argcount > co->co_argcount) && !(co->co_flags & CO_VARARGS)) {
        too_many_positional(tstate, co, argcount, defcount, fastlocals);
        goto fail;
    }

    /* Add missing positional arguments (copy default values from defs) */
    if (argcount < co->co_argcount) {
        Py_ssize_t m = co->co_argcount - defcount;
        Py_ssize_t missing = 0;
        for (i = argcount; i < m; i++) {
            if (GETLOCAL(i) == NULL) {
                missing++;
            }
        }
        if (missing) {
            missing_arguments(tstate, co, missing, defcount, fastlocals);
            goto fail;
        }
        if (n > m)
            i = n - m;
        else
            i = 0;
        for (; i < defcount; i++) {
            if (GETLOCAL(m+i) == NULL) {
                PyObject *def = defs[i];
                Py_INCREF(def);
                SETLOCAL(m+i, def);
            }
        }
    }

    /* Add missing keyword arguments (copy default values from kwdefs) */
    if (co->co_kwonlyargcount > 0) {
        Py_ssize_t missing = 0;
        for (i = co->co_argcount; i < total_args; i++) {
            PyObject *name;
            if (GETLOCAL(i) != NULL)
                continue;
            name = PyTuple_GET_ITEM(co->co_varnames, i);
            if (kwdefs != NULL) {
                PyObject *def = PyDict_GetItemWithError(kwdefs, name);
                if (def) {
                    Py_INCREF(def);
                    SETLOCAL(i, def);
                    continue;
                }
                else if (_PyErr_Occurred(tstate)) {
                    goto fail;
                }
            }
            missing++;
        }
        if (missing) {
            missing_arguments(tstate, co, missing, -1, fastlocals);
            goto fail;
        }
    }

    /* Allocate and initialize storage for cell vars, and copy free
       vars into frame. */
    for (i = 0; i < PyTuple_GET_SIZE(co->co_cellvars); ++i) {
        PyObject *c;
        Py_ssize_t arg;
        /* Possibly account for the cell variable being an argument. */
        if (co->co_cell2arg != NULL &&
            (arg = co->co_cell2arg[i]) != CO_CELL_NOT_AN_ARG) {
            c = PyCell_New(GETLOCAL(arg));
            /* Clear the local copy. */
            SETLOCAL(arg, NULL);
        }
        else {
            c = PyCell_New(NULL);
        }
        if (c == NULL)
            goto fail;
        SETLOCAL(co->co_nlocals + i, c);
    }

    /* Copy closure variables to free variables */
    for (i = 0; i < PyTuple_GET_SIZE(co->co_freevars); ++i) {
        PyObject *o = PyTuple_GET_ITEM(closure, i);
        Py_INCREF(o);
        freevars[PyTuple_GET_SIZE(co->co_cellvars) + i] = o;
    }

    /* Handle generator/coroutine/asynchronous generator */
    if (co->co_flags & (CO_GENERATOR | CO_COROUTINE | CO_ASYNC_GENERATOR)) {
        PyObject *gen;
        int is_coro = co->co_flags & CO_COROUTINE;

        /* Don't need to keep the reference to f_back, it will be set
         * when the generator is resumed. */
        Py_CLEAR(f->f_back);

        /* Create a new generator that owns the ready to run frame
         * and return that as the value. */
        if (is_coro) {
            gen = PyCoro_New(f, name, qualname);
        } else if (co->co_flags & CO_ASYNC_GENERATOR) {
            gen = PyAsyncGen_New(f, name, qualname);
        } else {
            gen = PyGen_NewWithQualName(f, name, qualname);
        }
        if (gen == NULL) {
            return NULL;
        }

        _PyObject_GC_TRACK(f);

        return gen;
    }

    retval = _PyEval_EvalFrame(tstate, f, 0);

fail: /* Jump here from prelude on failure */

    /* decref'ing the frame can cause __del__ methods to get invoked,
       which can call back into Python.  While we're done with the
       current Python frame (f), the associated C stack is still in use,
       so recursion_depth must be boosted for the duration.
    */
    if (Py_REFCNT(f) > 1) {
        Py_DECREF(f);
        _PyObject_GC_TRACK(f);
    }
    else {
        ++tstate->recursion_depth;
        Py_DECREF(f);
        --tstate->recursion_depth;
    }
    return retval;
}
```

_PyEval_EvalFrame()是帧评估函数（the frame evaluation function）interp->eval_frame()的装饰器。interp->eval_frame()可以设置为自定义函数。为什么有人要这么做？例如，它允许将 JIT 编译器添加到 CPython 中，通过将the default evaluation function替换为将编译的机器代码存储在代码对象中并运行该函数的函数。[PEP 523](https://www.python.org/dev/peps/pep-0523/)在 CPython 3.6 中引入了此功能。

默认情况下，interp->eval_frame()设置为 _PyEval_EvalFrameDefault()。此函数定义在Python/ceval.c，近 3，000 行。然而，今天，我们只对一个感兴趣。1336行开始了我们等待了这么久的事情：the evaluation loop 评估循环。

# 结论

我们今天讨论了很多。我们首先概述了 CPython 项目，编译了 CPython，并研究其源代码，研究了初始化阶段。总的来说，我认为这应该让你了解在开始解释字节码之前CPython做什么。之后发生的事情是下一篇文章的主题。

同时，为了巩固我们今天学到的东西，并了解更多，我真的建议你找一些时间来探索CPython源代码自己。我敢打赌，你读完这篇文章后有很多问题，所以你应该有一些东西要找。