## Creating a workspace

    ./bin/create_tiledb_workspace <workspace_dir>

The above command will create a workspace if the _\<workspace_directory\>_ does not exist. If _\<workspace_directory\>_  
exists, then the directory is not touched/modified.

We recommend running the above command on each node sequentially _before_ running any MPI processes to import data. This 
ensures that the workspace directory gets created in a consistent manner before any data gets imported into TileDB 
arrays.

## Creating a histogram
Given a list of VCF/CSV files, the following command will output a uniform histogram showing the number of variant calls per 
column interval across all files. Such a histogram can be used to partition data in a balanced manner across TileDB 
array instances (possibly scattered across nodes).

The histogram program takes as input a configuration JSON file similar to that used by
[[the import program|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]]. An example file is shown 
below:

    {   "column_partitions" : [ {"begin": 0 }  ],
        "callset_mapping_file" : "test_inputs/callsets/full3.json",
        "vid_mapping_file" : "test_inputs/broad_vid.json",
        "size_per_column_partition": 3000,
        "num_converter_processes" : 1,
        "max_histogram_range" : 3999999999,
        "num_bins" : 10000,
        "num_parallel_vcf_files" : 1
    }  

Most of the parameters are identical to those used by the import program.
* _column_partitions_: The _begin_ and _end_ fields within each dictionary specify the range of columns over which the 
histogram should be constructed. Any VCF/CSV lines outside the range will be ignored.
* _num_converter_processes_ (mandatory, type: int): Set this field to 1 (or any value \> 0).
* _max_histogram_range_ and _num_bins_ (mandatory, type: int64): These two parameters control the histogram that is 
produced. The first parameter determines the maximum value in the histogram - if there is a column position in a file 
above this value, the program will exit with an exception. The second parameter controls the number of bins in the 
produced histogram - each bin's size will be _max_histogram_range_/_num_bins_.

The _[[callset_mapping_file|Importing-VCF-data-into-GenomicsDB#samplescallsets]]_ needs one extra parameter to work correctly:

    {
        "callsets" : {
            "HG00141" : {
                "row_idx" : 0,
                "idx_in_file": 0,
                "filename": "/home/karthikg/broad/non_variant_db/bcftools/test_inputs/t0.vcf.gz"
            },
            "HG01958" : {
                "row_idx" : 1,
                "idx_in_file": 0,
                "filename": "/home/karthikg/broad/non_variant_db/bcftools/test_inputs/t1.vcf.gz"
            },
            "HG01530" : {
                "row_idx" : 2,
                "idx_in_file": 0,
                "filename": "/home/karthikg/broad/non_variant_db/bcftools/test_inputs/t2.vcf.gz"
            }
        },
        "file_division" : [
            [
                "/home/karthikg/broad/non_variant_db/bcftools/test_inputs/t0.vcf.gz",
                "/home/karthikg/broad/non_variant_db/bcftools/test_inputs/t1.vcf.gz",
                "/home/karthikg/broad/non_variant_db/bcftools/test_inputs/t2.vcf.gz"
                ]
        ]
    }

* _file_division_ (mandatory, type: list of list of strings): This parameter should be a list containing a single entry. 
The single entry should be a list of all the files over which the histogram should be constructed.

Running the tool:

    ./bin/vcf_histogram <histogram_json>

The program's output consists of lines that look like this:

    Histogram: [
    0,399999,104
    400000,799999,42
    800000,1199999,5270
    ]

Each line represents one bin in the histogram - the first bin is the column interval \[0-399999\] (inclusive) and it 
contains 104 calls across all samples.

## Doing a diff between VCFs
To validate the output VCF produced by a program, we use the vcfdiff tool. This tool is able to take into account 
re-ordered samples, re-ordered alleles and non-intersecting reference intervals. Note that the input VCFs must be block 
compressed and indexed.

    ./bin/vcfdiff [-t <tolerance>] [-l <loader.json>] [-p <column_partition_idx>] gold.vcf.gz test.vcf.gz

Without any of the optional parameters, the program compares gold and test and prints lines in which the two differ 
(including missing and extra lines).
* _-t_ (optional, type: float, default:10<sup>-5</sup>): Sets the threshold when comparing floating point values. Two 
floating point values are considered close enough if either the absolute difference or the relative difference between 
them is less than the threshold.
* _-l_ (optional, type: string): Specifies the path to the JSON file used by the
[[the import program|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]].
* _-p_ (optional, type: int): Specifies the index of the column partition in the _\<loader.json\>_ file for which the 
test VCF has been produced.

    A user may have partitioned his/her TileDB array into multiple instances and may produce a combined VCF for each 
instance separately. To compare each instance of the VCF with a golden VCF that may span the whole genome, the user can 
pass the import configuration and the specific column partition index for which the comparison has to be performed. For 
example, if the data is partitioned into two column ranges \[0-1.49B\] and \[1.5B-4B\] during the import process, then 
passing value 0 for the parameter _-p_ in vcfdiff will compare only those lines in the range \[0-1.49B\].
