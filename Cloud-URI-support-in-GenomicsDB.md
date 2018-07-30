There are two levels of Cloud URI support in GenomicsDB. Using URIs to read in VCF files is covered [here](Reading-VCF-files-directly-from-S3-or-GCS-or-over-http(s).md). 

Cloud URIs can also be specified to all tools and GenomicsDB classes```(com.intel.genomicsdb.importer.GenomicsDBImporter and com.intel.genomicsdb.reader.GemomicsDBFeatureReader)```  that create workspaces, load data into arrays and query the arrays from the workspaces via the loader and query json files. The following schemes are currently supported -

* HDFS e.g. hdfs://my-master:9000/my_workspace
* EMRFS e.g. s3://my-bucket/my_repository/my_workspace
* GCS e.g. gs://my-bucket/my_workspace

Examples:

* create_tiledb_workspace hdfs://my_master:9000/ws
* consolidate_tiledb_array gs://my-bucket/ws/gdb_partition_1
* Set workspace paths in the JSON files to point to either local filesystems(e.g. /my_home/my_workspace) or to cloud URIs(e.g hdfs://my-master:9000/my_workspace).

The support for the Cloud URIs in GenomicsDB is through Hadoop's support for distributed file systems(HDFS). HDFS runs on JVM and GenomicsDB relies on libhdfs to access HDFS Java API. Follow instructions to install and configure HDFS from either

* [Single Node HDFS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html) or
* [Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html)

Check that libjvm is configured into the library paths```(${LD_LIBRARY_PATH})```. Also, configure the JAVA classpath using ```hadooop classpath --glob```. 
  
EMRFS and GCS come configured to work with HDFS directly.

* EMRFS is an implementation of HDFS and GenomicsDB works out of the box as long as the buckets are accessible from HDFS. 
* For GCS, install the [GCS Cloud Connector](https://cloud.google.com/dataproc/docs/concepts/connectors/install-storage-connector) to work with gs:// URIs. 
    Basically, set up a service account and download the service key saved in the JSON format. Either ```set env GOOGLE_APPLICATION_CREDENTIALS``` to the downloaded JSON file or configure hadoop appropriately. See configured hadoop properties in core-site.xml if you follow that route.

```xml
		<property>
			<name>google.cloud.auth.service.account.enable</name>
			<value>true</value>
		</property>
		<property>
			<name>google.cloud.auth.service.account.json.keyfile</name>
			<value>/home/myself/GCS.json</value>
		</property>
		<property>
			<name>fs.gs.project.id</name>
			<value>braided-visitor-fdfdf</value>
		</property>
		<property>
			<name>fs.gs.working.dir</name>
			<value>/</value>
		</property>
```

