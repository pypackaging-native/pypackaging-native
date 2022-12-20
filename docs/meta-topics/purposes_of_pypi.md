# The multiple purposes of PyPI

## Current state

PyPI serves more than one purpose. In its own words, [from the pypi.org front
page](https://pypi.org/), it is a repository of software for the Python
programming language that

1. "package authors use to distribute their software"
2. "helps you find and install software developed and shared by the Python community"

For (2), we can further distinguish:

- installing packages from binaries (wheels) for a select set of supported platforms
- installing packages from a source distribution (sdist) on any platform


### Package authors distributing their software

Publishing versions of Python package as source code on PyPI is the original
purpose of PyPI. It is the way authors "announce" that there is a new version -
if it's not on PyPI, then effectively it does not exist as a reusable, public
Python package. It is *the* authoritative flow of source code flow from authors
to end users, downstream dependents, and (importantly) to all redistributors of
the package. That last category includes:

- operating system providers: Linux distros, Apple for macOS, Microsoft for Windows, ...
- standalone packaging systems: Conda, Spack, Nix, Homebrew, Chocolatey, ...
- sysadmins for deployments inside companies, universities, HPC centers, ...
- developers of Python distributions:
  [Anaconda Distribution](https://www.anaconda.com/products/distribution),
  [ActiveState Python](https://www.activestate.com/products/python/),
  [WinPython](https://winpython.github.io/),
  [RStudio](https://solutions.rstudio.com/python/), ...
- maintainers of other wheel indexes:
  [piwheels](https://www.piwheels.org/) (for Raspberry Pi),
  [Christoph Gohkle's Windows wheels](https://www.lfd.uci.edu/~gohlke/pythonlibs/) (discontinued June '22),
  [Alpine Wheels](https://github.com/alpine-wheels/), ...
- developers of standalone apps and embedded systems containing Python packages,
- and probably quite a few more types of consumers,


### Users installing Python packages from binaries

Installing from binaries typically means "install wheels with `pip`"[^1].
Today, wheels are responsible of the vast majority of package installs from
PyPI, and they are a very convenient way for most Python developers to quickly
install up-to-date versions of the packages they need in their development
environment.

[^1]:
    There are other installers besides `pip`, such as
    [pypa/installer](https://installer.readthedocs.io/en/stable/).Usage
    of alternative installers isn't widespread yet.

The number of types of wheels which PyPI allows uploading is growing. It is a
(sparse) matrix of:

- Python interpreter[^2]: CPython, PyPy
- Operating system: Windows, macOS, Linux
- CPU architecture: i686, x86-64, aarch64/arm64, ppc64le, s390x
- `libc` flavor: glibc, musl libc

A good overview is maintained in[the cibuildwheel docs](https://cibuildwheel.readthedocs.io/en/stable/).

[^2]:
    Note that each minor version of CPython and PyPy has to be treated
    separately (unless one is using the stable ABI), so typically a package
    will have 3 or 4 wheels for different minor versions of CPython with the
    same OS, CPU and libc flavors.

When installing a wheel, its declared runtime dependencies will also be installed.
Those dependencies are found in the metadata inside the wheel.
[`pip`'s dependency resolver](https://pip.pypa.io/en/stable/topics/dependency-resolution/) 
takes care of this process. Dependency resolution and management is an
oft-debated and nontrivial topic in general. Challenges in this area are not
specific to packages with native code; once we have wheels, they should behave
similarly to pure Python wheels: dependencies are only on other packages on
PyPI, everything else should be present within the wheel itself.

### Users installing Python packages from sdists

Installing a package from an sdist typically implies using `pip`. `pip`
downloads the sdist, builds a wheel from it (using a `pyproject.toml` build
backend hook, or invoking `setuptools` if such a hook is not present), and then
installs it. Installing runtime dependencies then works the same way as
described in the section above.

The key issue here is dependencies that are needed at build time. It is
currently not possible to describe build dependencies other than those that can
be found on PyPI. And *every* package containing native code has such
dependencies. In some cases it's only a C compiler - which is usually present
on (non-Windows) end user machines. Once one needs other compilers, shared
libraries from the system, or other non-PyPI dependencies, things become
fragile very quickly. This topic is discussed in
[Metadata handling on PyPI](../key-issues/pypi_metadata_handling.md).


## Problems

The different purposes of PyPI not being separated well enough results in an
important problem for package authors. Due to these two "mix-ups of purposes":

1. Uploading an sdist makes it available to redistributors (purpose 1) *and*
   causes it to be available for users to install from source.
2. `pip` prefers installing from a wheel, but automatically tries to install
   from sdist if a wheel cannot be found for the most recent version on PyPI.

authors of packages that are hard to build from source have a difficult choice
to make. If they *do upload an sdist*, they may get a flood of bug reports from
users with failed builds. If they *don't upload an sdist* but only wheels, they
(a) disrupt the flow of source code to redistributors, (b) diminish the value
of PyPI as the authoritative archive of Python packages, and (c) are doing
something that makes it harder to apply security best practices (ideally
binaries for open source projects always come with source code and can be
recreated from them).

We'll also remark here that wheels have to be built from source under a very
specific set of conditions (e.g., in a manylinux Docker container), which are
typically not met on end user systems. Locally built wheels may end up in
`pip`'s cache, which is then "polluted" with a wheel that may not work well.
There are only a few packaging systems that mix building from source with
binary caches - Nix and Spack are two good examples. That *only* works reliably
because the builds are deterministic enough; the package managers know
everything that is relevant (`libc`, compilers, native dependencies) so
binaries in a shared cache can be relied on and really do function like a
cache. For PyPI/pip/wheels that is not the case, and as a result this mixing
may work in simple cases and lead to unpredictable problems for more complex
builds.


## History

Wheels are a relatively recent addition to PyPI - the first release of the
`wheel` package happened in 2012,
[PEP 427 – The Wheel Binary Package Format 1.0](https://peps.python.org/pep-0427/)
was accepted in 2012, and it took a couple of years before it became the norm
to upload wheels for Windows, macOS and Linux for every release. 

Before wheels arrived on the scene, the `setuptools`-specific
[Egg format](https://setuptools.pypa.io/en/stable/deprecated/python_eggs.html)
(`.egg` files) was regularly used to provide binaries. `easy_install` plus eggs
gave a similar, if less polished, install experience to `pip` plus wheels.

Even before eggs and wheels, it was possible to upload binaries to PyPI. For
example, NumPy uploaded `.exe` installers for Windows (created with NSIS) for
years, to help users avoid having to build from source[^3]. To use those
installers, they had to be manually downloaded and run. These kinds of
installers were not common however, they only made sense for packages that were
difficult to build from source.

[^3]:
    The `.exe` and `.whl` files and their upload dates for
    [the NumPy 1.6.0 release](https://pypi.org/project/numpy/1.6.0/#files)
    illustrate how long `.exe` files remained in use, and when wheels were
    added (retroactively in that case).


## Relevant resources

- [Twitter thread (2021 - Steve Dower, Juan Luis Cano Rodríguez, Ralf Gommers)](https://twitter.com/zooba/status/1415440484181417998)
  about pros/cons and importance of uploading sdist's to PyPI.


## Potential solutions or mitigations

- Update package installers to not install from sdist by default if a wheel
  cannot be found (see [pip#9140](https://github.com/pypa/pip/issues/9140)).
- Update PyPI so the three purposes are better separated. E.g., allow upload of
  sdist's for archival and "source code to distributors flow" without making
  them available for direct installation.
