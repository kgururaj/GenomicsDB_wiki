### Example tools
Examples are built in the directory bin/.

* Simple text format dump of variants intersecting given column range

        ./bin/example_libtiledb_variant_driver <workspace> <array> <begin> <end>

* Dummy operation - median over samples for the PL vector for a single position (column position).

        ./bin/gt_example_query_processor -w <workspace> -A <array> -p <position>

* Dummy operation - median over samples for the PL vector for a whole list of positions. The positions file has one column value per line. 
  
        ./bin/gt_example_query_processor -w <workspace> -A <array> -P <positions_file>

### Merging multiple sorted CSVs into a single CSV file
Note: if you are using a relatively new version of GNU sort, you can use the 'parallel' command line argument. If not, remove the 'parallel' argument.
    
    sort --parallel=8 -T <tmp_dir> -t, -m -k2,2n -k1,1n <list_of_csv_files>  > merged.csv

### Defining a TileDB array schema
* All binaries in the directory: tiledb_cmd/bin/release/
* Manpages: man manpages/man/tiledb_define_array.1 

         tiledb_define_array \
         -w <workspace> \
         -A <array_name> \
         -a END,REF,ALT,QUAL,FILTER,BaseQRankSum,ClippingRankSum,MQRankSum,ReadPosRankSum,DP,MQ,MQ0,DP_FORMAT,MIN_DP,GQ,SB,AD,PL,GT \
         -d samples,position \
         -D 0,<num_samples>-1,0,4000000000 \
         -t int64,char:var,char:var,float,int:var,float,float,float,float,int,float,int,int,int,int,int:4,int:var,int:var,int:var,int64 \
         -o column-major \
         -c 1000 \
         -s 5 \
         -z GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP,GZIP 
         
  * _\<workspace\>_: Directory (preferably on the local disk) where the array data will be stored
  * _\<array_name\>_ : Arbitrary string, TileDB creates a directory _\<workspace\>/StorageManager/\<array_name\>_ and stores data inside the directory
  * _\<num_samples\>_ : Number of samples being imported (must be exact)

## Loading
Manpages: man manpages/man/tiledb_load_csv.1 

Command to run if you have a merged, sorted CSV file:

    tiledb_load_csv -w <workspace> -A <array> -p <merged_sorted_csv_file> -m sorted

Command to run if you have multiple, sorted CSV files

    tiledb_load_csv -w <workspace> -A <array> -m sorted -p <directory_containing_csv_files>

If you are not sure whether your CSV files are sorted, simply omit the "-m sorted" argument

    tiledb_load_csv -w <workspace> -A <array> -p <directory_containing_csv_files>