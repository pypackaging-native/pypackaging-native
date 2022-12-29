# Glossary

## Acronyms

| Acronym | ... stands for | Explanation |
|---|---|---|
| ABI | Application Binary Interface | See [here](./background/binary_interface.md) |
| API | Application Programming Interface | The sum total of available functions, classes, etc. of a given program |
| AVX | Advanced Vector eXtensions | Various extensions to the x86 instruction set (AVX, AVX2, AVX512), evolution after SSE |
| BLAS | Basic Linear Algebra Subprograms | Specification resp. implementation for low-level linear algebra routines |
| BOLT | Binary Optimization and Layout Tool | See [here](./background/compilation_concepts.md)|
| `cffi` | The C FFI for Python | |
| CI | Continuous Integration | Testing all changes made to a given software, resp. the infrastructure that makes this possible |
| CPU | Central Processing Unit | Main circuitry for executing machine instructions on a computer; contrast GPU |
| CUDA | Compute Unified Device Architecture | Parallel Computing Framework for nVidia GPUs |
| GCC | GNU Compiler Collection | Main compiler family (for C / C++ / Fortran etc.) on linux |
| GNU | GNU's Not Unix | Collection of free software packages under GPL License |
| GPL | (GNU) General Public License | Foundational "copyleft" license of the free software movement |
| glibc | GNU C Library | Widely used implemetation of the C standard library |
| FFI | Foreign Function Interface | Calling functions written in a different language than the one currently used |
| GPU | Graphics Processing Unit | Specialized circuitry for quickly computing graphics-related tasks |
| ILP64 | - | Name used for the standard 64-bit interface to BLAS/LAPACK. Also see "(64 bit) Data Models" below |
| IR | Intermediate Representation | Language-agnostic yet still semantic representation of code within a compiler |
| LAPACK | Linear Algebra PACKage | Standard software library for numerical linear algebra |
| JIT | Just-in-time Compilation | Compiling code just before execution; used in CUDA, pytorch, PyPy, numba etc. |
| LLVM | - | Cross-platform compiler framework, home of Clang, MLIR, BOLT etc. |
| LTO | Link-Time Optimization | See [here](./background/compilation_concepts.md)|
| LTS | Long-Term Support | Version of a given software/library/distribution designated for long-term support |
| musl | - | An alternative implementation of the C standard library |
| MPI | Message Passing Interface | Standard for message-passing in parallel computing |
| MLIR | Multi-Level IR | Higher-level IR within LLVM; used i.a. in machine learning frameworks |
| MSVC | Microsoft Visual C++ | Main compiler on windows |
| NEP | Numpy Enhancement Proposal | See [here](https://numpy.org/neps/) |
| OpenMP | Open Multi Processing | Multi-platform API for enabling multi-processing in C/C++/Fortran |
| PEP | Python Enhancement Proposal | See [here](https://peps.python.org/pep-0000/) |
| PGO | Profile-Guided Optimization | See [here](./background/compilation_concepts.md)|
| PSF | Python Software Foundation | See [here](https://www.python.org/psf-landing/) |
| PyPA | Python Packaging Authorithy | Group which maintains core set of projects in Python packaging |
| PyPI | Python Package Index | Main index where packages get installed from |
| PyPy | - | An implementation of the Python specification in (quasi-)Python, with JIT-capabilities |
| QEMU | Quick EMUlator | Predominant emulation framework on linux |
| RHEL | Redhat Enterprise Linux | Commercial distribution with some of the longest-running support timelines |
| ROCm | Radeon Open Compute | Software stack for AMD GPUs; comparable to CUDA |
| SIMD | Single Instruction, Multiple Data | CPU-specific instructions that can process more data in a single instruction |
| SSE | Streaming SIMD Extensions | Various extensions to the x86 instruction set (SSE, SSE2, SSE3, SSSE3, SSE4) for SIMD |
| UCRT | Universal C Runtime | Windows equivalent to glibc/musl | 

## Terms

| Term | Explanation | Examples / References |
|---|---|---|
| ABI Break | Failing to maintain the ABI | See [here](./background/compilation_concepts.md) |
| Binary Compatibility | Succeeding to maintain the ABI (e.g. across versions / upgrades) | See [here](./background/compilation_concepts.md)|
| Calling Convention | Agreed-upon contract with describes how to interact with a given CPU (family) | See [here](https://en.wikipedia.org/wiki/Calling_convention) |
| Cross-compilation | Compiling _on_ a given CPU architecture _for_  another CPU architecture | See [here](./background/compilation_concepts.md) |
| (64 bit) Data Models | Choice of bit-widths for `int`/`long` integer types | ILP32, ILP64, LP64; see [here](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models) |
| Distribution | An entity distributing (consistent) binary artefacts, often for a given platform | CentOS, Debian, Fedora, Ubuntu, RHEL...</br>Chocolatey, Conda, Spack, ... |
| Emulation | Pretending to run on a different CPU architecture; this can be used to avoid cross-compilation | See [here](https://en.wikipedia.org/wiki/Emulator) |
| Linker | A tool to correctly find the required third-party symbols for a given project | GNU's gold, LLVM's lld, [mold](https://github.com/rui314/mold) |
| Symbol | A compiled version of a function | See [here](./background/compilation_concepts.md) |
