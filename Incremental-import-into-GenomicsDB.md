See [[the terminology page|Basic-TileDB-GenomicsDB-terminology]] for definitions of bulk import and incremental import.

The _vcf2tiledb_ program supports incremental imports by appending new rows to an existing TileDB array. By default, the 
program will import all the callsets specified in the 
_[[callset_mapping_file|Importing-VCF-data-into-GenomicsDB#samplescallSets]]_, appending rows to an existing TileDB 
array if needed.

If the _callset_mapping_file_ contains samples/CallSets corresponding to existing rows in the array (same row index 
values), then the newly supplied data will be assumed to be the latest version for those samples/CallSets. Note that 
this does not imply that the old data for the sample/CallSet is completely deleted. For example, if row 0 had data at 
column 5 and _vcf2tiledb_ is invoked again for row 0 with data at column 6, the updated array will contain data for both 
columns 5 and 6 for row 0.

We provide some parameters for convenience - these parameter are not mandatory. Users may prefer to keep one single 
_callset_mapping_file_ for a given array at all times and append new samples/CallSets to this file. Hence, the 
_vcf2tiledb_ import program must be notified that only the new samples/CallSets should be imported from this file - this 
is achieved by the following two parameters that must be added to the 
_[[loader_config_file|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]]_:
* _lb_callset_row_idx_ (optional, type:int64, default: 0): If specified in the loader configuration file, then the 
import program will only import samples/CallSets with row index \>= _lb_callset_row_idx_. For example, assuming an array 
already has 100 rows (row indexes: 0-99), the user can import additional samples by appending sample/CallSet information 
to the _callset_mapping_file_ (from row index 100) and specify in the loader configuration that only samples with row 
index \>= 100 should be imported.
* _ub_callset_row_idx_ (optional, type:int64, default: INT64_MAX): The upper bound on the row index for 
samples/CallSets that should be imported. Provided for completeness.
