# Lack of a build farm for PyPI

There is no centralized service (or "build farm") for building wheels for
packages on PyPI. This is not for lack of agreement on whether that is desirable
or not (it's the top item on
[the PSF's fundable packaging improvements](https://github.com/psf/fundable-packaging-improvements/blob/master/FUNDABLES.md)),
but because it's a huge amount of work to create and maintain such a service.

Benefits of a build farm include:

- Easier to build wheels, which means less effort for package authors and a
  higher percentage of packages having wheels available,
- Ability to use a consistent build environment, leading to higher quality
  wheels and fewer issues with binary compatibility between different packages
  (see, e.g., [managing ABI compatibility](../key-issues/abi.md) and
  [complex C++ dependencies](../key-issues/native-dependencies/cpp_deps.md)),
- Ability to roll out upgrades to compiler toolchains or rebuilds of many
  packages at once (e.g., for a new Python version),
- Detecting of some classes of issues with releases at upload-to-PyPI time
  rather than when users open bug reports (e.g., ensure all `.py` files in a
  package can actually be imported),
- Opportunity to grow a packaging community, with collective expertise,
  resources and policies.

Two package repositories that are fairly similar to PyPI, CPAN for Perl and
CRAN for R, both don't allow hosting packages on the repository unless the
package passes a series of tests. This includes tests that attempt to build the
library in an isolated environment to ensure that all dependencies are
correctly listed, and running unit tests. From anecdotal evidence, the average
quality of those packages is higher, and it's easier to redistribute them in
other packaging systems. Pretty much all system package managers as well as
Conda and Spack follow similar designs - at least a build should pass and the
package should be importable.


## Current state

This topic has been discussed on and of, however there is no concrete effort
happening in this direction as of Dec 2022.

One early attempt at a solution was
[conda-press](https://github.com/conda-incubator/conda-press), which aimed to
repackage conda packages into wheels, has not been updated since 2019. There
were some unresolved design questions or issues with it, and it is unclear if
the approach would result in problem-free wheels for the more complex packages.

Even if `conda-press` isn't going anywhere, the conda-forge infrastructure is
perhaps the most suitable infrastructure to use as a base to create a new build
farm from. It would still be a lot of work to adapt it to PyPI - but far less
than starting from scratch.

A build farm needs maintainers and its own community around it. That community
doesn't exist, and bootstrapping it isn't easy. However, there *are* places
where maintainers have coalesced around common tooling. `cibuildwheel` is
probably the most central point. Now that `multibuild` is abandoned,
`cibuildwheel` is being used to build wheels on CI systems that are free for
open source projects to use by many of the most popular projects with native
code.


## Problems

A build farm has many advantages, listed above. Not having a build farm means
not having those advantages.


## History

TODO


## Relevant resources

TODO


## Potential solutions or mitigations

"Build a build farm" is easy to write down here, but it's a huge effort and it
is unclear how to plan and fund such an effort.
