**_This page contains outdated information - see [[this page|Importing-VCF-gVCF-data-into-TileDB]] for the current methodology._**

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
[Detailed description](https://github.com/Intel-HLS/TileDB/wiki/Producing-GVCFs-from-TileDB
)

# Defining and creating a new array
[Detailed description](https://github.com/Intel-HLS/TileDB/wiki/Using-the-variant-specific-customizations#defining-the-array)

# Importing the created CSV files into TileDB
[Detailed description](https://github.com/Intel-HLS/TileDB/wiki/Using-the-variant-specific-customizations#loading)

