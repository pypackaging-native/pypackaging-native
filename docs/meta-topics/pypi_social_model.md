# PyPI's author-led social model and its limitations

PyPI has an "author-led social model", meaning that individual package authors
(or groups of authors) control if and when packages get published on PyPI and
what they contain. PyPI itself has maintainers, but their purview is the
functioning of the PyPI service/infrastructure - dealing with file size limits
and issues around names of packages on PyPI (see
[PEP 541 - Package Index Name Retention](https://peps.python.org/pep-0541)) is
the very limited extent to which they get involved in what happens for any
particular package name on PyPI.

An author-led model is common for language-specific package repositories -
[npm](https://www.npmjs.com/) (JavaScript),
[crates.io](https://crates.io/) (Rust),
[CRAN](https://cran.r-project.org/) (R),
[RubyGems](https://rubygems.org/) (Ruby), and
[CPAN](https://www.cpan.org/)(Perl) and work roughly the same way[^1].

[^1]:
    PyPI has perhaps the least amount of rules and uniformity of all of the
    mentioned language-specific repositories, however the big picture is the
    same for all of them.

This contrasts with system package managers, which typically have a more
centralized social model - they typically have a set of policies to specify
that packaging gets done a certain way, and mechanisms and tools for a central
team to enforce those policies.


## Current state

There are pros and cons to an author-led social model. Pros include:

- Package authors have few constraints, and can release their package on PyPI
  at the time and with the content that they alone determine.
- Because authors are in control, the latest version of a package is (almost)
  always available on PyPI. Typically well before it makes its way into a
  system package manager[^2].
- Users get a new release very quickly - this helps package development, which
  benefits from fast feedback cycles.
- The package contents can be changed in whatever way the authors desire. For
  distributions there are sometimes issues with packagers patching a package in
  a way that the original authors disagree with.

[^2]:
    For Conda, Spack, or Homebrew, the delay of a package updating may be days
    to weeks. For a Linux distros like Debian it could be weeks (unstable) to
    years (stable).

Those benefits are compelling, and most language specific repositories work
this way. So what are the cons?

First, there is an important difference between source releases and
distributing binaries here (also see [The multiple purposes of PyPI](./purposes_of_pypi.md)).
For source releases, it seems clear that authors *should* be in full control -
it's their source code after all. The vast majority of system package managers
don't deal with distributing source code at all. The cons of an author-led
model are all related to binaries:

- A key issue is that *coordination across packages* is extremely difficult,
  but necessary. For distributing a working set of binaries that depend on each
  other, there are myriad coordination points:

    - around ABI management and a common toolchain (see
      [Depending on packages for which an ABI matters](../key-issues/abi.md)),
    - about building [native dependencies](../key-issues/native-dependencies/index.md)
      once instead of having every consumer of that dependency rebuild and
      vendor it,
    - about upgrades and rolling out support for new platforms, Python
      versions, or interpreters like PyPy or Pyston,
    - about runtimes that should be unique in an environment (e.g., see the
      [content about OpenMP](../key-issues/native-dependencies/blas_openmp.md)).

- The ability to do *system integration and integration testing* is very
  limited. With PyPI, there is a limited amount of integration testing that
  package authors do in an ad-hoc fashion (e.g., downstream package authors may
  test pre-releases). For everything else, the user is the integrator.
- The *requirement to avoid external dependencies* through vendoring or static
  linking of dependencies in wheels is directly caused by the author-led social
  model. This is a significant problem, as discussed in detail in content on
  [native dependencies](../key-issues/native-dependencies/index.md).

During the introduction of support for binaries on PyPI in the form of wheels,
this social model seems to have only been used implicitly - it was more or less
a given (wheels replaced eggs, which already had similar characteristics). The
key issue of vendoring wasn't touched upon in
[PEP 427 – The Wheel Binary Package Format 1.0](https://peps.python.org/pep-0427/),
and only appears in
[PEP 513 – A Platform Tag for Portable Linux Built Distributions](https://peps.python.org/pep-0513/#bundled-wheels-on-linux)
where it gets a significant amount of consideration:

!!! quote "Bundled Wheels on Linux (PEP 513)"
    While we acknowledge many approaches for dealing with third-party library
    dependencies within `manylinux1` wheels, we recognize that the `manylinux1`
    policy encourages bundling external dependencies, a practice which runs
    counter to the package management policies of many linux distributions'
    system package managers. The primary purpose of this is cross-distro
    compatibility. Furthermore, `manylinux1` wheels on PyPI occupy a different
    niche than the Python packages available through the system package
    manager.

    ...

    The model described in this PEP is most ideally suited for cross-platform
    Python packages, because it means they can reuse much of the work that
    they’re already doing to make static Windows and OS X wheels. We recognize
    that it is less optimal for Linux-specific packages that might prefer to
    interact more closely with Linux’s unique package management functionality
    and only care about targeting a small set of particular distos.

In hindsight[^3], the viewpoint in the quote above was problematic. Linux is
not special here, aside from there being so many distros. All the major issues
around coordination, system integration and avoiding external dependencies
apply equally on Windows, macOS and Linux. And in practice, complex projects
with native code have similar issues on all platforms.

[^3]:
    Hindsight is 20/20. PEP 513 and the introduction of `manylinux` was still
    an impressive feat; some challenging technical problems around dealing with
    ABI compatibility across a large collection of Linux distros were solved in
    that process.

??? question "Who should be hitting the bugs in a new release?"

    This is a bit of a philosophical question, and only partially related to
    the author-led vs. centralized social model. However, it is also an
    important question. Packages with native code tend to have an above-average
    amount of unexpected issues for any given new version, and those issues
    take longer to fix. Package authors benefit from *fast, high-quality bug
    reports* for those issues. The average end user benefits from *stability*,
    and is often not able to generate the desired high-quality bug reports.

    So to answer the question: ideally, distro packagers, downstream package
    authors, and early adopters (power users who understand and can deal with
    the occasional hiccup) should be hitting the bugs in new releases.
    End users, who may not even realize that they're living on the edge, should not.
    A problem with PyPI/pip/wheels is that the average Python user is indeed
    living on the edge, and doesn't know it.

    !!! quote "Python usage in 2020 and 2021"

        From the [Python Developers Survey 2021 results](https://lp.jetbrains.com/python-developers-survey-2021/):

        *There are no great changes in the distribution of Python use cases over the
        years. Data analysis (51-54%), machine learning (36-38%), web development (45-48%),
        and DevOps (36-38%) are still the most popular fields for Python usage.*

    About half of all Python users are scientific, engineering, and data
    science users now. These users are not "developers" - they are scientists
    and engineers first, and programming is a tool to do their actual job. They
    shouldn't be the ones consuming `.0` releases the day they're available.
    See, e.g., also [this "using software" vs. "developing software"
    post](https://discourse.nixos.org/t/nix-vs-language-package-manager/7099/4).


## Problems

- It's difficult to do anything that requires coordination across projects. For
  example, rebuilding all projects for a new Python interpreter, toolchain
  change, or common library or protocol like Protobuf or DLPack (upgrading in
  sync after a possible breaking change is easier than staggered upgrades).
- The limitations of PyPI force themselves on Python projects as limitations on
  the project as a whole[^4]. Because there's a lot of user demand for PyPI and
  because package authors are responsible for building wheels themselves, they
  tend to not do anything that doesn't work well on PyPI. Even though the
  opportunity cost is large. They are not making such trade-offs for other
  package repositories - they tend to consider that the downstream packagers'
  problem.
- The large amount of extra effort for package authors when dealing with native
  dependencies. For any other packaging system, that cost gets shared across
  all users of the native dependency. For PyPI, there's a significant extra
  cost per project of rebuilding, vendoring, etc.
- Lack of understanding by the Python packaging community about the different
  types of Python users. In particular, what is often forgotten is the
  distinction between users who are developers (which includes ~100% of
  participants in Python packaging design discussions) and users who write
  Python to get their job done but are *not* developers (at least, they are not
  thinking about themselves as developers - Python is simply one tool in their toolbox).
  Most decisions around Python packaging only consider the developer role.

[^4]:
    "We can't use X because on PyPI we cannot guarantee property Y for X". Example:
    SciPy forbids using OpenMP, because on PyPI there is no way to ensure that
    only a single OpenMP runtime (and not `libgomp`) is used (see
    [here](https://github.com/scipy/scipy/issues/10239#issuecomment-979373613)).
    Similar for C++17 usage (blocked by `manylinux` for a long time), MPI (shared runtime),
    etc.

## History

Some history on introduction of wheels captured above. More history is TODO.


## Relevant resources

Some threads on language-specific vs. system package managers:

- [Usage of language-specific package managers](https://lists.debian.org/debian-java/2012/03/msg00061.html) Debian mailing list thread (2013),
- [Language Specific Package Managers](https://kevincox.ca/2013/10/18/language-specific-package-managers/) blog post by Kevin Cox (2013),
- [Maintaining language-specific module package stacks](https://discourse.ubuntu.com/t/maintaining-language-specific-module-package-stacks/10476) Ubuntu Discourse thread (2019),
- [Nix vs Language Package manager](https://discourse.nixos.org/t/nix-vs-language-package-manager/7099) Nix Discourse thread (2020),


## Potential solutions or mitigations

- Improve interoperability with system package managers, so that the strengths
  and weaknesses of language and system package managers can be combined by
  users.
- Separating source and binary distribution on PyPI better (as discussed
  [here](./purposes_of_pypi.md)) and/or building a build farm (as discussed
  [here](./no_build_farm.md)) may mitigate some of the issues.
- Develop an explicit shared understanding within the Python packaging
  community of where the author-led social model really breaks down and the
  effort of distributing wheels is prohibitive (e.g.,
  [the geospatial stack](../key-issues/native-dependencies/geospatial_stack.md)).
- Having draft releases on PyPI (see
  [pypi/warehouse#726](https://github.com/pypi/warehouse/issues/726)) would be a
  step in the right direction[^5] towards addressing the "coordination across
  projects" problem.

[^5]:
    Note though that the coordination across projects problem is mostly a
    social issue - who can decide on and enforce standard ways of doing things
    - rather than a PyPI infrastructure one.
