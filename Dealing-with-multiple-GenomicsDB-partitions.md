Please read [[ the wiki page | GenomicsDB-setup-in-a-multi-node-cluster ]] describing partitioning in the context of GenomicsDB.

Each import process (whether you are running _vcf2tiledb_ or using a _VCF2TileDB_ object in the Java write API) writes data to a single partition in column-major order (the order is independent of the partitioning scheme). You can use MPI to spawn multiple processes to write to multiple partitions in parallel or use a parallel shell (such as [GNU parallel] (https://www.gnu.org/software/parallel/man.html) for a single node, multi-partition setup or [pdsh] (https://linux.die.net/man/1/pdsh) for a multi-node setup) and pass the rank/partition index argument value for each process explicitly.

## What paths should I use for my workspaces/arrays/combined VCF files when dealing with multiple partitions?
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

## My input data is in multiple files, one file per sample/CallSet (or row) but I wish to import data into GenomicsDB  partition
Generally, input data, such as variants produced by variant calling pipelines, is located in multiple files (VCF), one file per sample/CallSet (or a small set of samples/CallSets per file). Each file contains data from all genomic positions for the sample/CallSet. Users may wish to import data from a set of such files into multiple TileDB/GenomicsDB partitions which may be located on different machines.

* If you are using a shared filesystem such as NFS, Lustre etc to host your input variant files (example VCF files), there is no blocker since every machine has access to all the files.
* If you do not have a shared filesystem, then you might run into issues. Copying all the input files to every machine on which a GenomicsDB partition will be hosted may not be a feasible solution since a machine may not have enough disk space to store all the data from all the samples (note that the machine must have enough space to store data from all the samples for the specific partition).

### Partitioned by row (or sample/CallSet)
Loading data into a TileDB/GenomicsDB array partitioned by row is relatively easy. Each import process for each partition needs to access only the files which contain the samples/CallSets assigned to the partition. Thus, if there is no shared filesystem, then only a subset of files, corresponding to the samples/CallSets assigned to the partition(s) on a machine, needs to be copied to the machine.
 
 You should be able to create the loader JSON and execute _vcf2tiledb_ to import data into different partitions.


Our current recommended solution is to split the each input file a. For example