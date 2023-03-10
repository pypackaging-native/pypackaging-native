---
description: pypackaging-native is a collection of content about key Python packaging topics and issues for projects using native code - with a focus in particular scientific, data science and ML/AI projects in the PyData ecosystem.
---

## Introduction

Packaging is an important and time-consuming part of authoring and maintaining
Python packages. This is particularly true for projects that are not pure
Python but contain code that needs to be compiled, and have to deal with
distributing compiled extensions and with build dependencies. Many projects in
the PyData ecosystem - which includes scientific computing, data science and
ML/AI projects - fall into that category. This site aims to provide an overview
of the most important Python packaging issues for such projects, with in-depth
explanations and references.

The content on this site is meant to provide insights and good reference
material. This will hopefully provide common ground when discussing potential
solutions for those problems or design changes in Python packaging as a whole
or in individual packaging tools.

The content is divided into "meta topics" and "key issues". Meta topics are
mainly descriptions of aspects of Python packaging that are more or less
inherent to the whole design of it, and consequences and limitations that
follow from that. Key issues are more specific pain points felt by projects
with native code. Key issues may also be more tractable to devise solutions or
workarounds for.

??? question "How are these topics chosen and ranked?"

    The initial list of topics was constructed by soliciting input from ~25
    people, who together are a representative subset of stakeholders:

    - maintainers of widely used PyData projects like NumPy, scikit-learn, Apache
      Arrow, CuPy, Matplotlib, SciPy, H5py, Jupyter Hub and Spyder,
    - maintainers of package repositories, package managers and build systems
      (Pip, PyPI, Conda, Conda-forge, Spack, Nix, `pypa/build`, Meson, and
      `numpy.distutils`),
    - engineers from hardware vendors like Intel and NVIDIA,
    - engineers responsible for deploying software for HPC users,
    - educators and organisers of user groups (WiMLDS, SciPy Lectures, Data Umbrella),

    Adding new topics and making changes to existing content on this site
    happens through community input
    [on GitHub](https://github.com/pypackaging-native/pypackaging-native).


## Meta topics

- [Build & package management concepts and terminology](meta-topics/build_steps_conceptual.md)
- [The multiple purposes of PyPI](meta-topics/purposes_of_pypi.md)
- [PyPI's author-led social model and its limitations](meta-topics/pypi_social_model.md)
- [Lack of a build farm for PyPI](meta-topics/no_build_farm.md)
- [Expectations that projects provide ever more wheels](meta-topics/user_expectations_wheels.md)

## Key issues

1. [Native dependencies](key-issues/native-dependencies/index.md)
   *This is, by some distance, the most important issue. Several types of
   native dependencies are discussed in detail:*

       - [BLAS, LAPACK and OpenMP](key-issues/native-dependencies/blas_openmp.md)
       - [The Geospatial stack](key-issues/native-dependencies/geospatial_stack.md)
       - [Complex C++ dependencies](key-issues/native-dependencies/cpp_deps.md)

2. [Depending on packages for which an ABI matters](key-issues/abi.md)
3. [Packaging projects with GPU code](key-issues/gpus.md)
4. [Metadata handling on PyPI](key-issues/pypi_metadata_handling.md)
5. [Distributing a package containing SIMD code](key-issues/simd_support.md)
6. [Unsuspecting users getting failing from source builds](key-issues/unexpected_fromsource_builds.md)
7. [Cross compilation](key-issues/cross_compilation.md)


## Contributing

All contributions are very welcome and appreciated! Ways to contribute include:

- Improving existing content on the website: extending or clarifying
  descriptions, adding relevant references, diagrams, etc.
- Providing feedback on existing content
- Proposing new topics for inclusion on the website, and writing the content for them
- ... and anything else you consider useful!

The content for this website is
[maintained on GitHub](https://github.com/pypackaging-native/pypackaging-native).


## Acknowledgements

- Initial development of this website was sponsored by [Intel](https://www.intel.com),
- Initial development effort was led by [Quansight Labs](https://labs.quansight.org/),
