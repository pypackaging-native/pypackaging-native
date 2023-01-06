# Other issues

This page contains a collection of issues that do come up in the context of
scientific and data science projects and packaging those, but are deemed less
high-impact than the key issues.


## Lack of support for symlinks in wheels

Shared libraries on Linux and other non-Windows platforms are often provided
and versioned via symlinks (examples:
[cupy#6261](https://github.com/cupy/cupy/issues/6261#issuecomment-1076525294),
[`libarrow` in this Discourse thread](https://discuss.python.org/t/symbolic-links-in-wheels/1945),
[pip#5919](https://github.com/pypa/pip/issues/5919), and `pypa/wheel` issues
[#203](https://github.com/pypa/wheel/issues/203#issuecomment-622598330),
[#400](https://github.com/pypa/wheel/issues/400), and
[#453](https://github.com/pypa/wheel/issues/453)). In order to build wheels
containing versioned shared libraries, symlink support is needed. In the
absence of that, the symlinks get materialized into full copies of the
symlinked files, blowing up wheel sizes.

A second use case for symlinks is for editable installs when the build system
uses out-of-place builds. Out-of-place builds are the only option in Meson, and
also good practice for CMake. For out-of-place builds, you end up with compiled
extension modules and generated files in the build directory, and .py files in
the source directory. To put those together into a working editable install,
the most straightforward solution is putting symlinks to all files in a wheel -
see [meson-python#47](https://github.com/mesonbuild/meson-python/issues/47).

It looks like there is an understanding now that symlink support is needed, and
that it requires a new wheel format spec (and hence a PEP) - see
[Clarifications to the wheel
specification](https://discuss.python.org/t/clarifications-to-the-wheel-specification/8141/33).

An experimental setuptools extension,
[wheel-axle](https://github.com/karellen/wheel-axle/), implements support for
producing a wheel containing symlinks.


## Dropping support for old manylinux versions is difficult

Due to how wheel tags work, they need to be explicitly recognized by build and
install tools. Old versions of `pip` tend to be used for years (especially in
Linux distros), which means that when a project starts distributing wheels in a
newer format (e.g., `manylinux2014` instead of `manylinux1`), those new wheels
will not be recognized for part of the user base for a long time. As a result,
projects are forced to *also* continue distributing the older format, to avoid
those users getting no wheels and a build from sdist instead. Being forced to
produce duplicate wheels for years is a lot of extra work and CI time. This is
in principle a problem on all platforms, it tends to show up more for Linux
because of the combination of old `pip` versions and more changes to platform
tags (we've had `manylinux1`, `manylinux2010`, `manylinux2014` and now, with
PEP 600, "perennial manylinux" - but that still requires agreeing on new glibc
versions to start shipping in practice).

## Wheel build tooling is implemented in a scattered fashion

When working with native dependencies, one must use a tool to vendor
dependencies that aren't part of the platform by wheel standards. There are at
least three different tools for this: `auditwheel` (Linux), `delocate` (macOS)
and `delvewheel` (Windows). They have the same job, but are three independent
projects with different capabilities. This is bad from a usability perspective,
and when improvements to this tooling needs to be made, the discussion may have
to be had multiple times (example: adding an `--exclude` option to not vendor
certain libraries: [auditwheel#368](https://github.com/pypa/auditwheel/pull/368)).

This scattering issue can also be observed in the many support packages to deal
with metadata, wheel tags, and other aspects of producing wheels, e.g.:
[packaging](https://github.com/pypa/packaging),
[distlib](https://github.com/pypa/distlib),
[pyproject-hooks](https://github.com/pypa/pyproject-hooks), and
[pyproject-metadata](https://github.com/FFY00/pyproject-metadata). And with
`pip` and `build` not using the same UX for things like `--config-settings` and
`--no-isolation/--no-build-isolation`.


## Bootstrapping and circular dependencies of Python packaging tools

Python packaging tools have a bit of a bootstrapping issue, which is a problem
for other packaging systems when they want to incorporate those packages. If
one wants to build and install `pip`/`setuptools`/`wheel` from source, one
needs `pip`/`setuptools`/`wheel` already installed. Same for `pypa/build` and
`pypa/installer` and `flit` (`poetry` does better here, it uses `setuptools` to
build itself). This is getting better - `pip` vendors all of its runtime
dependencies so it can produce a wheel to install itself, and `flit` now
vendors a TOML parser - but there is still a ways to go. See this
[Bootstrapping a specific version of pip](https://discuss.python.org/t/bootstrapping-a-specific-version-of-pip/12306/18)
thread for some discussion on this.


## No good way to install headers or non-Python libraries

If a library provides functionality that is meant to be used from C or C++ code
in another package, one needs to install headers and libraries. To make that
work well, those headers and libraries should be installed in a place where
other tools can find them. There are standard places for this on a system, e.g.
for a prefix `/usr` the headers may go into `/usr/include/` and the libraries
in `/usr/lib`. This is *technically* possible with wheels, but recommended
against because the install process may clobber system files. As a result, what
projects like NumPy, Pybind11 and PyArrow end up doing is installing into their
own tree under `site-packages/pkgname` (which certainly won't be on a search
path), and then recommending that consuming packages query the location with a
`get_include` function. E.g.:
```python
import pyarrow

pyarrow.get_include()
```
This isn't great, because it assumes that the build tool can run Python - and
that breaks under cross-compilation. This would work much better under
Conda/Spack/Homebrew/etc., however the packages themselves have to decide where
to install to. Hence they choose to always install inside site-packages, due to
the limitations that PyPI/wheels impose.


## More issues

*These still have to be worked out:*

- UX for build and install tools is painful and easy to shoot oneself in the
  foot with (e.g., most users and maintainers don't understand the details of
  build isolation)
- Tooling will often assume virtualenvs only, and/or deal with environment
  activation when it really shouldn't.
