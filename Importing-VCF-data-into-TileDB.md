The import program can handle block compressed and indexed VCFs, gVCFs, BCFs and gBCFs. For brevity, we will only use the term VCF.

## Preliminaries
* Use the branch _multi_node_loader_.
* You need htslib while [[compiling|Compiling-VariantDB]].

## Organizing your data
* All your VCFs must be block compressed and indexed.
* In a multi-node environment, you must decide:
    * How to [[partition your data in TileDB|VariantDB-setup-in-a-multi-node-cluster]].
    * How your VCF files are accessed by the import program:
        * On a shared filesystem (NFS, Lustre etc) accessible from all nodes.
        * On a local filesystem accessible only by the node on which the filesystem is mounted.


