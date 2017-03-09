## Query result format
The sample query tools provided with the GenomicsDB repo return data in a JSON format. Query results may be returned as 
_VariantCalls_ or _Variants_ depending on command line parameters passed by the user.

_VariantCall_ is in some ways similar to a [GACall as defined by 
GA4GH](http://ga4gh.org/ga4gh_api.html#/schema/org.ga4gh.GACall). A _VariantCall_ object contains information stored for 
a given sample/CallSet for a given location/genomic interval in TileDB, i.e, it contains all the fields stored in the 
corresponding TileDB array cell. A sample _VariantCall_ formatted as a JSON is shown below:

    {
        "row": 2,
        "interval": [ 17384, 17386 ],
        "fields": {
            "REF": "GAT",
            "ALT": [ "GT","<NON_REF>" ],
            "PL": [ 1018,0,1116,1137,1224,2361 ],
            "BaseQRankSum": [ 1.046 ]
        }
    }

The above example shows a _VariantCall_ object for the sample/CallSet corresponding to row 2 in the TileDB at column
17384. Since the VariantCall is a deletion, it spans multiple columns and ends at column 17386 (inclusive). The _fields_ 
field is self explanatory.

_Variant_ is in some ways similar to [GAVariant as defined by 
GA4GH](http://ga4gh.org/ga4gh_api.html#/schema/org.ga4gh.GAVariant). In the GenomicsDB implementation, a _Variant_ is a 
set of _VariantCalls_ which meet the following properties:

1. All _VariantCalls_ span the exact same column interval
1. All _VariantCalls_ have the same value of REF
1. All _VariantCalls_ have the same value of ALT

A sample _Variant_ formatted as a JSON is shown below:

    {
        "interval": [ 17384, 17384 ],
            "common_fields" : {
                "REF": "G",
                "ALT": [ "A","<NON_REF>" ]
            },
            "variant_calls": [
            {
                "row": 0,
                "interval": [ 17384, 17384 ],
                "fields": {
                    "REF": "G",
                    "ALT": [ "A","<NON_REF>" ],
                    "BaseQRankSum": [ -2.096000 ]
                }
            },
            {
                "row": 2,
                "interval": [ 17384, 17384 ],
                "fields": {
                    "REF": "G",
                    "ALT": [ "A","<NON_REF>" ],
                    "BaseQRankSum": [ 1.046000 ]
                }
            }
        ]
    }

In the above example, the _Variant_ object corresponds to interval [17384, 17384] with "G" as the reference allele and 
"A", "\<NON_REF\>" as the alternate alleles. It consists of two _VariantCalls_ at rows 0 and 2.

## JSON configuration file for a query
A sample JSON query is shown below:

    {
        "workspace" : "/tmp/ws/",
        "array" : "t0_1_2",
        "query_column_ranges" : [ [ [0, 100 ], 500 ] ],
        "query_row_ranges" : [ [ [0, 2 ] ] ],
        "vid_mapping_file": "tests/inputs/vid.json",
        "callset_mapping_file": "tests/inputs/callset_mapping.json",
        "query_attributes" : [ "REF", "ALT", "BaseQRankSum", "MQ", "MQ0", "ClippingRankSum", "MQRankSum", "ReadPosRankSum", "DP", "GT", "GQ", "SB", "AD", "PL", "DP_FORMAT", "MIN_DP" ]
    }

Most of the fields are self-explanatory (take a look at the [[terminology section|Basic-TileDB-GenomicsDB-terminology]]).
The following fields are mandatory:
* _workspace_ (type: string or list of strings)
* _array_ (type: string or list of strings)
* _query_column_ranges_ : This field contains is a list of lists. Each member list of the outer list is handled by one 
MPI process. Each member list contains the column ranges that are to be queried by the MPI process. Each element of the 
inner list can be a single integer, representing the single column position that needs to be queried or a list of size 2 
representing the column range that needs to be queried.

In the above example, the process will query column range \[0-100\] (inclusive) and the position 500 and return all 
_VariantCalls_ intersecting with these query ranges/positions.

For a more detailed explanation as to why this field is a list of lists (and other fields are lists of strings), we 
refer the reader to the [[wiki page explaining how we use MPI in the context of GenomicsDB|MPI-with-GenomicsDB]].
* _query_attributes_ : List of strings specifying attributes to be fetched

Optional field(s):
* _query_row_ranges_ : Similar to _query_column_ranges_ but for rows. If this field is omitted, all rows are assumed 
to be included in the query.
* _vid_mapping_file_ and _callset_mapping_file_ (type: string or list of strings): Paths to JSON files specifying the [[vid mapping | Importing-VCF-data-into-GenomicsDB#information-about-vcfs-for-the-import-program]] and [[ callset mapping | Importing-VCF-data-into-GenomicsDB#samplescallsets ]]. Note that this feature is available since commit [f92c02e56](https://github.com/Intel-HLS/GenomicsDB/commit/f92c02e5684b5eb33484eee6aba1a07ca640b4ee).

  These two fields are optional because a user might specify them in the loader JSON while creating an array and pass the loader JSON to the query tool(s) (see below). This allows the user to write many different query JSON files without repeating the information.

  If the _vid_mapping_file_ and/or _callset_mapping_file_ parameters are specified in both the loader and query JSON files and passed to the query tool(s), then the parameter value in the query JSON gets precedence.

## Producing _VariantCalls_

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json> --print-calls

The _\<loader.json\>_ file is the 
[[configuration file used to import data into the GenomicsDB array|Importing-VCF-data-into-GenomicsDB]].

Output data is sent to stdout and informational messages are sent to stderr.

The user can specify the _vid_mapping_file_ and _callset_mapping_file_ parameters in the query JSON and drop the loader from the argument list.

    ./bin/gt_mpi_gather -j <query_with_mapping.json> --print-calls

## Producing CSV

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json> --print-csv

The CSV produced is identical to that [[required by the import program|Importing-CSVs-into-GenomicsDB]] - the fields and the order
in which they are printed in the CSV lines are determined by the value of _query_attributes_ in _\<query.json\>_.

The user can specify the _vid_mapping_file_ and _callset_mapping_file_ parameters in the query JSON and drop the loader from the argument list.

    ./bin/gt_mpi_gather -j <query_with_mapping.json> --print-csv

## Producing _Variants_

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json>

The user can specify the _vid_mapping_file_ and _callset_mapping_file_ parameters in the query JSON and drop the loader from the argument list.

    ./bin/gt_mpi_gather -j <query_with_mapping.json>

## Producing combined gVCF
We provide a mechanism to export the data in GenomicsDB into a combined VCF/BCF similar to that produced by the [GATK 
CombineGVCF 
tool](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantutils_CombineGVCFs.php).

When producing a combined VCF, the options described in [[this wiki page|Combined-VCF-options]] can be specified in the query JSON file.

Command:

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json> --produce-Broad-GVCF [-O <output_format>]

Output format can be one of the following strings: "z[0-9]" (compressed VCF),"b[0-9]" (compressed BCF) or "bu" (uncompressed BCF).
If nothing is specified, the default is uncompressed VCF.

The user can specify the _vid_mapping_file_ and _callset_mapping_file_ parameters in the query JSON and drop the loader from the argument list.

    ./bin/gt_mpi_gather -j <query_with_mapping.json> --produce-Broad-GVCF [-O <output_format>]


## Using MPI for parallel querying
Please take a look at the [[wiki page explaining how we use MPI in the context of GenomicsDB|MPI-with-GenomicsDB]] 
first. Once you setup your JSON configuration files and MPI hostfiles correctly:
    
    mpirun -n <num_processes> -hostfile <hostfile> ./bin/gt_mpi_gather -j <query.json> -l <loader.json> [<other_args>]

To produce a combined GVCF with MPI your TileDB array partitions must be partitioned by column. Each MPI 
process will produce a separate combined VCF file corresponding to the query column range assigned to it.

## Java/JNI interface
The Java interface of GenomicsDB implements the [FeatureReader](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/tribble/FeatureReader.html) 
interface in htsjdk. The interface provides an iterator over [VariantContext](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/variantcontext/VariantContext.html) objects for the queried interval(s). Each VariantContext object is the combined VCF record for a given location over all samples. The combine operation is performed by the GenomicsDB native library and the results are obtained by the Java interface through JNI calls.

Take a look at our [example code](https://github.com/Intel-HLS/GenomicsDB/tree/master/example/java/test_genomicsdb_jar)
for using this interface.

Since the Java interface of GenomicsDB exports combined VCF records (VariantContext objects), the options described in [[this wiki page|Combined-VCF-options]] can be specified in the query JSON file.

## Using Spark(TM) for multi-node querying
In addition to MPI, GenomicsDB includes a Spark interface for parallel queries. Spark is a generalized in-memory distributed map-reduce runtime developed by University of California, Berkeley. Various distributions of Spark are available from Apache, Cloudera(R), Hortonworks(R) and Databricks. The idea is that users will be able to run libraries such as Spark SQL, MLLib, Spark.ML or other Spark-based genomics tools such as GATK (4.0) Hellbender to analyze variants from GenomicsDB. For more detailed information on Spark please visit the [Apache Spark homepage](spark.apache.org).

The use of Spark interface mandates that variants be first loaded to GenomicsDB using the MPI-based loader. The variants are partitioned or sharded across multiple machines either by samples or genomic positions in a shared-nothing manner. This means that individual partitions are agnostic of the data residing in other partitions. Resilient Distributed Datasets or RDDs are the unit of data in memory in Spark. RDDs can be viewed as partitioned collection of objects. The objective of this interface is to provide RDDs of multi-sample variant contexts. In the first version, a RDD parititon in a machine contains data from the local GenomicsDB instance. Therefore, if GenomicsDB was partitioned by columns or genomics positions across multiple nodes, RDD partition will contain data by positions.

Please take a look at our example codes in [Java](https://github.com/Intel-HLS/GenomicsDB/blob/master/src/main/java/com/intel/genomicsdb/GenomicsDBJavaSparkFactory.java) or [Scala](https://github.com/Intel-HLS/GenomicsDB/blob/master/src/main/scala/com/intel/genomicsdb/GenomicsDBScalaSparkFactory.scala). The interface provides two APIs - 1) GenomicsDBRDD.getVariantContexts() where a new type of RDDs containing variant contexts (derived from Spark RDD base class) are returned and 2) sparkContext.newAPIHadoopRDD() which also returns RDD objects of variant contexts, but lets the user continue to use standard HadoopRDD interfaces from Spark.

To run the interface, first start a Spark instance. For example a simple local instance of Apache Spark can be started using:

    $ $SPARK_HOME/sbin/start-all.sh
    
Then, run the provided submit script as:

    $ ./bin/genomicsdb-spark-submit.sh spark://localhost:7077 /path/to/loaderjson /path/to/queryjson /path/to/hostfile
    
The loader and query JSON files are explained in detail in [Querying GenomicsDB](https://github.com/Intel-HLS/GenomicsDB/wiki/Querying-GenomicsDB) page. The hostfile is a text file containing list of hostnames or IPs of host machines containing GenomicsDB partitions. For example, a simple hostfile for local instance looks like:

    $ more hostfile
    localhost
    $
