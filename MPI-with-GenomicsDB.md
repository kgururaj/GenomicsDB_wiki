From [Wikipedia](https://en.wikipedia.org/wiki/Message_Passing_Interface): "Message Passing Interface (MPI) is a 
standardized and portable message-passing system designed by a group of researchers from academia and industry to 
function on a wide variety of parallel computers. The standard defines the syntax and semantics of a core of library 
routines useful to a wide range of users writing portable message-passing programs in different computer programming 
languages such as Fortran, C, C++ and Java." MPI libraries allow parallel programs to operate and communicate across 
multiple machines over a wide variety of network fabrics (such as Ethernet, Infiniband etc).

A comprehensive discussion of MPI is beyond the scope of this wiki - we refer the reader to online tutorials such as the 
one provided by [LLNL](https://computing.llnl.gov/tutorials/mpi/).

In GenomicsDB, we use MPI for the following purposes:
* Loading data into multiple TileDB partitions that may be scattered across multiple machines
* Querying multiple TileDB partitions that may be scattered across multiple machines

## Fields in JSON configuration files for MPI programs
In a multi-node cluster, the user is free to set the location of an array on each machine independently.  Thus, all 
machines may host their array partitions at the same path on their private filesystems or at different paths.  
Additionally, during querying, the user may wish to launch multiple MPI processes to process data from a single array 
partition for performance reasons (for example, a given array could have 1 TB of data and the user might wish to divide 
up the querying task into 8 subtasks, each dealing with 0.125 TB of data). Hence, many fields in the configuration files
could take single element values or a list of values. When a field takes single element value, this indicates to the 
program that all MPI processes will use the same value for that field. When a list of values is specified for a field, 
each MPI process picks the value in the list corresponding to the rank of the process. For example, a query with 4 MPI 
processes will have ranks 0-3 for its processes and process 2 will pick element 2 (0-based) from the list. In this case, 
the length of the list must be equal to the number of MPI processes launched.

The following fields may take single element values or list of values:
* _workspace_
* _array_
* _callset_mapping_file_
* _vid_mapping_file_
* _vcf_header_filename_
* _reference_genome_
* _vcf_output_filename_

We show an example configuration JSON file to clarify. Assume we launch 2 MPI processes to run a query - each process 
takes as input the following JSON configuration file (same file for both processes).

        {
          "workspace" :  [ "/tmp/ws", "/mnt/ws" ],
          "array" : "t0_1_2",
          "callset_mapping_file" : "/share/callset.json",
          "vid_mapping_file" : "/share/vid.json",
          "vcf_header_filename" : "/share/test_inputs/template_vcf_header.vcf",
          "vcf_output_filename" : ["/share/vcf1", "/share/vcf2" ]
          "reference_genome" : "/opt/Homo_sapiens_assembly19.fasta"
        }

The above configuration implies that MPI process 0 will access the workspace located at _"/tmp/ws"_ while process 1 will 
access the workspace at _"/mnt/ws"_. Both processes access the same array named _"t0_1_2"_ located within their 
respective workspaces. The paths for _"callset_mapping_file"_, _"vid_mapping_file"_,  _"vcf_header_filename" and
_"reference_genome"_ are identical for both processes. The 2 processes produce distinct output files at "/share/vcf1" 
and "/share/vcf2" respectively since the field _"vcf_output_filename"_ is a list of strings.

### Special fields during the loading process
* _column_partitions_: When using multiple MPI processes to load data into multiple column partitions, the user must 
specify the partitioning information as a list of dictionaries as described in the [[loading page | 
Importing-VCF-data-into-GenomicsDB#execution-parameters-for-the-import-program]]. The length of the list must match the 
number of MPI processes which must be equal to the number of partitions that the user wishes to create.
* _row_partitions_: Same as _column_partitions_.

### Special fields while querying
* _query_column_ranges_ : Each MPI process can query a list of column ranges. Assume we have 2 MPI processes querying 
GenomicsDB with the following _query_column_ranges_ setting:

          "query_column_ranges" : [ [ [0, 100 ], 500  ], [ [0, 10000000] ] ],

The above example specifies that _query_column_ranges_ is a list of size 2. Each element of the outermost list is also a 
list and specifies the column ranges that the corresponding MPI process will query. In the above example, the MPI 
process with rank 0 will query the column range \[0-100\] and the single position 500. MPI process 1 will query the 
column range \[0-10000000\].

The length of the outermost list MUST be EITHER equal to the number of MPI processes launched or just 1 (implying that 
all MPI processes query the same list of column ranges).
*_query_row_ranges_:  Same format as _query_column_ranges_, but for rows.

## Running MPI programs
MPI is a standard and there are multiple implementations of the standard. We have used 3 implementations on GNU/Linux: 
[OpenMPI](https://www.open-mpi.org/), [MPICH](https://www.mpich.org/) and 
[MVAPICH2](http://mvapich.cse.ohio-state.edu/). A comprehensive discussion of the various parameters in the different 
imlpementations is beyond the scope of this wiki. We provide sample commands that worked for us:

* OpenMPI:
** Sample command on Ethernet based clusters:

             mpirun --mca btl_tcp_if_include <network> -n <num_processes> -hostsfile <hostsfile> -x LD_LIBRARY_PATH -x PATH --bind-to none <genomicsdb_program> <args>

The _\<network\>_ arguments specifies a subnet such as 192.168.1.0/24.
** Sample hostsfile. The number after _max_slots_ represents the number of processes to run in that machine. If left 
empty, the MPI implementation generally sets it to the number of cores in that machine.

             host0.cluster.org max_slots=2
             host1.cluster.org max_slots=6

* MPICH:
** Sample command:

             mpirun -n <num_processes> -f <hostsfile> -genvlist PATH,LD_LIBRARY_PATH <genomicsdb_program> <args>

The network must specify a subnet such as 192.168.1.0/24.
** Sample hostsfile

             host0.cluster.org:2
             host1.cluster.org:6

* MVAPICH2:
** Sample command:

             mpirun_rsh -n <num_processes> -hostsfile <hostsfile> LD_LIBRARY_PATH=$LD_LIBRARY_PATH PATH=$PATH <genomicsdb_program> <args>

The network must specify a subnet such as 192.168.1.0/24.
** Sample hostsfile

             host0.cluster.org:2
             host1.cluster.org:6

## Common pitfalls
* The JSON configuration files passed as command line arguments to the GenomicsDB programs should be accessible on all 
machines at the exact same path. If you are on a shared filesystem, keep the configuration files there (these are small 
files). Else you need to copy the files to the same location on all machines.
* MPI not installed at the same location in all machines
* Different versions of MPI installed.
* Missing environment variables: The launched MPI programs will *NOT* see the environment from which mpirun is invoked.  
You need to explicitly set/pass the required environment variables (see sample commands listed above for passing PATH 
and LD_LIBRARY_PATH variables).
