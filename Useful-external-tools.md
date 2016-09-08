## Useful [bcftools](https://github.com/samtools/bcftools) commands
### Collapse multiple lines with the same genomic position into a single line

    bcftools norm -m +any [-O <output_format> -o <output>] <input_file>

Output format can be one of the following strings: "z" (compressed VCF),"b" (compressed BCF) or "bu" (uncompressed BCF). 
If nothing is specified, the default is uncompressed VCF. If the _-o_ parameter is omitted, the output is printed to 
stdout.
### Producing block compressed VCFs/BCFs

    bcftools view [-O <output_format> -o <output>] <input_file>

The output file name should ideally end with the suffix ".\[vcf\|bcf\].gz".
### Generating the index for a block compressed VCF/BCF file

    bcftools index [-f] <file>

The above command will create a CSI index. To produce a tabix index:

    bcftools index [-f] -t <file>

The _-f_ parameter will cause bcftools to overwrite an existing index file.

**NOTE:** Older versions of bcftools required the user to pass _-m0_ instead of _-t_ for creating a tabix index.

## Sorting [[CSV files before an import|Importing-CSVs-into-GenomicsDB]]
You must have [GNU coreutils](http://www.gnu.org/software/coreutils/coreutils.html) installed in your system.

    sort [-T <tmp_directory>] -t, -k2,2n -k1,1n -o <sorted_output.csv> <input.csv>

If you have a list of sorted CSV files and wish to merge them into a single sorted CSV file:

    sort [-T <tmp_directory>] -m -t, -k2,2n -k1,1n -o <sorted_output.csv> <input_csv_list>

