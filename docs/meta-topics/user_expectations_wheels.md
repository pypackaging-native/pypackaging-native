# Expectations that projects provide ever more wheels

Most users expect `pip install project-name` to work on all platforms. When
building from source is difficult, that expectation translates into requests to
projects to provide more wheels. Building many wheels can result in a lot
of maintenance work and load on CI systems.


## Current state

See [this summary](purposes_of_pypi.md#users-installing-python-packages-from-binaries)
for the combination of operating systems, Python interpreters, CPU
architectures and `libc` flavors which are supported by wheel specs/tags. The
[`cibuildwheel` matrix](https://cibuildwheel.readthedocs.io/en/stable/) adds up
to 70 combinations - and that doesn't include multiple `manylinux` versions,
`universal2` for macOS, i686 (32-bit) Linux, or some architectures that were
added in [PEP 599](https://peps.python.org/pep-0599/) (`armv7l`, `ppc64`).
Nor does it include tags that are likely to be added ([WASM](https://discuss.python.org/t/support-wasm-wheels-on-pypi/21924)), or for which tags exist but aren't supported by PyPI
([Pyston](https://github.com/mesonbuild/meson-python/issues/142#issuecomment-1262773682),
[AIX](https://github.com/pypa/pip/issues/6922)).

In practice, the less popular platforms are not supported by most projects and
the number of wheels they upload to PyPI is in the 20-40 range. Examples:

- [Kivy 2.1.0](https://pypi.org/project/Kivy/2.1.0/#files) (20 wheels),
- [NumPy 1.23.5](https://pypi.org/project/numpy/1.23.5/#files) (27 wheels),
- [Numba 0.56.4](https://pypi.org/project/numba/0.56.4/#files) (27 wheels),
- [Mypy 0.991](https://pypi.org/project/mypy/0.991/#files) (29 wheels),
- [asyncpg 0.27.0](https://pypi.org/project/asyncpg/0.27.0/#files) (35 wheels),
- [Pydantic 1.10.2](https://pypi.org/project/pydantic/1.10.2/) (35 wheels),
- [PyGame 2.1.2](https://pypi.org/project/pygame/2.1.2/#files) (57 wheels).

That is still a large amount of wheels, and maintaining support for them is
often problematic.

A key ingredient for building wheels is availability of CI systems which
support the target platforms. Most projects use 3-4 different CI systems,
and/or make use of cross-compilation when a target platform isn't natively
supported (e.g., macOS arm64 CI runners were unavailable for 2 years, and are
still only available on Cirrus CI as of Dec 2022, and PowerPC (`ppc64le`) and
IBM Z (`s390x`) are only available on Travis CI). `cibuildwheel` has lowered
the effort to build wheels compared to
[`multibuild`](https://github.com/multi-build/multibuild/), however the CI
system requirements haven't changed. Each CI system requires maintenance - and
self-hosted CI runners are even worse in this respect. That maintenance cost
can be shared much more easily for packaging systems with centralized builds;
for building wheels they have to be paid by every project (see [Lack of a build
farm for PyPI](no_build_farm.md) for more details).


## Problems

The primary problem is the large amount of maintenance effort on CI systems for
wheel building. This is particularly painful because:

- there are usually one a couple of maintainers (perhaps even a single
  person) who are responsible for or have expertise in wheel building,
- issues that show up are often specific to the platform and hence
  cannot be debugged locally on the maintainer's development machine,
- there are a lot of "layers" to wade through: CI system, cibuildwheel,
  pip, build backend, backend, build system.

Another issue is that a lot of discussion needed each time there is a request
to any project to support a new platform.


## History

TODO


## Relevant resources

Links to key issues, forum discussions, PEPs, blog posts, etc.

- [Supporting WASM wheels on PyPI](https://discuss.python.org/t/support-wasm-wheels-on-pypi/21924) thread on Discourse (2022)
- [Reducing effort spent on wheels?](https://mail.python.org/archives/list/numpy-discussion@python.org/thread/46HT2SYDHBNLOC6N5RTXI7CN32YWIJWR/#46HT2SYDHBNLOC6N5RTXI7CN32YWIJWR) thread on the NumPy mailing list (2021)

Related to platform support:

- [PEP 11 - CPython platform support](https://peps.python.org/pep-0011/),
- [NEP 29 - Recommend Python and NumPy version support as a community policy standard](https://numpy.org/neps/nep-0029-deprecation_policy.html),
- [SciPy Toolchain Roadmap & Official Builds](http://scipy.github.io/devdocs/dev/toolchain.html#official-builds).


## Potential solutions or mitigations

- Reducing the number of wheels needed. [HPy](https://hpyproject.org/) is
  probably the most promising way of achieving this.
- Defining a "supported platforms" policy that can be used across projects to
  make and document decisions around this topic.
- A build farm for PyPI (see [this meta topic](no_build_farm.md)).
- Having PyPI integrate better with other packaging systems, so users can
  obtain pure Python packages from PyPI in combination with hard-to-build
  packages from a package manager with support for the platform of interest.
- Manage user expectations better, and
  [be honest about limitations of PyPI/wheels](https://discuss.python.org/t/wanting-a-singular-packaging-tool-vision/21141/92).
  Users are often better served by a distribution that is built and tested in a
  consistent fashion than by downloading a standalone interpreter and then
  installing packages from PyPI. This is often not even presented on install
  pages as an option, let alone recommended.
