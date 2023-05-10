# Packaging projects with GPU code

Modern [Graphics Processing Units](https://en.wikipedia.org/wiki/Graphics_processing_unit)
(GPUs) can be used, in addition to their original purpose (rendering graphics),
for high-performance numerical computing. They are particularly important for
deep learning, but also widely used for data science and traditional scientific
computing and image processing application.

GPUs from NVIDIA using the [CUDA](https://en.wikipedia.org/wiki/CUDA)
programming language are dominant in deep learning and scientific computing as
of today. With both AMD and Intel releasing GPUs and other programming
languages for them ([ROCm](https://en.wikipedia.org/wiki/ROCm),
[SYCL](https://en.wikipedia.org/wiki/SYCL),
[OpenCL](https://en.wikipedia.org/wiki/OpenCL)), the landscape may become more diverse in the future.
In addition, Google provides
[Tensor Processing Units](https://en.wikipedia.org/wiki/Tensor_Processing_Unit)
access in Google Cloud Platform, and a host of startups are developing custom
accelerator hardware for high-performance computing applications.

Prominent projects which rely on GPUs and are either Python-only or widely used
from Python include [TensorFlow](https://www.tensorflow.org/),
[PyTorch](https://pytorch.org/), [CuPy](https://cupy.dev/),
[JAX](https://github.com/google/jax), [RAPIDS](https://rapids.ai/),
[MXNet](https://mxnet.incubator.apache.org), [XGBoost](https://xgboost.ai/),
[Numba](https://numba.pydata.org/), [OpenCV](https://opencv.org/),
[Horovod](https://horovod.ai/) and [PyMC](https://www.pymc.io).

Packaging such projects for PyPI has been, and still is, quite challenging.


## Current state

As of May 2023, PyPI and Python packaging tools are completely unaware of
GPUs, and of CUDA. There is no way to mark a package as needing a GPU in sdist
or wheel metadata, or as containing GPU-specific code (CUDA or otherwise). A
GPU is hardware that may or may not be present in a machine that a Python
package is being installed on - `pip` and other installers are unaware of this.
If wheels contain CUDA code, they require the CUDA Toolkit (a specific version
of it at that) to be installed. Again, installers do not know this and there is
no way to express this dependency. The same will be true for ROCm and other
types of GPU hardware and languages.

NVIDIA has made steps towards better support for CUDA on PyPI. Various
library components of the CTK have been packaged as wheels and are now
distributed on PyPI, such as
[nvidia-cublas-cu11](https://pypi.org/project/nvidia-cublas-cu11/). Python
wrappers around CUDA libraries have been consolidated into CUDA Python
([website](https://developer.nvidia.com/cuda-python), 
[PyPI package](https://pypi.org/project/cuda-python)), but this package assumes
that the CUDA runtime and driver are already installed since it only provides
Python bindings to the API. Many other projects remain hosted on
[NVIDIA's Private PyPI Index](https://pypi.org/project/nvidia-pyindex/), which
also includes rebuilds of TensorFlow and other packages.

A single CUDA version supports a reasonable range of GPU architectures. New
CUDA versions get released regularly, and - because they come with increased
performance or new functionality - it may be necessary or desirable to build
new wheels for that CUDA version. If only the supported CUDA version is
different between two wheels, the wheel tags and filename will be identical.
Hence it is not possible to upload more than one of those wheels under the same
package name.

Historically, this required projects to produce packages specific CUDA minor
versions. Projects would either support only one CUDA version on PyPI, or create
different packages. PyTorch and TensorFlow do the former, with TensorFlow
supporting only a single CUDA version, and PyTorch providing more wheels for
other CUDA versions and a CPU-only version in a separate wheelhouse (see
[pytorch.org/get-started](https://pytorch.org/get-started/locally/)).
CuPy provides a number of packages: [`cupy`](https://pypi.org/project/cupy/),
[`cupy-cuda102`](https://pypi.org/project/cupy-cuda102/),
[`cupy-cuda110`](https://pypi.org/project/cupy-cuda1110/),
[`cupy-cuda111`](https://pypi.org/project/cupy-cuda111/),
[`cupy-rocm-4-3`](https://pypi.org/project/cupy-rocm-4-3/),
[`cupy-rocm-5-0`](https://pypi.org/project/cupy-rocm-5-0/).
As of CUDA 11, CUDA promises [minor version
compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/index.html#minor-version-compatibility),
which allows building packages compatibility across an entire CUDA major
version. CuPy now leverages this to produce wheels like 
[`cupy-cuda11x`](https://pypi.org/project/cupy-cuda11x/) and
[`cupy-cuda12x`](https://pypi.org/project/cupy-cuda12x/) that work for any CUDA
11.x or CUDA 12.x version, respectively, that a user has installed.

GPU packages tend to result in very large wheels. This is in large part because
compiled GPU libraries must support a number of architectures, leading to large
binary sizes. These effects are compounded by the requirements imposed by the
manylinux standard for Linux wheels, which results in many large libraries being
bundled into a single wheel. This is true in particular for deep learning
packages because they link in [cuDNN](https://developer.nvidia.com/cudnn). For
example, recent `manylinux2014` wheels for TensorFlow are 588 MB
([2.11.0 files](https://pypi.org/project/tensorflow/2.11.0/#files)), and for
PyTorch those are 890 MB ([1.13.0
files](https://pypi.org/project/torch/1.13.0/#files)). The problems around
and causes of GPU wheel sizes were discussed in depth in
[this Packaging thread on Discourse](https://discuss.python.org/t/what-to-do-about-gpus-and-the-built-distributions-that-support-them/7125).

So far we have only discussed individual projects containing GPU code. Those
projects are the most fundamental libraries in larger stacks of packages
(perhaps even whole ecosystems). Hence, other projects will want to declare a
dependency on them. This is currently quite difficult, because of the implicit
coupling through a shared CUDA version. If a project like PyTorch releases a
new version and bumps the default CUDA version used in the `torch` wheels, then
any downstream package which also contains CUDA code will break unless it has
an exact `==` pin on the older `torch` version, and then releases a new version
of its own for the new CUDA version. Such synchronized releases are hard to do.
If there were a way to declare a dependency on CUDA version (e.g., through a
metapackage on PyPI), that strong coupling between packages would not be
necessary.


Other package managers typically do have support for CUDA:

- Conda: provides all CUDA versions through a [`cudatoolkit` conda-forge package](https://anaconda.org/conda-forge/cudatoolkit)
  and a virtual [`__cuda` package](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-virtual.html#managing-virtual-packages),
- Spack: supports building with or without CUDA, and allows specifying
  supported GPU architectures:
  [docs](https://spack.readthedocs.io/en/latest/build_systems/cudapackage.html#cuda).
  CUDA itself can be specified as externally provided, and is recommended to be
  installed directly from NVIDIA: [docs](https://spack.readthedocs.io/en/latest/containers.html#cuda).
  ROCm is supported [in a similar fashion](https://spack.readthedocs.io/en/latest/build_systems/rocmpackage.html),
- Ubuntu: provides one CUDA version per Ubuntu release: [`nvidia-cuda-toolkit` package](https://packages.ubuntu.com/jammy/nvidia-cuda-toolkit),
- Arch Linux: provides one CUDA version: [`cuda` package](https://archlinux.org/packages/community/x86_64/cuda/).

Those package managers typically also provide CUDA-related development tools,
and build all the most popular deep learning and numerical computing packages
for the CUDA version they ship.


## Problems

The problems around GPU packages include:

User-friendliness:

- Installs depend on a specific CUDA or ROCm version, and `pip` does
  not know about this. Hence installs may succeed, followed by errors at
  runtime,
- CUDA or ROCm must be installed through another package manager or a direct
  download from the vendor. And the other package manager upgrading CUDA or
  ROCm may silently break the installed Python package,
- Wheels may have to come from a separate wheelhouse, requiring install commands like
  `python -m pip install torch --extra-index-url https://download.pytorch.org/whl/cu116`
  which are easy to get wrong,
- The very large download sizes are problematic for users on slow network
  connections or plans with a maximum amount of bandwidth usage for a given
  month (`pip` potentially downloading multiple wheels because of backtracking
  in the resolver is extra painful here).

Maintainer effort:

- Keeping wheel sizes below either the 1 GB hard limit or the current PyPI file
  size or total project size limits can be a lot of work (or even impossible),
- Hosting your own wheelhouse to support multiple CUDA or ROCm versions is a
  lot of work,
- Depending on another GPU package is difficult, and likely requires a `==` pin,
- A dependency on CUDA, ROCm, or a specific version of them cannot be
  expressed in metadata, hence maintaining build environments is more
  error-prone than it has to be.

For PyPI itself:

- The large amount of space and bandwidth consumed by GPU packages.
  [pypi.org/stats](https://pypi.org/stats/) shows under "top projects by total
  package size" that many of the largest package are GPU ones, and that
  together they consume a significant fraction (estimated at ~20% for the ones
  listed in the top 100) of the total size for all of PyPI.

## History

Support for GPUs and CUDA has been discussed on and off on distutils-sig and the Packaging Discourse:

- [Environment markers for GPU/CUDA availability](https://mail.python.org/archives/list/distutils-sig@python.org/thread/LXLF4YSC4WUZOYRX65DW7CESIX7UUBK5/#LXLF4YSC4WUZOYRX65DW7CESIX7UUBK5)
  thread on distutils-sig (2018),
- [The next manylinux specification](https://discuss.python.org/t/the-next-manylinux-specification/1043/42?u=rgommers)
  thread on Discourse (2019), with a specific comment about presence/absence of
  GPU hardware and CUDA libraries being out of scope,
- [What to do about GPUs? (and the built distributions that support them)](https://discuss.python.org/t/what-to-do-about-gpus-and-the-built-distributions-that-support-them/7125)
  on a Packaging thread on Discourse (2021),

None of the suggested ideas in those threads gained traction, mostly due to a
combination of the complexity of the problem, difficulty of implementing
support in packaging tools, and lack of people to work on a solution.


## Relevant resources

TODO


## Potential solutions or mitigations

Potential solutions on the PyPI side include:

- add specific wheel tags or metadata for the most popular libraries,
- make an environment marker or selector package approach work,
- improve interoperability with other package managers, in order to be able to
  declare a dependency on a CUDA or ROCm version as externally provided,
