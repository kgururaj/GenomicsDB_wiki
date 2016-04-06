**NOTE: The CSV import functionality is still being tested internally - the public repo does not yet have the 
functionality to import CSVs. We will push this functionality shortly.**

## Preliminaries
* You need libcsv while [[compiling|Compiling-GenomicsDB]].

## CSV format
Given a variant call at a specific position in the genome for a particular CallSet/sample, the CSV file format supported 
by GenomicsDB contains one line describing the call. Essentially, each line in the CSV describes one cell in the TileDB 
array.

The exact format of the CSV is shown below:

     <row>,<begin_column>,<end_column>,<REF>,<concatenated_ALT>,<QUAL>,<FILTER_field_specification>[,<other_fields_specification>]

Fixed fields:
* _row_ (mandatory, type: int64): Row number in TileDB for this sample/CallSet
* _begin_column_ (mandatory, type: int64): Column in TileDB at which the variant call begins.
* _end_column_ (mandatory, type: int64): Column in TileDB at which the variant call ends (inclusive).
* _REF_ (mandatory, type:string): Reference allele at the given position.
* _concatenated_ALT_ (mandatory, type: string): Concatenated alternate alleles separated by the character '|'. For
example, if a given call has two alternate alleles _TAG_ and _TG_, then the concatenated string would be _TAG|TG_.
* _QUAL_ (optional, type: float): Represents the _QUAL_ field in a VCF. It can be left empty implying that the value was 
missing for this call.
* _FILTER_field_specification_ (mandatory, type: variable length list of integers): Each element of the FILTER field list 
is an integer representing the FILTERs that this call failed (similar to how a BCF represents FILTERs). The first 
element of this field is an integer displaying the number of elements in the list. A value of 0 indicates that the list 
is empty.

Additional fields can be optionally specified
* _\<other_fields_specification\>_: The format depends on the type of field:
    * String type fields (fixed or variable length strings): The field should contain the string - an empty field 
indicates that the field is missing.
    * Fixed length field (int or float): The field should contain exactly _N_ elements where _N_ is the length of the field 
(fixed constant). One or more elements may be left empty to indicate that those elements are missing.
    * Variable length field (int or float): The first element of this field should be an integer denoting the number 
of elements in the field for this call. It should then be followed by the elements of this field. An empty or missing 
field can be specified by setting the first element (field length) to 0 - no other elements should follow an empty field.

### Example
The following line contains 2 fields in addition to the fixed fields:
* _SB_: Fixed length field of 4 integers
* _PL_: Variable length field of integers

        2,1857210,1857210,G,A|T,894.77,0,,,,,6,923,0,599,996,701,1697

The line specifies the variant call for row id 2, beginning at column 1857210 and ending at 1857210. The _REF_ 
allele is 'G' and the call has 2 alternate alleles 'A' and 'T' (SNVs). The _QUAL_ value is 894.77 and there are no 
_FILTERs_ specified (hence FILTER field length = 0). The _SB_ field is missing - denoted by the 4 empty fields. The _PL_ 
field consists of 6 integers - the length appears first (since _PL_ is a variable length field) followed by the elements 
\[923,0,599,996,701,1697\].

### Special fields
* _GT_ is represented in the CSV as a variable length list of integers - each element of the list refers to the allele 
index (0 for reference allele, 1 for the first alternate allele and so on). The length of the list represents the ploidy 
of the sample/CallSet and must be specified in the CSV line (since _GT_ is treated as a variable length list).

## Organizing your data
* All CSV files imported into a TileDB array must respect the number and order of fields as defined in the 
[[_vid_mapping_file_|Importing-VCF-data-into-GenomicsDB#fields-information]].
* The import program cannot handle CSV files where 2 lines have the same value of _row_ and _begin_column_ - this 
restriction is similar to that imposed on [[loading VCFs|Importing-VCF-data-into-GenomicsDB#organizing-your-data]].
* The import program cannot handle CSV files where 2 lines have overlapping column intervals for the same row id.
* Other requirements are the same as described in the wiki page for [[importing VCF data|Importing-VCF-data-into-GenomicsDB#organizing-your-data]].

## Information about CSVs for the import program
Information is passed to the import program through JSON files that are largely identical to
[[those described for VCF import|Importing-VCF-data-into-GenomicsDB#information-about-vcfs-for-the-import-program]]. The 
only difference is in the [[_callset_mapping_file_|Importing-VCF-data-into-GenomicsDB#samplescallsets]]. 

    {
        "callsets" : { 
            "HG00141" : {
                "row_idx" : 0,
                "idx_in_file": 0,
                "filename": "test_outputs/merged_java_alt3.list.csv"
            },
            "HG01530" : {
                "row_idx" : 1,
                "idx_in_file": 1,
                "filename": "test_outputs/merged_java_alt3.list.csv"
            },
            "HG01958" : {
                "row_idx" : 2,
                "idx_in_file": 0,
                "filename": "test_outputs/HG01958.csv"
            }
        }
        "sorted_csv_files" : [
            "test_outputs/HG01958.csv"
        ],
        "unsorted_csv_files" : [
            "test_outputs/merged_java_alt3.list.csv"
        ]
    }

By default, the import program assumes all input files are VCFs. The files (file paths) listed under the fields  
_sorted_csv_files_ and _unsorted_csv_files_  are marked as CSV files. Sorted CSV files are assumed to be sorted in 
[[column-major order|Basic-TileDB-GenomicsDB-terminology]].

Unsorted CSV files are first sorted by invoking the
[GNU sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) command internally. To import 
unsorted CSV files, you must have GNU coreutils installed in your system and the command sort in your PATH. The sorted 
CSV file is stored in a temporary directory - generally /tmp but can be set while running the import program. The 
temporary directory must be large enough to store all the sorted CSV files and the user must have write permission to 
the directory.

## Running the program

        ./bin/vcf2tiledb <loader_json>

Specifying a temporary directory to store sorted CSV files:

        ./bin/vcf2tiledb -T <tmp_directory> <loader_json>

