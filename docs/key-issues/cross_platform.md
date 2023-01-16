# Cross-platform installation

The historical assumption of compilation is that the platform where the code is
compiled will be the same as the platform where the final code will be executed
(if not literally the same machine, then at least one that is CPU and ABI
compatible at the operating system level). This is a reasonable assumption for
most desktop platforms; however, for some platforms, this isn't the case.

On mobile platforms, an app is compiled on a desktop platform, and transferred
to the mobile device (or a simulator) for testing. The compiler is not executed
on device. Therefore, it must be possible to build a binary artefact for a CPU
architecture and a ABI that is different the platform that is running the
compiler.

Cross compilation issues also emerge when dealing with continuous
integration/deployment (CI/CD). CI/CD platforms (such as Github Actions)
generally provide the "common" architectures - often only x86-64 - however, a
project may want to produce binaries for other platforms (e.g., ARM support for
Raspberry Pi devices; PowerPC or s390 for mainframe/server devices; or for
mobile platforms). These binaries won't run natively on the host CI/CD system
(without some sort of emulation); but code can be compiled for the target
platform.

macOS also experiences this as a result of the Apple Silicon transition. Apple
has provided the tools to compile [fat binaries](multiple_architectures.md) on
x86_64 hardware; however, in this case, the host platform (macOS on x86_64) will
still be one of the outputs of the compilation process, and the resulting binary
will run on the CI/CD system.

## Current state

Native compiler and build toolchains (e.g., autoconf/automake, CMake) have long
supported cross-compilation; however, such cross-compilation capabilities for any
given project tend to bitrot and break easily unless they are exercised regularly.

CPython's build system includes some support for cross-compilation. This support
is largely based on leveraging autoconf's support for cross compilation. This
support wasn't well integrated into distutils and the compilation of the binary
portions of stdlib. The removal of distutils in Python 3.12 represents an
improvement the overall situation, but there is still a long way to go before
the ecosystem as a whole has fully integrated the consequences of this change.

The specification of PEP517 means cross-platform compilation support has been
largely converted into a concern for individual build systems to manage.

## Problems

There is currently a gap in communicating target platform details to the
build system. While a build system like autoconf or CMake may support
cross-platform compilation, and a project may be able to cross-compile binary
artefacts, invocation of the PEP517 build interface currently assumes that the
platform running the build will be the platform that ultimately runs the Python
code. As a result, `sys.platform`, or the various attributes of the `platform`
library can't be used as part of the build process.

`pip` provides limited support for installing binaries for a different platform
by specifying a `--platform`, `--implementation` and `--abi` flags; however,
these flags only work for the selection of pre-built binary artefacts, and are
therefore constrained to the set of platform and ABI tags published by the
author.

## History

Tools like [crossenv](https://github.com/benfogle/crossenv) can be used to trick
Python into performing cross-platform builds. These tools use path hacks and
overrides of known sources of platform-specific details (like `distutils`) to
provide a cross-compilation environment. However, these solutions tend to be
somewhat fragile as they aren't first-class citizens of the Python ecosystem.

[The BeeWare Project](https://beeware.org) also uses a version of these
techniques. On both platforms, BeeWare provides a custom package index that
contains pre-compiled binaries ([Android](https://chaquo.com/pypi-7.0/);
[iOS](https://anaconda.org/beeware/repo)). These binaries are produced using a
set of tooling
([Android](https://github.com/chaquo/chaquopy/tree/master/server/pypi);
[iOS](https://github.com/freakboy3742/chaquopy/tree/iOS-support/server/pypi))
that is analogous to the tools used by conda-forge to build binary artefacts.

## Relevant resources

TODO

## Potential solutions or mitigations

At it's core, what is required is a recognition that the use case of
cross-platform builds is something that the Python ecosystem should support.

In concrete terms, for native modules, this would require some combination of:

1. Extension of the PEP517 interface to allow communicating the desired target
   platform as part of a binary build; or

2. Formalization of the "platform identification" interface that can used by
   PEP517 build backends to identify the target platform, so that tools like
   `crossenv` can provide a reliable proxied environment for cross-platform
   builds.

3. Clear separation of metadata associated with the definition of build and
   target platforms, rather than assuming that build and target platform will
   always be the same.

4. Extension of the PEP517 interface to report when a build steps (e.g., running
   a code generation tool) cannot be run on the target hardware.
