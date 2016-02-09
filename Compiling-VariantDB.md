## Requirements:
* An MPI compiler, library and runtime (we have tested with reasonably new versions of OpenMPI and MVAPICH2).
* A relatively new version of gcc (C++-11 compatible). I have been testing with gcc-4.9.1.
* [Rapidjson library](https://github.com/miloyip/rapidjson): Parameters are passed to TileDB tools/examples through a JSON file - Rapidjson is . The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

* If you wish to use any functionality in TileDB that will require parsing/creating a VCF/gVCF/BCF, then you need [htslib](https://github.com/samtools/htslib). We maintain a [fork of htslib](https://github.com/kgururaj/htslib) with some modifications that are needed for use with VariantDB.

        git clone https://github.com/kgururaj/htslib
        cd htslib
        git checkout intel_fixes
        make -j 8

## Building
* Get the right branch based on what you wish to do - see the other pages for which branch to get. If you do not know which branch to use, the branch *variant_master_merge* is your best bet.
* Make sure you have your new gcc version in your PATH.
* Compiling in release mode

        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8

* Compiling in debug mode (for developers)

        #debug mode - assertions enabled, can use gdb for stepping, no OPENMP (can enable with the OPENMP=1 flag)
        make MPIPATH=<mpi_package_dir>/bin/ RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=debug -j 8

* Compiling with support for parsing/creating a gVCF/VCF/BCF: compile htslib as described above.

        make MPIPATH=<mpi_package_dir>/bin/ HTSDIR=<htslib_directory> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8