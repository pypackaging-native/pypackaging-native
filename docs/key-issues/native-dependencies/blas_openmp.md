# BLAS, LAPACK and OpenMP

BLAS, LAPACK and OpenMP are key libraries for scientific computing. BLAS and
LAPACK provide linear algebra functionality, and OpenMP provides primitives for
parallel computing on shared memory machines. They're written in C, C++,
Fortran, and even some assembly code. They are typically not packaged on PyPI
(MKL is the exception). The most popular libraries for BLAS and LAPACK are
OpenBLAS and MKL; they provide _both_ BLAS and LAPACK[^1]. NumPy and SciPy depend
on BLAS and LAPACK; scikit-learn depends on OpenMP. BLAS and LAPACK libraries
themselves can be built to use OpenMP _or_ pthreads.

[^1]:
    Note that it is possible to build OpenBLAS without LAPACK support. This is
    a bad idea, however Arch Linux does do this anyway (as of Dec 2022, see
    [scipy#17465](https://github.com/scipy/scipy/issues/17465)).
    Accounting for such nonstandard choices by individual packagers of system
    dependencies makes dealing with native dependencies extra difficult.

Two things make dealing with these native dependencies extra challenging:

1. There are *multiple independent implementations* of the BLAS, LAPACK and
   OpenMP API specifications. They adhere to the same API, but may be built
   with different ABI, make different choices for symbol names for 64-bit
   builds, etc.
2. All packages in an environment should use *the same library* for each of
   BLAS, LAPACK and OpenMP, because otherwise issues with threading control
   will occur. This is difficult to guarantee if wheels must be self-contained;
   it's clearly at odds with vendoring.


??? question "What are BLAS and LAPACK exactly?"

    BLAS (Basic Linear Algebra Subprograms) are routines for basic vector and
    matrix operations. LAPACK (Linear Algebra PACKage) builds on top of BLAS
    and provides more complex linear algebra routines. Both have a reference
    implementation in the [Netlib repository](https://netlib.org/), written in
    Fortran: [BLAS](https://netlib.org/blas/),
    [LAPACK](https://netlib.org/lapack/). BLAS also has a reference C API, named
    [CBLAS](https://netlib.org/blas/#_cblas) which is widely implemented;
    LAPACK correspondingly has a C API named
    [LAPACKE](https://netlib.org/lapack/lapacke.html), however that is less
    widely implemented.

    A lot of libraries implement the BLAS and LAPACK interfaces. This is
    typically done to obtain maximum performance, and optimized for particular
    hardware. The performance improvements over the Netlib version are often in
    the 10x-100x range, and given how critical linear algebra is to scientific
    computing and deep learning, these libraries and their performance
    characteristics are of major importance.

    Well-known implementations include [OpenBLAS](https://github.com/xianyi/openblas),
    [Intel MKL](https://en.wikipedia.org/wiki/Math_Kernel_Library),
    [Apple Accelerate](https://developer.apple.com/accelerate/),
    [ATLAS](https://math-atlas.sourceforge.net/) (no longer updated, but still
    shipped especially by Linux distros), [BLIS](https://github.com/flame/blis)
    (BLAS-only) and [libflame](https://github.com/flame/libflame) (LAPACK-only,
    builds on BLIS), AMD's
    [AOCL-BLIS and AOCL-libFLAME](https://developer.amd.com/amd-aocl/blas-library/),
    and [ARM Performance Libraries](https://developer.arm.com/Tools%20and%20Software/Arm%20Performance%20Libraries).
    Those are all for CPU; there are more libraries for GPUs and other
    accelerators, as well as libraries supporting sparse and graph data
    structures.

??? question "What is OpenMP exactly?"
    
    [OpenMP](https://www.openmp.org/) (Open Multi-Processing) is an API that
    supports shared-memory parallel programming in C, C++ and Fortran. It is
    widely used, from regular laptop/desktop to large supercomputers. There are
    many implementations, often shipped together with compilers. Commonly used
    implementations (which we may find vendored in wheels on PyPI) include
    the GCC implementation (`libgomp`), the LLVM implementation (`libomp`), and the
    Intel implementation (`libiomp`). For a comprehensive overview, see
    [this overview](https://www.openmp.org/resources/openmp-compilers-tools/).


## Current state

NumPy vendors [OpenBLAS](https://github.com/xianyi/openblas) as a shared
library in the wheels it uploads to PyPI. SciPy also vendors OpenBLAS, in the
same manner as NumPy - just not the same version, NumPy moved to ILP64 (64-bit)
OpenBLAS while SciPy uses the default 32-bit build. Those OpenBLAS libraries
are built with pthreads. In conda-forge and Homebrew on the other hand,
OpenBLAS is built with OpenMP. The OpenBLAS build for wheels requires a
separate repository with CI - see
[macPython/openblas-libs](https://github.com/MacPython/openblas-libs). The
build is maintained mostly by NumPy and SciPy maintainers - and as can be seen
from the history in the repository, it's a lot of work.
To make matters worse, NumPy and SciPy want tight control over the version of
OpenBLAS they ship with, because another version may cause segfaults or wildly
incorrect numerical results. Hence upgrading is very hard. Things have gotten
a little better recently though, and it may be an option to start depending on
OpenBLAS as a separate package with a version range rather than an exact pin.
[openblas-libs#86](https://github.com/MacPython/openblas-libs/issues/86)
discussed in-progress work to create a separate wheel for OpenBLAS; not ideal
to have to maintain that, but an improvement over vendoring separate versions
in NumPy and SciPy wheels.

Scikit-learn depends on OpenMP directly for an increasing amount of its
parallel execution, and vendors `libomp`/`libgomp` in its wheels. It depends on
SciPy for linear algebra functionality - in particular, the Cython interface
to BLAS and LAPACK that SciPy provides. The scikit-learn team also created 
[`threadpoolctl`](https://github.com/joblib/threadpoolctl), a separate package
specifically to control the parallelism across NumPy, SciPy, scikit-learn, and
any installed BLAS, LAPACK and OpenMP libraries.

PyTorch statically links MKL (except on macOS) and vendors OpenMP (`libiomp` on
Windows/macOS, `libgomp` on Linux) in its wheels.

TensorFlow uses [Eigen](https://eigen.tuxfamily.org) for linear algebra
functionality. This is a header-only C++ library, different from BLAS/LAPACK
implementations and easier for distribution. For parallel execution, TensorFlow
does use OpenMP. It is statically linked rather than vendored.

Deep learning libraries are often their own "silo", providing all functionality
in one coherent framework. When they are installed side-by-side with other
packages, there are issues though with conflicting libraries. However, the
problem is worse in the PyData stack. Here is what the situation look for just
NumPy, SciPy and scikit-learn:

![Scikit-learn with dependencies - BLAS, LAPACK, OpenMP](../../assets/images/blas_lapack_openmp.png#only-light)
![Scikit-learn with dependencies - BLAS, LAPACK, OpenMP](../../assets/images/blas_lapack_openmp_darkmode.png#only-dark)
<figcaption>
Build and runtime dependencies for scikit-learn - showing how BLAS, LAPACK and
OpenMP are included in an environment when installing either from wheels on
PyPI, or from conda-forge or a similar such system that includes all required
dependencies as separate packages.
</figcaption>

As we can see, there are multiple issues that are unique to PyPI:

- We have multiple copies of OpenBLAS rather than just one. Also different
  versions, breaking the "one version rule",
- No runtime switching of BLAS libraries, because of vendoring,
- Mixing pthreads and OpenMP parallelism, resulting in oversubscription issues,
- A diamond dependency, where NumPy and SciPy both depend on OpenBLAS. Making
  upgrades that not synchronized harder, even if those projects will be able to
  release a single OpenBLAS wheel in the future.

When for example PyTorch is installed, the problems multiply - now we also have
a second vendored OpenMP library in addition to two BLAS/LAPACK libraries.

!!! example "Example: PyTorch & OpenMP"

    PyTorch makes heavy use of OpenMP. And has a lot of issues with it - partly
    because OpenMP is inherently complex, and partly because of packaging
    limitations. One recurring problem is deadlocks in combination with
    `multiprocessing` (see, e.g.,
    [pytorch#17199](https://github.com/pytorch/pytorch/issues/17199)).
    These are caused by the GCC implementation of OpenMP (`libgomp`) not being
    fork-safe, and `multiprocessing` defaulting to `'fork'`. The LLVM
    (`libomp`) and Intel (`libiomp`) implementations *are* fork-safe, therefore
    if the dependency could be expressed in package metadata, it would be
    easier to avoid the problem. The dependency of PyTorch on OpenMP is
    completely implicit in `pyproject.toml`/`setup.py` metadata however.

    Here is another example issue, where a PyTorch build inside a virtualenv
    picks up an OpenMP implementation in an uncontrolled fashion, because it's
    an implicit dependency and hence build results get affected by another
    package pulling in an OpenMP library: 
    [pytorch#18398](https://github.com/pytorch/pytorch/issues/18398).

    *Note: there is an active discussion
    ([cpython#84559](https://github.com/python/cpython/issues/84559),
    [Discourse thread](https://discuss.python.org/t/switching-default-multiprocessing-context-to-spawn-on-posix-as-well/21868) - Dec'22)
    to change the default `multiprocessing` context away from `'fork'`.*
    

System package managers usually have a way of dealing with multiple
implementations of an API, through building against a reference package with
stubs as a "virtual dependency". This can be a generic mechanism, or specific
to the dependency type.

Conda manages BLAS, LAPACK and OpenMP (and other such complex dependencies,
like MPI) through
[mutex metapackages](https://docs.conda.io/projects/conda/en/latest/user-guide/concepts/packages.html#mutex-metapackages),
which ensure a single version of an implementation is installed and
[allow users to switch between implementations](https://conda-forge.org/docs/maintainer/knowledge_base.html#switching-blas-implementation). Spack uses
[virtual dependencies](https://spack.readthedocs.io/en/latest/basic_usage.html#virtual-dependencies)
which work in a similar fashion.

Linux package managers may do something similar
(e.g., Debian uses `update-alternatives`
[applied to `libblas` and `liblapack`](https://wiki.debian.org/DebianScience/LinearAlgebraLibraries)),
or they may use a more a more specific way of managing BLAS and LAPACK, such as
[Fedora using Flexiblas](https://docs.fedoraproject.org/en-US/packaging-guidelines/BLAS_LAPACK/#_user_level_selection).

??? info "BLAS/LAPACK demuxing[^2]: FlexiBLAS, libblastrampoline & SciPy's `cython_blas`/`cython_lapack`"

    Supporting multiple BLAS and LAPACK libraries is quite challenging,
    because of API and ABI incompatibilities between them. In particular:

    - API's using 32-bit (LP64) or 64-bit (ILP64) integers as indices of
      vectors/matrices (see
      [here](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models)
      for details). Typically a BLAS/LAPACK library supports both at the
      source level, and how the library is built determines which one is used.
      ILP64 builds may or may not use a symbol suffix - `64_` being the common
      one, but that's not a universal convention.
    - g77 vs. gfortran ABI. MKL and Apple Accelerate use the g77 ABI (or "f2c
      calling convention"), while all other common libraries use the gfortran
      ABI.

    In addition, libraries may implement different versions of the reference Netlib APIs,
    and CBLAS and LAPACKE support are not universal. To deal with this, a
    *demuxing library* (or wrapper library) may be a good solution. Its job is to
    provide a uniform API and ABI to any package using it. There are several
    examples of this: [FlexiBLAS](https://www.mpi-magdeburg.mpg.de/projects/flexiblas)[^3]
    is a standalone library to do this at the C/Fortran level,
    [libblastrampoline](https://github.com/JuliaLinearAlgebra/libblastrampoline)
    does this for Julia, and SciPy's `scipy.linalg.cython_[blas|lapack]` submodules
    provide a C/Cython interface to other Python packages.

    If other Python packages can accept a dependency on SciPy, that's their
    best option. SciPy itself has to deal with the full complexity of
    discovering and building against arbitrary BLAS/LAPACK libraries.

[^2]:
    The term demuxing here stands for demultiplexing, in the sense of splitting a single
    "signal" (e.g. _multiply these two matrices_) into several channels for the various
    supported implementations of BLAS/LAPACK.

[^3]:
    Note that FlexiBLAS is GPLv3 licensed, with a LGPL-like runtime exception that only
    covers the Netlib-equivalent part of its API (that may not be enough, see
    [scipy#17362](https://github.com/scipy/scipy/issues/17362)).


## Problems

1. Building from source is quite difficult. Dependencies on BLAS, LAPACK and
   OpenMP cannot be expressed, and end users will typically not have it
   installed, or they do have it but it's not found, or it's the wrong version.
   In addition, variations between system package managers are not easy to
   handle (e.g., Fedora fails to ship pkg-config files for OpenBLAS, Arch Linux
   fails to include LAPACK in OpenBLAS, etc.).
2. Building wheels is way too much work. Maintainers of each project using BLAS
   and LAPACK are responsible for building it themselves (which is painful) and
   then vendoring it. In addition, in the case of OpenBLAS, it requires
   vendoring `libgfortran`.
3. Deadlocks due to use of `multiprocessing` in combination with `libgomp`.
4. Issues due to using more than one library of the same type. See for example
   [scipy#15050](https://github.com/scipy/scipy/issues/15050) (*"Performance
   issue on macOS arm64 (M1) when installing from wheels (2x libopenblas)"*) and
   [openblas#3187 ](https://github.com/xianyi/OpenBLAS/issues/3187)
   (*"Slowdown when using openblas-pthreads alongside openmp based parallel code"*).
5. As a result of the above-mentioned problems with OpenMP, SciPy bans usage of
   OpenMP completely even though it would be of significant interest to use
   OpenMP. Scikit-learn is gradually expanding its usage of OpenMP, but is
   running into problems.
6. No runtime switching of implementations when installing from wheels. This
   makes testing, benchmarking and development work more difficult.
7. MKL does provide wheels but cannot be linked to due to the wheel spec. And
   MKL itself is >100 MB in size, so vendoring it isn't reasonable.

## History

TODO


## Relevant resources

- [scipy#10239 - can OpenMP be used portably](https://github.com/scipy/scipy/issues/10239#issuecomment-795030817) 
- [array-api#4 - Parallelism - what do libraries offer, and is there an API aspect to it](https://github.com/data-apis/array-api/issues/4)
- OpenBLAS build config for vendoring into wheels:
  [MacPython/openblas-libs](https://github.com/MacPython/openblas-libs/)


## Potential solutions or mitigations

- A mitigation to not make things worse: stay with the status quo, do not use
  more OpenMP. Not a very satisfactory one though.
- Build wheels for OpenBLAS and perhaps also OpenMP and maintain those on PyPI.

    - Issue: who takes responsibility for these, and decides on changes over
      time (possibly breaking ones)?

    - See this gradual evolution plan discussed between scikit-learn and SciPy
      maintainers:
      [scipy#15050](https://github.com/scipy/scipy/issues/15050#issuecomment-975318631)

- Larger changes to PyPI/wheels to get to parity with system package managers.
  This will require dealing with several "meta topics", like
  [a build farm](meta-topics/no_build_farm.md) and
  [PyPI's author-led social model](meta-topics/pypi_social_model.md),
  in addition to implementing something like virtual packages.
- Making it possible to express dependencies on libraries outside of PyPI.
- Making a distinction on PyPI between pure Python packages and other packages.
  With the latter set of packages all being provided by a system package
  manager.

