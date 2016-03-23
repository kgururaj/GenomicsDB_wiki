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
1 All _VariantCalls_ span the exact same column interval
1 All _VariantCalls_ have the same value of REF
1 All _VariantCalls_ have the same value of ALT

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
"A", "<NON_REF>" as the alternate alleles. It consists of two _VariantCalls_ at rows 0 and 2.

## JSON configuration file for a query
A sample JSON query is shown below:

    {
        "workspace" : "/tmp/ws/",
        "array" : "t0_1_2",
        "query_column_ranges" : [ [ [0, 100 ], 500 ] ],
        "query_row_ranges" : [ [ [0, 2 ] ] ],
        "query_attributes" : [ "REF", "ALT", "BaseQRankSum", "MQ", "MQ0", "ClippingRankSum", "MQRankSum", "ReadPosRankSum", "DP", "GT", "GQ", "SB", "AD", "PL", "DP_FORMAT", "MIN_DP" ],
        "vcf_header_filename" : ["test_inputs/template_vcf_header.vcf"],
        "reference_genome" : "/opt/Homo_sapiens_assembly19.fasta"
    }

Most of the fields are self-explanatory (take a look at the [[terminology section|Basic-TileDB-GenomicsDB-terminology]]).
The following fields are mandatory:
* _workspace_ (type: string or list of strings)
* _array_ (type: string or list of strings)
* _query_column_ranges_ : This field contains is a list of lists. Each member list of the outer list is handled by one 
MPI process. Each member list contains the column ranges that are to be queried by the MPI process. Each element of the 
inner list can be a single integer, representing the single column position that needs to be queried or a list of size 2 
representing the column range that needs to be queried.

In the above example, the process will query column range [0, 100] (inclusive) and the position 500 and return all 
_VariantCall_ intersecting with these query ranges/positions.

For a more detailed explanation as to why this field is a list of lists (and other fields are lists of strings), we 
refer the reader to the [[wiki page explaining how we use MPI in the context of GenomicsDB|MPI-with-GenomicsDB]].
* _query_attributes_ : List of strings specifying attributes to be fetched

Optional field(s):
* _query_row_ranges_ : Similar to _query_column_ranges_ but for rows. If this field is omitted, all rows are assumed 
to be included in the query.

## Producing _VariantCalls_

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json> --print-calls

The \<loader.json\> file is the [[configuration file used to import data into the GenomicsDB 
array|Importing-VCF-data-into-GenomicsDB]].

Output data is sent to stdout and informational messages are sent to stderr.

## Producing _Variants_

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json>

## Producing combined gVCF
We provide a mechanism to export the data in GenomicsDB into a combined VCF/BCF similar to that produced by the [GATK 
CombineGVCF 
tool](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantutils_CombineGVCFs.php).  
Additional fields are needed in the query JSON file to produce a combined VCF:

    "vcf_header_filename": "/share/template_vcf_header.vcf",
    "reference_genome" : "/opt/Homo_sapiens_assembly19.fasta",
    "vcf_output_filename" : "/share/out.vcf"

* _vcf_header_filename_ (type:string, mandatory): Path to a template VCF header file. All lines in this template will be 
present in the header of the combined VCF(s). This template should **NOT** contain sample/callset names (i.e. the line
starting with #CHR). Contigs present in the _vid_mapping_filename_ (as described in the loader JSON file) will be added 
to the combined GVCF, if not present in the template header. The template header should contain information about fields 
(INFO, FILTER, FORMAT).
* _reference_genome_ : (type:string, mandatory): Path to reference genome (indexed FASTA file).
* _vcf_output_filename_ (optional, type:string or list of strings): If producing a combined GVCF, then this parameter 
specifies the path at which the output VCF will be created. If this parameter is omitted, then the output VCF is printed 
on stdout.

    ./bin/gt_mpi_gather -j <query.json> -l <loader.json> --produce-Broad-GVCF [-O <output_format>]

Output format can be one of the following strings: "z" (compressed VCF),"b" (compressed BCF) or "bu" (uncompressed BCF). 
If nothing is specified, the default is uncompressed VCF.

##Using MPI for parallel querying
Please take a look at the [[wiki page explaining how we use MPI in the context of GenomicsDB|MPI-with-GenomicsDB]] 
first. Once you setup your JSON configuration files and MPI hostfiles correctly:
    
    mpirun -n <num_processes> -hostfile <hostfile> ./bin/gt_mpi_gather -j <query.json> -l <loader.json> [<other_args>]

* Note that to produce a combined GVCF with MPI your TileDB array partitions must be partitioned by column. Each MPI 
process will produce a separate combined VCF file corresponding to the query column range assigned to it.
