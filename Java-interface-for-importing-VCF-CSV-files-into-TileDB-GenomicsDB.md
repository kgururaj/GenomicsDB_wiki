We provide a Java wrapper class called GenomicsDBImporter for importing data into TileDB/GenomicsDB - there are two modes of using this class. Both modes require some common parameters:
* _loaderJSON_ (type: _String_): Path to loader JSON file (as described on [[this page|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]]).
* _rank_ (type: _int_): Rank of the process (similar to the MPI rank). When the loader JSON has multiple partitions, this parameter specifies the partition index the current process should write to. See [[ this section | Importing-VCF-data-into-GenomicsDB#running-the-program ]] for an explanation of how the rank/partition index parameter controls the loader program.
* _lbRowIdx_ and _ubRowIdx_ (type: _long_): Lower bound and upper bound of row indexes to be imported into TileDB/GenomicsDB - useful when [[adding new samples to an existing array|Incremental-import-into-GenomicsDB]].

# Mode 1: Importing VCF/CSV files
In this mode, the functionality of the GenomicsDBImporter class is identical to the _vcf2tiledb_ executable described in the previous sections. It provides multiple variants of a function _write()_ that does the actual loading. The parameters for the _write()_ call are the four parameters described above.

Take a look at our [example code](https://github.com/Intel-HLS/GenomicsDB/blob/master/example/java/TestGenomicsDB.java) for using this interface.

## Caveats
For portability reasons, the native library packaged into the GenomicsDB JAR file distributed on Maven Central is built **WITHOUT** MPI and OpenMP support. Hence, when using the GenomicsDB JAR from Maven Central:
* The loading process will be slower compared to a JAR built on your cluster/system with MPI and OpenMP enabled.
* None of the MPI functionality described below will work.

## Loading data into multiple TileDB partitions using the Java interface
We use the TestGenomicsDB class as the driver in our examples.
### Using MPI
For more information on how to use MPI in the context of GenomicsDB, see [[this page|MPI-with-GenomicsDB]] first.

    mpirun -n <num_partitions> -hostfile <hostfile> <MPI_args> java TestGenomicsDB -load <loader.json> 0 <lbRowIdx> <ubRowIdx>

The MPI runtime assigns the rank (partition index) for each process - pass 0 to TestGenomicsDB from the command line.

### Manually specifying the rank of each process (without MPI)
Since the different processes don't communicate in the current setup, you can run them without MPI and specify the rank/partition index manually:

    ssh host0 "java TestGenomicsDB -load <loader.json> 0 <lbRowIdx> <ubRowIdx>"
    ssh host1 "java TestGenomicsDB -load <loader.json> 1 <lbRowIdx> <ubRowIdx>"
    ...

Note that if you do not use MPI and you do not specify the rank on the command line, only the first partition's data is loaded. This is equivalent to running the program with rank 0.

# Mode 2: Java API for importing [VariantContext](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/variantcontext/VariantContext.html) objects
This mode allows developers to import data into GenomicsDB by using functions of the class GenomicsDBImporter without having to write intermediate files. For example, a variant annotation tool written in Java can use this API to directly write results to a GenomicsDB instance.

A GenomicsDBImporter object can be constructed using the same parameters described in the previous section. The primary function call to load VariantContext objects is _addSortedVariantContextIterator_. This function is passed an iterator over VariantContext objects that occur in column-major order (see the [[terminology section|Basic-TileDB-GenomicsDB-terminology]]). Multiple such iterators can be provided to a GenomicsDBImporter object. The GenomicsDBImporter class uses each iterator to obtains VariantContext objects and serializes these objects into a byte buffer. The byte buffer is passed to the C++ modules of GenomicsDB. Internally, GenomicsDB will 'merge' the buffers ensuring that data is loaded in column-major order across all samples/CallSets.

The user still needs to provide [[information about callsets | Importing-VCF-data-into-GenomicsDB#samplescallsets]] while using the API. Each VariantContext object has a corresponding VCF header. The header may contain multiple samples/CallSets. The user must provide a unique name and a unique TileDB row index for each sample/CallSet. There are two ways of providing this information which are described later in the section.

Typically, the caller code will look similar to:

    GenomicsDBImporter loader(loaderJSONFile, rank, lbRowIdx, ubRowIdx);
    loader.addSortedVariantConextIterator(streamName1, vcfHeader1, iterator1, bufferCapacity, streamFormat, sampleIndexToInfo1);
    loader.addSortedVariantConextIterator(streamName2, vcfHeader2, iterator2, bufferCapacity, streamFormat, sampleIndexToInfo2);
    loader.importBatch();
    assert loader.isDone();

* _streamName_ (type: _String_): Arbitrary string - must be unique for each invocation of addSortedVariantConextIterator(). For a given GenomicsDBImporter object, this parameter identifies each stream uniquely.
* _vcfHeader_ (type: _VCFHeader_): [htsjdk VCF header object](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/vcf/VCFHeader.html) for this stream.
* _iterator_ (type: _Iterator\<VariantContext\>_)
* _bufferCapacity_ (type: _long_): Size (in bytes) of the byte buffer used to pass data to the C++ modules of GenomicsDB. Larger values can possibly improve performance at the expense of memory consumption. Recommended value - around 10KiB. Note that internally the class may increase the buffer size if needed - hence, it's functionally safe to assign small values (including 0).
* _streamFormat_ (type: _VariantContextWriterBuilder.OutputType_): Format in which VariantContext objects are serialized in the byte buffer that is passed to the GenomicsDB C++ modules. This parameter can take one of two values:
  * [VariantContextWriterBuilder.OutputType](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/variantcontext/writer/VariantContextWriterBuilder.OutputType.html).BCF_STREAM: Objects are serialized in the BCF2 format - recommended for performance.
  * [VariantContextWriterBuilder.OutputType](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/variantcontext/writer/VariantContextWriterBuilder.OutputType.html).VCF_STREAM: Objects are serialized in the VCF format.
* _sampleIndexToInfo_ (type: _Map\<Integer, GenomicsDBImporter.SampleInfo\>_):   Holds sample/CallSet mapping information.
  * Map key (type: _Integer_): Index of the sample/CallSet in the VCF header. This is identical to the parameter [[ idx_in_file explained previously | Importing-VCF-data-into-GenomicsDB#samplescallsets]]. If the header has _N_ samples, but the map has _M_ entries, _M_ \< _N_, then only the samples/CallSets specified in the Map will be imported into GenomicsDB.
  * Map value (type: _GenomicsDBImporter.SampleInfo_): Contains two pieces of information
    * name (type: _String_): Name of the sample/CallSet - must be unique across all samples/CallSets in the TileDB/GenomicsDB array.
    * rowIdx (type: _long_): Row index of the sample/CallSets in the TileDB array - must be unique across all samples/CallSets.

  Sample information objects can be constructed by invoking the constructor `GenomicsDBImporter.SampleInfo sampleInfo(name, rowIdx);`.

  If you are absolutely sure that the sample names in your VCF header file are globally unique and you wish to import data from all the samples in the header, you can use a utility function provided by GenomicsDB to update the map object.

        rowIdx = GenomicsDBImporter.initializeSampleInfoMapFromHeader(sampleIndexToInfo, vcfHeader, rowIdx);

    * _rowIdx_ (type: _long_): the argument passed to the function contains the starting value from which row indexes are assigned to the samples/CallSets in the header. So, if the header has 3 samples and the argument value is 100, then the 3 samples will be assigned row indexes 100, 101 and 102.
    * _return value_ (type: _long_): Next available row index - in the above example, the returned value will be 103.

If the method described above is too cumbersome to use for providing sample/CallSet mapping, you can still use the _callset_mapping_file_ [[as described previously | Importing-VCF-data-into-GenomicsDB#samplescallsets ]] to specify the same information. You can pass a parameter called _stream_name_ (instead of _filename_) for specifying the source of information for each sample/CallSet. For example:

    {
        "callsets" : { 
            "HG00141" : {
                "row_idx" : 0,
                "idx_in_file": 0,
                "stream_name": "HG00141_stream"
            }
        }
    }

You must use the same value of _stream_name_ while calling _addSortedVariantContextIterator_. Since the sample/CallSet mapping information is specified in the JSON file, you can set the argument _sampleIndexToInfo_ to _null_ while invoking _addSortedVariantContextIterator_.

The recommended way is to use the JSON file for specifying CallSet/sample mapping, since the JSON file can be re-used for queries after the data is loaded. The Java API provides a way to specify this mapping purely for completeness.

## Example driver program
See our [example driver program](https://github.com/Intel-HLS/GenomicsDB/blob/master/example/java/TestBufferStreamGenomicsDBImporter.java) that demonstrates the API. The driver program takes as input the following command line parameters:

    -iterators loader.json stream_name_to_filename.json [bufferCapacity rank lbRowIdx ubRowIdx]

The JSON file _stream_name_to_filename.json_ contains entries of the form _\<stream_name\>: \<filename\>_. For example:

    {
        "HG00141_stream": "/data/HG00141.vcf.gz",
        "HG01598_stream": "/data/HG001598.vcf.gz"
    }

Note that the above file is only needed by our example driver program and not required for general use of the API. The driver opens the VCF files, creates iterators and uses the GenomicsDBImporter API to load data into the TileDB/GenomicsDB array.

## Possible sources of error (specific to the Java API):
* You cannot use a GenomicsDBImporter object for anything once the _importBatch()_ function has been called. If you do,  the program will fail with an exception.
* Each iterator provided must traverse VariantContext objects in column major order. So, if chromosomes "1", "2" and "3" are laid out in that order in the TileDB column space and an iterator provides VariantContext objects from chromosomes "1", "3" and "2" (in that order), then the column major order is violated and the program will fail with an exception.
* In a [[ multi-partition TileDB/GenomicsDB array | GenomicsDB-setup-in-a-multi-node-cluster]], a given GenomicsDBImporter object loads data into a single partition (as specified by the _rank_ parameter). If a VariantContext object that does not belong to the current partition is encountered, the program will fail with an exception.

  The parameter _ignore_cells_not_in_partition_ can be set to true in the [[ loader JSON file | Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program ]] - this will cause the import program to silently ignore VariantContext objects that do not belong to the current partition.
