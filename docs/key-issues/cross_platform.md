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

A microcosm of this problem exists on macOS as a result of the Apple Silicon
transition. Most CI systems don't provide native ARM hardware, but most
developers will still want ARM64-compatible build artefacts. Apple has provided
the tools compile [fat binaries](multiple_architectures.md) on x86_64 hardware;
however, in this case, the host platform (macOS on x86_64) will still be one of
the outputs of the compilation process. For mobile platforms, the computer that
compiles the code will not be able to execute the code that has been compiled.

## Potential solutions or mitigations

Compiler and build toolchains (e.g., autoconf/automake) have long supported
cross-compilation; however, these cross-compilation capabilities are easy to
break unless they are exercised regularly.

In the Python space, tools like [crossenv](https://github.com/benfogle/crossenv)
also exist; these tools use a collection of path hacks and overrides of known
sources of platform-specific details (like `distutils`) to provide a
cross-compilation environment. However, these solutions tend to be somewhat
fragile as aren't first-class citizens of the Python ecosystem.
