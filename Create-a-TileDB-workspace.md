A TileDB workspace must be created before importing any data into a TileDB array - please refer to the [[terminology page|Basic-TileDB-GenomicsDB-terminology]] to find out what a workspace means.

## Creating a workspace

    ./bin/create_tiledb_workspace <workspace_dir>

The above command will create a workspace if the _\<workspace_directory\>_ does not exist. If _\<workspace_directory\>_  
exists, then the directory is not touched/modified.

We recommend running the above command on each node sequentially _before_ importing any data. This 
ensures that the workspace directory gets created in a consistent manner before any data gets imported into TileDB 
arrays.