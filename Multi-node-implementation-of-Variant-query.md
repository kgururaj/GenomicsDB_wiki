Read [this page](https://github.com/Intel-HSS/TileDB/wiki/Using-the-variant-specific-customizations) for instructions on how to compile and run the single node version of the variant library first.

## Requirements:
* An MPI library
* [Rapidjson library](https://github.com/miloyip/rapidjson): I use this library to pass the query configuration in a json file to TileDB. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

## Compiling

Get the right branch of the TileDB repo for the multi node implementation.

    git checkout multi_node_var
    #make sure you have the new gcc version in your PATH
    #release mode - O3, NDEBUG - assertions disabled
    make MPIPATH=/usr/lib64/openmpi/bin/  BUILD=release RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include  -j 8
    #debug mode - assertions enabled, can use gdb for stepping
    make MPIPATH=/usr/lib64/openmpi/bin/ BUILD=release RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include  -j 8
    #Verbose output, with profiling enabled
    make MPIPATH=/usr/lib64/openmpi/bin/ RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include -j 8 BUILD=release VERBOSE=1 DO_PROFILING=1

## Running the MPI program
* Single node, simple interval query:

        ./variant/example/bin/gt_mpi_gather -w <workspace> -A <array> <begin> <end> -O <output_format>

  * Currently, \<output_format\> can only be _Cotton-JSON_ (case sensitive).
  * The following attributes will be queried: \[ "REF", "ALT", "BaseQRankSum", "AD", "PL" \] \(hard-coded\).
* Multi node, simple interval query:

        /opt/openmpi-1.10/bin/mpirun -np 2 -hostfile test_inputs/c10_14 --mca btl_tcp_if_include 192.168.100.0/24 -x LD_LIBRARY_PATH ./variant/example/bin/gt_mpi_gather -w <workspace> -A <array> <begin> <end> -O <output_format>
  The arguments to mpirun in this example specify the following:
  * -np 2 : Run 2 MPI processes
  * -hostfile \<file\> : This is a text file containing the list of hosts on which the MPI processes should be launched. See points 19-21 on the [OpenMPI FAQ page](https://www.open-mpi.org/faq/?category=running#mpirun-hostfile) for more information.
  * --mca btl_tcp_if_include 192.168.100.0/24 : Some machines may have multiple network interfaces. This option specifies that when using the TCP/IP interface, MPI should use the network interface with the IP address in the range 192.168.100.1-254. This is highly specific to the network and will likely not work on other clusters. If your cluster has InfiniBand (IB), consider using the IB interface for higher bandwidth. See the OpenMPI page for information.
  * -x LD_LILBRARY_PATH : This ensures that the env variable LD_LIBRARY_PATH is the same across all nodes (propagated from the node where mpirun is executed). This is needed if your libraries are in a non-standard location (for example, if you have a custom gcc version).
    