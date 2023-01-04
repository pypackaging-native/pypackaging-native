# Unsuspecting users getting failing from-source builds

## Current state

When a project makes a release, it typically uploads one sdist (a source
distribution) and multiple wheels (binary installers). Wheels are primarily
meant to make the installation experience better and faster. For projects which
contain code that needs to be compiled (e.g., C/C++/Cython), installing from
the sdist is challenging. The sdist metadata does not even allow expression the
required dependencies (e.g., a compiler - see
[native dependencies](native-dependencies/index.md)). Hence installing from an sdist often
goes wrong. Why does a user get an sdist when they didn't expect one? This can
happen in quite a few circumstances:

- Shortly after the release of a new Python version, most projects will not yet
  have wheels for that new Python version uploaded to PyPI. So when a user
  installs the "latest and greatest" Python and type `pip install somepackage`,
  they are likely to see `pip` try to install the sdist of the highest version
  of `somepackage`.
- In case new hardware becomes available. A recent example is macOS arm64: it
  took many scientific projects over a year before they were able to build
  `arm64` or `universal2` wheels and upload them to PyPI. All users which used
  a native arm64 Python were getting builds from sdist.
- Users who use an old `pip` version (e.g. the `pip` shipped with their distro
  on a typical HPC cluster) which does not have support for recent `manylinux`
  versions may see `pip` try to install from an sdist even though there are
  (say) `manylinux2014` wheels for the package.
- Installs from sdist may happen if a project tags a release but there's a
  problem on a particular platform for which it normally uploads wheels.
  Especially if this is a less popular platform (e.g., `ppc64le`) the release
  manager may just go ahead with the release, and aim to upload the missing
  wheels later.
- If a project is uploading a new version and the person doing the release
  isn't careful to upload all wheels first and the sdist afterwards, then users
  on any platform may see installers try to install from the sdist. This
  mistake is easy to make, and can lead to a lot of failed installs quickly if
  a package is popular (e.g., the download rate for `numpy` is ~2000/minute).

Many years ago, users were expecting to build from source. Today, in 2022, the
scientific Python ecosystem has tens of millions of users. The vast majority of
those users are not expecting, and are often unable, to build from source when
they type a command like `pip install scikit-learn`.


## Problems

There clearly are a lot of issues due to installing from an sdist when the user
did not intend to do that. For users:

- Failed installs, often with confusing error messages and after a possibly
  time-consuming build step.
- Installs that appear to succeed but have issues that show up at runtime as a
  result of building against incorrect or mismatching libraries. This ranges
  from import errors due to missing symbols in shared libraries to segfaults
  and silently wrong numerical results.

For `pip` maintainers: a _lot_ of bug reports they have to deal with because
the user thinks `pip` is the cause rather than the package they tried to
install.

For maintainers of projects with compiled code:

- A lot of bug reports that are very time-consuming to address. Issues are
  often not reproducible, and bug reports typically do not contain enough
  information to be able to understand if the problem is user error or an
  actual bug in the project.
- A lot of time spent carefully managing build dependencies and their versions
  in `pyproject.toml` (see, e.g., the
  [oldest-supported-numpy metapackage](https://github.com/scipy/oldest-supported-numpy))
  which has as its only purpose to serve as a build dependency which pins
  `numpy` to the correct version (typically the lowest version for which there
  are wheels on PyPI) per platform and Python version/interpreter.
- Being forced to support platforms that are already end of life, because the
  project does not have a good way of dropping support for older `manylinux`
  flavors (see, e.g.,
  [numpy#19192](https://github.com/numpy/numpy/issues/19192)).


## History

TODO


## Relevant resources

TODO


## Potential solutions or mitigations

1. Do not upload sdists to PyPI at all. This is the approach taken by many of
   the projects with the most complex builds - for example PyTorch, TensorFlow,
   MXNet, jaxlib, and Ray. It is necessary to then delete every single sdist
   for any version of the package from PyPI - if there was a single sdist for
   version 0.1.0, even yanking that is not enough (PyTorch found this out the
   hard way, some long-running issues were closed when deleting old yanked
   sdists).
2. Change the behavior of installers to not use sdists by default. Make it easy
   for users to opt in to installing from source, but by default only look for
   wheels and error out with a clear message if no wheels matching the users'
   platform and Python interpreter are found.
   _The `pip` maintainers recently agreed to take this direction, see_
   [pypa/pip#9140](https://github.com/pypa/pip/issues/9140).
   Also note that it's recommended to upload wheels even for projects that are
   pure Python, because installs are faster (metadata in a wheel is static, no
   need to run `setup.py - see
   [this blog post](https://pradyunsg.me/blog/2022/12/31/wheels-are-faster-pure-python/)
   for a more detailed explanation). There are very few packages which would be
   unable to upload a wheel.
3. Let individual packages determine the behavior of installers (try to install
   from sdist, or error out) via metadata on PyPI somehow.
4. Individual solutions for some of the separate issues. For example, reduce
   the load on `pip` maintainers via better error messages, and let projects
   who want to drop support for old `manylinux` versions detect the `pip`
   version in their build scripts/files, and error out if a too old version is
   detected.

