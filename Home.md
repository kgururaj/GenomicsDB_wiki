# Introduction
[TileDB](https://github.com/Intel-HLS/TileDB) is a system for efficiently storing, querying and accessing sparse matrix/array data. TileDB is developed by researchers at the [Intel Science and Technology Center for Big Data] (http://istc-bigdata.org/#&panel1-1).

GenomicsDB is built on top of the TileDB system. Variant data is sparse by nature (sparse relative to the whole genome) and hence TileDB is a perfect fit for storing such data.

The GenomicsDB stores variant data in a 2D TileDB array where:
* Each column corresponds to a genomic position (chromosome + position)
* Each row corresponds to a sample in a VCF (or CallSet in the GA4GH terminology)
* Each cell contains data for a given sample/CallSet at a given position; data is stored in the form of TileDB cell attributes.
* Cells are stored in column major order - this makes accessing cells with the same column index (i.e. data for a given genomic position over all samples) fast.
* Variant interval/gVCF interval data is stored in a cell at the start of the interval. The END is stored as a cell attribute. When queried for a given genomic position, the query library performs an efficient sweep to determine all intervals that intersect with the queried position.


#Typical methodology for importing variant data into GenomicsDB 

* Assign unique row ids to each sample/CallSet. Sample/CallSet names must be unique
* Assign unique column ranges to each chromosome/contig in the “flattened” column space of TileDB array. Also, all chromosomes must be from the same reference genome - results are undefined if you mix and match chromosomes from different reference genomes.
* Define a list of fields/attributes you wish to store in GenomicsDB.
* Produce a CSV file with a list of cells and attributes for each sample/CallSet
* Import CSV files into TileDB
