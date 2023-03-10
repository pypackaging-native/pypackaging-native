# Depending on packages for which an ABI matters

When a library exposes an API in a compiled language, any other library or
package that uses that API has to concern themselves with the
[ABI](https://en.wikipedia.org/wiki/Application_binary_interface) (Application
Binary Interface) as soon as the API provider and API consumer are distributed
separately. As a rule, it's best for ABI compatibility if the different
packages are built with the same compiler toolchain, command-line flags and
other configuration options. If that is not possible or guaranteed, one has to
start thinking carefully about whether the two build environments will result
in packages that are ABI-compatible.

There are a lot of things with an ABI that a Python library maintainer may have
to care about:

- The C standard library (the `libc` flavor)
- The C++ standard library (`libstdc++` old vs. new string ABI, `libc++`, MSVC runtime, etc.)
- Fortran (gfortran vs. g77 ABI)
- CPython
- Python packages with C APIs: (NumPy, SciPy)
- Python packages with C++ APIs (PyTorch, Apache Arrow - see
  [complex C++ dependencies](native-dependencies/cpp_deps.md))
- Common non-Python dependencies for scientific computing
  [BLAS, LAPACK, OpenMP](native-dependencies/blas_openmp.md),
  MPI[^1] (see [mpich.org/abi](https://www.mpich.org/abi/)).

[^1]:
    `h5py` is an example of a project that support MPI but, despite regular
    user requests, does not ship MPI-enabled wheels. MPI has multiple
    implementations, and does not have a stable ABI (see, e.g.,
    [mpi-issues#654](https://github.com/mpi-forum/mpi-issues/issues/654) for
    (lack of) MPI ABI stability).

For some more background on ABI, see [here](../background/binary_interface.md).


## Current state

A package manager has to know about ABI, either implicitly or explicitly. In
the case of PyPI, it's implicit. There are conventions that anyone publishing a
wheel should adhere to. For example, on Windows and macOS, use compilers and
flags compatible with those used to produce the Python installers published on
[python.org](https://www.python.org/downloads/). And on Linux, what the various
manylinux PEPs say (more complex, so best to use the `manylinux`-provided
Docker images).

Other package managers are more explicit about managing ABI, to varying
degrees. They also have the advantage of being able to enforce using a
consistent compiler toolchain. Inspecting the dependency tree of a SciPy
install (which uses the Python and NumPy C APIs/ABIs) will show this:

=== "PyPI"

    ```bash
    $ pipdeptree -p scipy
    scipy==1.9.3
      - numpy [required: >=1.18.5,<1.26.0, installed: 1.23.5]
    ```

=== "conda-forge"

    ```bash
    $ # Note: output edited to remove duplicate packages and python's dependencies
    $ mamba repoquery depends scipy --tree

    scipy[1.9.3]
      ├─ libgfortran-ng[12.1.0]
      │  └─ libgfortran5[12.1.0]
      ├─ libgcc-ng[12.1.0]
      │  ├─ _libgcc_mutex[0.1]
      │  └─ _openmp_mutex[4.5]
      │     └─ llvm-openmp[14.0.4]
      │        └─ libzlib[1.2.13]
      ├─ libstdcxx-ng[12.1.0]
      ├─ python_abi[3.10]
      │  └─ python[3.10.8]
      └─ numpy[1.23.3]
         ├─ libblas[3.9.0]
         │  └─ libopenblas[0.3.21]
         ├─ libcblas[3.9.0]
         ├─ liblapack[3.9.0]
      ├─ libblas already visited
      ├─ liblapack already visited
    ```

=== "Spack"

    ```bash
    $ # Note: output edited to remove build-only dependencies
    $ ./spack spec py-scipy%gcc

    py-scipy@1.9.2%gcc@12.2.0 arch=linux-endeavourosrolling-skylake_avx512
        ^openblas@0.3.21%gcc@12.2.0~bignuma~consistent_fpcsr+fortran~ilp64+locking+pic+shared symbol_suffix=none threads=none arch=linux-endeavourosrolling-skylake_avx512
        ^python@3.9.13%gcc@12.2.0+bz2+ctypes+dbm~debug+libxml2+lzma~nis~optimizations+pic+pyexpat+pythoncmd+readline+shared+sqlite3+ssl~tix~tkinter~ucs4+uuid+zlib patches=0d98e93,4c24573,f2fd060 arch=linux-endeavourosrolling-skylake_avx512
        ^py-numpy@1.23.3%gcc@12.2.0+blas+lapack patches=873745d arch=linux-endeavourosrolling-skylake_avx512
    ```

=== "Arch Linux"

    ```bash
    $ # Note: output edited to remove duplicates and some transitive dependencies
    $ pactree python-scipy

    python-scipy
    └─python-numpy
      ├─cblas
      │ └─openblas provides blas
      │   └─gcc-libs
      │     └─glibc>=2.27
      ├─lapack
      │ └─openblas provides blas
      └─python
        ├─bzip2
        │ ├─glibc
        ├─expat
        ├─gdbm
        ├─libffi
        ├─libnsl
        ├─libxcrypt
        ├─openssl
        └─zlib
    ```

=== "Homebrew"

	```zsh
	% # Note: output edited to remove duplicate packages
	% brew deps --tree --installed scipy

	scipy
	├── gcc
	│   ├── gmp
	│   ├── isl
	│   ├── libmpc
	│   ├── mpfr
	│   └── zstd
	│       ├── lz4
	│       └── xz
	├── numpy
	│   └── openblas
	│       └── gcc
	├── openblas
	│   └── gcc
	└── python@3.11
	    ├── mpdecimal
	    ├── openssl@1.1
	    ├── sqlite
	    └── xz
	```
For example, we see `python_abi` and `libgcc_mutex` in the conda-forge output;
detailed compiler, BLAS interface, CPU architecture and library info in the
Spack output; and `glibc` version info in the Arch Linux output.

In general, the more dependencies and more languages one uses, the more ABI
compatibility starts to matter. If the Python C API is the only thing used by a
package, the rules are relatively straightforward: rebuild for every minor
Python version (unless one can use the
[limited API](https://docs.python.org/3/c-api/stable.html), then it's even
easier), with compatible compilers.

It already gets a lot harder as soon as one uses the NumPy C API. Which many
packages do, often via Cython. While the NumPy ABI is much more stable than the
CPython one (NumPy hasn't broken compatibility in a nontrivial way in over a
decade), one still has to understand the rules for building against NumPy:

!!! example "Example: Using the NumPy C API"

    NumPy has a C API, which Cython, SciPy, and many other packages use. That
    ABI is forward but not backward compatible, meaning if you use it then you
    must build your wheels against the lowest NumPy version that you expect
    your users to use. So if you build against version `1.X.Y` then the runtime
    requirement you get is `numpy>=1.X.Y`. That lowest version may depend on
    Python version and platform. There is no good way to express a dependency
    like that in `pyproject.toml`, or even to keep track of what the lowest
    version should be. Because of that, the
    [`oldest-supported-numpy`](https://github.com/scipy/oldest-supported-numpy/)
    metapackage is being used by projects that depend on `numpy` as an
    imperfect hack to obtain the correct `numpy==` build requirement pins per
    Python version and platform. It can be used like so:

    ```toml
    [build-system]
    requires = [
        "oldest-supported-numpy",
        # your other build requirements here
    ]
    ```
    Each time NumPy adds support for a new Python version or platform,
    `oldest-supported-numpy` is updated so that each user of the NumPy C API
    does not have to do that.

    Despite this complexity, the solution is imperfect -
    `oldest-supported-numpy` is unable to communicate back the correct
    `numpy>=` runtime requirement, so those requirements are generally
    incorrect for all packages using NumPy (all wheels will have a generic
    `numpy>=1.Y.Z` for the lowest value of `1.Y.Z` across all wheels).

    See [Adding a dependency on NumPy](https://numpy.org/devdocs/dev/depending_on_numpy.html#adding-a-dependency-on-numpy)
    for more details.


The most difficult cases arise with dependencies outside of Python/PyPI (also see
[native dependencies](native-dependencies/index.md)). At that point one can
pick up arbitrary libraries during the build phase of a package, and with
complex dependencies the ABI of that dependency may be unknown or have to be
introspected. [This paper on SciPy's Cython API for BLAS and LAPACK](https://conference.scipy.org/proceedings/scipy2015/pdfs/ian_henriksen.pdf)
explains how and why SciPy added a Cython API to hide the ABI variations across
BLAS/LAPACK implementations from other Python packages.

These three "levels of complexity" cases are illustrated by these diagrams of
package stacks:

=== "pure Python only"

    ![Anchoring a package stack - pure Python only](../assets/images/anchoring_a_package_stack_pure_only.png#only-light)
    ![Anchoring a package stack - pure Python only](../assets/images/anchoring_a_package_stack_pure_only_darkmode.png#only-dark)
    <figcaption>
    Package stack: CPython and pure Python packages only
    </figcaption>

=== "pure & C API-using"

    ![Anchoring a package stack - pure Python only](../assets/images/anchoring_a_package_stack_pure_plat.png#only-light)
    ![Anchoring a package stack - pure Python only](../assets/images/anchoring_a_package_stack_pure_plat_darkmode.png#only-dark)
    <figcaption>
    Package stack: CPython, pure Python & Python C API-using packages
    </figcaption>

=== "complete stack"

    ![Anchoring a package stack - pure Python only](../assets/images/anchoring_a_package_stack_full.png#only-light)
    ![Anchoring a package stack - pure Python only](../assets/images/anchoring_a_package_stack_full_darkmode.png#only-dark)
    <figcaption>
    Package stack: all the way down to `libc`
    </figcaption>

C++ APIs are a problem onto themselves. When a library is written in C++ but
wants to expose an API with a stable ABI, it often exposes a C rather than a
C++ API with `extern "C"`. Keeping ABI stability has a large cost though, it
takes a lot of extra work and may prevent changes to internals of a library.
Hence it's not always done. For example, PyTorch has a large C++ API and does
not promise any ABI stability:

!!! example "Example: Using the PyTorch C++ API"

    PyTorch is mostly written in C++, and exposes a C++ API in addition to its Python API.
    C++ ABI stability is a tricky topic - it's implementation-defined, and
    because name mangling can change between compilers (or even compiler
    versions), mixing binaries built with different compilers isn't possible.
    PyTorch does not attempt to provide a stable ABI; even bug fix releases
    aren't guaranteed to be compatible. As a result, all packages using
    PyTorch's C API must use a runtime requirement like:
    ```toml
    [project]
    dependencies = [
        "torch == X.Y.Z",  # X.Y.Z is the version of PyTorch used to build against
    ]
    ```

    This requirement also implies *synchronized releases*. If PyTorch does a
    new release, the team ensures that simultaneous releases of `torchvision`,
    `torchaudio`, `torchtext`, `torchdata` and other dependent packages are
    made. This level of coordination doesn't scale well though, and therefore
    may limit the use of the PyTorch C++ API - especially by community open
    source projects whose authors may not have the bandwidth to keep up with
    releases.

    The project description for [pypi/torchvision](https://pypi.org/project/torchvision/)
    is a good example to illustrate the tight version coupling.


## Problems

A problem for PyPI and wheels is that there is little coordination on compiler
toolchains, ABI changes, etc. So maintainers of every package are on their own
trying to figure this out. Other package managers don't have this problem -
they build everything with a consistent toolchain (as much as possible at
least), either including `libc` or on top of the `libc` provided by the
operating system. See [no build farm](../meta-topics/no_build_farm.md) and
[PyPI's author-led social model](../meta-topics/pypi_social_model.md) for more details.

CPython breaks its ABI every minor release (unless one's needs are limited,
then there is the limited API). This has a huge cost: packages have to build
wheels for every minor Python release.

NumPy is effectively forced to do the opposite: NumPy needs to *not*
break its ABI in order to avoid exploding the build matrix of every project
using its C API. This has a large opportunity cost; there are a lot of
improvements and cleanups that NumPy cannot implement. If PyPI either had a
build farm or could be disregarded for binaries with ABI stability
requirements, NumPy would have broken ABI compatibility multiple times by
now[^2], with positive impacts on maintainability, performance, and
functionality.

[^2]:
    Actually, NumPy did break its ABI once, in the 1.4.0 release (2010). This
    resulted in some mayhem and very long and painful discussions. The ABI was
    unbroken, and hasn't been touched since.

Python packaging build tooling has no understanding of ABI constraints, making it
hard to add runtime version constraints to a wheel that are correct (and
tighter than those in the corresponding sdist).

Manylinux versions still using the old C++ ABI, while most C++ projects want to
use the new ABI. `manylinux_2_28` may change this finally, but isn't yet in use.
See [complex C++ dependencies](native-dependencies/cpp_deps.md) for more
details on this topic.


## History

TODO


## Relevant resources

- ["Circumventing the Linker: using SciPy’s BLAS and LAPACK within Cython"](https://conference.scipy.org/proceedings/scipy2015/pdfs/ian_henriksen.pdf),
  Ian Henriksen, SciPy 2015.
- ["`archspec`: A library for detecting, labeling, and reasoning about microarchitectures"](https://tgamblin.github.io/pubs/archspec-canopie-hpc-2020.pdf),
  Culpo et al. (2020).
- ["C++ binary compatibility between Visual Studio versions"](https://learn.microsoft.com/en-us/cpp/porting/binary-compat-2015-2017?view=msvc-170)
- GCC docs on [`libstdc++` dual ABI](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_dual_abi.html)
  and on [ABI Policy and Guidelines](https://gcc.gnu.org/onlinedocs/libstdc++/manual/abi.html).
- NumPy docs [for downstream package authors](https://numpy.org/devdocs/dev/depending_on_numpy.html).
- [PEP 384 - Defining a Stable ABI](https://peps.python.org/pep-0384/).
- [PEP 652 - Maintaining the Stable ABI](https://peps.python.org/pep-0652/).
- Python docs on [C API Stability](https://docs.python.org/3/c-api/stable.html).
- ["Let’s get rid of the stable ABI, but keep the limited API"](https://discuss.python.org/t/lets-get-rid-of-the-stable-abi-but-keep-the-limited-api/18458)
  thread on Discourse (2022).
- [trailofbits/abi3audit](https://github.com/trailofbits/abi3audit).
- ["ABI compatibility in Python: How hard could it be?"](https://blog.trailofbits.com/2022/11/15/python-wheels-abi-abi3audit/) (2022).
- [HPy - A better C API for Python](https://hpyproject.org/).
- Conda-forge FAQ entry on ["How to handle breaking of a package due to ABI incompatibility?"](https://conda-forge.org/docs/user/faq.html#faq-abi-incompatibility).


## Potential solutions or mitigations

- Expressing constraints imposed by an ABI on versions at runtime can be done
  with something like conda-forge's
  [`pin_compatible` and `run_exports` features](https://conda-forge.org/docs/maintainer/pinning_deps.html).
  See [meson-python#29](https://github.com/mesonbuild/meson-python/issues/29)
  for ideas about implementing that in a build backend.
- HPy and its [hybrid and universal ABIs](https://docs.hpyproject.org/en/latest/overview.html#target-abis)
  may be the way forward for improvements in constraints that the CPython ABI
  imposes, as well as enabling use of alternative interpreters.
- Coordinated rebuilds that are needed because of `==` runtime constraints
  would be a lot easier with a build farm (as discussed
  [here](../meta-topics/no_build_farm.md)).
