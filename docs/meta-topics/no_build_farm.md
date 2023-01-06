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

This topic has been discussed on and off, however there is no concrete effort
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

!!! info "The cost of a build farm"

    Build farms come with substantial-to-massive costs in terms of build &
    hosting infrastructure, automation that needs to be maintained, and
    generally human attention/intervention for cases that go wrong.

    These costs already appear for the above-mentioned "build and verify" model
    used for CRAN. However, when wishing to follow the widely-used practice
    among distributions to rely predominantly on shared libraries, this brings
    a massive further increase in necessary automation and maintenance (though
    the benefits are such that this is the norm rather than the exception).

    In particular, doing so needs careful tracking which packages are exposed to
    any given (shared) library's ABI, the ABI stability of that library as
    expressed through its version scheme, rebuilding all dependent packages once
    an ABI-changing library version is released, and finally representing all
    that accurately enough in metadata so that the resolver will only choose
    compatible package combinations for a given environment.

    Due to the limitation of only having a single shared library on the path
    searched by the linker for symbols at runtime (see [here](../background/compilation_concepts.md#linkers)
    for more details), this kind of ecosystem-wide rebuild needs to be done
    relatively quickly. This is because any given package that has a release
    in the meantime can only be compiled for either the old or for the new ABI
    (with very few exceptions for transitions that are more onerous, otherwise
    one quickly suffers a combinatorial explosion of required build variants),
    and _not_ moving the ecosystem as a whole to the new baseline essentially
    means a bifurcation which packages can be co-installed with each other.

    For an impression of the amount of effort required for the maintenance of
    such an undertaking, see for example the permanently ongoing so-called "migrations"
    in conda-forge, e.g. [here](https://conda-forge.org/status/#other_migrations)
    and [here](https://conda-forge.org/status/#closed_migrations). While a lot
    of rebuilds can be automated (requiring infrastructure that is maintained
    and operated), the initial integration of a library needs to be done
    manually, and a persistent percentage of packages (say ~1-10%) will fail
    any given migration for various reasons, requiring further intervention of
    a dedicated group of build farm maintainers (occasionally requiring
    backporting or even authoring patches against the library sources). This is
    not unique to conda-forge, but a reality for distributions that follow this
    model, from Debian, Fedora and Ubuntu to Gentoo, vcpkg etc.


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
