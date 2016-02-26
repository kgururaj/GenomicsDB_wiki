The import program can handle block compressed and indexed VCFs, gVCFs, BCFs and gBCFs. For brevity, we will only use the term VCF.

## Preliminaries
* Use the branch _multi_node_loader_.
* You need htslib while [[compiling|Compiling-VariantDB]].

## Organizing your data
* All your VCFs must be block compressed and indexed. [Bcftools](https://github.com/samtools/bcftools) is one good option for compressing and indexing.
* In a multi-node environment, you must decide:
    * How to [[partition your data in TileDB|VariantDB-setup-in-a-multi-node-cluster]].
    * How your VCF files are accessed by the import program:
        * On a shared filesystem (NFS, Lustre etc) accessible from all nodes.
        * If you are partitioning data by rows, your files can be scattered across local filesystems on multiple machines; each filesystem accessible only by the node on which it is mounted. Currently, such a distribution isn't supported for column partitioning.

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
        * _type_ (string): Supported values are "int", "float", "char"
    * Optional elements
        * _vcf_field_class_ (list of strings, default empty list):  Specifies whether the field is a "FILTER", "INFO" or "FORMAT" field in the VCF terminology. Note that a field in a VCF can belong to multiple classes. See the field "DP" in the above example JSON - it's both an "INFO" and a "FORMAT" field.
        * _length_ (string or integer, default 1): If the value of this field is an integer, the field is assumed to be a fixed length field. In the above example, "SB" is a fixed length field of 4 integers. The default value of 1 indicates that the field contains a single entry. If the value of the field is a string, then the length of the field is variable. Supported string values and their implications on the field length are listed below:
            * "A": Number of alternate alleles
            * "R": Number of alleles (including reference allele)
            * "G": Number of possible genotypes
            * "P": Ploidy
            * "VAR": The variable length field is terminated by a vector end entry.
            
            **NOTE:** It is important to specify the _length_ field correctly as the decision about which fields need re-ordering based on allele order during query time are determined by the _length_ field. For example, the field "PL" needs to be re-ordered if the allele order changes. The program knows about this because the _length_ field is set to "G".
 
### Samples/CallSets
The value of the field "_callset_mapping_file_" is a string containing the path to a JSON file that maps sample/CallSet names to row ids in the TileDB array. By separating the sample/CallSet mappings into a different JSON file, users can create many different callset mapping files and use them with a single contig mapping and fields information JSON (shown above).

An example callset mapping file is shown below:

    {
        "callsets" : { 
            "HG00141" : {
                "row_idx" : 0,
                "idx_in_file": 0,
                "filename": "/home/karthikg/broad/non_variant_db/bcftools/test_outputs/merged_java_alt3.list.vcf.gz"
            },
            "HG01530" : {
                "row_idx" : 1,
                "idx_in_file": 1,
                "filename": "/home/karthikg/broad/non_variant_db/bcftools/test_outputs/merged_java_alt3.list.vcf.gz"
            },
            "HG01958" : {
                "row_idx" : 2,
                "idx_in_file": 2,
                "filename": "/home/karthikg/broad/non_variant_db/bcftools/test_outputs/merged_java_alt3.list.vcf.gz"
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
            {"begin": 0, "workspace":"/tmp/ws", "array": "test0", "vcf_output_filename":"/tmp/test0.vcf.gz" }
            {"begin": 1000, "workspace":"/tmp/ws", "array": "test1", "vcf_output_filename":"/tmp/test1.vcf.gz" }
        ],
        "row_partitions" : [
            {"begin": 0, "workspace":"/tmp/ws", "array": "test0" },
            {"begin": 200, "workspace":"/tmp/ws", "array": "test1" }
        ],
        "vid_mapping_file" : "test_inputs/broad_vid.json",
        "callset_mapping_file" : "test_inputs/callsets/t6_7_8.json",
        "size_per_column_partition": 3000,
        "treat_deletions_as_intervals" : true,
        "vcf_header_filename": "test_inputs/template_vcf_header.vcf",
        "reference_genome" : "/data/broad/samples/joint_variant_calling/broad_reference/Homo_sapiens_assembly19.fasta",
        "num_parallel_vcf_files" : 1,
        "do_ping_pong_buffering" : true,
        "offload_vcf_output_processing" : true,
        "discard_vcf_index": true,
        "delete_and_create_tiledb_array" : true
    }

* _row_based_partitioning_ (optional, type: boolean, default value: _false_): Controls how your variant data is partitioned across multiple nodes/TileDB instances - see [[this wiki page for more information first|VariantDB-setup-in-a-multi-node-cluster]]. The default value is _false_, which implies your data is partitioned by columns.
* _produce_combined_vcf_ (optional, type: boolean, default value: _false_): The import program can produced a combined VCF identical to that produced by the [GATK CombineGVCF tool](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantutils_CombineGVCFs.php) if this parameter is set to _true_. Note that if you partition your array by rows, then you cannot produce a combined GVCF - it works only when the array is partitioned by columns.
* _produce_tiledb_array_ (optional, type: boolean,, default value _false_): The import program will produce the TileDB array if this parameter is set to _true_.
* _column_partitions_ (mandatory if _row_based_partitioning_=_false_, else optional): This field is a list/array of dictionaries. Each dictionary describes the column partition.
    * _begin_ (mandatory, type: integer): Describes the column id at which the current partition begins
    * _end_ (optional, type: integer): Describes the column id at which the current partition ends (inclusive)
    * _workspace, array_ (optional, type: strings): If creating a TileDB array, these parameters [[specify the directories where the TileDB array data will be stored|Basic-TileDB-terminology]].
    * _vcf_output_filename_ (optional, type:string): If producing a combined GVCF, then this parameter specifies the path at which the output VCF will be created. If this parameter is omitted but _produce_combined_vcf_ is _true_, then the output VCF is printed on stdout.

    The program sorts column partitions (in increasing order) and determines the end values (if not specified). For the last partition, if the end is not specified, then it's assumed to be INT64_MAX. In the above example, the column partitions are [0:999] and [1000:INT64_MAX] respectively.
* _row_partitions_ (mandatory if _row_based_partitioning_=_true_, else optional): This field is similar to _column_partitions_ but for rows. The field _vcf_output_filename_ is not meaningful for row partitioned arrays.
* _vid_mapping_file_ (mandatory, type:string): The vid mapping file [[described above|Importing-VCF-data-into-TileDB#information-about-vcfs-for-the-import-program]].
* _callset_mapping_file_ (optional, type:string): Same as the callset mapping file [[described above|Importing-VCF-data-into-TileDB#samplescallsets]]. This field value gets higher preference if the _vid_mapping_file_ also contains a field called _callset_mapping_file_. The idea is that the user can use a fixed _vid_mapping_file_ and work with many different sets of samples/CallSets by changing the _callset_mapping_file_ in this loader JSON config file.
* _size_per_column_partition_ (mandatory, type: int): The program reads small chunks of each input VCF file into a buffer in memory. Let's denote this chunk size as _X_. Note that _X_ must be large enough to hold at least 1 VCF line;i.e. _X_ must be bigger than the largest VCF line among all the input VCF files.

    To produce a column major array, the program allocates one buffer for every input sample/CallSet. Hence, the total size of all buffers put together is _\#samples * X_. The parameter _size_per_column_partition_ must be set to this value.

    Unfortunately, there is no good way to figure out what this parameter value should be. In our tests with the WGS gVCFs accessible to us, a value of 15KB for _X_ seemed to be sufficient (but a value of 1KB wasn't adequate). The program will fail with an exception if it encounters a line whose length is \> _X_; however, this could be deep into the program's execution. Setting large values would increase memory consumption, but if your sample count is 'small' enough (and your machine memory large enough), setting larger values should not be an issue.
