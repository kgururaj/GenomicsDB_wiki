Read [this page](https://github.com/Intel-HSS/TileDB/wiki/Using-the-variant-specific-customizations) for instructions on how to compile and run the single node version of the variant library first.

## Requirements:
* An MPI library
* [Rapidjson library](https://github.com/miloyip/rapidjson): I use this library to pass the query configuration in a json file to TileDB. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

## Compiling

Get the right branch of the TileDB repo for the multi node implementation.

    git checkout multi_node_var
    #make sure you have the new gcc version in your PATH
    #release mode - O3, NDEBUG - assertions disabled
    make MPIPATH=/usr/lib64/openmpi/bin/  BUILD=release RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include  -j 8
    #debug mode - assertions enabled, can use gdb for stepping
    make MPIPATH=/usr/lib64/openmpi/bin/ BUILD=release RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include  -j 8
    #Verbose output, with profiling enabled
    make MPIPATH=/usr/lib64/openmpi/bin/ RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include -j 8 BUILD=release VERBOSE=1 DO_PROFILING=1

## Running the MPI program
* Single node, simple interval query:

        ./variant/example/bin/gt_mpi_gather -w  <workspace> -A <array> <begin> <end> -O <output_format>

  * Currently, \<output_format\> can only be _Cotton-JSON_ (case sensitive).
  * The following attributes will be queried: \[ "REF", "ALT", "BaseQRankSum", "AD", "PL" \] \(hard-coded\).
* Multi node, simple interval query
    