# Platforms with multiple CPU architectures

In addition to any ABI requirements, a binary is compiled for a CPU
architecture. That CPU architecture defines the CPU instructions that can be
issued by the binary.

Historically, it could be assumed that an executable or library would be
compiled for a single CPU archicture. On the rare occasion that an operating
system was available for mulitple CPU architectures, it became the
responsibility of the user to find (or compile) a binary that was compiled for
their host CPU architecture.

However, on occasion, we see an operating system platform where multiple CPU
architectures are supported:

* In the early days of Windows NT, both x86 and DEC Alpha CPUs were supported
* Windows 10 supports x86, x86-64, ARMv7 and ARM64; Windows 11 supports x86-64
  and ARM64.
* Although Linux started as an x86 project, the Linux kernel is now available a
  wide range of other CPU architectures, including ARM64, RISC-V, PowerPC, s390
  and more.
* Apple transitioned Mac hardware from PowerPC to Intel (x86-64) CPUs, providing
  a forwards compatibility path for binaries
* Apple is currently transitioning Mac hardware from Intel (x86-64) to
  Apple Silicon (ARM64) CPUs, again providing a forwards compatibility
  path
* Apple supports ARMv6, ARMv7, ARMv7s, ARM64 and ARM64e on iOS
* Android currently supports ARMv7, ARM64, x86, and x86-64; it has historically
  also supported ARMv5 and MIPS

CPU architecture compatibility is a necessary, but not sufficient criterion for
determining binary compatibility. Even if two binaries are compiled for the same
CPU architecture, that doesn't guarantee [ABI compatibility](abi.md).

In some respects, CPU architecture compatibility could be considered a superset
of [GPU compatibility](gpus.md). When dealing with multiple CPU architectures,
there may be some overal with the solutions that can be used to support GPUs in
native binaries.

## Platform approaches for dealing with multiple architectures

Three approaches have emerged for handling multiple CPU architectures.

### Multiple binaries

The minimal solution is to distribute multiple binaries. This is the approach
that was used by Windows NT, and is currently supported by Linux. At time of
distribution, an installer or other downloadable artefact is provided for each
supported platform, and it is up to the user to select and download the correct
artefact.

### Archiving

The approach taken by Android is very similar to the multiple binary approach,
with some affordances and tooling to simplify distribution.

When building an Android project, each target architecture is compiled
independently. If a native binary library is required to compile the Android
application, a version must be provided for each supported CPU architecture. A
directory layout convention exists for providing a binary for each platform,
with the same library name. This yields an independent final binary (APK) for
each CPU architecture. When running locally, a CPU-specific APK will be
uploaded to the simulator or test device.

To simplify the process of distributing the application, at time of publication,
a single Android App Bundle (AAB) is generated from the multiple CPU-specific
APKs. This AAB contains binaries for all platforms that can be uploaded to an
app store.

When an end-user requests the installation of an app, the app store strips out the
binary that is appropriate for the end-user's device.

### Fat binaries

Apple has taken the approach of "fat" binaries. A fat binary is a single
executable or library artefact that contains code for multiple CPU
architectures.

Fat binaries can be compiled in two ways:

1. **Single pass** Apple has modified their compiler tooling with flags that
   allow the user to specify a single compilation command, and instruct the
   compiler to generate multiple output architectures in the output binary
2. **Multiple pass** After compiling a binary for each platform, Apple provides
   a call named `lipo` to combine multiple single-architecture binaries into a
   single fat binary that contains all platforms.

At runtime, the operating system loads the binary slice for the current CPU
architecture, and the linker loads the appropriate slice from the fat binary of
any dynamic libraries.

On macOS ARM hardware, Apple also provides Rosetta as a support mechanism; if a
user tries to run an binary that doesn't contain an ARM64 slice, but *does*
contain an x86-64 slice, the x86-64 slice will be converted at runtime into an
ARM64 binary. Complications can occur when only *some* of the binary is being
converted (e.g., if the binary being executed is fat, but a dynamic library
isn't).

iOS has an additional complication of requiring support for mutiple *ABIs* in
addition to multiple CPU archiectures. The ABI for the iOS simulator and
physical iOS devices are different; however, ARM64 is a supported CPU
architecture for both. As a result, it is not possible to produce a single fat
library that supports both the iOS simulator and iOS devices. Apple provides an
additional structure - the `XCFramework` - as a wrapper format for packaging
libraries that need to span multiple ABIs. When developing an application for
iOS, a developer will need to install binaries for both the simulator and
physical devices.

## Potential solutions or mitigations

Python currently provides `universal2` wheels to support x86_64 and ARM64 in a
single wheel. This is effectively a "fat wheel" format; the `.dylib` files
contained in the wheel are fat binaries containing both x86_64 and ARM64 slices.

However, "Universal2" is a macOS-specific definition that encompasses the scope
of the specific "Apple Silicon" transition ("Universal" wheels also existed
historically for the PowerPC to Intel transition). Even inside the Apple
ecosystem, iOS, tvOS, and watchOS all have different combinations of supported
CPU architectures.

A more general solution for naming multi-architecture binaries, similar to how a
wheel can declare compatibility with multiple CPython versions (e.g.,
`cp34.cp35.cp36-abi3-manylinux1_x86_64`) may be called for. In such a scheme,
`cp310-abi3-macosx_10_9_universal2` would be equivalent to
`cp310-abi3-macosx_10_9_x86_64.arm64`.

To support Android's multi-architecture approach, it may be necessary to extend
installation tools to allow for installing multiple versions of a wheel in one
installation pass. This can be emulated by making multiple independent calls to
to package installer tools; but that results in independent dependency
resolution, etc.
