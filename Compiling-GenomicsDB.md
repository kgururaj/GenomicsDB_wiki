## Requirements:
* An MPI compiler, library and runtime (we have tested with reasonably new versions of OpenMPI, MPICH and MVAPICH2).
*  gcc version >= 4.9.0 (C++-11 and OpenMP v4 compatible), we have been testing with gcc-4.9.1.
  
    Note that we use directives from OpenMP specification v4. This is supported on gcc versions >= 4.9.0. Without a new enough compiler, you will see compilation errors around the line listed below:

        #pragma omp parallel for default(shared) num_threads(m_num_parallel_vcf_files) reduction(l0_sum_up : combined_histogram)

    Alternately, you can disable using OpenMP by omitting the flag OPENMP=1 during compilation (see below). You may lose some performance during loading without OpenMP
* Dependencies from TileDB
    * Zlib headers and libraries
    * OpenSSL headers and libraries
* If you wish to import [[CSV data into TileDB|Importing-CSVs-into-GenomicsDB]], then you need 
[libcsv](https://sourceforge.net/projects/libcsv/). You also need to pass special flags while invoking make (see below).

    On RedHat based systems, if you have the [EPEL repo](https://fedoraproject.org/wiki/EPEL) installed and enabled, you 
can install the libcsv packages using yum:

         #As root
         yum -y install libcsv libcsv-devel

* *NOTE*: We use git submodules to pull in the remaining dependencies - you can skip directly to the [[building|Compiling-GenomicsDB#Building]] section if you do not wish to manually fetch and build the following dependencies.
* TileDB

        git clone https://github.com/Intel-HLS/TileDB.git

* [Rapidjson library](https://github.com/miloyip/rapidjson): Parameters are passed to TileDB tools/examples through a JSON file - Rapidjson is used to parse this JSON file. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

* [Htslib](https://github.com/samtools/htslib) for parsing and exporting VCFs. We maintain a [fork of htslib](https://github.com/Intel-HLS/htslib) with some modifications that are needed for use with GenomicsDB.

        git clone https://github.com/Intel-HLS/htslib
        cd htslib
        git checkout intel_mods
        make -j 8

## Building
* Get the right branch based on what you wish to do - see the other pages for which branch to get. If you do not know which branch to use, the *master* branch is your best bet.
* To get dependencies using git submodule, run:

        git clone --recursive https://github.com/Intel-HLS/GenomicsDB.git

* Make sure you have your new gcc version in your PATH.
* Assuming you want to use the dependencies pulled in by git and you have the MPI compiler (mpicxx) in your PATH
        
        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make BUILD=release OPENMP=1 -j 8

* Compiling in debug mode:

        #debug mode - assertions enabled, can use gdb for stepping, no OPENMP (can enable with the OPENMP=1 flag)
        make BUILD=debug -j 8

* If you do not have the MPI compiler in your PATH (note that the trailing slash is required):
        
        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release OPENMP=1 -j 8

* If you wish to import CSV data and libcsv is installed at a location where the compiler can find the header file and 
library (for example under /usr), then enable libcsv usage

        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release USE_LIBCSV=1 OPENMP=1 -j 8

    If you have downloaded libcsv [from sourceforge](https://sourceforge.net/projects/libcsv/) and compiled it at a 
custom location, then pass the directory to the make command


        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release LIBCSV_DIR=<libcsv_directory> OPENMP=1 -j 8

    The build process assumes that the library file is located in _\<libcsv_directory\>/.libs+ (the default location in the 
build process of libcsv).

If you have downloaded and compiled the dependencies manually, use the following commands:

* Compiling in release mode

        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ TILEDB_DIR=<TileDB_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8

* If you have the MPI compilers (mpicc, mpicxx) in your PATH, you can drop the MPIPATH argument

        #release mode
        make TILEDB_DIR=<TileDB_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8

* Compiling in debug mode (for developers)

        #debug mode - assertions enabled, can use gdb for stepping, no OPENMP (can enable with the OPENMP=1 flag)
        make MPIPATH=<mpi_package_dir>/bin/ TILEDB_DIR=<TileDB_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=debug -j 8

* Compiling with support for parsing/creating a gVCF/VCF/BCF: compile htslib as described above.

        make MPIPATH=<mpi_package_dir>/bin/ TILEDB_DIR=<TileDB_dir> HTSDIR=<htslib_directory> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8
