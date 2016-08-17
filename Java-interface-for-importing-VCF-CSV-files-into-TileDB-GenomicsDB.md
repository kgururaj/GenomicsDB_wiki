We provide a Java wrapper class called VCF2TileDB for importing data into TileDB/GenomicsDB - the functionality of this class is identical to the _vcf2tiledb_ executable described in the previous sections. It provides multiple variants of a function call _write()_ that does the actual loading. The parameters for the _write()_ call are listed below:

* _loaderJSON_ (type:string): Path to loader JSON file (as described on [[this page|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]]).
* _rank_ (type: int): Rank of the process (similar to the MPI rank). When the loader JSON has multiple partitions, this parameter specifies the partition index the current process should write to.
* _lbRowIdx_ and _ubRowIdx_ (type: int64): Lower bound and upper bound of row indexes to be imported into TileDB/GenomicsDB - useful when [[adding new samples to an existing array|Incremental-import-into-GenomicsDB]].

Take a look at our [example code](https://github.com/Intel-HLS/GenomicsDB/tree/java_load_api/example/java/test_genomicsdb_jar) for using this interface.

## Caveats
For portability reasons, the native library packaged into the GenomicsDB JAR file distributed on Maven Central is built **WITHOUT** MPI and OpenMP support. Hence, when using the GenomicsDB JAR from Maven Central:
* The loading process will be slower compared to a JAR built on your cluster/system with MPI and OpenMP enabled.
* None of the MPI functionality described below will work.

## Loading data into multiple TileDB partitions using the Java interface
We use the TestGenomicsDB class as the driver in our examples.
### Using MPI
For more information on how to use MPI in the context of GenomicsDB, see [[this page|MPI-with-GenomicsDB]] first.

    mpirun -n <num_partitions> -hostfile <hostfile> <MPI_args> java TestGenomicsDB -load <loader.json> 0 <lbRowIdx> <ubRowIdx>

The MPI runtime assigns the rank (partition index) for each process - just pass 0 to TestGenomicsDB.

### Manually specifying the rank of each process (without MPI)
Since the different processes don't communicate in the current setup, you can run them without MPI and specify the rank manually:

    ssh host0 "java TestGenomicsDB -load <loader.json> 0 <lbRowIdx> <ubRowIdx>"
    ssh host1 "java TestGenomicsDB -load <loader.json> 1 <lbRowIdx> <ubRowIdx>"
    ...

