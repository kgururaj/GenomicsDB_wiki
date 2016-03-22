## Requirements:
* An MPI compiler, library and runtime (we have tested with reasonably new versions of OpenMPI, MPICH and MVAPICH2).
* A relatively new version of gcc (C++-11 and OpenMP compatible). I have been testing with gcc-4.9.1.
* Dependencies from TileDB
    * Zlib headers and libraries
    * OpenSSL headers and libraries
* *NOTE*: We use git submodules to pull in the remaining dependencies - you can skip directly to the [[building|Compiling-GenomicsDB#Building]] section if you do not wish to manually fetch and build the following dependencies.
* TileDB

        git clone https://github.com/Intel-HSS/TileDB.git

* [Rapidjson library](https://github.com/miloyip/rapidjson): Parameters are passed to TileDB tools/examples through a JSON file - Rapidjson is used to parse this JSON file. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

* If you wish to use any functionality in GenomicsDB that will require parsing/creating a VCF/gVCF/BCF, then you need [htslib](https://github.com/samtools/htslib). We maintain a [fork of htslib](https://github.com/Intel-HSS/htslib) with some modifications that are needed for use with GenomicsDB.

        git clone https://github.com/Intel-HSS/htslib
        cd htslib
        git checkout intel_mods
        make -j 8

## Building
* Get the right branch based on what you wish to do - see the other pages for which branch to get. If you do not know which branch to use, the *master* branch is your best bet.
* To get dependencies, using git submodule:

        git clone --recursive <GenomicsDB_url> 

* Make sure you have your new gcc version in your PATH.
* Assuming you want to use the dependencies pulled in by git and you have mpicxx in your PATH
        
        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make BUILD=release OPENMP=1 -j 8

* Compiling in debug mode:

        #debug mode - assertions enabled, can use gdb for stepping, no OPENMP (can enable with the OPENMP=1 flag)
        make BUILD=debug -j 8

* If you do not have the MPI compiler in your PATH (note that the trailing slash is required):
        
        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release OPENMP=1 -j 8

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
