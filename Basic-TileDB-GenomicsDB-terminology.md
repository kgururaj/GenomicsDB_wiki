* _Cell_: Cell of an array (for example: _A[i][j]_ in a 2-D array)
* _Workspace_: A directory on the file system under which multiple TileDB arrays can be stored. TileDB stores some metadata files under the workspace directory. All GenomicsDB import/create programs assume that if the specified workspace 
directory exists, then it's a valid workspace directory. If the user points to an existing directory which doesn't 
contain the TileDB workspace metadata, the create/import programs will exit with an exception. If the directory doesn't 
exist, then the programs will initialize a new workspace correctly.
* _Array_: Name of a TileDB array
* Given a _workspace_ and _array_ name, the TileDB framework will store its data in the directory 
 _\<workspace\>_/_\<array\>_.
* _Column-major and row-major ordering_: Denotes the order in which cells are stored on disk by TileDB. 
    * Column major storage implies that cells belonging to the same column are stored contiguously on disk (cells belonging to the same column but different rows are sorted by row id). When cells are stored in column major order queries such as "retrieve all cells belonging to genomic position _X_" are fast due to high spatial locality. However, queries such as 
"retrieve cells for sample _Y_" are relatively slow. For GenomicsDB, we expect the former type of queries to be more 
frequent and hence, by default all arrays are stored in column major order (even when partitioning by [[rows across TileDB instances|GenomicsDB-setup-in-a-multi-node-cluster]]).
    * Row major ordering implies that cells belonging to the same row are stored contiguously on disk.
* _Bulk importing and incremental importing_: _Bulk importing_ of data implies that all the data is loaded at once into 
TileDB/GenomicsDB and the array is never modified later on. _Incremental importing_ implies that the array can be 
modified by adding new samples/CallSet over time. TileDB/GenomicsDB support both modes of operation - however, repeated 
incremental updates (for example, incrementally adding one sample/CallSet at a time, ~1000 times) may lead to reduced 
performance during querying. If you are interested in why this occurs, please read the [section on writing TileDB 
arrays](http://istc-bigdata.org/tiledb/tutorials/index.html#writing) in the TileDB website to understand the concept of 
fragments and how TileDB performs updates. Ideally, users should perform incremental imports a few times (~10s of times) 
per array and each import should contain a large number of samples.
