# Introduction
VariantDB is built on top of [TileDB](https://github.com/stavrospapadopoulos/TileDB) is a system for efficiently storing, querying and accessing sparse matrix/array data. TileDB is being developed by researchers at the [Intel Science and Technology Center for Big Data] (http://istc-bigdata.org/#&panel1-1). Variant data is sparse by nature (relative to the whole genome).

We store variant data in a 2D TileDB array where:
* Each column corresponds to a genomic position (chromosome + position)
* Each row corresponds to a sample in a VCF (or CallSet in the GA4GH terminology)
* Each cell contains data for a given sample/CallSet at a given position. Data is stored in the form of TileDB cell attributes.
* Variant interval/gVCF interval data is stored in a cell at the start of the interval. The END is stored as a cell attribute. When queried for a given genomic position, the query library performs an efficient left sweep to determine all intervals that intersect with the queried position.
* Cells are stored in column major order - this makes accessing cells with the same column index (i.e. data for a given genomic position over all samples) fast.

#Typical methodology for importing data into TileDB

* Assign unique row ids to each sample/CallSet. Sample/CallSet names must be unique
* Assign unique column ranges to each chromosome/contig in the “flattened” column space of TileDB array. Also, all chromosomes must be from the same reference genome - we have no idea what will happen if you mix and match.
* Produce a CSV file with a list of cells and attributes for each sample/CallSet
* Define TileDB array schema
* Import CSV files into TileDB