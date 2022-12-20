# Native dependencies

Depending on non-Python compiled dependencies ("native dependencies") is very
tricky. Python packaging works reasonably well when a project contains some
self-contained C, C++ or Cython code. Even then, issues can occur though - for
example because the dependency on a C or C++ compiler cannot be expressed in
package metadata. So any constraints on versions, compiler types, etc. can only
be documented and not enforced[^1].

[^1]:
    As a simple example, let's use Pythran - a Python to C++ transpiler that is
    gaining popularity and is used in SciPy and scikit-image. On Windows it
    needs Clang-cl rather than MSVC. This often goes wrong, because most users
    don't have Clang-cl installed.

C, C++ and Cython are not the only languages that need native dependencies -
Fortran, CUDA, and Rust are other commonly used languages, and there are more
languages that one may want to use (e.g., in the context of scientific
computing and GPUs, OpenCL, HIP and SYCL are of increasing interest).

Once such code starts to depend on APIs from non-Python projects, more problems
show up.


## Current state

One obvious and hard to deal with problem is that dependencies on libraries
that are not on PyPI cannot be expressed. When one types `pip install somepkg`
for a `somepkg` that has such dependencies on a platform that `somepkg` doesn't
provide wheels for, what is most likely to happen is a build failure halfway
through because a dependency is missing or is present but in an unexpected
configuration.

!!! example "Example: SciPy's build and runtime dependencies"

    SciPy has a few build-time dependencies and one runtime dependency
    (`numpy`) listed in its `pyproject.toml`. As a package with medium build
    complexity (more complex than projects with self-contained C/Cython
    extensions, but less than the likes of TensorFlow and PyArrow), it can
    serve as an example of what metadata can and cannot capture about
    dependencies. This diagram illustrates those dependencies:

    ![SciPy's build and runtime dependencies](../../assets/images/scipy_build_runtime_dependencies.png#only-light)
    ![SciPy's build and runtime dependencies](../../assets/images/scipy_build_runtime_dependencies_darkmode.png#only-dark)


    Out of those, these are the dependencies declared in `pyproject.toml`:
    `numpy`, `Cython`, `pybind11`, and `pythran` (also the build system
    dependencies: `meson-python`, `wheel`). And these are the dependencies that
    cannot be declared:

    - C/C++ compilers
    - Fortran compiler
    - BLAS and LAPACK libraries
    - `*-dev` packages for Python, BLAS and LAPACK, if headers are packaged separately
    - pkg-config or system CMake (for dependency resolution of BLAS/LAPACK)

    Finally, a number of native libraries (Boost, ARPACK, HiGHS, etc.) are
    vendored into SciPy. Unvendoring those (something system packagers would
    like) has been deemed infeasible[^2], those dependencies would also not be
    expressable and therefore make the build more fragile.

[^2]:
    With Meson as the new build system for SciPy, it is becoming possible to
    query the system for dependencies first, and only fall back to a vendored
    version if the system is not found. This may be done in the future.

Some projects do upload sdists but advise users to avoid them (or avoid PyPI
completely in favor of other package managers). Some other projects do not
upload sdists to PyPI to avoid users filing issues about failing builds. See
[purposes of PyPI](../../meta-topics/purposes_of_pypi.md) for more on this
topic.

Building wheels is challenging too. Wheels are required to be self-contained,
and therefore must vendor those non-Python dependencies. This means that a
project becomes responsible for (often) rebuilding the dependency, dealing with
the vendoring process (through `auditwheel`, `delocate`, `delvewheel`, etc. -
and sometimes those tools are not enough), and implementing static linking or
other ways of slimming down wheel sizes when vendoring large libraries. There
may be other issues with vendoring, like EULA's for libraries like CUDA and MKL
being ambiguous or outright forbidding redistribution.

Even when the vendoring hurdle is successfully taken and working wheels are
produced, there may be problems because vendoring may be the wrong solution
technically. E.g., runtimes are often designed with the assumption that they're
the only runtime on a given system - so having multiple vendored copied of a
runtime in different packages leads to conflicts.

Native dependencies is a huge topic, so to make the problems more concrete, a
number of cases are worked out:

- [BLAS, LAPACK and OpenMP](blas_openmp.md),
- [The Geospatial stack](geospatial_stack.md),
- [Complex C++ dependencies](cpp_deps.md).


## Problems

The key problems are (1) not being able to express dependencies in metadata,
and (b) the design of Python packaging and the wheel spec forcing vendoring
dependencies.

For more detailed explanations of problems, see the concrete cases linked above.


## History

- [PEP 426, section "Mapping dependencies to development and distribution activities"](https://peps.python.org/pep-0426/#mapping-dependencies-to-development-and-distribution-activities) (2012, withdrawn).
- [PEP 459 - "Standard Metadata Extensions for Python Software Packages"](https://peps.python.org/pep-0459/)
  (2013, withdrawn). *This is probably the most relevant PEP that has been
  proposed - it explicitly deals with native dependencies provided by the
  system.*
- Nathaniel Smith's [`pynativelib` proposal](https://github.com/njsmith/wheel-builders/blob/pynativelib-proposal/pynativelib-proposal.rst)
  (2016).
- [PEP 668 - Marking Python base environments as “externally managed”](https://peps.python.org/pep-0668/) (2021).

More history TODO


## Relevant resources

TODO


## Potential solutions or mitigations

From the ["Wanting a singular packaging tool/vision"](https://discuss.python.org/t/wanting-a-singular-packaging-tool-vision/21141/92)
Discourse thread (2022):

- Define "native requirements" metadata (even if `pip` ignores it)
- Allow (encourage) wheels with binaries to have tighter dependencies than their sdists
- Encode and expose more information about ABI in package requirements

Adding a mechanism to specify system dependencies that are needed by a Python
package seems like a tractable first step here.

- Provide a way for users to get pure Python packages from PyPI, and everything
  else from a system package manager.
- ... many other potential improvements (all a lot of work).
