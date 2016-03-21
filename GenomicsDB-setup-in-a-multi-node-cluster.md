GenomicsDB can be setup to store variant data across multiple nodes in a cluster. The user must decide how to partition data across multiple nodes. Currently, two primary modes of partitioning are supported by various import/query tools.

* Row partitioning: In this mode, for a given sample/CallSet (row), all the variant data resides in a single node. Data belonging to different samples/CallSets may be scattered across different machines.

* Column partitioning: In this mode, for a given genomic position (column), all the variant data across all samples/CallSets resides in a single node. Data is partitioned by genomic positions across different machines.

Which partitioning scheme is better to use is dependent on the queries/analysis performed by downstream tools. Here are some example queries for which the 'best' partitioning schemes are suggested.

* Query: fetch attribute X from all samples/CallSets for position Y (or small interval [Y1-Y2])
    * Row-based partitioning

    _Rationale_
    * For single position queries (or small intervals), partitioning the data by rows would likely provide higher performance. By accessing data across multiple nodes in parallel, the system will be able to utilize higher aggregate disk and memory bandwidth. In a column based partitioning, only a single node would service the request.
    * Simple data import step if the original data is organized as a file per sample/CallSet (for example VCFs). Just transfer the required subset of files to the correct machine and import into TileDB.

    _Con(s)_
    * A final aggregator may be needed since the data for a given position is scattered across machines. Some of the query tools we provide use MPI to collect the final output into a single node.

* Query: run analysis tool _T_ on all variants (grouped by column position) found in a large column interval \[Z1-Z2\] (or scan across the whole array).
    * Column-based partitioning

    _Rationale_
    * The user is running a query/analysis for every position in the queried interval. Hence, for each position, the system must fetch data from all samples/CallSets and run _T_. Partitioning by column reduces/eliminates any communication between nodes. For a sufficiently large query interval, the aggregate disk and memory bandwidth across multiple nodes can still be utilized.
    * No/minimal data aggregation step as all the data for a given column is located within a single node.

    _Con(s)_
    * Importing data into TileDB may become complex, especially if the initial data is organized as a file per sample/CallSet.
    
