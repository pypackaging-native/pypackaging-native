# Glossary

## Acronyms

| Acronym | ... stands for | Explanation |
|---|---|---|
| ABI | Application Binary Interface | See [here](./background/binary_interface.md) |
| API | Application Programming Interface | The sum total of available functions, classes, etc. of a given program |
| ARM | Advanced RISC Machines | Family of RISC architectures, second-most widely used processor family after x86 |
| AVX | Advanced Vector eXtensions | Various extensions to the x86 instruction set (AVX, AVX2, AVX512), evolution after SSE |
| BLAS | Basic Linear Algebra Subprograms | Specification resp. implementation for low-level linear algebra routines |
| BOLT | Binary Optimization and Layout Tool | See [here](./background/compilation_concepts.md#binary-optimization-and-layout-tool-bolt)|
| `cffi` | The C FFI for Python | See [here](https://cffi.readthedocs.io) |
| CI | Continuous Integration | Testing all changes made to a given software, resp. the infrastructure that makes this possible |
| CLI | Command Line Interface | |
| CPU | Central Processing Unit | Main circuitry for executing machine instructions on a computer; contrast GPU |
| CRAN | Comprehensive R Archive Network | Main index for R language packages, comparable to PyPI |
| CUDA | Compute Unified Device Architecture | Parallel Computing Framework for NVIDIA GPUs |
| DRY | Don't Repeat Yourself | Principle in software development aimed at reducing repetition |
| GCC | GNU Compiler Collection | Main compiler family (for C / C++ / Fortran etc.) on Linux |
| GUI | Graphical UI | |
| GNU | GNU's Not Unix | Collection of free software packages under GPL License |
| GPL | (GNU) General Public License | Foundational "copyleft" license of the free software movement |
| glibc | GNU C Library | Widely used implemetation of the C standard library |
| FFI | Foreign Function Interface | Calling functions written in a different language than the one currently used |
| GPU | Graphics Processing Unit | Specialized circuitry for quickly computing graphics-related tasks |
| ILP64 | - | Name used for the standard 64-bit interface to BLAS/LAPACK. Also see "(64 bit) Data Models" below |
| IR | Intermediate Representation | Language-agnostic yet still semantic representation of code within a compiler |
| LAPACK | Linear Algebra PACKage | Standard software library for numerical linear algebra |
| ISA | Instruction Set Architecture | Specification of an instruction set for a CPU; e.g. x86-64, arm64, ... |
| JIT | Just-in-time Compilation | Compiling code just before execution; used in CUDA, PyTorch, PyPy, Numba etc. |
| LLVM | - | Cross-platform compiler framework, home of Clang, MLIR, BOLT etc. |
| LTO | Link-Time Optimization | See [here](./background/compilation_concepts.md#link-time-optimization-lto)|
| LTS | Long-Term Support | Version of a given software/library/distribution designated for long-term support |
| musl | - | An alternative implementation of the C standard library |
| MPI | Message Passing Interface | Standard for message-passing in parallel computing |
| MLIR | Multi-Level IR | Higher-level IR within LLVM; used i.a. in machine learning frameworks |
| MSVC | Microsoft Visual C++ | Main compiler on Windows |
| NEP | Numpy Enhancement Proposal | See [here](https://numpy.org/neps/) |
| OpenMP | Open Multi Processing | Multi-platform API for enabling multi-processing in C/C++/Fortran |
| OS | Operating System | E.g. Linux, MacOS, Windows |
| PEP | Python Enhancement Proposal | See [here](https://peps.python.org/pep-0000/) |
| `pip` | Pip Installs Packages | Default installer for Python packages, distributed with CPython itself; see [here](https://pip.pypa.io/en/stable/) |
| PGO | Profile-Guided Optimization | See [here](./background/compilation_concepts.md#profile-guided-optimization-pgo)|
| PSF | Python Software Foundation | See [here](https://www.python.org/psf-landing/) |
| PyPA | Python Packaging Authorithy | Group which maintains core set of projects in Python packaging |
| PyPI | Python Package Index | Main index where packages get installed from |
| PyPy | - | An implementation of the Python specification in (quasi-)Python, with JIT capabilities |
| QEMU | Quick EMUlator | Predominant emulation framework on Linux |
| RHEL | Red Hat Enterprise Linux | Commercial distribution with some of the longest-running support timelines |
| RISC | Reduced Instruction Set Computer | Paradigm underlying many past and current CPU architectures |
| ROCm | Radeon Open Compute | Software stack for AMD GPUs; comparable to CUDA |
| `sdist` | Source DISTribution | An archive of the source code of a Python project with metadata | See [here](https://setuptools.pypa.io/en/latest/deprecated/distutils/sourcedist.html) |
| SIMD | Single Instruction, Multiple Data | CPU-specific instructions that can process more data in a single instruction |
| SIG | Special Interest Group | E.g., Distutils-SIG (now replaced by [Discourse](https://discuss.python.org/c/packaging/14)) |
| SSE | Streaming SIMD Extensions | Various extensions to the x86 instruction set (SSE, SSE2, SSE3, SSSE3, SSE4) for SIMD |
| TOML | Tom's Obvious Minimal Language | Configuration language chosen for `pyproject.toml`, `cargo` etc., see [here](https://toml.io) |
| UCRT | Universal C Runtime | Windows equivalent to glibc/musl |
| UI | User Interface | |
| UX | User eXperience | |
| VCS | Version Control System | Tool to keep track of changes in source code, e.g. `git` |
| `venv` | Virtual ENVironments | Python standard library module for creating environments; distinct from `virtualenv` | See [here](https://docs.python.org/3/library/venv.html) |

## Terms

| Term | Explanation | Examples / References |
|---|---|---|
| Architecture | In the context of packaging software, this generally refers to the CPU architecture (=ISA) | |
| ABI Break | Failing to maintain the ABI | See [here](./background/binary_interface.md#an-example-of-an-abi-break) |
| Binary Compatibility | Succeeding to maintain the ABI (e.g. across versions / upgrades) | See [here](./background/binary_interface.md)|
| Build Backend | Specifically in the context of `pyproject.toml` builds, the tool responsible for building a Python package | `setuptools`, `flit`, `hatch`, ... |
| Build Frontend | Specifically in the context of `pyproject.toml` builds, the tool used to trigger a build | Predominantly `pip` |
| Calling Convention | Agreed-upon contract with describes how to interact with a given CPU (family) | See [here](https://en.wikipedia.org/wiki/Calling_convention) |
| `cargo` | Package manager for the Rust language, often upheld as a positive example for installation UX | See [here](https://doc.rust-lang.org/cargo/) |
| Conda | Cross-platform package & environment manager, based on distribution channels separate from PyPI | See [here](https://docs.conda.io/) |
| Conda-forge | Community-led packaging effort for (predominantly) Python packages | See [here](https://conda-forge.org/) |
| Cross-compilation | Compiling _on_ a given platform _for_  another platform | See [here](./background/compilation_concepts.md#cross-compilation) |
| (64 bit) Data Models | Choice of bit-widths for `int`/`long` integer types | ILP32, ILP64, LP64; see [here](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models) |
| Distribution | An entity distributing (consistent) binary artefacts, often forming its own ecosystem | Incl. OS: Debian, Fedora, Ubuntu, RHEL...</br>OS-less: Chocolatey, Conda, Spack, ... |
| `distutils` | Python standard library module for building and installing packages; added in 1.6, to be removed in 3.12 | See [here](https://docs.python.org/3/library/distutils.html) |
| `easy_install` | Deprecated method for installing Python packages, superseded by `pip install` | See [here](https://setuptools.pypa.io/en/latest/deprecated/easy_install.html) |
| Egg | Historical format for distributing Python packages | See [here](https://setuptools.pypa.io/en/latest/deprecated/python_eggs.html) |
| Emulation | Pretending to run on a different CPU architecture; this can be used to avoid cross-compilation | See QEMU, resp. [here](https://en.wikipedia.org/wiki/Emulator) |
| Linker | A tool to correctly find the required third-party symbols for a given project | GNU's gold, LLVM's lld, [mold](https://github.com/rui314/mold) |
| Mamba | Alternative implementation of the `conda` CLI tool with a faster solver | See [here](https://mamba.readthedocs.io/) |
| Manylinux | Baseline tooling to allow distributing wheels across various Linux distributions | See [PEP 600](https://peps.python.org/pep-0600/) and the PEPs it replaces |
| `numpy.distutils` | Extension to `distutils`, adding i.a. support for BLAS/LAPACK, Fortran, SIMD etc. | See [here](https://numpy.org/doc/stable/reference/distutils.html) |
| Platform | Colloquially used as interchangeable with the OS, though really only fully specified by the target triple | |
| `pyproject.toml` | Standard metadata file for Python packages | See [PEP 517](https://peps.python.org/pep-0517) & [518](https://peps.python.org/pep-0518) |
| `setuptools` | Most widely used tool for building Python packages; new home of `distutils` | See [here](https://setuptools.pypa.io/) |
| Symbol | A compiled version of a function | See [here](./background/compilation_concepts.md#symbols-mangling-and-linkers) |
| Tarball | Colloquial name for various flavors of `.tar` archive files | See [here](https://en.wikipedia.org/wiki/Tar_(computing)) |
| (Target) Triple | Unambiguous specification of the platform for the purpose of (cross-)compiling software for it, usually `<arch>-<vendor>-<OS>` | See [PEP 11](https://peps.python.org/pep-0011/), resp. [here](https://clang.llvm.org/docs/CrossCompilation.html#target-triple) or [here](https://gcc.gnu.org/install/specific.html) |
| `virtualenv` | Installable module for handling virtual environments; largely a superset of `venv` | See [here](https://virtualenv.pypa.io/) |
| Wheel | A format for distributing and installing binary artefacts for Python packages; essentially a tarball plus metadata | See [here](https://wheel.readthedocs.io/) |
