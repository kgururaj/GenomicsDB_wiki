We provide a Java wrapper class called VCF2TileDB for importing data into TileDB/GenomicsDB - there are two modes of using this class.

# Mode 1: Importing VCF/CSV files
In this mode, the functionality of this class is identical to the _vcf2tiledb_ executable described in the previous sections. It provides multiple variants of a function call _write()_ that does the actual loading. The parameters for the _write()_ call are listed below:

* _loaderJSON_ (type:string): Path to loader JSON file (as described on [[this page|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]]).
* _rank_ (type: int): Rank of the process (similar to the MPI rank). When the loader JSON has multiple partitions, this parameter specifies the partition index the current process should write to.
* _lbRowIdx_ and _ubRowIdx_ (type: int64): Lower bound and upper bound of row indexes to be imported into TileDB/GenomicsDB - useful when [[adding new samples to an existing array|Incremental-import-into-GenomicsDB]].

Take a look at our [example code](https://github.com/Intel-HLS/GenomicsDB/tree/master/example/java/test_genomicsdb_jar) for using this interface.

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

Note that if you do not use MPI and you do not specify the rank on the command line, only the first partition's data is loaded. This is equivalent to running the program with rank 0.

# Mode 2: Java API for importing [VariantContext](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/variantcontext/VariantContext.html) objects
This mode allows developers to import data into GenomicsDB by using the VCF2TileDB class functions without having to write intermediate files - for example, a variant annotation tool written in Java can use this API to directly write results to a GenomicsDB instance.

A VCF2TileDB object can be constructed using the same parameters described in the previous section. The primary function call is _addSortedVariantContextIterator_. This functions provides an iterator over VariantContext objects that occur in column-major order (see the [[terminology section|Basic-TileDB-GenomicsDB-terminology]]). Multiple such iterators can be provided to the VCF2TileDB object. The VCF2TileDB class uses each iterator to serialize VariantContext objects into a byte buffer which is passed to the C++ modules of GenomicsDB. Internally, GenomicsDB will 'merge' the buffers ensuring that data is loaded in column-major order.

Typically, the caller code will look something similar to:

    VCF2TileDB loader(loaderJSONFile, rank, lbRowIdx, ubRowIdx);
    loader.addSorted
