# Application Binary Interface (ABI)

The Application Binary Interface (ABI) is a strange, emergent phenomenon.
Neither the C nor C++ standard recognize its existence, however it is ubiquitous
in discussions about the evolution of those standards.

This is broadly because, in a world where everything always gets recompiled from
scratch (essentially the purview of those standards), ABI would be irrelevant.
However, reality has shown that the always-recompile model is not practical in
the vast majority of contexts, and binary artifacts need to be distributed.

As soon as this is attempted, we run into all the many potential problems that
can appear at the interface between source code and the actual physical
execution that is baked into an artifact. For some common terms, it's
recommended to familiarize yourself with [compilation concepts](./compilation_concepts.md)
first.

Note that ABI appears for all distribution of binary artifacts, not just those
compiled from C/C++. The concepts are the same in all cases though, so for
brevity, we restrict ourselves to these languages here.

A key focus for binary packaging systems is _maintaining binary compatibility_.
If the packagers are not successful in maintaining that compability, we then get
an _ABI break_, which can lead to crashes, segfaults, corrupted data etc.
Unsurprisingly, a lot of effort goes into avoiding them.


## An example of an ABI break

Let's assume we are using a function `f` from a C library `libfoo`, and this
function takes an argument of type `long long`.
```C
extern int f (long long value);

int main () {
    f(1);
    return 0;
}
```

If we inspect the assembly
```as
mov     edi, 1
call    f
```
we see that a single register, `edi`, is used to pass the `value` argument to
the function `f`, which makes sense since our registers are 64 bits wide, and
`long long` matches this on unix platforms.

If we replace the first line with `extern int f (__int128_t value);`, the
assembly becomes:
```as
mov     edi, 1
xor     esi, esi
call    f
```
Here we see that we now need two registers, `edi` and `esi`, to correctly pass
`value` into the function `f`.

Imagine then a situation where we did not change our code away from `long long`,
but where the author of `libfoo` (that contains `f`) wants to upgrade to wider
integers because their users are asking for that. If we upgrade the shared
library `libfoo` (that our code is linked against) _without_ recompilation of
our own application, then our code would still run (because the linker would
find a symbol with the right name `f` due to the lack of name mangling).
However, because the artifact we had compiled expects `f` to take `long long`,
it would only set up the `edi` register correctly, but the upgraded symbol for
`f` would now use both `edi` and `esi`.

If we're "lucky", this only leads to a crash, but it might lead to pretty much
arbitrary behavior based on whatever happens to be in `esi`. This can include
anything from wrong results to Heisenbugs based on any other code that might
leave some random values in the `esi` register, all in a way that's generally
extremely hard to debug.

This is but a trivial example, there are innumerable ways for libraries you
rely on to break their ABI, including:

- Changing anything about a function signature
- Changing (almost) anything about templating (C++)
- Adding a data member or virtual functions to a class
- Making something inline that previously wasn’t
- Changing your compiler or any of many relevant compilation flags
- Etc.


## Interaction with shared libraries

What the above means is that, bascially, you can never change anything about a
C symbol that has been distributed to consumers in binary form, especially those
in shared libraries. Since some code effectively must use shared libraries
(not least the C standard library, in the form of glibc/[musl](https://musl.libc.org)
on linux, resp. the UCRT on Windows), this is one if not _the_ reason for the
importance of distributions, because those will ensure that all their packages
are compiled consistently against a given version of libc (and other libraries
like the C++ standard library).

Similarly, this is why distributions only upgrade their glibc across major
releases, because doing so for an LTS release would risk too much breakage,
even though libc takes extreme care about remaining backwards compatible.

## The different levels of ABI breaks

Fundamentally, ABI can break at pretty much any point that's involved in the
computation of our program. For the sake of clarity, let us divide these into
three different levels:

1. ABI breaks in third-party libraries
2. ABI breaks in compiler or due to compiler configuration
3. ABI breaks in the language standard

The further down we go this list, the more impactful an ABI break becomes, so
much so that the latter two happen rarely if ever. When they do happen, they
tend to leave long-lasting traces.

### ABI breaks on language level

The last substantial ABI break in C++ was when the standard committee decided
to standardize an implementation of `std::string` that made illegal a previously
deployed GCC-extension that used copy-on-write behavior. This meant that all
C++ code compiled with GCC against older C++ standards needed to be recompiled.

Due to the timelines involved with LTS distributions like RHEL, this took a long
time to percolate through the ecosystem.

Episodes like these have led to extreme reluctance in the C & C++ committees to
change anything that even remotely touches the ABI, even though there are often
substantial improvements left on the table due to this (famously, `std::regex`
is excruciatingly slow and cannot be fixed without breaking ABI). This leads to
extreme scrutiny (and therefore sluggish pace) in standards development, which
has its own knock-on effects (see section about Abseil below).

This is also the reason why there are no larger integer types (e.g. `int128`)
that are officially supported by the standard, because their introduction would
be an ABI [break](https://thephd.dev/intmax_t-hell-c++-c). Generally, the
contortions that the C/C++ committees put themselves through to avoid breaking
ABI are [spectacular](https://thephd.dev/binary-banshees-digital-demons-abi-c-c++-help-me-god-please)
(and have very explicit opportunity costs)[^1], but ease ABI problems for the levels
above.

[^1]:
    Despite this article being very colorful, it's worth noting that it was
    written by the current editor of the C language, as well as a prolific
    proposal author for both C & C++.

Finally, the unrealized performance gains left on the table due to ABI
stability (and millions of dollar cost of single percentage performance
pessimization) were what led Google to push hard for an ABI break in the C++
committee, and their [defeat](https://cor3ntin.github.io/posts/abi/) at the
C++ meeting in Prague is ultimately what led to them withdrawing their
substantial resources from compiler development (principally clang), and
start their own C++-alike language, [Carbon](https://github.com/carbon-language/carbon-lang).

### ABI breaks on compiler level

For completeness, we need to distinguish that here we are not talking about
ABI breaks in the compiler infrastructure (in many ways, compilers are less
exposed because using them, i.e. recompiling, drastically lessens the exposure
to ABI), but rather of ABI breaks in the artifacts produced by them.

Generally, compiler authors are almost as reluctant to break ABI as the language
committees, because the effects are largely the same. An exception was MSVC,
which for a long time used to change the ABI it generated with every release,
meaning that each Visual Studio release (before VS2015) required recompilation
of all involved binary artifacts. Starting from VS2015 (up to including VS2022
currently), MSVC has not broken ABI anymore, which means it's possible to, for
example, compile with VS2019 against a shared library produced by VS2017.

However, compilers expose a vast majority of flags, some of which have impacts
on the ABI of the produced artifacts. It's therefore essential to have some
degree of homogeneity (resp. control / auditability) about the compilation flags
being used in an ecosystem.

### ABI breaks in third party libraries

This is the most common case; various libraries make different kinds of promises
about the stability of their ABI. Some (certainly those lower in the stack, like
OpenSSL) promise stringent ABI stability (except across major versions), whereas
others might break ABI in every patch release.

Knowing which library versions are compatible how is a pretty involved job, but
services like [abi-laboratory](https://abi-laboratory.pro/tracker/timeline/openssl)
exist to ease this work.

For distributions that focus on using shared libraries, this means they need to
be able to track which packages are dependent on any given library, and then
rebuild all those in a short timespan, in order to roll out a new version of
that library (due to the fact that, absent explicit inline namespacing, there
can only be one version of a shared library in any given environment).

??? note "Contrast with default approach in Python packaging"

    Because the standard packaging in Python (wheels) cannot express a
    dependency on non-Python artifacts, the default approach is to fully copy
    ("vendor") the respective binaries into the wheel, and mangle their symbols
    in such a way that they do not get accidentally picked up from elsewhere.


## Abseil

Google's [Abseil](https://abseil.io/) project fills a particular role, which is
to backport advances in newer C++ standards, and make those facilities to
projects that still need to compile with older standard versions (c.f. the speed
of the standardization process due to ABI concerns above).

One prominent example of such usage is `absl::string_view`, which backports the
C++17 `std::string_view` back to C++11 & C++14 (this feature allows to heavily
cut down on useless copies involving strings, which has a substantial
performance impact). However, these backports are generally not ABI-compatible
with the implementations for later standard versions.

This puts Abseil in the curious position that the C++ standard version used to
compile it has an impact on its ABI. This is because Abseil will, by default,
pick standard facilities when available, and otherwise fall back to its
backports. As an example, `absl::string_view` compiled with C++17 will use the
C++17 `std::string_view` ABI, whereas for C++14 and below, it will have a
different ABI.

Due to the constraint around having only one shared library per environment,
Abseil strongly recommends against distribution of its binary artifacts,
especially in shared builds. In fact, the only mode of operation that is really
considered supported by upstream Abseil is compiling all dependencies with the
same C++ standard version. This is obviously incompatible with servicing a large
ecosystem, where some libraries might still require C++11, and some already
require C++20.

This issue would be _somewhat_ manageable if Abseil types were only ever used
internally in libraries, meaning things could be solved with a certain degree of
care which (static) builds of Abseil are available at build time. However, the
situation is exacerbated drastically by the fact that several projects are using
(or beginning to use) Abseil types in their public API, e.g. [protobuf](https://developers.google.com/protocol-buffers/docs/news/2022-08-03#abseil-dep).

In the worst case, this means a full bifurcation of the necessary builds, though
more realistically, it means that all Abseil consumers more or less need to
agree on a given ABI. Further alternatives [exist](https://github.com/conda-forge/abseil-cpp-feedstock/issues/45)
(e.g. _always_ using the backport types, even if newer C++ standards are used),
but are non-trivial to realize at scale.
