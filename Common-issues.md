* Verify that your JSON configuration files are syntactically correct before invoking GenomicsDB tools

  The GenomicsDB tools assume that you provide a syntactically correct JSON - always validate your JSON files using 
tools such as _json_verify_ before invoking GenomicsDB tools.

        cat <json_file> | json_verify

* You cannot move/copy the TileDB workspace/array directory arbitrarily after loading. If you do copy/move files to a different machine, ensure that the absolute path of the workspace/array directory is the same as that on the machine on which data is loaded/imported.
* I get an exception with the following message:

        Unhandled overlapping variants at columns <col1> and <col2> for row <row>

  TileDB/GenomicsDB cannot deal with overlapping variants within a single sample - [[more details documented here| Overlapping-variant-calls-in-a-sample]]. Workarounds exist for dealing with overlapping deletions and gVCF reference blocks (intervals with \<NON_REF\> as the only alternate allele). When overlapping variants which are neither deletions nor reference blocks are found in the input VCF file, then the above exception is thrown. Check whether bcftools can help you - see point 2 [[in the section on organizing your data|Importing-VCF-data-into-GenomicsDB#organizing-your-data]].
* I have both _row_partitions_ and _column_partitions_ in my loader JSON file and I get an exception message:

        Cannot have both "row_partitions" and "column_partitions" simultaneously in the JSON file

  A TileDB array in the context of GenomicsDB can be partitioned by rows or columns but not both simultaneously - see [[this page|GenomicsDB-setup-in-a-multi-node-cluster]] for more information.
* I have setup all my JSON files correctly, but the import program finishes almost immediately without importing any data from my VCFs:

  There could be many reasons, but here are the common issues we have seen users running into:
  * _Contig/chromosome names don't match in the _vid_mapping_file_ _and the input VCFs_: The contig/chromosome names in the VCF and the vid mapping JSON file MUST match EXACTLY. For example, if the vid file has a contig named _"1"_ (as per the 1000 genomes naming convention) while the VCF has a contig named _"chr1"_ (as per the UCSC convention), GenomicsDB will ignore all data corresponding to _"chr1"_.

* I have setup all my JSON files correctly, but the import program doesn't load data for some of the samples:
  
  There could be many reasons, but here are the common issues we have seen users running into:
  * _Incorrect value(s) of _idx_in_file_ _in the callset_mapping_file_: Note that _row_idx_ is the globally unique value of the TileDB row index corresponding to a given sample/CallSet. _idx_in_file_ is useful mostly for multi-sample VCFs and specifies the index of the sample in a given VCF. For single sample VCFs, this field should be 0 (or omitted altogether).