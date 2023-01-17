# Platforms with multiple CPU architectures

One important subset of ABI concerns is the CPU architecture for which a binary
artefact has been built. Attempting to run a binary on hardware that doesn't
match the CPU architecture (or architecture variant [^1]) for which the binary
was built will generally lead to crashes, even if the ABI being used is
otherwise compatible.

[^1] e.g., the x86-64 architecture has a range of well-known extensions, such as
     SSE, SSE2, SSE3, AVX, AVX2, AVX512, etc.

## Current state

Historically, it could be assumed that an executable or library would be
compiled for a single CPU archicture. On the rare occasion that an operating
system was available for mulitple CPU architectures, it became the
responsibility of the user to find (or compile) a binary that was compiled for
their host CPU architecture.

However, we now see operating system platforms where multiple CPU architectures
are supported:

* In the early days of Windows NT, both x86 and DEC Alpha CPUs were supported
* Windows 10 supports x86, x86-64, ARMv7 and ARM64; Windows 11 supports x86-64
  and ARM64.
* Due to its open source nature, Linux tends to support all CPU architectures for
  which someone is interested enough to author & provide support in the kernel,
  see [here](https://en.wikipedia.org/wiki/List_of_Linux-supported_computer_architectures).
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
there may be some overlap with the solutions that can be used to support GPUs in
native binaries.

Three approaches have emerged on operating systmes that have a need to manage
multiple CPU architectures:

### Multiple binaries

The minimal solution is to distribute multiple binaries. This is the approach
that is by Windows and Linux. At time of distribution, an installer or other
downloadable artefact is provided for each supported platform, and it is up to
the user to select and download the correct artefact.

### Archiving

The approach taken by Android is very similar to the multiple binary approach,
with some affordances and tooling to simplify distribution.

By default Android projects use Java/Kotlin, which produces platform independent
code. However, it is possible to use non-Java/Kotlin libraries by using JNI and
the Android NDK (Native Development Kit). If a project contains native code, a
separate compilation pass is performed for each architecture.

If a native binary library is required to compile the Android application, a
version must be provided for each supported CPU architecture. A directory layout
convention exists for providing a binary for each platform, with the same
library name.

The final binary artefact produced for Android distrobution uses this same
directory convention. A "binary" on Android is an APK (Android Application
Package) bundle; this is effectively a ZIP file with known metadata and
structure; internally, there are subfolders for each supported CPU architecture.
This APK is bundled into AAB (Android Application Bundle) format for upload to
an app store; at time of installation, a CPU-specific APK is generated and
provided to the end-user for installation.

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

To support the transition to Apple Silicon/M1 (ARM64), Python has introduced a
`universal2` architecture target. This is effectively a "fat wheel" format; the
`.dylib` files contained in the wheel are fat binaries containing both x86_64
and ARM64 slices.

iOS has an additional complication of requiring support for mutiple *ABIs* in
addition to multiple CPU archiectures. The ABI for the iOS simulator and
physical iOS devices are different; however, ARM64 is a supported CPU
architecture for both. As a result, it is not possible to produce a single fat
library that supports both the iOS simulator and iOS devices. Apple provides an
additional structure - the `XCFramework` - as a wrapper format for packaging
libraries that need to span multiple ABIs. When developing an application for
iOS, a developer will need to install binaries for both the simulator and
physical devices.

## Problems

At present, the Python ecosystem almost exclusively uses the "multiple binary"
solution. This serves the needs of Windows and Linux well, as it matches the
way end-users interact with binaries.

The `universal2` "fat wheel" solution also works well for macOS. The definition
of `universal2` is a hard-coded accomodation for one specific (albeit common)
multi-architecture configuration, and involves a number of specific
accomodations in the Python ecosystem (e.g., a macOS-specific architecture
lookup scheme).

Supporting iOS requires supporting between 2 and 5 architectures (x86_64 and
ARM64 at the minimum), and at least 2 ABIs - the iOS simulator and iOS device
have different (and incompatible) binary ABIs. At runtime, iOS expects to find a
single "fat" binary for the ABI that is in use. iOS effectively requires an
analog of `universal2` covering the 2 ABIs and multiple architectures. However:

1. The Python ecosystem does not provide an extension mechanism that would allow
   platforms to define and utilize multi-architecture build artefacts.

2. The rate of change of CPU architectures in the iOS ecosystem is more rapid
   than that seen on desktop platforms; any potential "universal iOS" target
   would need to be updated or versioned regularly. A single named target would
   also force developers into supporting older devices that they may not want to
   support.

Supporting Android also requires the support of between 2 and 4 architectures
(depending on the range of development and end-user configurations the app needs
to support). Android's archiving-based approach can be mapped onto the "multiple
binary" approach, as it is possible to build a single archive from multiple
individual binaries. However, some coordination is required when installing
multiple binaries. If an independent install pass (e.g., call to `pip`) is used
for each architecture, the dependency resolution process for each platform will
also be independent; if there are any discrepancies in the specific versions
available for each architecture (or any ordering instabilities in the dependency
resolution algorithm), it is possible to end up with different versions on each
platform. Some coordination between per-architecture passes is therefore
required.

## History

[The BeeWare Project](https://beeware.org) provides support for building both
iOS and Android binaries. On both platforms, BeeWare provides a custom package
index that contains pre-compiled binaries
([Android](https://chaquo.com/pypi-7.0/);
[iOS](https://anaconda.org/beeware/repo)). These binaries are produced using a
set of tooling
([Android](https://github.com/chaquo/chaquopy/tree/master/server/pypi);
[iOS](https://github.com/freakboy3742/chaquopy/tree/iOS-support/server/pypi))
that is analogous to the tools used by conda-forge to build binary artefacts.
These tools patch the source and build configurations for the most common Python
binary dependencies; on iOS, these tools also manage the process of merging
single-architecture, single ABI wheels into a fat wheel.

On iOS, BeeWare-supplied iOS binary packages provide a single "iPhone" wheel.
This wheel includes 2 binary libraries (one for the iPhone device ABI, and one
for the iPhone Simulator ABI); the iPhone simulator binary includes x86_64 and
ARM64 slices. This is effectively the "universal-iphone" approach, encoding a
specific combination of ABIs and architectures.

BeeWare's support for Android uses [Chaquopy](https://chaquo.com/chaquopy) as a
base. Chaquopy's binary artefact repository stores a single binary wheel for
each platform; it also contains a wrapper around `pip` to manage the
installation of multiple binaries. When a Python project requests the
installation of a package:

* Pip is run normally for one binary architecture
* The `.dist-info` metadata is used to identify the native packages -  both
  those directly requested by the user, and those installed as indirect
  requirements by pip
* The native packages are separated from the pure-Python packages, and pip is
  then run again for each of the remaining architectures; this time, only those
  specific native packages are installed, pinned to the same versions that pip
  selected for the first architecture.

[Kivy](https://kivy.org) also provides support for iOS and Android as deployment
platforms. However, Kivy doesn't support the use of binary artefacts like wheels
on those platforms; Kivy's support for binary modules is based on the broader Kivy
platform including build support for libraries that may be required.

## Relevant resources

To date, there haven't been extensive public discussions about the support of
iOS or Android binary packages. However, there were discussions around the
adoption of universal2 for macOS:

* [The CPython discussion about universal2
  support](https://discuss.python.org/t/apple-silicon-and-packaging/4516)
* [The addition of universal2 to
  CPython](https://github.com/python/cpython/pull/22855)
* [Support in packaging for
  universal2](https://github.com/pypa/packaging/pull/319), which declares the
  logic around resolving universal2 to specific platforms.

## Potential solutions or mitigations

There are two approaches that could be used to provide a general solution to
this problem, depending on whether the support of multiple architectures is
viewed as a distribution or integration problem.

### Distribution-based solution

The first approach is to treat the problem as a package distribution issue. In
this approach, artefacts stored in package repositories include all the ABIs and
CPU architectures needed to meaningfully support a given platform. This is the
approach embodied by the `universal2` packaging solution on macOS, and the iOS
solution used by BeeWare.

This approach would require agreement on any new "known" multi-ABI/arch tags, as
well as any resolution schemes that may be needed for those tags.

A more general approach to this problem would be to allow for multi-architecture
and multi-ABI binaries as part of the wheel naming scheme. A wheel can already
declare compatibility with multiple CPython versions (e.g.,
`cp34.cp35.cp36-abi3-manylinux1_x86_64`); it could be possible for a wheel to
declare multiple ABI or architecture inclusions. In such a scheme,
`cp310-abi3-macosx_10_9_universal2` would effectively be equivalent to
`cp310-abi3-macosx_10_9_x86_64.macosx_10_9_arm64`; an iPhone wheel for the same
package might be
`cp310-abi3-iphoneos_12_0_arm64.iphonesimulator_12_0_x86_64.iphonesimulator_12_0_arm64`.

This would allow for more generic logic based on matching name fragments, rather
than specific "known name" targets.

Regardless of whether "known tags" or a generic naming scheme is used, the
distribution-based approach requires modifications to the process of building
packages, and the process of installing packages.

### Integration-based solution

Alternatively, this could be treated as an install-time problem. This is the
approach taken by BeeWare/Chaquopy on Android.

In this approach, package repositories would continue to store
single-architecture, single-ABI artefacts. However, at time of installation, the
installation tool allows for the specification of multiple architectures/ABI
combinations. The installer then downloads a wheel for each architecture/ABI
requested, and performs any post-processing required to merge binaries for
multiple architectures into a single fat binary, or archiving those binary
artefacts in an appropriate location.

This approach is less invasive from the perspective of package repositories and
package build tooling; but would require significant modifications to installer
tooling.
