The following options are useful when producing a combined VCF (during the loading/importing or querying phase) similar to the one produced by the [GATK 
CombineGVCF 
tool](https://www.broadinstitute.org/gatk/guide/tooldocs/org_broadinstitute_gatk_tools_walkers_variantutils_CombineGVCFs.php).

Note that the Java interface produces combined VCF records ([VariantContext](https://samtools.github.io/htsjdk/javadoc/htsjdk/htsjdk/variant/variantcontext/VariantContext.html) objects) and hence, the following options are applicable when using the Java interface.

The options can be specified in the [[loader JSON file|Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]] if the combined VCF is being produced during the load/import phase. Otherwise, these options must be specified in the [[query JSON file|Querying-GenomicsDB#json-configuration-file-for-a-query]].

* _reference_genome_ : (type:string or list of strings, mandatory): Path to reference genome (indexed FASTA file).
* _vcf_header_filename_ (type:string or list of strings, optional): Path to a template VCF header file. All lines in this template will be present in the header of the combined VCF(s). This template should **NOT** contain sample/callset names (i.e. the line starting with #CHR). Contigs and fields present in the _vid_mapping_filename_ file will be added to the combined GVCF, if not present in the template header. If this field is omitted, then a simple header will be produced containing the contigs and fields described in the _vid_mapping_filename_ file.
* _max_diploid_alt_alleles_that_can_be_genotyped_ (type: int, optional, default: 50): For certain locations, the number of alternate alleles in the combined VCF record can get very large (we have seen ~700 alternate alleles for some sample sets). For such locations, the large size of the VCF record causes the program to consume a massive amount of memory. Additionally, in some cases, the VCF spec is unable to handle such large records (especially when there are fields such as _PL_ whose length depends on the number of genotypes). The parameter helps keep the size of the combined gVCF records in check. If the number of alternate alleles is greater than the value of _max_diploid_alt_alleles_that_can_be_genotyped_, then fields such as _PL_ are dropped for this VCF record. This fix is identical to the one implemented in GATK CombineGVCFs (including the default value).
* _vcf_output_filename_ (optional, type:string or list of strings): If producing a combined GVCF, then this parameter specifies the path at which the output VCF will be created. If this parameter is omitted, then the output VCF is printed 
on stdout.
* _vcf_output_format_ (type:string, optional, default _\<empty\>_): Output format can be one of the following strings: "z[0-9]" (compressed VCF),"b[0-9]" (compressed BCF) or "bu" (uncompressed BCF). If nothing is specified, the default is uncompressed VCF.
* _produce_GT_field_ (type: boolean, optional, default _false_): The _GT_ field in the combined VCF records is set to missing to match the output produced by GATK CombineGVCF. By setting _produce_GT_field_ to _true_, the _GT_ field will be retrieved from TileDB/GenomicsDB.
* _produce_GT_with_min_PL_value_for_spanning_deletions_ (type: boolean, optional, default _false_): This flag is applicable only when _produce_GT_field_ is true and only for [spanning deletions](https://software.broadinstitute.org/gatk/documentation/article.php?id=6926). By default (or when this flag is set to false), the _GT_ field for spanning deletions in the combined VCF records corresponds to the value stored in TileDB/GenomicsDB for the deletion. The allele indexes may get reassigned since the number of alleles in the spanning deletion may be reduced. For example:

                POS	REF	ALT	GT
      Original: 1000	ATGC	TTGC,A,<NON_REF>	0/2
      Spanning: 1001	T	*,<NON_REF>	0/1  #gets changed to 0/1 since number of alleles is reduced

  However, when this flag is set to true, the value of the GT field for spanning deletions corresponds to the genotype with the smallest likelihood value (_PL_ field). Thus, the _GT_ value in the spanning deletion could become 1/1.

  See the discussion in https://github.com/Intel-HLS/GenomicsDB/issues/161 for a detailed example, especially the comments by @ldgauthier.
* _index_output_VCF_ (type: boolean, optional, default _false_): If a compressed combined VCF file is being created (see _vcf_output_filename_ and _vcf_output_format_), setting this parameter to _true_ will create an index for the output file - tabix for compressed VCFs and csi for compressed BCFs.
* _produce_FILTER_field_ (type: boolean, optional, default _false_): The _FILTER_ field in the combined VCF records is set to missing to match the output produced by GATK CombineGVCF. By including the FILTER field in the list of queried attributes (or setting _scan_full_ to true) and setting _produce_FILTER_field_ to _true_, the _FILTER_ field will be retrieved from TileDB/GenomicsDB.
* _sites_only_query_ (type: boolean, optional, default _false_): When set to true, GenomicsDB will NOT produce any FORMAT fields in the resulting VCF records (no samples). ID, QUAL, FILTER and INFO fields will be produced.

Performance tuning options:
* _combined_vcf_records_buffer_size_limit_ (type: integer, optional, default 1048576): This parameter determines the size of the memory buffer (in bytes) to hold the combined VCF records in the following scenarios:
    * If the combined VCF records are being produced in the load/import phase and software pipelining is used to run multiple stages of the loader in parallel. Records are flushed to disk when this buffer is full.
    * If the Java interface is used to produce combined VCF records. Control returns to the Java code when this buffer is full.

### INFO and QUAL field combine operations
See [[this section|Importing-VCF-data-into-GenomicsDB#fields-information]] to find out how to specify combine operations for INFO and QUAL fields in the [[vid JSON file|Importing-VCF-data-into-GenomicsDB#information-about-vcfs-for-the-import-program]]. In particular, see the subsection labeled _VCF_field_combine_operation_.