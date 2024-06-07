# Cross compilation

The historical assumption of compilation is that the platform where the code is
compiled will be the same as the platform where the final code will be executed
(if not literally the same machine, then at least one that is CPU and ABI
compatible at the operating system level). This is a reasonable assumption for
most desktop platforms; however, for some platforms, this isn't the case.

On mobile platforms, an app is compiled on a desktop platform, and transferred
to the mobile device (or a simulator) for testing. The compiler is not executed
on device. Therefore, it must be possible to build a binary artefact for a CPU
architecture and an ABI that is different from the platform that is running the
compiler. The situation is similar for embedded devices.

Cross compilation issues also emerge when dealing with continuous
integration/deployment (CI/CD). CI/CD platforms (such as Github Actions)
generally provide the "common" architectures - often only x86-64 - however, a
project may want to produce binaries for other platforms (e.g., ARM support for
Raspberry Pi devices; PowerPC or s390x for mainframe/server devices; or for
mobile platforms). These binaries won't run natively on the host CI/CD system
(without some sort of emulation, for example with QEMU); but code can be
compiled for the target platform.

macOS also experiences this as a result of the Apple Silicon transition. Apple
has provided the tools to make cross compilation from x86-64 to arm64 as easy
as possible, as well as to compile [fat binaries](multiple_architectures.md)
(supporting x86-64 and arm64 at the same time) on both architectures. In the
latter case, the host platform will still be one of the outputs of the
compilation process, and the resulting binary will run on the CI/CD system.


## Current state

Native compiler and build toolchains (e.g., autoconf/automake, CMake, Meson) have long
supported cross-compilation; however, such cross-compilation capabilities for any
given project tend to bitrot and break easily unless they are exercised regularly.

CPython's build system includes some support for cross-compilation. This support
is largely based on leveraging autoconf's support for cross compilation. This
support wasn't well integrated into `distutils` and the compilation of the binary
portions of stdlib. The removal of `distutils` in Python 3.12 represents an
improvement the overall situation, but there is still a long way to go before
the ecosystem as a whole has fully integrated the consequences of this change.

The way build backend hooks in `pyproject.toml` are specified (see PEP 517)
means cross-platform compilation support has been partially converted into a
concern for individual build systems to manage.

In order to cross-compile a Python package, one needs a compiler toolchain as
well as two Python installs - one for the build system and one for the host
system.[^1] This can make it a little challenging to get started. If a compiler
toolchain is not already provided on the system of interest, it can be built
from source with, e.g., [crosstool-ng](https://crosstool-ng.github.io/) or
obtained from, e.g., [dockcross](https://github.com/dockcross/dockcross).
Or one can use a packaging system that has builtin support for cross-compilation.
[The Yocto Project](https://www.yoctoproject.org/),
[OpenEmbedded](https://www.openembedded.org/wiki/Main_Page) and
[Buildroot](https://buildroot.org/) are projects specifically focused on
cross-compilation for Linux embedded systems. More general-purpose packaging
ecosystems often have toolchains and supporting infrastructure to cross-compile
packages for their own needs - see, e.g., info for
[Void Linux](https://github.com/void-linux/void-packages#cross-compiling),
[conda-forge](https://conda-forge.org/),
[Debian](https://wiki.debian.org/CrossCompiling) and
[Nix](https://nixos.org/guides/cross-compilation.html).

[^1]:
    The "build", "host" and "target" terminology for identifying which system
    is which in a cross-compilation setup is not consistent across build
    systems and packaging tools. Always carefully check whether "build" means
    the machine on which the compilation is run and "host" the machine on which
    the produced binaries will run - or vice versa.

Tools like [crossenv](https://github.com/benfogle/crossenv) can be used to trick
Python into performing cross-platform builds. These tools use path hacks and
overrides of known sources of platform-specific details (like `sysconfig` and
`distutils`) to provide a cross-compilation environment. However, these
solutions tend to be somewhat fragile as they aren't first-class citizens of
the Python ecosystem.

[The BeeWare Project](https://beeware.org) also uses a version of these
techniques. For both the platforms it supports, BeeWare provides a custom
package index that contains pre-compiled binaries ([Android](https://chaquo.com/pypi-7.0/);
[iOS](https://anaconda.org/beeware/repo)). These binaries are produced using a
set of tooling ([Android](https://github.com/chaquo/chaquopy/tree/master/server/pypi);
[iOS](https://github.com/freakboy3742/chaquopy/tree/iOS-support/server/pypi))
that is analogous to the tools used by conda-forge to build binary artefacts.


## Problems

There is currently a gap in _communicating target platform details to the
build system_. While a build system like Meson or CMake may support
cross-platform compilation, and a project may be able to cross-compile binary
artefacts, invocation of a `pyproject.toml` build hook typically assumes that the
platform running the build will be the platform that ultimately runs the Python
code. As a result, `sys.platform`, or the various attributes of the `platform`
and `sysconfig` modules can't be used as part of the build process.

_Running Python code_ for the host (cross) platform is not possible (modulo
using an emulator), but Python packages have not taken this into account and
provided ways to avoid the need to run the host interpreter. For example,
`numpy` and `pybind11` ship headers and have `get_include()` functions in their
main namespaces to obtain the path to those headers. That is clearly a problem,
which packages dependending on those headers have to work around (often done by
patching those packages with hardcoded paths within a cross-compilation setup).

`pip` provides support for installing wheels for a different platform
by specifying a `--platform`, `--implementation` and `--abi` flags. However,
these flags only work for packages with wheels, not sdists. Therefore, for
cross compilation setups that rely on `pip` rather than another package manager
to install build dependencies, it is cumbersome in practice to prepare the host
(non-native) part of the cross build environment - a single missing `-none-any`
wheel for a dependency that is pure Python necessitates hacks to get it
installed.[^2]

[^2]:
    The correct solution - filing issues on each project asking them to upload
    a `-none-any` wheel next to their sdist - typically has a long lead time.
    Therefore [Briefcase](https://beeware.org/project/projects/tools/briefcase/), the
    packaging tool for Beeware, patches `pip` to allow installing projects from
    sdists when `--platform` is specified and only error out when the wheel
    build attempts to invoke a compiler. That way, pure Python packages can be
    installed directly.

## History

TODO


## Relevant resources

- ["Towards standardizing cross compiling "](https://discuss.python.org/t/towards-standardizing-cross-compiling/10357), Ben Fogle (2021),
- ["PEP xxxx - Standardized Config Settings for Cross-Compiling"](https://github.com/benfogle/peps/blob/master/pep-9999.rst), Ben Fogle (2021),
- [scipy#14812 - Tracking issue for cross-compilation needs and issues](https://github.com/scipy/scipy/issues/14812) (2021),


## Potential solutions or mitigations

At the core, what is required is a recognition that the use case of
cross-platform builds is something that the Python ecosystem should support.

In concrete terms, for native modules, this would require at least:

1. Making it possible to retrieve relevant metadata from a Python installation
   without having to run Python code.
2. Clear separation of metadata associated with the definition of build and
   target platforms, rather than assuming that build and target platform will
   always be the same.

In addition, to make cross-compilation easier to use and move from build system
specific configuration files - like a "toolchain file" for CMake or a "cross
file" for Meson - to a standardized version:

3. Extension of the `pyproject.toml` build interface to allow communicating the
   desired target platform as part of a binary build; or
4. Formalization of the "platform identification" interface that can used by
   build backends to identify the target platform, so that tools like
   `crossenv` can provide a reliable proxied environment for cross-platform
   builds.
