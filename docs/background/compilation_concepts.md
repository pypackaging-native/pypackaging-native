# Basic code compilation concepts

??? question "What does this have to do with Python?"

    Python as a glue language is essentially exposed to the sum of all problems
    that other languages have with their binary distribution, and this is a
    really vast field (even just for C/C++).

    Needless to say, adequately summarizing decades of work and developments
    that have led us to where we are now is not easy. If you find errors or
    things to improve, please open an issue or PR!

In order to get a computer to execute a given unit of work (say, application `X`
calling a function `f` from a previously compiled library `libfoo`), a lot of
preconditions have to be met:

- The function needs to have been compiled into instructions that the
  computer understands; a _symbol_.
- The symbol for `f` needs to be named (resp. "mangled") in a consistent manner
  between the compilation of the current code (for `X`), resp. the compilation
  of the library (`libfoo`, which contains the symbol for `f`).
- The symbol for `f` needs to be discoverable from within the currently
  running process; assuming `libfoo` is available on the machine where we are
  compiling, this is ensured by the _linker_.
- Variables passed to the function need to be loaded into the right CPU
  registers; this is highly dependent on the _calling convention_ of a given
  CPU, or rather, CPU family.
- The code (in `X`) calling a given symbol (e.g. for `f`) needs to be
  excruciatingly compatible with the actual implementation of that symbol
  (in `libfoo`).

It's not useful for the average programmer to consider this level of detail
when trying to get work done, but it is unfortunately unavoidable when
considering the realities of packaging and distributing software as
pre-compiled binary artifacts.


## Symbols, mangling and linkers

### Symbols

For any given function `f` you might have written in a language that needs to
be compiled (C, C++, ...), you can consider a symbol as a translation of your
function to a level that can actually be executed by your computer.

So, for example, a simple square function
```C
int square(int num) {
    return num * num;
}
```

will be [translated](https://godbolt.org/z/WT5Wa4bhq) to the following assembly
on x86-64 hardware:
```as
square:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     eax, DWORD PTR [rbp-4]
        imul    eax, eax
        pop     rbp
        ret
```
There is a lot of ceremony for reading in the argument into a register, setting
up another register for the result, while the actual work is done in:
```as
        imul    eax, eax
```
If you click the link above, you will see that this assembly looks completely
different when compiled for another processor architecture (e.g. arm64, as used
in Apple M1 and newer).

### Symbol name mangling

Note as well that, in C, the symbol `square` has exactly the same name as the
function - there is a 1:1 relationship, or in other words, there is no "mangling"
of the symbol name. This is because C has no concept of overloading functions
(i.e. having different functions of the same name but different signatures).

It also means that you can never change anything about a C symbol that has been
distributed to consumers in binary form. We return to this in the
[background content about ABI](./binary_interface.md).

In C++, the same function name can have several different signatures; for
example
```C++
template<class T>
int square(T num) {
    return num * num;
}
```
allows us to call `square` with all kinds of integers, floats, etc. The compiler
will keep track of which flavor of the function has been used in the program
and generate the respective symbols for each one of them. In order to
distinguish these symbols, they get names that look like gibberish, but really
simply bake the types of their input arguments into the identifier, so that we
can ensure the right symbol gets called when we actually execute the function.
For more details about the most widespread convention about this, see [here](https://github.com/itanium-cxx-abi/cxx-abi).

### Linkers

When building an executable or a library, any code that references functions
from third-party libraries needs to be resolved to the respective symbols, and
those need to be found and copied into the executable, or alternatively, loaded
at runtime.

The tool for this job is called the linker, and it's a hopefully invisible task,
at least, until things break (symbols not found, etc.). For an in-depth
introduction to linkers see [here](https://lwn.net/Articles/276782/). Note that
the field has evolved a lot since then, and new-generation linkers have appeared
(see e.g. [mold](https://github.com/rui314/mold)), but the basic operations have
remained essentially unchanged.

One crucial aspect about the way things have historically grown is that there
is no name-spacing of these symbols in any way. All symbols are fully global,
and this brings a lot of constraints. The linker will search for a given symbol
within different paths on the system, in order, but this is obviously very
fragile, in case symbols appear in several libraries on the linker path.

### Key take-aways

- Functions are compiled into symbols.
- Symbol names are 1:1 with function names in C, mangled according to their
  signature in C++.
- These symbols share a global name space.
- Symbols are picked by the linker in order of precedence on the path.


## Shared vs. static libraries

From a high-level point of view, libraries are collections of symbols, and can
come in two different flavors: static and dynamic. In very crude terms, static
libraries are only useful at compile time (i.e. compiling an app `X` against a
static library `libfoo` will copy the symbols from `libfoo` required by `X` into
the final artifact), whereas for dynamic libraries, the symbols will only be
_referenced_, and then loaded from the dynamic library at runtime.

As a rough overview:

| | Static `libfoo` | Dynamic `libfoo` |
|---|---|---|
| Compiling app `X` | Copies required symbols from `libfoo`<br/>into final artifact for `X` | References required symbols from `libfoo` |
| Running app `X` | - | Needs to load required symbols from `libfoo` |
| Storage footprints | O(#libs) for libraries using a given symbol | Symbol only saved once |
| Binary Interface | Self-contained | Sensitive to ABI breaks |

The trade-offs involved here are very painful. Either we duplicate symbols for
every artifact that uses them (and the bloat can be extreme, e.g. statically
linking the MSVC runtime[^1] can increase storage / memory footprint by 100MB),
or we face the constraints of subjecting us to ABI stability, and finding the
right symbols at runtime. For a further deep-dive into how Microsoft evolves its
UCRT (Universal C Runtime), see [here](https://mingwpy.github.io/ucrt.html).

[^1]:
    This is an extreme example, but it is used in the wild, see e.g. [protobuf](https://github.com/protocolbuffers/protobuf/blob/v21.12/CMakeLists.txt#L70).
    Note that Microsoft itself [discourages](https://devblogs.microsoft.com/cppblog/introducing-the-universal-crt/)
    this pattern, but still supports it:
    > We strongly recommend against static linking of the Visual C++ libraries,
    > for both performance and serviceability reasons, but we recognize that
    > there are some use cases that require static libraries and we will continue
    > to support the static libraries for those reasons.

In general, large parts of the wider computing ecosystem have found the
duplication inherent in static libraries unacceptable, and would rather deal
with the constraints of ABI stability. For example, standard functions like
`printf` (or standard mathematical functions, or ...) are used so pervasively
that copying those symbols into every produced binary would be extremely
wasteful.

It's worth noting that on Windows, symbols in shared libraries work quite
differently than on unix. Due to various reasons, symbols on Windows have to be
explicitly marked `__declspec(dllexport)` & `__declspec(dllimport)` in the
source with a macro that switches appropriately. This is quite an intrusive
change for libraries aiming to be cross-platform, and the reason that several
libraries developed primarily on unix do not support shared builds on Windows.
Even using workarounds like `CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS` still doesn't
cover all necessary symbols (e.g. global static data members).

Using the latter may also have a performance impact, as it keeps the compiler
from inlining calls to symbols that have been exported.

### Key take-aways

- Static libraries are only useful at compile-time; they cause duplication of
  symbols, but are much less susceptible to the intricacies of ABIs.
- Shared libraries need to be available both at compilation as well as at runtime,
  solve the symbol duplication, but are extremely susceptible to ABI breaks.
- Due to the global symbol namespace, there can only be one version / build of
  a shared library per environment (unless there is explicit versioning for the
  symbols or libraries themselves, or for C++, explicitly different inline
  namespaces are used).


## Foreign-Function Interface (FFI)

For any given task, a function may already exist in a library written in another
language than the one at hand. In order to avoid rewriting (and maintaining)
that functionality, it's desirable to have a way to call such functions from a
"foreign" language. A common example are the BLAS/LAPACK routines, which are
written in Fortran, but provide a C interface in addition to the Fortran one.

In addition to the considerations above, this needs to ensure that the types
between the language of the callee and the types of the language that the
function is implemented in are transformed correctly.


## Transpilation

Many Python projects do not deal with C/C++ directly, but use transpilers like
Cython or Pythran that _generate_ C resp. C++ code from (quasi-)Python source
code. Aside from almost always being exposed to the NumPy C API & ABI, these
modules compile into shared libraries themselves, with all the caveats
mentioned above that this implies. However, few projects expose their
cythonized functions as a C API, so there are generally fewer concerns about
ABI stability in this scenario.


## Cross-compilation

Many projects use public CI services, which might not offer some of the more
exotic or emerging architectures a project wants to support. This means that
publishing binary artifacts for such architectures can often only be achieved
with cross-compilation, i.e. compiling _on_ one architecture (e.g., x86-64),
_for_ another (e.g., aarch64).

A recent example where this was necessary at scale was the introduction of a
new processor architecture for Apple's M1 notebooks, for which (almost) no CI
agents were available. Distributors of packages for `linux-aarch64`,
`linux-ppc64le` or `windows-arm64` are often in similar situations.

The difficulty in cross-compilation is that it needs further attention to and
separation (both conceptually, as well as in metadata) of the different
requirements for the build environment (e.g. the necessary build tools that
are executed on the x86-64 agent), as well as the host environment (i.e. the
libraries that need to match the target architecture).

Additionally, many build procedures assume they can execute arbitrary code
(e.g. code generation) on the same architecture as the host, which is not a
given in this case and may need to be worked around.


## Performance optimization

### Compiler flags

TODO: Optimization levels; inlining functions; problems with `-Ofast`

For `-Ofast`, see in particular:
https://github.com/conda-forge/conda-forge.github.io/issues/1824

### Link-Time Optimization (LTO)

In the search for speed, it's possible to do a whole-program analysis after
everything has been compiled, and let the compiler identify which functions
can be inlined. This is generally out of scope for Python packaging, because
it is too involved. However, CPython release builds [make use](https://github.com/python/cpython/blob/main/README.rst#link-time-optimization)
of it.

### Profile-Guided Optimization (PGO)

If a program is instrumented (compiled with tracking capabilities) to profile
common usage patterns, it is possible to optimize the layout of the final
artifact to ensure that hot paths get preferred in branch prediction. This is
generally out of scope for Python packaging, because it is too involved. However,
CPython release builds [make use](https://github.com/python/cpython/blob/main/README.rst#profile-guided-optimization)
of it.

### Binary Optimization and Layout Tool (BOLT)

Further optimization of the produced binary artifacts can be achieved by
arranging their layout to avoid cache misses under profiled behavior. The
respective [tool](https://github.com/llvm/llvm-project/blob/main/bolt/README.md)
is based on LLVM and still under heavy development, and not suitable for all
usecases. It is generally out of scope for Python packaging, because it is too
involved. However, CPython [added](https://github.com/python/cpython/blob/main/Doc/using/configure.rst#performance-options)
experimental support as of 3.12.
