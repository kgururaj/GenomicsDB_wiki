Read [this page](https://github.com/Intel-HSS/TileDB/wiki/Using-the-variant-specific-customizations) for instructions on how to compile and run the single node version of the variant library first.

## Requirements
* You need htslib to output VCF/BCF records
* You need the latest version of bcftools to read information about samples and contigs from SQLite. Install bcftools and htslib as described in [this Wiki page](https://github.com/kgururaj/bcftools/wiki/Using-bcftools-for-TileDB)
* You need to have the SQLite tables populated correctly with sample, contig and fields mapping.
* An MPI library
* [Rapidjson library](https://github.com/miloyip/rapidjson): I use this library to pass the query configuration in a json file to TileDB. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

## Compiling

Get the right branch of the TileDB repo.

    git checkout variant_master_merge
    #make sure you have the right gcc version in your PATH
    #release mode - O3, NDEBUG - assertions disabled
    make MPIPATH=/opt/mvapich2-2.2a/bin/ BCFTOOLSDIR=<bcftools_dir> HTSDIR=<htslib_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include/ BUILD=release -j 8
    #debug mode - assertions enabled, can use gdb for stepping
    make MPIPATH=/opt/mvapich2-2.2a/bin/ BCFTOOLSDIR=<bcftools_dir> HTSDIR=<htslib_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include/ BUILD=debug -j 8
    
Other compilation options include DO_PROFILING=1, VERBOSE=1

## Running the program
* Currently, the program has been tested on a single node only without MPI.
* Configurable queries: Query information is passed to the program using a JSON file. 
         
        ./variant/example/bin/gt_mpi_gather -j <json_file> --produce-Broad-GVCF -O <output_format>

  Output format can be one of the following strings: "z" (compressed VCF),"b" (compressed BCF) or "bu" (uncompressed BCF). If nothing is specified, the default is uncompressed VCF.

        {
          "workspace" :  [ "../configs/ws", "/mnt/app_hdd/scratch/karthikg/VCFs/tiledb_csv/v1/arrays/" ],
          "array" : [ "t0_1_2_GT", "GT10" ],
          "query_column_ranges" : [ [ [12000, 13000 ]  ], [ [0, 10000000] ] ],
          "query_row_ranges" : [ [ [100, 3000 ]  ], [ [4000, 10000] ] ],
          "query_attributes" : [ "REF", "ALT", "BaseQRankSum", "MQ", "MQ0", "ClippingRankSum", "MQRankSum", "ReadPosRankSum", "DP", "GT", "GQ", "SB", "AD", "PL", "DP_FORMAT", "MIN_DP" ],
          "sqlite" : "/mnt/app_hdd/scratch/karthikg/VCFs/samples_and_fields.sqlite",
          "vcf_header_filename" : "test_inputs/template_vcf_header.vcf",
          "vcf_output_filename" : "/home/output.vcf",
          "reference_genome" : "/data/broad/samples/joint_variant_calling/broad_reference/Homo_sapiens_assembly19.fasta"
        }

  * "workspace" (mandatory): List of strings, specifying the locations of the workspace directory for each MPI process launched. The length of this list MUST be equal to the number of MPI processes launched. Alternately, this parameter can be a string (not a list), which implies that all MPI processes access the workspace at the same directory location.
  * "array" (mandatory) : Similar to the workspace parameter. Note that the arrays MUST have exactly identical schemas.
  * "query_column_ranges" (mandatory): Each MPI process can query a list of column ranges. For example, the list \[ \[ 0, 100 \], 500 \] specifies that the MPI process should query column interval \[0-100\] and the single position 500. The parameter "query_column_ranges" is a list of such lists. The length of the outer list MUST be EITHER equal to the number of MPI processes launched OR just 1 (implying that all MPI processes query the same column ranges).
  * "query_row_ranges" (optional): Same format as "query_column_ranges", but for rows. Can be omitted, in which case all rows of the array will be queried.
  * "query_attributes" (mandatory): List of strings specifying attributes to be fetched. Note for producing the GVCF as required by Broad, the attributes listed above MUST be used.
  * "sqlite" (mandatory) : Path to SQLite file containing contig, sample name and fields mapping. Can be a single string or a list of strings (same semantics as workspace/array fields).
  * "vcf_header_filename" (mandatory) : Path to template VCF header file - contains only fields and contigs in the header. Can be a single string or a list of strings (same semantics as workspace/array fields). Use this [template header file] (https://github.com/Intel-HSS/TileDB/files/14269/template_vcf_header.txt) for producing GVCFs for Broad.
  * "vcf_output_filename" (optional) : Output VCF/BCF file path. Can be a single string or a list of strings (same semantics as workspace/array fields). If not specified, the program will print the VCF/BCF records to stdout.
  * "reference_genome" (mandatory) : Path to reference genome (fasta file) that was used to obtain the input variants in TileDB. Can be a single string or a list of strings (same semantics as workspace/array fields).

  NOTE: The RapidJSON library doesn't clearly flag syntax errors. I generally run the command "json_verify \< \<json_file\>" to check the syntax first.

