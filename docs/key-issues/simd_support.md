# Distributing a package containing SIMD code

[Single Instruction, Multiple Data](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data)
(SIMD) instructions are instructions that are CPU-specific, and can yield
significant performance gains compared to regular, portable C/C++ code. Each
popular modern CPU architecture has its own SIMD instruction sets.

Using SIMD instructions in a Python package is quite difficult, because there
is no way to specify, in either metadata or wheel tags, what CPU features are
needed on the target machine in order to use a given wheel.

??? question "What does code containing SIMD instructions look like?"

    This code fragment shows how to use a single SSE2 instruction on an x86-64 CPU.
    It defines a `mul` function which multiplies two double precision floating
    point vectors:
    ```C
    #include <immintrin.h>
    __m128d mul(__m128d a, __m128d b)
    {
        return _mm_mul_pd(a, b);
    }
    ```
    If the CPU supports the instruction and the code gets compiled with the
    needed compiler flag (`-msse2`), the `mul` function will work and will be
    faster than using regular multiplication in C/C++.

    As a more real-world example, here is a code fragment from a `sin` function
    for 32-bit float data from NumPy code:
    ```C
    #if NPY_SIMD_F32 && NPY_SIMD_FMA3
        if (is_mem_overlap(src, steps[0], dst, steps[1], len) ||
            !npyv_loadable_stride_f32(ssrc) || !npyv_storable_stride_f32(sdst)
        ) {
            for (; len > 0; --len, src += ssrc, dst += sdst) {
                simd_sincos_f32(src, 1, dst, 1, 1, SIMD_COMPUTE_SIN);
            }
        } else {
            simd_sincos_f32(src, ssrc, dst, sdst, len, SIMD_COMPUTE_SIN);
        }
    #else
        for (; len > 0; --len, src += ssrc, dst += sdst) {
            const float src0 = *src;
            *dst = npy_sinf(src0);
        }
    #endif
    ```

??? question "How important is use of SIMD code?"

    Code with SIMD instructions is typically a lot more difficult to read and
    maintain than regular C or C++ code. The speedups can be large however,
    so the implementation effort and the maintenance burden may be worth it.
    For basic and heavily used functionality like element-wise math functions
    (`abs`, `sqrt`, `multiply`, etc.), typical gains are in the `1.x - 10` range,
    and sometimes even `>10`). Here are a few benchmark results for:

    - OpenCV color conversion functionality, ~25x faster on ARM CPUs with NEON:
      [opencv#19883](https://github.com/opencv/opencv/pull/19883)
    - NumPy's `absolute`, `reciprocal`, `sqrt`, `square` functions, for
      SSE/AVX2 (x86-64), NEON (aarch64/arm64), and VSX (ppc64le):
      [numpy#16247](https://github.com/numpy/numpy/pull/16247)
    - PyTorch `softmax`, `min` and `max` 3x-4x faster for `bfloat16` with
      AVX2/AVX512 on x86-64:
      [pytorch#55202](https://github.com/pytorch/pytorch/pull/55202#issuecomment-812284545),
      and up to 2x-10x with `uint8` for `+`, `>>`, `min`:
      [pytorch#89284](https://github.com/pytorch/pytorch/pull/89284#issuecomment-1323640288)
    - Using AVX2 instead of SSE in SciPy's 2-D Fourier transforms:
      [scipy#16984](https://github.com/scipy/scipy/issues/16984)

    It is safe to say that performance gains that large, for single-threaded
    execution in libraries that are so widely used, are extremely important.


## Current state

As of December 2022, there is no support on PyPI, in the wheel spec, or in any
widely used packaging tool for binaries containing SIMD instructions. Nor a
plan to implement such support. The only relevant metadata is the "platform
compatibility tag" in a wheel name, first defined in [PEP 425](https://peps.python.org/pep-0425/)
and now maintained under [PyPA specifications in the Python Packaging User
Guide](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/#platform-compatibility-tags).
A platform tag defines a CPU family, for example `x86_64` for 64-bit x86 CPUs
and `aarch64` for 64-bit ARM CPUs.

Projects that want to distribute wheels containing SIMD instructions have
effectively three choices:

1. Make a single choice of SIMD instructions to include.
2. Build extension modules with multiple SIMD flavors inside, detect CPU
   capabilities at runtime, and then dynamically choose the optimal binary
   code.
3. Create separate packages on PyPI with a different package name but the same
   import name, and containing wheels with newer instructions. Then let users
   manually install those alternative packages.

Choice (1) implicitly defines what CPUs are supported by their package. Given
that unsupported instructions result in very obscure errors, this means
targeting SIMD instruction sets that are at least 10 years old (sometimes more).
Choice (2) results in improved performance, because newer SIMD instructions can
be used. However, this comes at the cost of a large amount of code complexity
and larger wheel sizes.

In practice, only the largest and most widely used projects are able to make
choice (2). And they indeed do so - TensorFlow, PyTorch, OpenCV, NumPy, and
MXNet all have their own machinery and methods to work with SIMD instructions.

There are not many examples of choice (3). The ones that do exist, e.g.
[Pillow-SIMD](https://github.com/uploadcare/pillow-simd) and
[Intel(R) Extension for scikit-learn](https://github.com/intel/scikit-learn-intelex),
tend to be forks by a third party rather than packages created by the original
development team.

Distributing binaries with SIMD instructions is not something many other packaging
systems have an answer for. The exception is Spack, which has builtin capabilities
through [`archspec`](https://github.com/archspec/archspec) for installing
optimized binaries. This will even be surfaced in its resolver; individual
package entries will contain a tag like `-skylake_avx512` (microarchitecture +
highest supported instruction set). The
[`archspec` paper](https://tgamblin.github.io/pubs/archspec-canopie-hpc-2020.pdf)
is worth reading for a thorough discussion of the design aspects of integrating
support for SIMD instructions, and dealing with CPU compatibility in a
packaging system in a more granular fashion.


## Problems

Writing SIMD instructions is a specialized skill, however it can be effective
to do so in only a few performance hotspots of the code. So it is often
worthwhile, if it weren't for the problems around distributing wheels on PyPI.
To illustrate how prohibitively expensive in terms of developer time the
dynamic dispatch solution is: NumPy only gained support for it in 2020, and
SciPy still does not have it (it chooses SSE3 instructions, first released in
2005, as the most recent instructions that are allowed to be used on x86).

The "distribute separate wheels under a different package name (choice 3 above)
is so user-unfriendly, and also fairly labor-intensive, that we cannot think of
a single open source project that does this on PyPI.

The "choose a baseline and compile only for that" (choice 1 above) is the
easiest choice that still allows using some SIMD instructions - and the
difference between some (e.g. up to SSE3) and none at all can still be very
large in terms of performance gain. However, this still leaves some users with
old or nonstandard CPUs out in the cold, and it forces package authors to come
up with a method for choosing that maximum feature set. The rule of thumb that
NumPy and SciPy came up with is: if the number of users with incompatible CPUs
stays below 0.5% (as determined by some publicly available data from browser
and gaming vendors), then it's okay to use a particular feature. This is not
ideal, but tends to lead to few complaints in practice.


## History

Distributing packages containing SIMD code on PyPI came up a number of times on
the distutils-sig mailing list, as well as more recently on Discourse:

- [Handling the binary dependency management problem](https://mail.python.org/pipermail/distutils-sig/2013-December/023238.html)
  thread on distutils-sig (2013)
- [Warning about potential problems for wheels](https://mail.python.org/archives/list/distutils-sig@python.org/thread/4TXZXRVYKFTPJSWSSBOU4AWODM7YHF6N/#GR33CY6HJF5V36XG3R2IGWAB4LG4STFG)
  thread on distutils-sig (2015)
- [Status update on the NumPy & SciPy vs SSE problem?](https://mail.python.org/archives/list/distutils-sig@python.org/thread/EFXFPZYKDPR74RJ7D7EKKRSILZRBZEDT/#TAOR6F2R2DVX5KDORMCGP4LAUC5HWT24)
  thread on distutils-sig (2016)
- [Archspec: a library for labeling optimized binaries](https://discuss.python.org/t/archspec-a-library-for-labeling-optimized-binaries/3149)
  on a Packaging thread on Discourse (2020)
- [Idea: selector packages](https://discuss.python.org/t/idea-selector-packages/4463)
  on a Packaging thread on Discourse (2020)

Even before wheels existed, NumPy and SciPy were already distributing `.exe`
Windows installers for three SIMD flavors (no SIMD, up to SSE2, and up to SSE3,
see for example [pypi.org/project/numpy/1.5.1/#files](https://pypi.org/project/numpy/1.5.1/#files).


## Relevant resources

Links to key issues, forum discussions, PEPs, blog posts, etc.

- [NEP 38 - Using SIMD optimization instructions for performance](https://numpy.org/neps/nep-0038-SIMD-optimizations.html)
  (see also the "Related Work" section in that NEP for more relevant projects)
- `archspec` - a library for detecting, labeling, and reasoning about microarchitectures:
  [GitHub repo](https://github.com/archspec/archspec), [paper](https://tgamblin.github.io/pubs/archspec-canopie-hpc-2020.pdf)
- `pytorch/cpuinfo` - CPU INFOrmation library: [GitHub repo](https://github.com/pytorch/cpuinfo)
- `xsimd` - C++ wrappers for SIMD intrinsics and parallelized, optimized mathematical functions:
  [GitHub repo](https://github.com/xtensor-stack/xsimd)
- Spack's [docs on support for specific microarchitectures](https://spack.readthedocs.io/en/latest/basic_usage.html#support-for-specific-microarchitectures)


## Potential solutions or mitigations

There are few potential solutions on the Python packaging side that look promising:

- New wheel tags for specific microarchitectures is a blunt instrument, and
  there are too many microarchitectures to consider for this to work well,
- Using a library like `archspec` by packaging tools is very likely too complicated,
- The [selector packages idea](https://discuss.python.org/t/idea-selector-packages/4463)
  seemed promising at first, but seems to have fallen out of favor now.

The most likely path forward to improve the current situation is to make it easier to
share and reuse infrastructure for CPU feature detection and runtime dispatch. With `archspec`
and `pytorch/cpuinfo` there are two solid libraries available for feature detection.
The NumPy and Meson projects are planning to collaborate to make the "multiple
compilation for different CPU capabilities" part available as a build system
feature. If the runtime dispatch part could be implemented as a standalone,
vendorable component, perhaps it will become easier for other projects to go
this route.
