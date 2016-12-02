Please read [[ the wiki page | GenomicsDB-setup-in-a-multi-node-cluster ]] describing partitioning in the context of GenomicsDB.

Each import process (whether you are running vcf2tiledb or using a VCF2TileDB object in the Java write API) writes data to a single partition in column-major order (the order is independent of the partitioning scheme). You can use MPI to write to multiple partitions in parallel or simply use a parallel shell (such as npm-run-all for single hostor pdsh 

Many users wish to partition their GenomicsDB instances by column (or genomic position) since it provides higher performance for downstream queries and analysis. 