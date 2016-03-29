* _Cell_: Cell of an array (for example: _A[i][j]_ in a 2-D array)
* _Workspace_: A directory in the machine under which multiple TileDB arrays can be stored. TileDB stores some metadata 
files under the workspace directory. All GenomicsDB import/create programs assume that if the specified workspace 
directory exists, then it's a valid workspace directory. If the user points to an existing directory which doesn't 
contain the TileDB workspace metadata, the create/import programs will exit with an exception. If the directory doesn't 
exist, then the programs will initialize a new workspace correctly.
* _Array_: Name of a TileDB array
* Given a _workspace_ and _array_ name, the TileDB framework will store its data in the directory 
 _\<workspace\>_/_\<array\>_.
* _Column-major and row-major ordering_: Denotes the order in which cells are stored on disk by TileDB. Column major 
storage implies that cells belonging to the same column are stored contiguously on disk (cells belonging to the same 
column but different rows are sorted by row id). Thus, queries such as "get me all cells belonging to genomic position 
_X_" are fast in arrays that are stored in column major order because of high spatial locality. However, queries such as 
"get me all cells for sample _Y_" are relatively slow. For GenomicsDB, we expect the former type of queries to be more 
frequent and hence, by default all arrays are stored in column major order (even when partitioning by [[rows across TileDB instances|GenomicsDB-setup-in-a-multi-node-cluster]]).

    Row major ordering implies that cells belonging to the same row are stored contiguously on disk.
