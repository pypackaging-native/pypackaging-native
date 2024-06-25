# Metadata handling on PyPI

Metadata for sdists and wheels on PyPI is contained in those sdists and wheels
themselves, in a form as prescribed by the
[Core metadata specifications](https://packaging.python.org/en/latest/specifications/core-metadata/).
That metadata format is typically produced by build backends. The source of
that metadata is information that package authors maintain within their
project. It historically lived in `setup.py`, and may still live there today -
however it now preferably should be specified in `pyproject.toml`
(see [PEP 621 - Storing project metadata in pyproject.toml](https://peps.python.org/pep-0621/)).
Specifying that metadata is relatively straightforward, and is mostly in decent
shape (even though there are some loose ends[^1]) - with the exception of
native dependencies, as discussed [on this key issue page](native-dependencies/index.md).

[^1]:
    Example of a not unimportant loose end:
    [PEP 639 - Improving License Clarity with Better Package Metadata](https://peps.python.org/pep-0639/)
    is still in Draft status and not supported by PyPI as of Dec 2022.


??? question "Can package metadata be queried for PyPI?"

    *From [this Discourse thread](https://discuss.python.org/t/query-package-metadata-from-pypi-org/21962)
    (Dec 2022):*

    !!! quote ""

        `dist-info-metadata` is available in the simple API for wheels (and sdists
        if they follow PEP 643). But PyPI doesn’t expose that data (yet).

        The PyPI JSON API includes metadata, but it is unreliable as it is at the
        project release level, so it doesn’t take into account the possibility of
        different wheels having different metadata. But it’s the nearest you can get
        right now.

    See also [the Project Metadata Table in BigQuery](https://warehouse.pypa.io/api-reference/bigquery-datasets.html#project-metadata-table).

    Similarly, metadata *about* the package can be queried. For example,
    [Libraries.io](https://libraries.io/pypi/) provides an easy to use overview
    of projects including reverse dependencies. And there are a number of ways
    of querying package download statistics, see for example
    [pypistats.org](https://pypistats.org/) for a UI with quick numbers, and the
    [Analyzing PyPI package downloads](https://packaging.python.org/en/latest/guides/analyzing-pypi-package-downloads/)
    page of the Python Packaging User Guide which lists a number of tools to
    enable more in-depth analysis.

While specifying metadata for a package is relatively straightforward in most
cases[^2], the same cannot be said for the workflows around dealing with
problems in metadata.

[^2]:
    See *Example: Using the NumPy C API* [on this page](abi.md#current-state)
    for a case where getting the right metadata into a wheel is very difficult.


## Current state

By design, metadata in artifacts on PyPI is (a) immutable and (b) contained
within the artifact itself rather than available separately. Both of those
design aspects can be problematic.

### Impact of immutable metadata

When a package author discovers an issue with their release or with a
particular artifact in a release, there are good solutions. In the former case,
a release can be [yanked](https://pypi.org/help/#yanked). In the latter case, a
new artifact with a higher build number can be uploaded.
[This blog post by Brett Cannon](https://snarky.ca/what-to-do-when-you-botch-a-release-on-pypi/)
(2021) explains what to do in those cases in detail. Where immutability
becomes a problem is when the issue is in the metadata - in particular, when a
build or runtime dependency changes something that breaks a package. It's a
problem because:

1. Doing a new release for complex packages with native code is very expensive.
   It may be days of full-time work (builds may have to run on multiple CI
   systems, and those configs tend to degrade pretty quickly for older
   releases), and therefore it may not be feasible to do on short notice.
2. Having affected users deal with the situation by themselves is also very
   expensive. The most popular packages have hundreds of thousands or even
   millions of users, so even if only 1% of users[^3] are affected by a problem
   with a dependency, that is still an unacceptably large amount of work (and
   probably lots of complaints on the issue tracker).

[^3]:
   The number of users on platforms without wheel support on PyPI is on the
   order of 1%, and that is a set of users that is frequently affected by
   issues with build dependencies.

(2) is often advocated for by Python packaging experts, in particular by having
users apply post-hoc constraints through a
[constraints file](https://pip.pypa.io/en/stable/user_guide/#constraints-files).
(2) is the worst solution though in the case of large-scale breakage, both
because of the large numbers of users that each need to take action and because
*users are, more often than not, not developers*. Instead, they're (data)
scientists, engineers, business analysts and so on. They don't want to, and
shouldn't need to, understand things like constraints files. If the metadata
needs patching, the far better solution would be to patch them on PyPI. And
this is not possible, because artifacts are immutable.

Depending on the situation, these are the most common ways that an issue with a
dependency gets dealt with:

- Bite the bullet and do a new release of the affected package,
- Convince the authors of the dependency that broke things to unbreak them
  again (e.g., undo removal of a deprecated or private API),
- Or even temporarily yank the dependency that broke things.

Because this kind of situation happens frequently, it may also be a good idea to
add upper bounds on version specifiers of dependencies. No one likes upper
bounds, because they result in incompatibilities and make dependency resolution
more difficult. A lot of effort has been spent discussing the issues with upper bounds
(e.g., see [this blog post](https://iscinumpy.dev/post/bound-version-constraints/) and
[this Discourse thread](https://discuss.python.org/t/requires-python-upper-limits/12663));
package authors are caught between a rock and a hard place though - the problem
is immutability of metadata.

!!! quote "On upper bounds - Matthias Bussonnier"

    I echo many sentiments here that 1) I hate that some projects have to put
    an upper bound [in their metadata], but 2) they do it because removing the
    upper bound is worse.

Experience with other package managers that *are* able to patch metadata shows
that this is a much nicer experience. For example, conda-forge uses
["repo data patching"](https://conda-forge.org/docs/orga/guidelines.html#fixing-broken-packages),
while Spack and Nix build from source (with a binary cache providing many
common build configs) and the
[Spack repo](https://github.com/spack/spack/tree/develop/var/spack/repos/builtin/packages)
and [Nixpgs repo](https://github.com/NixOS/nixpkgs/tree/master/pkgs/development/python-modules)
contain metadata for all packages and can therefore be updated via a PR.
As a result, upper bounds that are present on PyPI can typically be left out
safely in these package managers; applying new constraints later is cheap.

Managing the necessary upper bounds itself is an exercise that may have to be
repeated for each release, and takes time and effort. See for example
[this part of the SciPy developer guide](https://docs.scipy.org/doc/scipy-1.9.3/dev/core-dev/index.html#updating-upper-bounds-of-dependencies).


### Metadata contained within artifacts

Each wheel has its own metadata contained within the artifact. It can be
different metadata than that for the sdist for which it came - and this is more
likely to happen for packages with native code. Wheels for packages with native
code also tend to be larger -from tens of MBs for the likes of NumPy, SciPy,
Pandas and PyArrow to many hundreds of MBs for deep learning packages.

Downloading such large packages in order to access the metadata is clearly
suboptimal. Especially if that metadata then shows a conflict and Pip has to
backtrack. Also during debugging install issues this is a significant problem -
when one wants to go through a number of wheels and compare differences with
for example the `METADATA` or `RECORD` files, the current process is slow and
bandwidth-intensive.

The solution seems obvious: make metadata separately accessible from wheels.
Luckily, the solution for this is currently in progress:

!!! note "PEP 658 - Serve Distribution Metadata in the Simple Repository API"

    [PEP 658](https://peps.python.org/pep-0658/) (accepted) proposes to make the
    metadata file in the `.dist-info` directory of a wheel separately available.
    This should solve the problems identified in this section. Support is
    [already implemented in `pip`](https://github.com/pypa/pip/pull/11111).
    Implementation in PyPI is still pending, see
    [warehouse#8254](https://github.com/pypi/warehouse/issues/8254).

There are also issues around packages who don't yet use static metadata in
`pyproject.toml`, and reliable metadata for sdists being only relatively
recently available ([PEP 643, Nov 2020](https://peps.python.org/pep-0643/)).
With dynamic metadata or `setup.py` usage, sdists have to be built in order
to obtain the metadata. This is a general packaging issue however, not specific to
packages with native code, and not nearly as much of a problem as the other
issues discussed higher up. See, e.g.,
[pip#1884](https://github.com/pypa/pip/issues/1884) and
[this thread](https://discuss.python.org/t/pip-download-just-the-source-packages-no-building-no-metadata-etc/4651/12)
for details.


## Problems

The most important problem is the need to add upper bounds on version
specifications of dependencies.


## History

- Older analyses of PyPI dependencies include
  [this one from Olivier Girardot](https://ogirardot.wordpress.com/2013/01/05/state-of-the-pythonpypi-dependency-graph/) (2013),
  [this one from Martin Thoma](https://martin-thoma.com/analyzing-pypi-metadata/) (2015), and
  [this one from Kevin Gullikson](https://kgullikson88.github.io/blog/pypi-analysis.html) (2016).
- The [Requires-Python upper limits](https://discuss.python.org/t/requires-python-upper-limits/12663) Discourse thread (2021) went into detail on issues around specifying the upper bound of supported Python versions.

TODO: add more history


## Relevant resources

TODO


## Potential solutions or mitigations

- Making metadata editable. This would require a PEP and be a large effort.
- The impact of issues with build dependencies, and hence the need to add upper
  bounds, would be much less if Pip did not install from sdist by default, as
  discussed in
  [Unsuspecting users getting failing from-source builds](unexpected_fromsource_builds.md).
- Fix issues in installers. E.g., Poetry and PDM should not propagate upper
  bounds the way they currently do, as discussed in
  [this thread](https://discuss.python.org/t/requires-python-upper-limits/12663).
  Pip also needs to continue reducing the amount of excessive backtracking, and
  use the separate metadata available soon with PEP 568 to reduce the impact of
  that backtracking. See
  [Possible ways to reduce backtracking](https://pip.pypa.io/en/latest/topics/dependency-resolution/#possible-ways-to-reduce-backtracking)
  in the Pip docs for current mitigation options available to users.
