GenomicsDB can be setup to store variant data across multiple partitions of an array. All the data belonging to one partition of an array lives on a single filesystem. Thus, by creating multiple partitions, users can store data possibly across multiple hosts/nodes in a cluster. Array partitioning is useful when the data to be stored and queried is very large and cannot fit within a single machine/node. Or the user might wish to store array partitions in different nodes so that downstream queries and analysis can be run in a distributed manner for scalability and/or performance. 

The user must decide how to partition data across multiple nodes in a cluster:
* How many nodes should be used to store the data?
* How many partitions should reside on each node? A single node can hold multiple partitions (assuming the node has enough disk space).
* What mode should be used for partitioning the data?
  Two modes of partitioning are supported by various import/query tools.
  * Row partitioning: In this mode, for a given sample/CallSet (row), all the variant data resides in a single partition. Data belonging to different samples/CallSets may be scattered across different partitions.
  * Column partitioning: In this mode, for a given genomic position (column), all the variant data across all samples/CallSets resides in a single partition. Data is partitioned by genomic positions.

  Which partitioning scheme is better to use is dependent on the queries/analysis performed by downstream tools. Here are some example queries for which the 'best' partitioning schemes are suggested.

  * Query: fetch attribute X from all samples/CallSets for position Y (or small interval [Y1-Y2])
      * Row-based partitioning

      _Rationale_
      * For single position queries (or small intervals), partitioning the data by rows would likely provide higher performance. By accessing data across multiple partitions that may be located in multiple nodes in parallel, the system will be able to utilize higher aggregate disk and memory bandwidth. In a column based partitioning, only a single partition would service the request.
      * Simple data import step if the original data is organized as a file per sample/CallSet (for example VCFs). Just import data from the required subset of files to the correct partition.

      _Con(s)_
      * A final aggregator may be needed since the data for a given position is scattered across machines. Some of the query tools we provide use MPI to collect the final output into a single node.

  * Query: run analysis tool _T_ on all variants (grouped by column position) found in a large column interval \[Z1-Z2\] (or scan across the whole array).
      * Column-based partitioning

      _Rationale_
      * The user is running a query/analysis for every position in the queried interval. Hence, for each position, the system must fetch data from all samples/CallSets and run _T_. Partitioning by column reduces/eliminates any communication between partitions. For a sufficiently large query interval, the aggregate disk and memory bandwidth across multiple nodes can still be utilized.
      * No/minimal data aggregation step as all the data for a given column is located within a single partition.

      _Con(s)_
      * Importing data into TileDB may become complex, especially if the initial data is organized as a file per sample/CallSet.