# Glossary

## Acronyms

| Acronym | Explanation | References |
|---|---|---|
| ABI | Application Binary Interface | See [here](./background/binary_interface.md) |
| API | Application Programming Interface | The sum total of available functions, classes, etc. of a given program |
| BLAS | Basic Linear Algebra Subprograms | Specification resp. implementation for low-level linear algebra routines |
| BOLT | Binary Optimization and Layout Tool | See [here](./background/compilation_concepts.md)|
| `cffi` | The C FFI for Python | |
| CI | Continuous Integration | Testing all changes made to a given software, resp. the infrastructure that makes this possible |
| CPU | Central Processing Unit | Main circuitry for executing machine instructions on a computer; contrast GPU |
| CUDA | Compute Unified Device Architecture | Parallel Computing Framework for nVidia GPUs |
| GCC | GNU Compiler Collection | Main compiler family (for C / C++ / Fortran etc.) on linux |
| glibc | GNU C Library | Widely used implemetation of the C standard library |
| FFI | Foreign Function Interface | Calling functions written in a different language than the one currently used |
| GPU | Graphics Processing Unit | Specialized circuitry for quickly computing graphics-related tasks |
| ILP64 | - | Name used for the standard 64-bit interface to BLAS/LAPACK. Also see "(64 bit) Data Models" below |
| LAPACK | Linear Algebra PACKage | Standard software library for numerical linear algebra |
| LLVM | - | Cross-platform compiler framework, home of Clang, MLIR, BOLT etc. |
| LTO | Link-Time Optimization | See [here](./background/compilation_concepts.md)|
| LTS | Long-Term Support | Version of a given software/library/distribution designated for long-term support |
| musl | - | An alternative implementation of the C standard library |
| MPI | Message Passing Interface | Standard for message-passing in parallel computing |
| MSVC | Microsoft Visual C++ | Main compiler on windows |
| NEP | Numpy Enhancement Proposal | See [here](https://numpy.org/neps/) |
| OpenMP | Open Multi Processing | Multi-platform API for enabling multi-processing in C/C++/Fortran |
| PEP | Python Enhancement Proposal | See [here](https://peps.python.org/pep-0000/) |
| PGO | Profile-Guided Optimization | See [here](./background/compilation_concepts.md)|
| PSF | Python Software Foundation | See [here](https://www.python.org/psf-landing/) |
| PyPA | Python Packaging Authorithy | Group which maintains core set of projects in Python packaging |
| PyPI | Python Package Index | Main index where packages get installed from |
| QEMU | Quick EMUlator | Predominant emulation framework on linux |
| RHEL | Redhat Enterprise Linux | Commercial distribution with some of the longest-running support timelines |
| ROCm | Radeon Open Compute | Software stack for AMD GPUs; comparable to CUDA |
| SIMD | Single Instruction, Multiple Data | CPU-specific instructions that can process more data in a single instruction |
| UCRT | Universal C Runtime | Windows equivalent to glibc/musl | 

## Terms

| Term | Explanation | Examples |
|---|---|---|
| ABI Break | Failing to maintain the ABI | See [here](./background/compilation_concepts.md) |
| Binary Compatibility | Succeeding to maintain the ABI (e.g. across versions / upgrades) | See [here](./background/compilation_concepts.md)|
| Calling Convention | Agreed-upon contract with describes how to interact with a given CPU (family) | |
| Cross-compilation | Compiling _on_ a given CPU architecture _for_  another CPU architecture | See [here](./background/compilation_concepts.md) |
| (64 bit) Data Models | Choice of bit-widths for `int`/`long` integer types | See [here](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models) |
| Distribution | An entity distributing (consistent) binary artefacts, often for a given platform | CentOS, Debian, Fedora, Ubuntu, RHEL...</br>Chocolatey, Conda, Spack, ... |
| Emulation | Pretending to run on a different CPU architecture; this can be used to circumvent cross-compilation | |
| Symbol | A compiled version of a function | See [here](./background/compilation_concepts.md) |
