## Compiling

You need some MPI library on your system and a relatively new version of gcc (C++-11 compatible). I have been testing with gcc-4.9.1.

Get the right branch

    git checkout variant_master_merge
    #make sure you have the new gcc version in your PATH
    make MPIPATH=/usr/lib64/openmpi/bin/  SPEED=1   -j 8

Examples are built in variant/example/bin

    ./variant/example/bin/example_libtiledb_variant_driver <workspace> <array> <begin> <end>
    #Dummy genotyping operation - median over samples for the PL vector
    ./variant/example/bin/gt_example_query_processor -w <workspace> -A <array> -p <position>


    
