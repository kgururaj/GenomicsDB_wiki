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

## My input data is in multiple files, one file per sample/CallSet (or row) and I wish to import data into TileDB/GenomicsDB  partition(s)
Generally, input data, such as variants produced by variant calling pipelines, is located in multiple files (VCF), one file per sample/CallSet (or a small set of samples/CallSets per file). Each file contains data from all genomic positions for the sample/CallSet. Users may wish to import data from a set of such files into multiple TileDB/GenomicsDB partitions which may be located on different machines.

* If you are using a shared filesystem such as NFS, Lustre etc to host your input variant files (example VCF files), there is no blocker since every machine has access to all the files.
* If you do not have a shared filesystem, then you might run into issues. Copying all the input files to every machine on which a GenomicsDB partition will be hosted may not be a feasible solution since a machine may not have enough disk space to store all the data from all the samples.

### Partitioned by row (or sample/CallSet)
Importing data into a TileDB/GenomicsDB array partitioned by row is relatively easy. Each import process for each partition needs to access only the files which contain the samples/CallSets assigned to the partition. Thus, even if there is no shared filesystem, then only a subset of input files, corresponding to the samples/CallSets assigned to the partition(s) on a machine, needs to be copied to the machine. Once each machine can access the files assigned to the partitions on the machine, _vcf2tiledb_ can be run to import data.

### Partitioned by column (or genomic position)
Certain tools (for example, GATK GenotypeGVCFs) require data from all samples for each genomic position on which they operate. When running such an analysis on a large number (or range) of genomic positions, it might be beneficial to partition the data by columns. The reason is that all the data is local to a single machine which minimizes inter-machine network traffic.

For each column partition, the import process needs to access data from all samples. We describe some common scenarios:

* If you have all your input files (such as VCFs) on a shared filesystem (such as NFS, Lustre etc), there are no significant changes to the import process. You should be able to create the loader JSON and execute _vcf2tiledb_ to import data into different partitions. Since every machine can access all the files, all the _vcf2tiledb_ processes can access and load data from different regions of the files.

* If there is no shared filesystem, then copying all the input files to every machine that hosts a partition may not be feasible. Our current recommended solution is to split each input file into multiple files, one file per partition. Each split file contains data only for one partition. This way only the split files can be copied to a particular machine. Split files can be significantly smaller than the original input files.

  The _vcf2tiledb_ executable can be used to split the input files as per the column partitions specified in the loader JSON file. Prepare the loader and callset mapping files as [[described previously | Importing-VCF-data-into-GenomicsDB ]] with the original input VCF files. Then:

        ./bin/vcf2tiledb <loader.json> --split-files --split-all-partitions

  The argument `--split-files` causes _vcf2tiledb_ to only split the files in the callset mapping JSON file as per the column partitions specified in the loader JSON configuration file. The split files are created in the same directory as the original files. For example, if one of the input files is `/data/HG01958.vcf.gz`, then the split files are `/data/partition_0_HG01958.vcf.gz, /data/partition_1_HG01958.vcf.gz, ...`.

  The argument `--split-all-partitions` causes the program to create the split files for all the column partitions. If the argument is omitted, then split files for only the column partition corresponding to the rank of the _vcf2tiledb_ process are created.

  Other arguments:
  
  * _--split-callset-mapping-file_ (optional): By specifying this argument, the program will create callset mapping files which contain paths to the split files. The newly created callset mapping files are identical to the original callset mapping file except that the input file paths are set to the split files. Thus, if there are _N_ partitions, _N_ callset mapping JSON files are created, one for each partition.

    The program will also create a loader JSON identical to the original loader JSON except that the _callset_mapping_file_ field will be replaced by a list of paths to the newly created callset mapping files. For example, if the loader is located at `/configs/loader.json`

        {
            "column_partitions": [
                { "begin": 0 },
                { "begin": 1000000 }
            ],
            "callset_mapping_file": "/configs/callset_mapping.json"
        }
    The loader produced by the program will be at `/configs/partition_0_loader.json` (irrespective of the number of partitions) and will look like:

        {
            "column_partitions": [
                { "begin": 0 },
                { "begin": 1000000 }
            ],
            "callset_mapping_file": [
                "/configs/partition_0_callset_mapping.json",
                "/configs/partition_1_callset_mapping.json"
            ]
        }
    The produced loader and callset JSON files can then be used directly to import data into TileDB/GenomcisDB.
  * _--split-files-results-directory=\<directory\>_ (optional, type: string): Specifies the path to a directory in which the split files will be created - split files are not created in the same directory as the input file. This also applies to newly created callset and loader JSON files.

  _Creating a split file for one column partition for a single input VCF file_: Utility option in case the user wishes to explicitly control the naming of files through a script.

        ./bin/vcf2tiledb <loader.json> --rank=<rank> --split-files --split-output-filename=<output_path> <input.vcf.gz>

  If the `--rank` argument is omitted, the default rank is 0, so the split file will contain data corresponding to partition index 0.