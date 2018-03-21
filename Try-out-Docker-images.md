## Docker images

We provide a couple of Docker images to try out GenomicsDB at Dockerhub: 
https://hub.docker.com/r/intelhlsgenomicsdb/

WARNING: these images are useful for quickly trying out GenomicsDB - we don't recommend using them 
for production data/setups.

The images can:
* Import a bunch of VCF files into a GenomicsDB partition
* Query the partition and return results

## Importing VCF data into GenomicsDB
We explain with an example. On the host machine, the working directory looks like this:
```
├── reference
│   ├── Homo_sapiens_assembly19.dict
│   ├── Homo_sapiens_assembly19.fasta
│   └── Homo_sapiens_assembly19.fasta.fai
├── test_query.json
├── vcfs
│   ├── t0.vcf.gz
│   ├── t0.vcf.gz.tbi
│   ├── t1.vcf.gz
│   ├── t1.vcf.gz.tbi
│   ├── t2.vcf.gz
│   ├── t2.vcf.gz.tbi
│   ├── t6.vcf.gz
│   ├── t6.vcf.gz.tbi
│   ├── t7.vcf.gz
│   ├── t7.vcf.gz.tbi
│   ├── t8.vcf.gz
│   └── t8.vcf.gz.tbi
├── vid_GT_only_Homo_sapiens_assembly19.json
```

We are going to import all the VCFs in the directory _vcfs/_ into a GenomicsDB partition. Note that 
the VCFs must be block compressed and indexed.

The import command:

```bash
#!/bin/bash
export IMPORT_TAG=0.9.2-93da4b0-0.6

mkdir -p $PWD/workspace

docker run -v $PWD:/data \
  intelhlsgenomicsdb/vcf_importer:$IMPORT_TAG \
  vcf_importer \
  -R /data/reference/Homo_sapiens_assembly19.fasta \
  -V /data/vid_GT_only_Homo_sapiens_assembly19.json \
  -i /data/vcfs \
  --range 1:1-10000000 \
  -C /data/  \
  -o /data/workspace
```

The command imports the data in the VCF files into a GenomicsDB array called _TEST0_ in the directory 
called workspace in the working directory. _vcf_importer_ is a script that reads the VCF headers, 
creates the callsets json file and invokes the _vcf2tiledb_ executable to import the data into 
GenomicsDB. Arguments to the script:
* -R: path to reference genome
* -V: path to vid json file for the specific reference assembly
* -i: directory containing the block compressed and indexed VCF files you wish to import
* --range: contig range for which you wish to import data
* -C: path to callsets.json. If this is a directory, a _callsets.json_ will be created in this 
directory (from the VCF headers).  If this is a file, then the file will be treated as an input 
_callsets.json_ file for the import process.
* -o: directory in which the GenomicsDB array named _TEST0_ will be created. This directory must exist 
before the import command is invoked (hence, the mkdir command above).

## Querying the data
The one extra input to the query command is a file containing the intervals/positions to be queried.  
Here is an example:
```
cat test_query.json
[
    [
        {
            "1": [ 12140, 13000 ]
        },
        {
            "1": 17385
        },
        {
            "1": 8029501
        }
    ]
]
```

You should modify the inner list to define which positions/intervals you wish to query. For example, 
the first dictionary in the inner list specifies a query for chromosome interval 12140 to 13000 for 
chromosome 1 while the second dictionary specifies a query for position 17385.

The query command:

```bash
export QUERY_TAG=0.9.2-93da4b0-0.5

docker run -v $PWD:/data \
  intelhlsgenomicsdb/genomicsdb_querier:$QUERY_TAG \
  genomicsdb_querier.py \
  -R /data/reference/Homo_sapiens_assembly19.fasta \
  -C /data/callsets.json \
  -V /data/vid_GT_only_Homo_sapiens_assembly19.json \
  -o /data/workspace \
  --positions /data/test_query.json \
  [ --print-AC | --print-calls ]
```

_genomicsdb_querier.py_ is a wrapper around _gt_mpi_gather_ that creates a temporary query JSON file 
using the inputs provided. Arguments:
* -R: path to reference genome
* -C: path to _callsets.json_. This could be produced by the import command (above) or user defined.
* -V: path to vid JSON file.
* -o: path to workspace containing the _TEST0_ GenomicsDB array
* --positions: path to file containing list of query positions/intervals described above.
* --print-AC: print allele counts
* --print-calls: print _VariantCalls_ in the format described [[ here|Querying-GenomicsDB#query-result-format ]]

## Notes
* The data used in the example is in the import container under /test_data (except for the reference genome which is big).
* Paths to all files/directories must be valid inside the Docker container - ensure that your 
volumes are 'mounted' at the correct location.
* IMPORT_TAG and QUERY_TAG are different.