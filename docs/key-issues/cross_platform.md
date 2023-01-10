# Cross-platform installation

The historical assumption of compilation is that the platform where the code is
compiled will be the same as the platform where the final code will be executed
(if not literally the same machine, then at least one that is CPU and ABI
compatible at the operating system level). This is a reasonable assumption for
most desktop projects; However, for mobile platforms, this isn't the case.

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

## Potential solutions or mitigations

Compiler and build toolchains (e.g., autoconf/automake) have long supported
cross-compilation; however, these cross-compilation capabilities are easy to
break unless they are exercised regularly.

In the Python space, tools like [crossenv](https://github.com/benfogle/crossenv)
also exist; these tools use a collection of path hacks and overrides of known
sources of platform-specific details (like `distutils`) to provide a
cross-compilation environment. However, these solutions tend to be somewhat
fragile as they aren't first-class citizens of the Python ecosystem.
