* Verify that your JSON configuration files are syntactically correct before invoking GenomicsDB tools

    The GenomicsDB tools assume that you provide a syntactically correct JSON - always validate your JSON files using 
tools such as _json_verify_ before invoking GenomicsDB tools.

        cat <json_file> | json_verify

* You cannot move/copy the TileDB workspace/array directory arbitrarily after loading. If you do copy/move files to a different machine, ensure that the absolute path of the workspace/array directory is the same as that on the machine on which data is loaded/imported.
* I get an exception with the following message:

        Unhandled overlapping variants at columns <col1> and <col2> for row <row>

    TileDB/GenomicsDB cannot deal with overlapping variants within a single sample - [[more details documented here| Overlapping-variant-calls-in-a-sample]]. Workarounds exist for dealing with overlapping deletions and gVCF reference blocks (intervals with \<NON_REF\> as the only alternate allele). When overlapping variants which are neither deletions nor reference blocks are found in the input VCF file, then the above exception is throw. Check whether bcftools can help you - see point 2 [[in the section on organizing your data|Importing-VCF-data-into-GenomicsDB#organizing-your-data]].
