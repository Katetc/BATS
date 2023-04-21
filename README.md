Build and test structure
========================

The Build And Test Structure (BATS) is primarily intended to allow users and developers to
quickly generate a set of regression tests for use with the Land Ice Validation
and Verification toolkit [LIVVkit](https://github.com/LIVVkit/LIVVkit).

BATS is a [Python 3.6](https://www.python.org/) module that is primarily
controlled by command line options.
BATS requires:

Python Packages
* [python-numpy](https://pypi.python.org/pypi/numpy)
* [python-scipy](https://pypi.python.org/pypi/scipy)
* [python-netCDF4](https://pypi.python.org/pypi/netCDF4)
* [python-matplotlib](https://pypi.python.org/pypi/matplotlib)

External Packages
* [NetCDF 4.3.0+](http://www.unidata.ucar.edu/software/netcdf/)
* [NCO (NetCDF Operators) 4.4.0](http://nco.sourceforge.net/)
* [HDF5 1.8.6](https://www.hdfgroup.org/HDF5/)

If you have a working install of CISM, and you installed the suggested packages
in the [CISM users manual](https://escomp.github.io/cism-docs),
you'll likely already have everything you need. If you haven't previously built
CISM on your machine, we suggest following the installation instructions as they
are laid out in the users manual first.

(Note: on some High Performance Computing platforms, a
`setup_PLATFORM.bash` script has been provided which will attempt to load the
needed modules.)

If you are having any troubles with dependencies, open an issue on the
[issue tracker](https://github.com/LIVVkit/BATS/issues)!

Installation and Usage
======================
Since BATS is most useful as a companion to the Land Ice Validation and Verification
Toolkit [LIVVkit](https://github.com/LIVVkit/LIVVkit), it is reccomended to create an
Anaconda Python environment, with LIVVkit as the only requirement.

Firstly be sure Anaconda / Mamba is loaded or installed:

- On Cheyenne: `module load conda/latest`
- On Perlmutter `module load python/3.9-anaconda-2021.11`
- Workstations or other HPC see
[Mamba](https://mamba.readthedocs.io/en/latest/installation.html)

```sh
conda env create -n livv -c conda-forge livvkit
conda activate livv
```
this will install everything needed to run LIVVkit and BATS, and load the environment.

Then, clone the BATS repository, change into the directory:

```sh
git clone git@github.com:LIVVkit/BATS.git
cd BATS
```

If, for example you are using the Cheyenne supercomputer, setup the correct modules:
```sh
source setup_cheyenne
```

On other platforms, ensure that the compiler you intend to use (GCC, Intel, etc.) is
loaded, as are CMake >= 3.7, as are the netCDF, and pnetCDF libraries.

You are now ready to run BATS!

The main BATS run script is `build_and_test.py`. BATS is primarily controlled via
options specified at the command line. To see the full list of options, run:

```sh
./build_and_test.py --help
```

Example test
============
If the location of your CISM source code is stored in the environment variable `$CISM`,
and your desired output directory (say `/glade/scratch/${USER}/cism_tests/branchname_a`)
is `$OUTDIR` you can invoke BATS by:

```sh
./build_and_test.py --platform cheyenne-intel --cism-dir ${cism} --build_dir ./build --out-dir $OUTDIR
```

This will build CISM, setup the test, and if on an HPC system print out instructions on
running the tests, or on a workstation it will run the tests, and output will be directed
as described below in "How BATS works".

To use LIVVkit, to analize the differences, point the `$cism` environment variable at your
updated code, and re-run, directing output to a new output directory
(e.g. :`/glade/scratch/${USER}/cism_tests/branchname_b`), then run LIVVkit's validation
suite

```sh
REF=/glade/scratch/${USER}/cism_tests/branchname_a/cheyenne-intel/CISM_glissade
TEST=/glade/scratch/${USER}/cism_tests/branchname_b/cheyenne-intel/CISM_glissade
livv -v $TEST $REF -o branch_vs_another -s
```
Which will generate a results website, in `./branch_vs_another` directory and serve
it locally. See [the LIVVkit docs](https://livvkit.github.io/Docs/quickstart.html#basic-usage)
for a quickstart guide, and more detailed instructions.


How BATS works
==============

BATS works very similar to how you would build CISM and then run one or more CISM
tests. BATS will build a version of CISM, and then either run a set of
regression tests if you are using a personal computer (PC), or setup a series of
regression tests and generate a job submission script if you are using a high
performance computer (HPC).

That is, if CISM is located in `$CISM`  on ORNL's HPC
platform titan, for example, and you want to run the `$CISM/higher-order/dome`
test using the gnu compiler, you would typically build CISM by:

```sh
cd $CISM/builds/titan-gnu/
source titan-gnu-cmake
make -j 8
```

which would build a parallel version of CISM into the
`$CISM/builds/titan-gnu/cism_driver/`
directory. Then you would go to the dome test directory, create a link to the
CISM driver, and run the test:

```sh
cd $CISM/tests/higher-order/dome
ls -s $CISM/builds/titan-gnu/cism_driver/cism_driver cism_driver
./runDome.py
```

You would have to repeat this process for each individual test you'd like
to run. BATS, however, automates these steps. On titan, running BATS like so

```sh
cd $CISM/tests/regression/
source setup_titan.bash
./build_and_test.py -p titan -c gnu -b ./build
```

will result in BATS generating all the CMake build file into a new directory
called `build`, and building CISM into the directory `build/cism_driver/`. BATS
will then run a set of CISM's higher-order tests:

* Dome (at a variety of resolutions and processor counts)
* Circular Shelf
* Confined Shelf
* ISMIP-HOM a and c (at 20 and 80 km resolutions)
* ISMIP-HOM f
* Stream

All of the files associated with each test will be output to a directory called
`reg_test/titan-gnu/CISM_glissade/` which has a leaf-node directory structure that
provides some metadata for each test:

```sh
 reg_test/
    └── PLATFORM-COMPILER/
        └── CISM_glissade/
            ├── CMakeCache.txt
            |-- TEST/
                |-- CASE/
                    |-- DOF/ (degrees of freedom)
                        |-- PROCESSORS/
                            |-- [OPTIONAL TEST SPECIFIC DIRS]
                                |-- files.ext
        --------------------------------------------
            ├── all_jobs/
            │    ├── platform_job.small
            │    ├── platform_job.small_timing_?
            │    ├── platform_job.large
            │    └── platform_job.large_timing_?
            ├── submit_all_jobs.bash
            └── clean_timing.bash
```

where `DOF` is an s-prefixed integer indicating the test's size (units are test
specific), and `PROCESSORS` is p-prefixed integer indicating the number of
processors used to run the test. Everything below the dashed line will only
appear on HPC systems. `submit_all_jobs.bash` will submit all the jobs in the
`jobs/` directory and `clean_timing.bash` will clean out any repeated test run
for performance profiling such that only the timing files remain (to be used
once all jobs have finished). This `reg_test/titan-gnu/CISM_glissade`
directory is formatted to be used with LIVVkit 2.1+ directly.

BATS is designed to be flexible and work with any LIVVkit usage scenario. In
order to do that, BATS provides a number of options to configure which system
you are using, when/where/how CISM is built, the destination of the output
directory, and which tests are run. For more information on these topics, see:


* Detailed discussion of [BATS options](https://github.com/LIVVkit/LIVVkit/wiki/BATS-options)
* [Out-of-source builds](https://github.com/LIVVkit/LIVVkit/wiki/CISM-out-of-source-builds)
* [The reg_test directory](https://github.com/LIVVkit/LIVVkit/wiki/BATS-reg-test) structure
* [LIVVkit usage](https://github.com/LIVVkit/LIVVkit/wiki/Usage)

Authors
=============

- Joseph H. Kennedy, ORNL
    - github : @jhkennedy
- Michael E. Kelleher, ORNL
    - github @mkstratos