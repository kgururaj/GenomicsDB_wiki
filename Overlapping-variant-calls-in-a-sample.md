TileDB/GenomicsDB currently handle small variants - SNVs, small insertions and deletions. An important property that is required for correct operation (as of 7/8/2016) is that for a given sample/Callset/TileDB row, no two variant calls should overlap (there is no issue if variant calls from different samples/CallSets/rows overlap).

However, variant callers such as the GATK HaplotypeCaller do occasionally produce overlapping calls for a given sample. Here is a small example:

    #CHROM  POS     REF     ALT
    1       1000    TGA     T,<NON_REF>
    1       1001    G       A,<NON_REF>
    1       1003    A       <NON_REF>

The deletion at position 1:1000 spans positions 1:[1000-1002]; however, another variant call is produced at position 1001. Such a configuration by itself cannot yet be handled by TileDB/GenomicsDB (as of 7/8/2016).

## Workaround
Since we are interested in dealing with multiple samples/CallSets in TileDB/GenomicsDB, we determined what the GATK CombineGVCFs tool would do when the above file is passed as input. When dealing with position 1001, the GATK tool ignores the deletion that spans positions 1:[1000-1002] and considers the SNV at position 1001 as the sole variant call at the location for the given sample.

We mimic this functionality in TileDB/GenomicsDB while storing the variant calls in the TileDB array (and while creating a combined gVCF). The deletion at position 1:1000 is truncated to end at 1:1000 since a new variant call appears at location 1:1001. This satisfies the requirement that no two variant calls for a given sample overlap. A query for position 1:1001 in TileDB/GenomicsDB will return the SNV and not the deletion. A query for position 1:1002 will return no calls.

Note that had the variant call at location 1:1001 not existed, the deletion would span location 1:[1000-1002]. Queries for positions 1:1001 and 1:1002 in TileDB/GenomicsDB would have returned the deletion.

Overlapping reference blocks (intervals with \<NON_REF\> as the only alternate allele) are handled in an identical manner.
## What would a query on an indexed VCF file corresponding to the above sample return?
If the variant calls in the above sample are stored in an block compressed and indexed VCF file and [htslib](https://github.com/samtools/htslib)/[bcftools](https://github.com/samtools/bcftools) used to query location 1:1001, then both the deletion and the SNV at position 1:1001 are returned. A query for position 1:1002 would return just the deletion.

## Implications for partitioned arrays/VCFs in TileDB/GenomicsDB
* Assume an array/VCF partition begins at 1:1001. This partition will not contain the deletion and will begin with the SNV at 1:1001.
* Assume an array/VCF partition begins at 1:1002. This partition will not contain the deletion or the SNV. The first call for this sample in the partition will be the reference block at 1:1003.

## Summary
The treatment of overlapping intervals in TileDB/GenomicsDB feels like a hack and tied to the VCF format and unfortunately, it is. TileDB/GenomicsDB cannot deal with overlapping variant calls within a single sample (as of 7/8/2016). Fixing this is a matter of finding time for the TileDB/GenomicsDB developers since technical solutions exist. The tougher questions that are outside the scope of TileDB/GenomicsDB are the following:
* How do downstream bioinformatic tools interpret overlapping variant calls?
* Is the VCF-based structure suitable for dealing with such variant calls? Or should a [graph based approach](https://github.com/ga4gh/schemas/wiki/Human-Genome-Variation-Map-%28HGVM%29-Pilot-Project) be the way forward?