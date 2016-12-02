Please read [[ the wiki page | GenomicsDB-setup-in-a-multi-node-cluster ]] describing partitioning in the context of GenomicsDB.

Each import process (whether you are running _vcf2tiledb_ or using a _VCF2TileDB_ object in the Java write API) writes data to a single partition in column-major order (the order is independent of the partitioning scheme). You can use MPI to spawn multiple processes to write to multiple partitions in parallel or use a parallel shell (such as [GNU parallel] (https://www.gnu.org/software/parallel/man.html) for a single node, multi-partition setup or [pdsh] (https://linux.die.net/man/1/pdsh) for a multi-node setup) and pass the rank/partition index argument value for each process explicitly.

## What directory/array/output name should I use for my workspaces/arrays when dealing with multiple partitions?
Each partition must be located on a separate physical directory on disk to avoid clobbering another partition's data. This can be achieved in multiple ways depending on how you organize your data in your cluster:

### Scenario 1: Partitions are located on the the local disks of a different machines.
In this case, the _workspace_ (and the _array_) parameter(s) in the loader JSON file can have the same value for all partitions. Since partitions are located on the local disks/filesystems of different machines, no conflicts are possible.

Such a scenario allows you to simplify the loader JSON (and all other JSON files) - since the _workspace_ and _array_ can be set to the same value for all partitions. So, instead of:

    {
        "column_partitions" : {[
            { "begin": 0, "workspace": "/workspace", "array": "variants_array" },
            { "begin": 10000, "workspace": "/workspace", "array": "variants_array" }
        ]}
    }

you can simply use:

    {
        "workspace": "/workspace",
        "array": "variants_array",
        "column_partitions" : {[
            { "begin": 0 },
            { "begin": 10000 }
        ]}
    }


### Scenario 2: Partitions are located on a shared filesystem (such as NFS, Lustre etc) or multiple partitions may be hosted on a single filesystem
In this case, each partition must be located on a separate directory. Hence, fields such _workspace_, _array_, _vcf_output_filename_ etc should be unique for each partition. This information can be specified in the loader and query JSON files as lists (either as list of strings or as a list of dictionaries with the parameter specified in each dictionary).

Many users wish to partition their GenomicsDB instances by column (or genomic position) since it provides higher performance for downstream queries and analysis. 