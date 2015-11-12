## Compiling

Requirements:
* An MPI library
* A relatively new version of gcc (C++-11 compatible). I have been testing with gcc-4.9.1.

Get the right branch

    git checkout variant_master_merge
    #make sure you have the new gcc version in your PATH
    #release mode - O3, NDEBUG - assertions disabled
    make MPIPATH=/usr/lib64/openmpi/bin/  BUILD=release   -j 8
    #debug mode - assertions enabled, can use gdb for stepping
    make MPIPATH=/usr/lib64/openmpi/bin/  BUILD=debug   -j 8

Examples are built in variant/example/bin

    ./variant/example/bin/example_libtiledb_variant_driver <workspace> <array> <begin> <end>

    #Dummy genotyping operation - median over samples for the PL vector for a single position
    ./variant/example/bin/gt_example_query_processor -w <workspace> -A <array> -p <position>

    #Dummy genotyping operation - median over samples for the PL vector for a whole list of positions
    ./variant/example/bin/gt_example_query_processor -w <workspace> -A <array> -P <positions_file>

## Merging multiple sorted CSVs into a single CSV file
Note: if you are using a relatively new version of GNU sort, you can use the 'parallel' command line argument. If not, remove the 'parallel' argument.
    
    sort --parallel=8 -T <tmp_dir> -t, -m -k2,2n -k1,1n <list_of_csv_files>  > merged.csv

## Defining the array
All binaries in the directory: tiledb_cmd/bin/release/
Manpages: man manpages/man/tiledb_load_csv.1 

    tiledb_define_array \
    -w $ws \
    -A $array_name \
    -a END,REF,ALT,QUAL,FILTER,BaseQRankSum,ClippingRankSum,MQRankSum,ReadPosRankSum,DP,MQ,MQ0,DP_FORMAT,MIN_DP,GQ,SB,AD,PL,GT \
    -d samples,position \
    -D 0,$num_samples-1,0,4000000000 \
    -t int64,char:var,char:var,float,int:var,float,float,float,float,int,float,int,int,int,int,int:4,int:var,int:var,int:var,int64 \
    -o column-major \
    -c 1000 \
    -s 5

  * _ws_: Directory (preferably on the local disk) where the array data will be stored
  * _array_name_ : Arbitrary string, TileDB creates a directory $ws/$array_name and stores data inside the directory
  * _num_samples_ : Number of samples being imported (must be exact)

## Loading

    tiledb_load_csv -w <workspace> -A <array> -p <merged_sorted_csv_file> -m sorted

    

## TODO
* Scan operation not implemented
