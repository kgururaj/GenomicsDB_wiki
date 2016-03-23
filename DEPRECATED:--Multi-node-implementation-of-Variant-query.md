**_This page contains outdated information - see [[this page|Querying-GenomicsDB]] for the current methodology._**

## Running the MPI program
* Output data is sent to stdout and informational messages are sent to stderr.
* Output data is gathered at the root MPI process with rank == 0.
* Single node, simple interval query:

        ./variant/example/bin/gt_mpi_gather -w <workspace> -A <array> <begin> <end> -O <output_format>

  * Currently, \<output_format\> can only be _Cotton-JSON_ (case sensitive).
  * The following attributes will be queried: \[ "REF", "ALT", "BaseQRankSum", "AD", "PL" \] \(hard-coded\).
* Multi node, simple interval query:

        /usr/lib64/openmpi/bin/mpirun -np 2 -hostfile test_inputs/c10_14 --mca btl_tcp_if_include 192.168.100.0/24 -x LD_LIBRARY_PATH ./variant/example/bin/gt_mpi_gather -w <workspace> -A <array> <begin> <end> -O <output_format>

  The arguments to mpirun in this example specify the following:
  * -np 2 : Run 2 MPI processes
  * -hostfile \<file\> : This is a text file containing the list of hosts on which the MPI processes should be launched. See points 19-21 on the [OpenMPI FAQ page](https://www.open-mpi.org/faq/?category=running#mpirun-hostfile) for more information.
  * --mca btl_tcp_if_include 192.168.100.0/24 : Some machines may have multiple network interfaces. This option specifies that when using the TCP/IP interface, MPI should use the network interface with the IP address in the range 192.168.100.1-254. This is highly specific to the network and will likely not work on other clusters. If your cluster has InfiniBand (IB), consider using the IB interface for higher bandwidth. See the OpenMPI page for information.
  * -x LD_LIBRARY_PATH : This ensures that the env variable LD_LIBRARY_PATH is the same across all nodes (propagated from the node where mpirun is executed). This is needed if your libraries are in a non-standard location (for example, if you have a custom gcc version).
  
  Note that all the nodes MUST have identical array names and schemas as well as identical locations for the workspace.
* Use the command line argument "--skip-query-on-root" in setups where the TileDB array data is located on data nodes, but the results must be gathered at a non-data root node. However, the array schema (but not the data) MUST be available on the root node.
* Configurable queries: Query information is passed to the MPI program using a JSON file.
         
        ./variant/example/bin/gt_mpi_gather -j <json_file> -O <output_format>
        {
          "workspace" :  [ "../configs/ws", "/mnt/app_hdd/scratch/karthikg/VCFs/tiledb_csv/v1/arrays/" ],
          "array" : [ "t0_1_2_GT", "GT10" ],
          "query_column_ranges" : [ [ [12000, 13000 ]  ], [ [0, 10000000] ] ],
          "query_row_ranges" : [ [ [100, 3000 ]  ], [ [4000, 10000] ] ],
          "query_attributes" : [ "REF", "ALT", "BaseQRankSum", "AD", "PL" ]
        }

  * "workspace" (mandatory): List of strings, specifying the locations of the workspace directory for each MPI process launched. The length of this list MUST be equal to the number of MPI processes launched. Alternately, this parameter can be a string (not a list), which implies that all MPI processes access the workspace at the same directory location.
  * "array" (mandatory) : Similar to the workspace parameter. Note that the arrays MUST have exactly identical schemas.
  * "query_column_ranges" (mandatory): Each MPI process can query a list of column ranges. For example, the list \[ \[ 0, 100 \], 500 \] specifies that the MPI process should query column interval \[0-100\] and the single position 500. The parameter "query_column_ranges" is a list of such lists. The length of the outer list MUST be EITHER equal to the number of MPI processes launched OR just 1 (implying that all MPI processes query the same column ranges).
  * "query_row_ranges" (optional): Same format as "query_column_ranges", but for rows. Can be omitted, in which case all rows of the array will be queried.
  * "query_attributes" (mandatory): List of strings specifying attributes to be fetched.

  NOTE: The RapidJSON library doesn't clearly flag syntax errors. I generally run the command "json_verify \< \<json_file\>" to check the syntax first.