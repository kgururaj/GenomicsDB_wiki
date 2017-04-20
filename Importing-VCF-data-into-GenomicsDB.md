The import program can handle block compressed and indexed VCFs, gVCFs, BCFs and gBCFs. For brevity, we will only use the term VCF.

## Preliminaries
* You need htslib while [[compiling|Compiling-GenomicsDB]].

## Organizing your data
* All your VCFs must be block compressed and indexed. [Bcftools](https://github.com/samtools/bcftools) is one good option for compressing and indexing.
* The VCF format allows you to have multiple lines with the same position (identical chromosome+pos). The import program 
can *NOT* handle such VCFs. Make sure you use tools like [bcftools](https://github.com/samtools/bcftools) to collapse 
multiple lines with the same position into a single line. Example commands are [[provided here|Useful-external-tools#collapse-multiple-lines-with-the-same-genomic-position-into-a-single-line]].
* In a multi-node environment, you must decide:
    * How to [[partition your data in TileDB|GenomicsDB-setup-in-a-multi-node-cluster]].
    * How your VCF files are accessed by the import program:
        * On a shared filesystem (NFS, Lustre etc) accessible from all nodes.
        * If you are partitioning data by rows, your files can be scattered across local filesystems on multiple machines; each filesystem accessible only by the node on which it is mounted. Currently, such a setup isn't supported for column partitioning.

## Information about VCFs for the import program
There are 3 pieces of information that the import program needs:
* Mapping contigs to disjoint column intervals: By assigning disjoint column intervals to each contig in the VCF, we are creating a "flattened" genomic co-ordinate space. Each position in a contig will correspond to a particular column value in TileDB. TileDB column ids are 0-based. Reminder: the position information displayed in a VCF (but not a BCF) is 1-based.
* Mapping row id to samples/CallSets: Each sample/CallSet name must be assigned a unique row id in a TileDB array
* Fields information: The user must pass information about the fields that are required to be imported into TileDB from the VCF.

The three pieces of information listed above are passed to the import tool through a JSON file which we refer to as the **_vid_mapping_file_**. A sample JSON file is shown:

    {
        "contigs": {
            "1": {
                "length": 249250621, 
                "tiledb_column_offset": 0
            }, 
            "2": {
                "length": 243199373, 
                "tiledb_column_offset": 249250621
            }
        },
        "callset_mapping_file" : "/home/user/callsets.json",
        "fields" : {
            "END":{ "vcf_field_class":["INFO"], "type":"int" },
            "BaseQRankSum":{ "vcf_field_class" : ["INFO"], "type":"float" },
            "DP": { "vcf_field_class":["INFO","FORMAT"], "type":"int" },
            "SB":{ "vcf_field_class" : ["FORMAT"], "type":"int", "length":4 },
            "GT": { "vcf_field_class":["FORMAT"], "type":"int", "length":"P" },
            "AD": { "vcf_field_class":["FORMAT"], "type":"int", "length":"R" },
            "PL": { "vcf_field_class":["FORMAT"], "type":"int", "length":"G" }
        }
    }

### Contigs
* The "_contigs_" field is a dictionary of _\<contig_name\> : \<contig_info\>_ entries.
    * It's the user's responsibility to ensure that each contig/chromosome is identified by a unique name in the JSON file.
* _\<contig_info\>_ is a dictionary with the following mandatory elements:
    * _length_ (integer): Specifies the length of the contig
    * _tiledb_column_offset_ (integer): Specifies the column id at which the current contig begins in the TileDB array. In the above example, contig "1" spans columns [0:249250620] and contig "2" begins at column 249250621. It's the user's responsibility to ensure that no two contigs have overlapping column intervals.

### Fields information
* The "_fields_" field is a dictionary of _\<field_name\> : \<field_info\>_ entries.
* _\<field_info\>_ is a dictionary with the following elements
    * Mandatory elements
        * _type_ (string): Supported values are "int", "float", "char", "flag"
        * NOTE: the "flag" type was added in [pull request 82](https://github.com/Intel-HLS/GenomicsDB/issues/82)
    * Optional elements
        * _vcf_field_class_ (list of strings, default empty list):  Specifies whether the field is a "FILTER", "INFO" or "FORMAT" field in the VCF terminology. Note that a field in a VCF can belong to multiple classes. See the field "DP" in the above example JSON - it's both an "INFO" and a "FORMAT" field.
        * _length_ (string or integer, default 1): If the value of this field is an integer, the field is assumed to be a fixed length field. In the above example, "SB" is a fixed length field of 4 integers. The default value of 1 indicates that the field contains a single entry. If the value of the field is a string, then the length of the field is variable. Supported string values and their implications on the field length are listed below:
            * "A": Number of alternate alleles
            * "R": Number of alleles (including reference allele)
            * "G": Number of possible genotypes
            * "P": Ploidy
            * "VAR": The variable length field is terminated by a vector end entry.
            
            **NOTE:** It is important to specify the _length_ field correctly as the decision about which fields need re-ordering based on allele order during query time are determined by the _length_ field. For example, the field "PL" needs to be re-ordered if the allele order changes. The program knows about this because the _length_ field is set to "G".
        * _VCF_field_combine_operation_ (string, default _unknown_ for most fields): This field is useful when producing [[combined VCF records|Querying-GenomicsDB#producing-combined-gvcf]] - it allows the user to specify how INFO and QUAL field values should be produced in the combined VCF. Valid options for this field are:
            * "sum": Sum over valid input values
            * "mean"
            * "median"
            * "element_wise_sum": Valid for vector fields
            * "concatenate": Valid for variable length vector fields - fields whose _length_ is set to "_VAR_" (described in the sub-section above).
            * "move_to_FORMAT": Converts the field to a FORMAT field and copies data for each sample/CallSet into the FORMAT section of the VCF.

            By default, GenomicsDB performs the following operations for certain VCF fields to match the output produced by the [GATK CombineGVCF tool](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantutils_CombineGVCFs.php).
            * INFO fields:
                * BaseQRankSum, ClippingRankSum, MQRankSum, ReadPosRankSum, MQ, MQ0, ExcessHet: median
                * RAW_MQ: sum
            * QUAL field: set to missing

            An INFO field will be imported into the TileDB/GenomicsDB array irrespective of whether the combine operation is specified (in the JSON or in the source code) or not. However, if an INFO field needs to be produced in the [[ combined VCF | Querying-GenomicsDB#producing-combined-gvcf]] or the [[ combined VariantContext objects in the Java interface | Querying-GenomicsDB#javajni-interface ]], it needs to have a valid combine operation specified (either in the vid JSON or in the source code). If the combine operation is not specified, the INFO field will not be included in the combined VCF and VariantContext records. A warning message is posted for each INFO field that will be omitted.

### Samples/CallSets
The value of the field "_callset_mapping_file_" is a string containing the path to a JSON file that maps sample/CallSet names to row ids in the TileDB array. By separating the sample/CallSet mappings into a different JSON file, users can create many different callset mapping files and use them with a single contig mapping and fields information JSON (shown above).

An example callset mapping file is shown below:

    {
        "callsets" : { 
            "HG00141" : {
                "row_idx" : 0,
                "idx_in_file": 0,
                "filename": "test_outputs/merged_java_alt3.list.vcf.gz"
            },
            "HG01530" : {
                "row_idx" : 1,
                "idx_in_file": 1,
                "filename": "test_outputs/merged_java_alt3.list.vcf.gz"
            },
            "HG01958" : {
                "row_idx" : 2,
                "idx_in_file": 2,
                "filename": "test_outputs/merged_java_alt3.list.vcf.gz"
            }
        }
    }

* _\<callsets\>_ is a dictionary of _\<sample/callset name> : \<callset_info\>_ entries
    * It's the user's responsibility to ensure that each sample/CallSet is identified by a unique name in the JSON file.
* _\<callset_info\>_ is a dictionary containing the following entries:
    * Mandatory entries
        * _filename_ (string): path to the VCF file
        * _row_idx_ (integer): row id in TileDB corresponding to this sample/CallSet. It's the user's responsibility to ensure that no two samples/CallSets have the same row id.
    * Optional entries
        * _idx_in_file_ (integer, default 0): VCF files can contain multiple samples. This parameter specifies the sample idx (0-based) of the current sample in the VCF file's header. For the above example JSON, the header line corresponding to samples in the VCF file is shown:

                #CHROM  POS     ID      REF     ALT     QUAL    FILTER  INFO    FORMAT  HG00141 HG01530 HG01958

            Hence, the parameter _idx_in_file_ takes values 0, 1 and 2 for samples HG00141, HG01530 and HG01958 respectively.
            
            The default value is 0 because the program assumes that the VCF has a single sample/CallSet. Again, it's the user's responsibility to ensure this parameter is correct.

## Execution parameters for the import program
The import program needs additional parameters that control how the program runs and are independent of the input VCF files. Such parameters are passed through another JSON file which we refer to as the **_loader_config_file_**. An example loader JSON file is shown below:

    {
        "row_based_partitioning" : false,
        "produce_combined_vcf": true,
        "produce_tiledb_array" : true,
        "column_partitions" : [
            {"begin": 0, "workspace":"/tmp/ws", "array": "test0", "vcf_output_filename":"/tmp/test0.vcf.gz" },
            {"begin": 1000, "workspace":"/tmp/ws", "array": "test1", "vcf_output_filename":"/tmp/test1.vcf.gz" }
        ],
        "vid_mapping_file" : "test_inputs/broad_vid.json",
        "callset_mapping_file" : "test_inputs/callsets/t6_7_8.json",
        "max_num_rows_in_array" : 100,
        "size_per_column_partition": 3000,
        "treat_deletions_as_intervals" : true,
        "num_parallel_vcf_files" : 1,
        "delete_and_create_tiledb_array" : false,
        "fail_if_updating": true,
        "vcf_header_filename": "test_inputs/template_vcf_header.vcf",
        "vcf_output_format": "z",
        "reference_genome" : "/data/broad/samples/joint_variant_calling/broad_reference/Homo_sapiens_assembly19.fasta",
        "do_ping_pong_buffering" : true,
        "offload_vcf_output_processing" : true,
        "discard_vcf_index": true,
        "compress_tiledb_array" : true,
        "disable_synced_writes" : true,
        "segment_size" : 1048576,
        "num_cells_per_tile" : 1000,
        "ignore_cells_not_in_partition": false
    }

* General parameters
  * _row_based_partitioning_ (optional, type: boolean, default value: _false_): Controls how your variant data is partitioned across multiple nodes/TileDB instances - see [[this wiki page for more information first|GenomicsDB-setup-in-a-multi-node-cluster]]. The default value is _false_, which implies your data is partitioned by columns.
  * _produce_combined_vcf_ (optional, type: boolean, default value: _false_): The import program can produced a combined VCF identical to that produced by the [GATK CombineGVCF tool](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantutils_CombineGVCFs.php) if this parameter is set to _true_. Note that if you partition your array by rows, then you cannot produce a combined GVCF - it works only when the array is partitioned by columns.
  * _produce_tiledb_array_ (optional, type: boolean,, default value _false_): The import program will produce the TileDB array if this parameter is set to _true_.
  * _column_partitions_ (mandatory if _row_based_partitioning_=_false_, else optional): This field is a list/array of dictionaries. Each dictionary describes the column partition.
      * _begin_ (mandatory, type: integer): Describes the column id at which the current partition begins
      * _end_ (optional, type: integer): Describes the column id at which the current partition ends (inclusive)
      * _workspace, array_ (optional, type: strings): If creating a TileDB array, these parameters [[specify the directories where the TileDB array data will be stored|Basic-TileDB-GenomicsDB-terminology]].
      * _vcf_output_filename_ (optional, type:string): If producing a combined GVCF, then this parameter specifies the path at which the output VCF will be created. If this parameter is omitted but _produce_combined_vcf_ is _true_, then the output VCF is printed on stdout.

    The program sorts column partitions (in increasing order) and determines the end values (if not specified). For the last partition, if the end is not specified, then it's assumed to be INT64_MAX. In the above example, the column partitions are [0:999] and [1000:INT64_MAX] respectively.
  * _row_partitions_ (mandatory if _row_based_partitioning_=_true_, else optional): This field is similar to _column_partitions_ but for rows. The field _vcf_output_filename_ is not meaningful for row partitioned arrays.   Example:

            "row_partitions" : [
                {"begin": 0, "workspace":"/tmp/ws", "array": "test0" },
                {"begin": 200, "workspace":"/tmp/ws", "array": "test1" }
            ],

    ***YOU SHOULD NOT HAVE _row_partitions_ and _column_partitions_ SIMULTANEOUSLY IN ONE LOADER JSON CONFIGURATION FILE.***
  * _vid_mapping_file_ (mandatory, type:string): The vid mapping file [[described above|Importing-VCF-data-into-GenomicsDB#information-about-vcfs-for-the-import-program]].
  * _callset_mapping_file_ (optional, type:string): Same as the callset mapping file [[described above|Importing-VCF-data-into-GenomicsDB#samplescallsets]]. This field value gets higher preference if the _vid_mapping_file_ also contains a field called _callset_mapping_file_. The idea is that the user can use a fixed _vid_mapping_file_ and work with many different sets of samples/CallSets by changing the _callset_mapping_file_ in this loader JSON config file.
  * _max_num_rows_in_array_ (optional, type: int64, default: INT64_MAX): TileDB requires that the dimensions of the array 
be specified while creating the array. This parameter specifies the maximum number of rows that are to be stored in the 
TileDB array. Setting this parameter is optional.
  * _size_per_column_partition_ (mandatory, type: int): During the importing process, the program reads small chunks from 
each input VCF file into a buffer in memory. Let's denote this chunk size as _X_. Note that _X_ must be large enough to 
hold at least 1 VCF line;i.e. _X_ must be bigger than the largest VCF line among all the input VCF files.

      To produce a column major array, the program allocates one buffer for every input sample/CallSet. Hence, the total size of all buffers put together is _\#samples * X_. The parameter _size_per_column_partition_ must be set to this value.

      Unfortunately, there is no good way to figure out what this parameter value should be. In our tests with the WGS gVCFs accessible to us, a value of 10KB for _X_ seemed to be sufficient (but a value of 1KB wasn't adequate). The program will fail with an exception if it encounters a line whose length is \> _X_; however, this could be deep into the program's execution. Setting large values would increase memory consumption, but if your sample count is 'small' enough (and your machine memory large enough), setting larger values should not be an issue.
  * _treat_deletions_as_intervals_ (type: boolean, optional, default: _false_): Consider the following lines in a VCF:

            #CHR	POS	REF	ALT
            1	100	GATC	GC
            1	500	T	C

    In an indexed VCF, querying for position 101 would return the deletion allele since it overlaps the queried position (even though, by the VCF convention, it begins at location 100). Hence, we can think of a deletion as an interval and any query within that interval should return the deletion. By setting this flag to _true_, all deletions are treated as intervals. Without this flag, a GenomicsDB query for position 101 would *NOT* return the deletion.
  * _num_parallel_vcf_files_ (type:integer, optional, default: 1): This parameter controls the number of VCF files that are opened and read in parallel by the loader program. Increasing this number could improve (decrease) loading time.
  * _delete_and_create_tiledb_array_ (type: boolean, optional, default: _false_): If set to _true_, the program will delete existing data in the array referred to by the _workspace_ and _array_ fields. By default, the loader program will only update/add to the existing array and not delete previous data.
  * _fail_if_updating_ (type: boolean, optional, default: _false_): If set to _true_, the program will fail with an exception if you are trying to update an existing array - creating a fresh array will succeed. This option is useful if you are not sure if your mapping information for samples/CallSets and contigs is consistent with respect to an existing array.

* Parameters for producing a combined VCF file during the loading phase
  
  When producing a combined VCF, the options described in [[this wiki page|Combined-VCF-options]] can be specified in the loader JSON file.

* Parameters relevant when using the write API (for example the Java write API):
  * _ignore_cells_not_in_partition_ (type: boolean, optional, default: _false_): When using the write API (for example, the [[ Java write API | Java-interface-for-importing-VCF-CSV-files-into-TileDB-GenomicsDB#mode-2-java-api-for-importing-variantcontext-objects ]]) for importing data into a [[ multi-partition TileDB/GenomicsDB array  | GenomicsDB-setup-in-a-multi-node-cluster ]], if records NOT belonging to the current partition are encountered, an exception will be thrown. Setting this parameter to _true_ will cause the program to silently ignore such records.

    If you are not using the write API, this parameter is not useful.

* Parameters for developers/tuners.
  * _do_ping_pong_buffering_ (type: boolean, optional, default: _false_): Enabling this option runs the multiple stages in the loader in parallel using [OpenMP's sections directive](https://computing.llnl.gov/tutorials/openMP/#SECTIONS) (software pipelining).
  * _offload_vcf_output_processing_ (type: boolean, optional, default: _false_): When producing a combined gVCF, enabling this option offloads the processing associated with serializing the VCF record into  a character buffer, compression and writing to disk to another thread. This reduces the burden on the critical thread in the combine GVCF process.
  * _discard_vcf_index_ (type: boolean, optional, default: _true_): The loader program traverses each VCF in column order. 
This is done by sorting contigs in increasing order of their offsets (as described in the _vid_mapping_file_) and then 
using the indexed VCF reader from [htslib](https://github.com/samtools/htslib) to traverse the VCF in the sorted contig 
order. The program by default uses the input VCF's index only when switching from one contig to another since records 
belonging to a single contig are contiguous and in non-decreasing order in the VCF. When a new contig is seen, the index 
is re-loaded into memory (from disk) and the program moves to the next contig (in the sorted contig order). Once the 
file pointer moves to the next contig, the index structures are dropped from memory.

      Indexes are not stored in memory for the duration of the program to ensure that the loader program is tractable when 
dealing with a large number of inputs. In our tests, for WES gVCFs, each index structure consumed about 6 MB of memory. 
For WGS gVCFs, each index consumed around 40 MB of memory. This becomes an issue when dealing with \>= 1000 files.
  * _compress_tiledb_array_ (type: boolean, optional, default: _true_): Determines whether the files storing TileDB data 
on disk are compressed.
  * _tiledb_compression_level_ (type: integer, optional, default: 6): Zlib compression level to use.
  * _disable_synced_writes_ (type: boolean, optional, default: _false_): Determines whether TileDB uses the O_SYNC flag while writing to disk. Disabling synced writes likely improves performance. The performance improvement is significant when the array is compressed since tiles are written one at a time to disk when compression is enabled. Compressed tiles are relatively small in size (few KBs) and using synced writes slows down the loading process significantly.

      However, bear in mind that disabling synced writes implies that data may not committed to disk till after the end of the import program (the kernel may decide to buffer pages in memory). If you are certain that no concurrent reads will occur during or immediately after the import process, disabling synced writes is likely to give you a performance boost without affecting correctness. Use with care.
  * _segment_size_ (type: int64_t, optional, default: 10MB): Buffer size in bytes allocated for TileDB attributes during 
the loading process. Should be large enough to hold one cell worth of data. Small buffer sizes (< 4KB) may lead to low
performance since data is flushed to disk each time the buffer is full.
  * _num_cells_per_tile_ (type: int64_t, optional, default: 1000): Controls number of cells per tile. Very small values 
can lead to low performance during loading and querying (since a tile is a unit of indexing in TileDB)

## Running the program
* If you have a single partition (in _column_partitions_ or _row_partitions_) in your loader JSON configuration file, 
simply run:

        ./bin/vcf2tiledb <loader_json>

* If you have multiple partitions, you can use MPI to create the partitions in parallel. For more information on how to 
use MPI in the context of GenomicsDB, see [[this page|MPI-with-GenomicsDB]].

        mpirun -n <num_partitions> [-hostfile <hostfile>] ./bin/vcf2tiledb <loader_json>

* If you do not wish to use MPI, you can specify the rank (partition index) on the command line. For example:

        ssh host0 "vcf2tiledb -r 0 <loader.json>"
        ssh host1 "vcf2tiledb -r 1 <loader.json>"
        ....

* Note that if you do not use MPI and you do not specify the rank on the command line, only the first partition's data is loaded. This is equivalent to running _vcf2tiledb_ with rank 0.
