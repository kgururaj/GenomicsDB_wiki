Since GenomicsDB is a project under development, we do expect bugs to show up from time to time.

Please do go through the [[common issues wiki page|Common-issues]] before filing a bug report. 

### Build TileDB/GenomicsDB in debug mode
See the [[build wiki|Compiling-GenomicsDB#building]] page for details on building TileDB/GenomicsDB. Omit the _OPENMP=1_ and _BUILD=release_ parameters - instead add the parameter _BUILD=debug_. Typically, the following commands should be adequate

    make clean-all
    make BUILD=debug <other build options - Java, libcsv etc>

### Enable core dumps
A core dump file contains the state of the program at the time of the crash and can be used to debug the program offline, even on a different machine, without any input data.

However, on most systems, core dump generation is disabled by default. You can find out how to enable generation of core dump files on the web; for example, ["Enable Linux Core Dump"](http://www.idimmu.net/2013/06/21/enable-linux-core-dump/) and ["HOWTO enable core-dumps"](http://en.linuxreviews.org/HOWTO_enable_core-dumps) for GNU/Linux and [" Generating core dumps in el capitan "](https://forums.developer.apple.com/thread/43006) for MacOSX.

Ultimately, if you run _ulimit -c_, you should see a non-zero value (or the string _unlimited_).

### Producing core dump files
Run your program; when it crashes/aborts, the system will produce a core dump file. Generally, the core dump file is created in the program working directory and is named _core.\<pid\>_

### Create an issue on Github
**WARNING**: the core dump contains program state at the time of the crash - this means that it will contain a small amount of the original data (from the TileDB/GenomicsDB array or VCF files). It is **YOUR RESPONSIBILITY** to ascertain that uploading core dump files to the public GenomicsDB Github repo isn't a violation of your institute policies and regulations.

Create an [issue on Github](https://github.com/Intel-HLS/GenomicsDB/issues) and provide the following information:
* System type: GNU/Linux or MacOSX
* Description of the error
* Label the issue as a bug
* The git commit id of the GenomicsDB source used

Attach the following to the Github issue:
* The exact executable/library binary used to produce the core dump file
* The core dump file

**WARNING**: the core dump contains program state at the time of the crash - this means that it will contain a small amount of the original data (from the TileDB/GenomicsDB array or VCF files). It is **YOUR RESPONSIBILITY** to ascertain that uploading core dump files to the public GenomicsDB Github repo isn't a violation of your institute policies and regulations.