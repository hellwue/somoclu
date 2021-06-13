# Somoclu
[![Ubuntu2004](https://github.com/peterwittek/somoclu/actions/workflows/u2004.yml/badge.svg)](https://github.com/peterwittek/somoclu/actions/workflows/u2004.yml)

Somoclu is a massively parallel implementation of self-organizing maps. It exploits multicore CPUs, it is able to rely on MPI for distributing the workload in a cluster, and it can be accelerated by CUDA. A sparse kernel is also included, which is useful for training maps on vector spaces generated in text mining processes.

Key features:

* Fast execution by parallelization: OpenMP, MPI, and CUDA are supported.
* Multi-platform: Linux, macOS, and Windows are supported.
* Planar and toroid maps.
* Rectangular and hexagonal grids.
* Gaussian and bubble neighborhood functions.
* Both dense and sparse input data are supported.
* Large maps of several hundred thousand neurons are feasible.
* Integration with [Databionic ESOM Tools](http://databionic-esom.sourceforge.net/).
* [Python](https://somoclu.readthedocs.io/), [R](https://cran.r-project.org/web/packages/Rsomoclu/), [Julia](https://github.com/peterwittek/Somoclu.jl), and [MATLAB](https://github.com/peterwittek/somoclu/tree/master/src/MATLAB) interfaces for the dense CPU and GPU kernels.

For more information, refer to the manuscript about the library [1].

# Usage

## Basic Command Line Use

Somoclu takes a plain text input file -- either dense or sparse data. Example files are included.

    $ [mpirun -np NPROC] somoclu [OPTIONs] INPUT_FILE OUTPUT_PREFIX

Arguments:

    -c FILENAME              Specify an initial codebook for the map.
    -d NUMBER                Coefficient in the Gaussian neighborhood function
                             exp(-||x-y||^2/(2*(coeff*radius)^2)) (default: 0.5)
    -e NUMBER                Maximum number of epochs
    -g TYPE                  Grid type: square or hexagonal (default: square)
    -h, --help               This help text
    -k NUMBER                Kernel type
                                0: Dense CPU
                                1: Dense GPU
                                2: Sparse CPU
    -l NUMBER                Starting learning rate (default: 0.1)
    -L NUMBER                Finishing learning rate (default: 0.01)
    -m TYPE                  Map type: planar or toroid (default: planar)
    -n FUNCTION              Neighborhood function (bubble or gaussian, default: gaussian)
    -p NUMBER                Compact support for Gaussian neighborhood
                             (0: false, 1: true, default: 0)
    -r NUMBER                Start radius (default: half of the map in direction min(x,y))
    -R NUMBER                End radius (default: 1)
    -s NUMBER                Save interim files (default: 0):
                                0: Do not save interim files
                                1: Save U-matrix only
                                2: Also save codebook and best matching
    -t STRATEGY              Radius cooling strategy: linear or exponential (default: linear)
    -T STRATEGY              Learning rate cooling strategy: linear or exponential (default: linear)
    -v NUMBER                Verbosity level, 0-2 (default: 0)
    -x, --columns NUMBER     Number of columns in map (size of SOM in direction x)
    -y, --rows    NUMBER     Number of rows in map (size of SOM in direction y)

Examples:

    $ somoclu data/rgbs.txt data/rgbs
    $ mpirun -np 4 somoclu -k 0 --rows 20 --columns 20 data/rgbs.txt data/rgbs

With random initialization, the initial codebook will be filled with random numbers ranging from 0 to 1. Either supply your own initial codebook or normalize your data to fall in this range.

If the range of the values of the features includes negative numbers, the codebook will eventually adjust. It is, however, not advised to have negative values, especially if the codebook is initialized from 0 to 1. This comes from the batch training nature of the parallel implementation. The batch update rule will change the codebook values with weighted averages of the data points, and with negative values, the updates can cancel out.

The maps generated by the GPU and the CPU kernels are likely to be different. For computational efficiency, Somoclu uses single-precision floats. This occasionally results in identical distances between a data instance and the neurons. The CPU version will pick the best matching unit with the lowest coordinate values. Such sequentiality cannot be guaranteed in the reduction kernel of the GPU variant. This is not a bug, but it is better to be aware of it.

## Efficient Parallel and Distributed Execution

The CPU kernels use OpenMP to load multicore processors. On a single node, this is more efficient than launching tasks with MPI to match the number of cores. The MPI tasks replicated the codebook, which is especially inefficient for large maps.

For instance, given a single node with eight cores, the following execution will use 1/8th of the memory, and will run 10-20% faster:

    $ somoclu -x 200 -y 200 data/rgbs.txt data/rgbs

Or, equivalently:

    $ OMP_NUM_THREADS=8 somoclu -x 200 -y 200 data/rgbs.txt data/rgbs

Avoid the following on a single node:

    $ OMP_NUM_THREADS=1 mpirun -np 8 somoclu -x 200 -y 200 data/rgbs.txt data/rgbs

The same caveats apply for the sparse CPU kernel.

## Visualisation

The primary purpose of generating a map is visualisation. Apart from the Python interface, Somoclu does not come with its own functions for visualisation, since there are numerous generic tools that are capable of plotting high-quality figures. The R version integrates with [kohonen](https://cran.r-project.org/package=kohonen) and the MATLAB version with [somtoolbox](www.cis.hut.fi/somtoolbox/).

The output formats U-matrix and the codebook of the command-line version are compatible with [Databionic ESOM Tools](http://databionic-esom.sourceforge.net/) for more advanced visualisation.

# Input File Formats

One sparse and two dense data formats are supported. All of them are plain text files. The entries can be separated by any white-space character. One row represents one data instance across all formats. Comment lines starting with a hash mark are ignored.

The sparse format follows the [libsvm](http://www.csie.ntu.edu.tw/~cjlin/libsvm/) guidelines. The first feature is zero-indexed. For instance, the vector [ 1.2 0 0 3.4] is represented as the following line in the file:
0:1.2 3:3.4. The file is parsed twice: once to get the number of instances and features, and the second time to read the data in the individual threads.

The basic dense format includes the coordinates of the data vectors, separated by a white-space. Just like the sparse format, this file is parsed twice to get the basic dimensions right.

The .lrn file of [Databionic ESOM Tools](http://databionic-esom.sourceforge.net/) is also accepted and it is parsed only once. The format is described as follows:

% n

% m

% s1 s2 .. sm

% var_name1 var_name2 .. var_namem

x11 x12 .. x1m

x21 x22 .. x2m

. . . .

. . . .

xn1 xn2 .. xnm

Here n is the number of rows in the file, that is, the number of data instances. Parameter m defines the number of columns in the file. The next row defines the column mask: the value 1 for a column means the column should be used in the training. Note that the first column in this format is always a unique key, so this should have the value 9 in the column mask. The row with the variable names is ignore by Somoclu. The elements of the matrix follow -- from here, the file is identical to the basic dense format, with the addition of the first column as the unique key.

If the input file is sparse, but a dense kernel is invoked, Somoclu will execute and results will be incorrect. Invoking a sparse kernel on a dense input file is likely to lead to a segmentation fault.

# Interfaces

[Python](https://somoclu.readthedocs.io/), [Julia](https://github.com/peterwittek/Somoclu.jl), [R](https://cran.r-project.org/web/packages/Rsomoclu/), and [MATLAB](https://github.com/peterwittek/somoclu/tree/master/src/MATLAB) interfaces are available for the dense CPU and GPU kernels. MPI and the sparse kernel are not support through the interfaces. For respective examples, see the folders in src.

The Python version is also available in [PyPI](https://pypi.python.org/pypi/somoclu). You can install it with

    $ pip install somoclu

Alternatively, it is also available on [conda-forge](https://github.com/conda-forge/somoclu-feedstock):

    $ conda install somoclu

Some pre-built binaries in the wheel format or Windows installer are provided at [PyPI Dowloads](https://pypi.python.org/pypi/somoclu#downloads), they are tested with [Anaconda](https://www.continuum.io/downloads) distributions. If you encounter errors like `ImportError: DLL load failed: The specified module could not be found` when `import somoclu`, you may need to use [Dependency Walker](http://www.dependencywalker.com/) as shown [here](http://stackoverflow.com/a/24704384/1136027) on `_somoclu_wrap.pyd` to find out missing DLLs and place them at the write place. Usually right version (32/64bit) of `vcomp90.dll, msvcp90.dll, msvcr90.dll` should be put to `C:\Windows\System32` or `C:\Windows\SysWOW64`.

The wheel binaries for macOS are compiled with the system `clang++`, which means by default it is not parallelized. To use the parallel version on Mac, you can either use the version in [conda-forge](https://github.com/conda-forge/somoclu-feedstock) or compile it from source with your favourite OpenMP-friendly compiler. To get it working with the GPU kernel, you might have to follow the instructions at the [Somoclu - Python Interface](https://github.com/peterwittek/somoclu/tree/master/src/Python).

The R version is available on CRAN. You can install it with

    install.packages("Rsomoclu")

To get it working with the GPU kernel, download the source zip file and specify your CUDA directory the following way:

    R CMD INSTALL src/Rsomoclu_version.tar.gz --configure-args=/path/to/cuda

The Julia version is available on [GitHub](https://github.com/peterwittek/Somoclu.jl). The standard `Pkg.add("Somoclu")` should work.

For using the MATLAB toolbox, install SOM-Toolbox following the instructions at [ilarinieminen/SOM-Toolbox](https://github.com/ilarinieminen/SOM-Toolbox) and define the location of your MATLAB install to the configure script:

    ./configure --without-mpi --with-matlab=/usr/local/MATLAB/R2014a

For the GPU kernel, specify the location of your CUDA library for the configure script. More detailed instructions are in the [MATLAB source folder](https://github.com/peterwittek/somoclu/tree/master/src/MATLAB).

# Compilation & Installation

These are the instructions for compiling the core library and the command line interface. The only dependency is a C++ compiler chain -- GCC, ICC, clang, and VC were tested.

Multicore execution is supported through OpenMP -- the compiler must support this. Distributed systems are supported through MPI. The package was tested with OpenMPI. It should also work with other MPI flavours. CUDA support is optional.

## Linux or macOS

If you have just cloned the git repository first run

    $ ./autogen.sh

Then follow the standard POSIX procedure:

    $ ./configure [options]
    $ make
    $ make install

Options for configure

    --prefix=PATH           Set directory prefix for installation

By default Somoclu is installed into /usr/local. If you prefer a
different location, use this option to select an installation
directory.

    --without-mpi           Disregard any MPI installation found.
    --with-mpi=MPIROOT      Use MPI root directory.
    --with-mpi-compilers=DIR or --with-mpi-compilers=yes
                              use MPI compiler (mpicxx) found in directory DIR, or
                              in your PATH if =yes
    --with-mpi-libs="LIBS"  MPI libraries [default "-lmpi"]
    --with-mpi-incdir=DIR   MPI include directory [default MPIROOT/include]
    --with-mpi-libdir=DIR   MPI library directory [default MPIROOT/lib]

The above flags allow the identification of the correct MPI library the user wishes to use. The flags are especially useful if MPI is installed in a non-standard location, or when multiple MPI libraries are available.

    --with-cuda=/path/to/cuda           Set path for CUDA

Somoclu looks for CUDA in /usr/local/cuda. If your installation is not there, then specify the path with this parameter. If you do not want CUDA enabled, set the parameter to `--without-cuda`.

## Windows

Use the `somoclu.sln` under `src/Windows/somoclu` as an example Visual Studio 2015 solution. Modify the CUDA version or VC compiler version according to your needs.

The default solution enables all of OpenMP, MPI, and CUDA. The default MPI installation path is `C:\Program Files (x86)\Microsoft SDKs\MPI\`, modify the settings if yours is in a different path. The configuration default CUDA version is 9.1. Disable MPI by removing `HAVE_MPI` macro in the project properties (`Properties -> Configuration Properties -> C/C++ -> Preprocessor`). Disable CUDA by removing `CUDA` macro in the solution properties and uncheck CUDA in `Project -> Custom Build Rules`. If you open the solution without CUDA installed, please remove the following sections in `somoclu.vcxproj`:

```
  <ImportGroup Label="ExtensionSettings">
    <Import Project="$(VCTargetsPath)\BuildCustomizations\CUDA 9.1.props" />
  </ImportGroup>
```

and

```
  <ImportGroup Label="ExtensionTargets">
    <Import Project="$(VCTargetsPath)\BuildCustomizations\CUDA 9.1.targets" />
  </ImportGroup>
```

or change the version number according to which you installed.

The usage is identical to the Linux version through command line (see the relevant section).

# Acknowledgment

This work was supported by the European Commission Seventh Framework Programme under Grant Agreement Number FP7-601138 PERICLES and by the AWS in Education Machine Learning Grant award.

# Citation

1. Peter Wittek, Shi Chao Gao, Ik Soo Lim, Li Zhao (2017). Somoclu: An Efficient Parallel Library for Self-Organizing Maps. Journal of Statistical Software, 78(9), pp.1--21. DOI:[10.18637/jss.v078.i09](https://doi.org/10.18637/jss.v078.i09).
   arXiv:[1305.1422](https://arxiv.org/abs/1305.1422).
