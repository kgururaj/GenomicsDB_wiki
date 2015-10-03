Read [this page](https://github.com/Intel-HSS/TileDB/wiki/Using-the-variant-specific-customizations) for instructions on how to compile and run the single node version of the variant library first.

Requirements:
* An MPI library
* [Rapidjson library](https://github.com/miloyip/rapidjson): I use this library to pass the query configuration in a json file to TileDB. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

Get the right branch of the TileDB repo for the TileDB implementation.

    git checkout multi_node_var
    #make sure you have the new gcc version in your PATH
    #release mode - O3, NDEBUG - assertions disabled
    make MPIPATH=/usr/lib64/openmpi/bin/  BUILD=release RAPIDJSONDIR=<rapidjson_dir>  -j 8
    #debug mode - assertions enabled, can use gdb for stepping
    make MPIPATH=/usr/lib64/openmpi/bin/  BUILD=release RAPIDJSONDIR=<rapidjson_dir>  -j 8

make MPIPATH=/opt/openmpi-1.10/bin/  RAPIDJSON_INCLUDE_DIR=/home/karthikg/softwares/setup_files/rapidjson/include/ DO_PROFILING=1 USE_BIGMPI=/home/karthikg/broad/non_variant_db/variantDB/BigMPI/ -j 8 BUILD=release VERBOSE=1