# Complex C++ dependencies

Projects with a large amount of C++ code and dependencies include
[TensorFlow](https://www.tensorflow.org/),
[PyTorch](https://pytorch.org/),
[Apache Arrow](https://arrow.apache.org/),
[Apache MXNet](https://mxnet.apache.org),
[cuDF](https://github.com/rapidsai/cudf),
[Ray](https://www.ray.io/),
[ROOT](https://root.cern/), and
[Vaex](https://vaex.io).
Many other projects include some C++ components, like:
[scikit-learn](https://github.com/scikit-learn/scikit-learn/),
[SciPy](https://scipy.org/),
[NumPy](https://numpy.org/),
[CuPy](https://cupy.dev/), and
[Awkward Array](https://awkward-array.org/doc/main/).

To get an impression of the number and type of external dependencies that some
of these projects have, outside of their own large code bases, one can look at
the "third party" directories in their repositories:
[pytorch/third_party](https://github.com/pytorch/pytorch/tree/master/third_party),
[tensorflow/third_party](https://github.com/tensorflow/tensorflow/tree/master/third_party),
[mxnet/3rdparty](https://github.com/apache/mxnet/tree/master/3rdparty).

There are usually more build time dependencies that are specified elsewhere - some
on PyPI, some only listed in the documentation. Projects with dependencies like
that tend to not upload any sdist's to PyPI (users will see too many build
failures, see [purposes of PyPI](../../meta-topics/purposes_of_pypi.md) for
context), and have a hard time building wheels because of the requirement that
a wheel must be self-contained - which means a lot of vendoring and potential
issues.


## Current state

Wheels tend to work fine for packages that have some C, C++ or Cython code but
no external dependencies other than Python packages. Once a project has
dependencies on other C++ libraries, it has to build that other library and
vendor it as part of building its own wheel. Tools like `auditwheel`,
`delocate` and `delvewheel` help with the vendoring part, but building
everything in a consistent fashion is (a) a lot of work, and (b) may be very
difficult.

!!! quote "Why are wheels hard - Wes McKinney"

    The Python wheel binary standard was optimized for easy installation
    of packages with C extensions, but where those C extensions are simple
    to build. The best case scenario is that the C code is completely
    self-contained.

    If things are more complicated, things get messy:

    * Third party C libraries
    * Third party C++ libraries
    * Differing C++ ABI versions

    The constraint of wheels is that a package must generally be entirely
    self-contained, including all C/C++ symbols included via static
    linking or by including the shared library bundled in the wheel --
    which style of bundling works best may be different for Linux, macOS,
    and Windows.

Only very recently (Oct '22) was the requirement for wheels to be fully
self-contained loosened a little, allowing package authors to take
responsibility for the quality of their own wheels and avoiding vendoring of
libraries that are very large or had to be vendored into multiple wheels that
they have control over:
[auditwheel#368](https://github.com/pypa/auditwheel/pull/368). That change is
an improvement, but doesn't change the big picture - package authors still
cannot rely on other native dependencies on the system easily, and may have to
maintain their own separate wheels (e.g., if SciPy and NumPy want to rely on a
shared `libopenblas` wheel, they are the ones who have to do all the work for
that).

!!! example "Example: PyArrow"

    PyArrow is the Python API of Apache Arrow. Apache Arrow is a C++ project
    with many C and C++ dependencies. Large and complicated ones. In 2019 the
    need to vendor the likes of OpenSLL, gRPC, and Gandiva (which in turn
    relies on LLVM) made it too hard to build wheels for PyPI at all - because
    all dependencies must  be built from source and vendored into a wheel,
    which is a major endeavour. As of Dec 2022, there are wheels again, without
    OpenSSL and Gandiva in them.
    *TODO: figure out in more detail what changed here*

    To understand the C/C++ dependencies of PyArrow, let's look at the
    dependency tree in conda-forge and at the libraries vendored into a
    manylinux wheel on PyPI:

    ??? note "PyArrow dependency tree (conda-forge)"

        ```bash
        $ # Note: output edited to remove duplicate packages and python/numpy dependencies
        $ mamba repoquery depends pyarrow --tree

        pyarrow[9.0.0]
          ├─ libgcc-ng[12.2.0]
          │  ├─ _openmp_mutex[4.5]
          │  │  ├─ _libgcc_mutex[0.1]
          │  │  └─ libgomp[12.2.0]
          ├─ libstdcxx-ng[12.2.0]
          ├─ numpy[1.23.5]
          ├─ parquet-cpp[1.5.1]
          │  └─ arrow-cpp[9.0.0]
          │     ├─ gflags[2.2.2]
          │     ├─ c-ares[1.18.1]
          │     ├─ libbrotlienc[1.0.9]
          │     │  └─ libbrotlicommon[1.0.9]
          │     ├─ libbrotlidec[1.0.9]
          │     ├─ zstd[1.5.2]
          │     ├─ aws-sdk-cpp[1.8.186]
          │     │  ├─ aws-c-event-stream[0.2.7]
          │     │  │  ├─ aws-c-common[0.6.2]
          │     │  │  ├─ aws-checksums[0.1.11]
          │     │  │  └─ aws-c-io[0.10.5]
          │     │  │     ├─ aws-c-cal[0.5.11]
          │     │  │     └─ s2n[1.0.10]
          │     │  ├─ libcurl[7.86.0]
          │     │  │  ├─ krb5[1.19.3]
          │     │  │  │  ├─ libedit[3.1.20191231]
          │     │  │  │  └─ keyutils[1.6.1]
          │     │  │  ├─ libssh2[1.10.0]
          │     │  │  └─ libnghttp2[1.47.0]
          │     │  │     └─ libev[4.33]
          │     ├─ lz4-c[1.9.3]
          │     ├─ libthrift[0.16.0]
          │     │  └─ libevent[2.1.10]
          │     ├─ libutf8proc[2.8.0]
          │     ├─ snappy[1.1.9]
          │     ├─ re2[2022.06.01]
          │     ├─ glog[0.6.0]
          │     ├─ libabseil[20220623.0]
          │     ├─ libprotobuf[3.21.10]
          │     ├─ orc[1.8.0]
          │     ├─ libgrpc[1.49.1]
          │     │  ├─ zlib[1.2.13]
          │     └─ libgoogle-cloud[2.3.0]
          │        ├─ libcrc32c[1.1.2]
          ├─ python_abi
          ├─ python
          ├─ numpy
          └─ arrow-cpp
        ```

    ??? note "PyArrow vendored libraries (PyPI wheels)"

        ```bash
        $ ls pyarrow/*.so  # pyarrow/ is from an unzipped manylinux wheel
        /home/rgommers/Downloads/pyarrow/_compute.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_csv.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_dataset.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_dataset_orc.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_dataset_parquet.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_exec_plan.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_feather.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_flight.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_fs.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_gcsfs.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_hdfs.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_hdfsio.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_json.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/libarrow_python_flight.so
        /home/rgommers/Downloads/pyarrow/libarrow_python.so
        /home/rgommers/Downloads/pyarrow/lib.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_orc.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_parquet.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_parquet_encryption.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_plasma.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_pyarrow_cpp_tests.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_s3fs.cpython-311-x86_64-linux-gnu.so
        /home/rgommers/Downloads/pyarrow/_substrait.cpython-311-x86_64-linux-gnu.so
        ```

    We see that [Apache Parquet](https://github.com/apache/parquet-format) and
    [Substrait](https://github.com/substrait-io/substrait), two separate
    projects, have to be vendored, and a lot of Apache Arrow C++ components are
    packed into a single wheel. What can't be seen from the wheel contents is
    that some of the extension modules were built with other complex
    dependencies like `protobuf` and `glog` (which therefore also need to be
    built as part of the wheel build process). Such dependencies raise the
    possibility of symbol conflicts when other packages are built with
    different versions of those libraries, or in different ways. This can
    result in hard to debug crashes or "undefined symbol" problems.
    [This blog post by Uwe Korn](https://uwekorn.com/2019/09/15/how-we-build-apache-arrows-manylinux-wheels.html)
    describes some of the issues in detail, including problems installing
    PyArrow side by side with TensorFlow and Turbodbc.


C++ ABI has been an issue for quite a while. Most C++ developers and projects
want to use the new ABI, however due to the old manylinux standard and
depending on `devtoolset` forces the use of the old C++ ABI
(`_GLIBCXX_USE_CXX11_ABI=0`). Projects using modern C++14/17 typically want
to use the new ABI. This is still quite difficult. It's now possible with
`manylinux_2_28`, but requires building duplicate sets of wheels[^1] (also
`manylinux2014` for compatibility with older distros like Ubuntu 18.04 and
CentOS 8, that will still use the old ABI).

As an example: PyTorch added `libtorch` builds with the new ABI to its own
download server in 2019 already
([pytorch#17492](https://github.com/pytorch/pytorch/issues/17492)), however the
issue for matching wheels is still open
([pytorch#51039](https://github.com/pytorch/pytorch/issues/51039)) as of Dec '22.

[^1]:
    See [manylinux#1332](https://github.com/pypa/manylinux/issues/1332)
    and [pytorch#51039](https://github.com/pytorch/pytorch/issues/51039#issuecomment-1183263853)
    for details.


## Problems

- Requirement for a wheel being completely self-contained, forcing vendoring of
  external C++ dependencies. Building external dependencies is a lot of effort,
  and error prone.

    - It also prevents splitting up a wheel into multiple dependent ones. This
      may be desirable because of binary size or maintainability.

- The old C++ ABI still being the default.
- Symbol clashes when different libraries vendor the same external dependency.


## History

Early history TODO.

The RAPIDS projects was forced to drop wheels completely from May 2019 to Oct
2022, because the `manylinux` required a too old C++ version, and made it
impossible to create compliant wheels with the RAPIDS C++14 code base. See
[this blog post for details](https://medium.com/rapids-ai/rapids-0-7-release-drops-pip-packages-47fc966e9472).

Apache Arrow's issues with wheels and the amount of effort they take were laid out
in detail in [this mailing list post by Wes McKinney](https://lists.apache.org/thread/mxvp4mcx01mvox8jckgszyg0h65ddlkn).

[This long discussion on `pypa/packaging-problems#25`](https://github.com/pypa/packaging-problems/issues/25)
touches on many of the key pain points around wheels and projects with complex
C/C++ dependencies (interspersed with some "packaging politics").


## Relevant resources

TODO


## Potential solutions or mitigations

- Better interaction/integration between PyPI/wheels/pip and other package
  managers, where dealing with C++ dependencies is easier.
- ... ?
