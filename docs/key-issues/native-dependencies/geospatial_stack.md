# The Geospatial stack

Python users have a rich set of packages for geospatial data I/O, manipulation,
analytics and visualization available to them. Those include
[xarray](https://xarray.dev/),
[Shapely](https://shapely.readthedocs.io),
[Geopandas](https://geopandas.org),
[Rasterio](https://github.com/rasterio/rasterio),
[Fiona](https://github.com/Toblerity/Fiona),
[GDAL](https://gdal.org/),
[pyproj](https://github.com/pyproj4/pyproj),
[PySAL](https://pysal.org/),
[Folium](https://github.com/python-visualization/folium),
[Geoviews](https://geoviews.org/), and
[descartes](https://pypi.org/project/descartes/).
Users typically use multiple of these packages together. It has always been
difficult to set up a working Python environment for that. Especially when
installing from PyPI.


## Current state


This is the warning displayed in the Geopandas documentation
([v0.12.2 install page](https://geopandas.org/en/v0.12.2/getting_started/install.html))
for installing with `pip`:

!!! warning

    When using pip to install GeoPandas, you need to make sure that all
    dependencies are installed correctly.

    - [`fiona`](https://fiona.readthedocs.io/) provides binary wheels with the
      dependencies included for Mac and Linux, but not for Windows.
      Alternatively, you can install [`pyogrio`](https://pyogrio.readthedocs.io/)
      which does have wheels for Windows.
    - [`pyproj`](https://github.com/pyproj4/pyproj),
      [`rtree`](https://github.com/Toblerity/rtree), and
      [`shapely`](https://shapely.readthedocs.io/)
      provide binary wheels with dependencies included for Mac, Linux, and
      Windows.

    Depending on your platform, you might need to compile and install their
    C dependencies manually. We refer to the individual packages for more
    details on installing those.
    Using conda (see above) avoids the need to compile the dependencies yourself.

The description tells a clear story: there are four dependencies with native
code, *and those then have other native dependencies that may not be included*.
Why weren't those dependencies all vendored? Likely because it was simply too
hard - building for example [GDAL](https://gdal.org/) correctly is notoriously
difficult. Also, while GDAL is a large C/C++ library, it has a Python API and
*is* present on PyPI but does not provide wheels
([its PyPI project description](https://pypi.org/project/GDAL/) recommends
using conda). That brings up a question - should other packages express a
dependency on GDAL, knowing it probably won't build, or try to vendor it
in their own wheels?[^1]

[^1]:
    GDAL seems to be using the "flow of source code to distributors" function
    of PyPI here (see ["purposes of PyPI"](../../meta-topics/purposes_of_pypi.md)).
    Unofficial wheels used to be provided by
    [Christoph Gohlke on his website](https://www.lfd.uci.edu/~gohlke/pythonlibs/#gdal).

We can use conda-forge to look at the full dependency tree for
Geopandas, which shows how many native dependencies this one pure Python
package has:

??? example "Geopandas dependency tree"

    This is the dependency tree when installing only `geopandas` from
    conda-forge (duplicate entries removed from tree):

    ```txt
    $ mamba create -n geo-env geopandas
    $ mamba activate geo-env
    $ mamba repoquery depends geopandas --tree

    geopandas[0.12.2]
      ├─ fiona[1.8.22]
      │  ├─ attrs[22.1.0]
      │  │  └─ python[3.11.0]
      │  │     ├─ bzip2[1.0.8]
      │  │     │  └─ libgcc-ng[12.2.0]
      │  │     │     ├─ _libgcc_mutex[0.1]
      │  │     │     └─ _openmp_mutex[4.5]
      │  │     │        └─ libgomp[12.2.0]
      │  │     ├─ ld_impl_linux-64[2.39]
      │  │     ├─ libffi[3.4.2]
      │  │     ├─ libnsl[2.0.0]
      │  │     ├─ libsqlite[3.40.0]
      │  │     │  └─ libzlib[1.2.13]
      │  │     ├─ libuuid[2.32.1]
      │  │     ├─ ncurses[6.3]
      │  │     ├─ openssl[3.0.7]
      │  │     │  ├─ ca-certificates[2022.12.7]
      │  │     ├─ readline[8.1.2]
      │  │     ├─ tk[8.6.12]
      │  │     ├─ tzdata[2022g]
      │  │     ├─ xz[5.2.6]
      │  │     └─ pip[22.3.1]
      │  │        ├─ setuptools[65.5.1]
      │  │        ├─ wheel[0.38.4]
      │  ├─ click[8.1.3]
      │  ├─ click-plugins[1.1.1]
      │  ├─ cligj[0.7.2]
      │  ├─ gdal[3.6.1]
      │  │  ├─ hdf5[1.12.2]
      │  │  │  ├─ libcurl[7.86.0]
      │  │  │  │  ├─ krb5[1.20.1]
      │  │  │  │  │  ├─ keyutils[1.6.1]
      │  │  │  │  │  ├─ libedit[3.1.20191231]
      │  │  │  │  │  ├─ libstdcxx-ng[12.2.0]
      │  │  │  │  ├─ libnghttp2[1.47.0]
      │  │  │  │  │  ├─ c-ares[1.18.1]
      │  │  │  │  │  ├─ libev[4.33]
      │  │  │  │  ├─ libssh2[1.10.0]
      │  │  │  ├─ libgfortran-ng[12.2.0]
      │  │  │  │  └─ libgfortran5[12.2.0]
      │  │  ├─ libgdal[3.6.1]
      │  │  │  ├─ blosc[1.21.3]
      │  │  │  │  ├─ lz4-c[1.9.3]
      │  │  │  │  ├─ snappy[1.1.9]
      │  │  │  │  └─ zstd[1.5.2]
      │  │  │  ├─ cfitsio[4.2.0]
      │  │  │  ├─ expat[2.5.0]
      │  │  │  ├─ freexl[1.0.6]
      │  │  │  ├─ geos[3.11.1]
      │  │  │  ├─ geotiff[1.7.1]
      │  │  │  │  ├─ jpeg[9e]
      │  │  │  │  ├─ libtiff[4.4.0]
      │  │  │  │  │  ├─ lerc[4.0.0]
      │  │  │  │  │  ├─ libdeflate[1.14]
      │  │  │  │  │  ├─ libwebp-base[1.2.4]
      │  │  │  │  ├─ proj[9.1.0]
      │  │  │  │  │  └─ sqlite[3.40.0]
      │  │  │  │  └─ zlib[1.2.13]
      │  │  │  ├─ giflib[5.2.1]
      │  │  │  ├─ hdf4[4.2.15]
      │  │  │  ├─ icu[70.1]
      │  │  │  ├─ json-c[0.16]
      │  │  │  ├─ kealib[1.5.0]
      │  │  │  ├─ libiconv[1.17]
      │  │  │  ├─ libkml[1.3.0]
      │  │  │  │  ├─ boost-cpp[1.78.0]
      │  │  │  ├─ libnetcdf[4.8.1]
      │  │  │  │  ├─ curl[7.86.0]
      │  │  │  │  ├─ libxml2[2.10.3]
      │  │  │  │  ├─ libzip[1.9.2]
      │  │  │  ├─ libpng[1.6.39]
      │  │  │  ├─ libpq[15.1]
      │  │  │  ├─ libspatialite[5.0.1]
      │  │  │  │  ├─ librttopo[1.1.0]
      │  │  │  ├─ openjpeg[2.5.0]
      │  │  │  ├─ pcre2[10.40]
      │  │  │  ├─ poppler[22.12.0]
      │  │  │  │  ├─ cairo[1.16.0]
      │  │  │  │  │  ├─ fontconfig[2.14.1]
      │  │  │  │  │  │  ├─ freetype[2.12.1]
      │  │  │  │  │  ├─ fonts-conda-ecosystem[1]
      │  │  │  │  │  │  └─ fonts-conda-forge[1]
      │  │  │  │  │  │     ├─ font-ttf-dejavu-sans-mono[2.37]
      │  │  │  │  │  │     ├─ font-ttf-inconsolata[3.000]
      │  │  │  │  │  │     ├─ font-ttf-source-code-pro[2.038]
      │  │  │  │  │  │     └─ font-ttf-ubuntu[0.83]
      │  │  │  │  │  ├─ libglib[2.74.1]
      │  │  │  │  │  │  ├─ gettext[0.21.1]
      │  │  │  │  │  ├─ libxcb[1.13]
      │  │  │  │  │  │  ├─ pthread-stubs[0.4]
      │  │  │  │  │  │  ├─ xorg-libxau[1.0.9]
      │  │  │  │  │  │  └─ xorg-libxdmcp[1.1.3]
      │  │  │  │  │  ├─ pixman[0.40.0]
      │  │  │  │  │  ├─ xorg-libice[1.0.10]
      │  │  │  │  │  ├─ xorg-libsm[1.2.3]
      │  │  │  │  │  ├─ xorg-libx11[1.7.2]
      │  │  │  │  │  │  ├─ xorg-kbproto[1.0.7]
      │  │  │  │  │  │  └─ xorg-xproto[7.0.31]
      │  │  │  │  │  ├─ xorg-libxext[1.3.4]
      │  │  │  │  │  │  └─ xorg-xextproto[7.3.0]
      │  │  │  │  │  ├─ xorg-libxrender[0.9.10]
      │  │  │  │  │  │  └─ xorg-renderproto[0.11.1]
      │  │  │  │  ├─ lcms2[2.14]
      │  │  │  │  ├─ nss[3.82]
      │  │  │  │  │  └─ nspr[4.35]
      │  │  │  │  └─ poppler-data[0.4.11]
      │  │  │  ├─ postgresql[15.1]
      │  │  │  │  ├─ tzcode[2022g]
      │  │  │  ├─ qhull[2020.2]
      │  │  │  ├─ tiledb[2.13.0]
      │  │  │  ├─ xerces-c[3.2.4]
      │  │  ├─ numpy[1.23.5]
      │  │  │  ├─ libblas[3.9.0]
      │  │  │  │  └─ libopenblas[0.3.21]
      │  │  │  ├─ libcblas[3.9.0]
      │  │  │  ├─ liblapack[3.9.0]
      │  │  │  └─ python_abi[3.11]
      │  ├─ munch[2.5.0]
      │  │  └─ six[1.16.0]
      │  ├─ shapely[2.0.0]
      ├─ folium[0.14.0]
      │  ├─ branca[0.6.0]
      │  │  ├─ jinja2[3.1.2]
      │  │  │  ├─ markupsafe[2.1.1]
      │  └─ requests[2.28.1]
      │     ├─ certifi[2022.12.7]
      │     ├─ charset-normalizer[2.1.1]
      │     ├─ idna[3.4]
      │     └─ urllib3[1.26.13]
      │        ├─ brotlipy[0.7.0]
      │        │  ├─ cffi[1.15.1]
      │        │  │  ├─ pycparser[2.21]
      │        ├─ cryptography[38.0.4]
      │        ├─ pyopenssl[22.1.0]
      │        ├─ pysocks[1.7.1]
      ├─ geopandas-base[0.12.2]
      │  ├─ packaging[22.0]
      │  ├─ pandas[1.5.2]
      │  │  ├─ python-dateutil[2.8.2]
      │  │  └─ pytz[2022.6]
      │  ├─ pyproj[3.4.1]
      ├─ mapclassify[2.4.3]
      │  ├─ networkx[2.8.8]
      │  ├─ scikit-learn[1.2.0]
      │  │  ├─ joblib[1.2.0]
      │  │  ├─ scipy[1.9.3]
      │  │  └─ threadpoolctl[3.1.0]
      ├─ matplotlib-base[3.6.2]
      │  ├─ contourpy[1.0.6]
      │  ├─ cycler[0.11.0]
      │  ├─ fonttools[4.38.0]
      │  │  ├─ brotli[1.0.9]
      │  │  │  ├─ brotli-bin[1.0.9]
      │  │  │  │  ├─ libbrotlidec[1.0.9]
      │  │  │  │  │  ├─ libbrotlicommon[1.0.9]
      │  │  │  │  ├─ libbrotlienc[1.0.9]
      │  │  ├─ munkres[1.1.4]
      │  ├─ kiwisolver[1.4.4]
      │  ├─ pillow[9.2.0]
      │  ├─ pyparsing[3.0.9]
      ├─ rtree[1.0.1]
      │  ├─ libspatialindex[1.9.3]
      └─ xyzservices[2022.9.0]
    ```

Building all those libraries in a consistent fashion as a set of wheels is
effectively infeasible, due to some of the "meta topics" around packaging on PyPI
([the author-led social model](../../meta-topics/pypi_social_model.md),
[lack of a build farm](../../meta-topics/no_build_farm.md)). And that's just the
tip of the iceberg. Geopandas is positioned relatively low in the stack:
[libraries.io tells us](https://libraries.io/pypi/geopandas/dependents) that it
has ~660 dependent packages on PyPI, quite a few of which are quite popular
themselves.


## Problems



## History

TODO


## Relevant resources

TODO


## Potential solutions or mitigations

The current state - the geospatial stack being almost uninstallable from PyPI,
and projects and users mostly using conda-forge - is hard to improve upon
unless some of the meta issues with PyPI that are the root causes of that
current state get addressed.
