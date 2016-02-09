# Introduction
VariantDB is built on top of [TileDB](https://github.com/stavrospapadopoulos/TileDB) is a system for efficiently storing, querying and accessing sparse matrix/array data. TileDB is being developed by researchers at the [Intel Science and Technology Center for Big Data] (http://istc-bigdata.org/#&panel1-1). Variant data is sparse by nature (relative to the whole genome).

We store variant data in a 2D TileDB array where:
* Each column corresponds to a genomic position (chromosome + position)
* Each row corresponds to a sample in a VCF (or CallSet in the GA4GH terminology)
* Each cell contains data for a given sample/CallSet at a given position. Data is stored in the form of TileDB cell attributes.
* Variant interval/gVCF interval data is stored in a cell at the start of the interval. The END is stored as a cell attribute. When queried for a given genomic position, the query library performs an efficient left sweep to determine all intervals that intersect with the queried position.
* Cells are stored in column major order - this makes accessing cells with the same column index (i.e. data for a given genomic position over all samples) fast.

#Importing data into TileDB for VCFs

* Assign unique row ids to each sample. Sample names must be unique
* Assign unique column ranges to each chromosome/contig in the “flattened” column space of TileDB array. Also, all chromosomes must be from the same reference genome - we have no idea what will happen if you mix and match.
* Produce a CSV file with a list of cells and attributes for each VCF
* Define TileDB array schema
* Import CSV files into TileDB

# Creating mappings from samples/contigs to row ids/column intervals

Mappings are stored in a SQLite DB. Development of a PostgreSQL based solution in progress.

Due to the limited nature of SQLite and our library built on top of it, we recommend that new mappings be added to the SQLite DB sequentially. Another way of putting it, is never run 2 processes which may update the SQLite DB in parallel.

Once new mappings are created, CSV files can be produced in parallel for different VCF files.

* Prep work for SQLite: [create the tables](https://github.com/kgururaj/bcftools/wiki/Using-bcftools-for-TileDB#prep-work)
* Obtain [htslib and bcftools](https://github.com/kgururaj/bcftools/wiki/Using-bcftools-for-TileDB
)
* Create new mappings for the sample names and contig names by running the program as [described here](https://github.com/kgururaj/bcftools/wiki/Using-bcftools-for-TileDB#consistent-sqlite-samples-numbering-across-nodes) sequentially for all VCF files.
* You may want to control the sample naming as [described here](https://github.com/kgururaj/bcftools/wiki/Using-bcftools-for-TileDB#sample-name).
* Put together:

        ./bcftools view -O t --sqlite=samples_and_fields.sqlite --tiledb-output-format=csv_v1  --header-only  test_inputs/t0.vcf
        # to control sample naming
        ./bcftools view -O t --sqlite=samples_and_fields.sqlite --tiledb-output-format=csv_v1  --header-only --tiledb-override-sample-name=HG01530 test_inputs/t0.vcf

* The above commands will create all the mappings in the SQLite DB in the file samples_and_fields.sqlite.

# Creating CSV files
[Details here](https://github.com/kgururaj/bcftools/wiki/Using-bcftools-for-TileDB#running)

Quick command:

    ./bcftools view -O t --sqlite=samples_and_fields.sqlite --tiledb-output-format=csv_v1  --tiledb-treat-deletions-as-intervals --tiledb-override-sample-name=HG01530 test_inputs/t0.vcf -o output.csv

# Obtaining and compiling TileDB
[Detailed description](https://github.com/Intel-HSS/TileDB/wiki/Producing-GVCFs-from-TileDB
)

# Defining and creating a new array
[Detailed description](https://github.com/Intel-HSS/TileDB/wiki/Using-the-variant-specific-customizations#defining-the-array)

# Importing the created CSV files into TileDB
[Detailed description](https://github.com/Intel-HSS/TileDB/wiki/Using-the-variant-specific-customizations#loading)

