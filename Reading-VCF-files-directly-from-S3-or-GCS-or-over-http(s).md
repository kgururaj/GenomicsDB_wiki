**EXPERIMENTAL**

GenomicsDB uses [htslib](https://github.com/samtools/htslib) to read VCF files. htslib can read data off http(s) URLs using libcurl. By using the REST API provided by object storage systems like S3, GCS and Ceph, htslib should be able to read indexed VCFs directly (rather than having to download them to a local drive first).

* Install libcurl development libraries

        centos: yum -y install libcurl-devel
        ubuntu: apt-get install libcurl4-openssl-dev
* Build GenomicsDB  with libcurl enabled -DENABLE_LIBCURL=1
* In the callsets mapping JSON, set path to the VCF file in the _"filename"_ attribute. Example paths:

        s3://bucket/file.vcf.gz
        gs://bucket/file.vcf.gz
   Note: the VCF index file should be at the same place as the VCF file (s3://bucket/file.vcf.gz.tbi in the above example).
* Credentials: the credentials for accessing restricted data must be available on the host/VM running the import process. There are multiple options for specifying the authentication information.
  * AWS/S3 (in decreasing order of priority)
    * Environment variables
      * AWS_ACCESS_KEY_ID
      * AWS_SECRET_ACCESS_KEY
      * AWS_SESSION_TOKEN
    * $HOME/.aws/credentials (ini file)
      * aws_access_key_id
      * aws_secret_access_key
      * aws_session_token
    * [$HOME/.s3cfg](https://s3tools.org/kb/item14.htm)
      * access_key
      * secret_key
      * access_token
      * host_base (useful if, for example, you are using Ceph with an S3 API)
  * GCS
    * [Setup a service account and get the JSON file containing the key](https://cloud.google.com/docs/authentication/getting-started)
    * Generate a token using the credentials file

            export GOOGLE_APPLICATION_CREDENTIALS=<path to JSON file with key>
            gcloud auth application-default print-access-token
    * On the machine where the import process will run, set the environment variable GCS_OAUTH_TOKEN to the token printed by the above command