1. _Version \> 0.3.0 and \<= 0.4.0_: If your git commit id is 
[6bc801d1b1881](https://github.com/Intel-HLS/GenomicsDB/commit/6bc801d1b1881bc5db6829054e7b336936bdc00c) or older and 
the commit id is  
[860400623ed4](https://github.com/Intel-HLS/GenomicsDB/commit/860400623ed44a155018d691c3427352b166701d) or newer, then
follow the instructions on this page.
1. _Version 0.3.0 or older_: If your git commit id is 
[e10bf412ddd35](https://github.com/Intel-HLS/GenomicsDB/commit/e10bf412ddd35c5aa2c4739aea266d9e5a460acd) or older, then 
follow the instructions on [[the page for building GenomicsDB 0.3 or older|Building-GenomicsDB-Version-0.3.0]].
1. _Version \> 0.4.0_: If your git commit id is 
[6bc801d1b1881](https://github.com/Intel-HLS/GenomicsDB/commit/6bc801d1b1881bc5db6829054e7b336936bdc00c) or newer, then
follow the instructions on the [[ current build page | Compiling-GenomicsDB ]].

## Requirements:
###Mandatory pre-requisites:
* We have tested TileDB/GenomicsDB on the following platforms:
    * GNU/Linux:
        * CentOS 6 and 7 (almost identical to RHEL 6 and 7). Most of our heavy testing is performed on CentOS-7 systems.
        * Ubuntu Trusty (14.04)
    * MacOSX: We have tested with version 10.11 (El Capitan).
* Dependencies from TileDB
    * Zlib headers and libraries
    * OpenSSL headers and libraries
    * Example installation commands:
        * On CentOS/RedHat systems:

                sudo yum -y install openssl-devel zlib-devel openssl-static

        * On Ubuntu systems:

                sudo apt-get install zlib1g-dev libssl-dev

        * On MacOSX, you can use [Homebrew](http://brew.sh/) to obtain the OpenSSL library.

                brew install openssl

*  C++ compiler: A C++ 2011 compiler.
    * gcc version >= 4.8. We have been testing with gcc-4.9.1.
    * We have tested with clang version >= 7.3.0 on MacOSX. 
    * Installing a new version of gcc/g++
        * CentOS/RedHat systems: You can use the [software collections](https://www.softwarecollections.org/en/docs/) repository and install the package [devtoolset-3](https://www.softwarecollections.org/en/scls/rhscl/devtoolset-3/) or [devtoolset-4](https://www.softwarecollections.org/en/scls/rhscl/devtoolset-4/).

                sudo yum install centos-release-scl
                sudo yum install devtoolset-3
                scl enable devtoolset-3 bash

        * Ubuntu: We use the Ubuntu Toolchain PPA to obtain new versions of gcc

                sudo add-apt-repository ppa:ubuntu-toolchain-r/test
                sudo apt-get update
                sudo apt-get install g++-4.9
                sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.9 60 

* [Google Protocol Buffer](https://github.com/google/protobuf) Google protocol buffer is a mandatory pre-requisite version 0.4.0 onward. We use protocol buffers to exchange configuration parameters, headers and callset/sample id to TileDB row index between Java and C++. Note that Ubuntu-14.04 LTS as well as CentOS 6 and 7 releases use protobuf version 2.5.0. However, we specifically depend on protobuf version 3.0.2. We recommend to build it locally and link it using appropriate environment variables and not overwrite existing system protobuf version. To build protobuf from source:

        git clone https://github.com/google/protobuf/tree/3.0.x
        cd protobuf
        autogen.sh
        ./configure --prefix=/path/to/local/installation --with-pic
        make -j4
        make install
        export PROTOBUF_LIBRARY=/path/to/local/installation # For our Makefile to work
        export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PROTOBUF_LIBRARY/lib
   
* *NOTE*: We use git submodules to pull in the remaining mandatory dependencies - you can skip directly to the 
[[optional pre-requisites|Compiling-GenomicsDB#optional-pre-requisites]] section if you do not wish to manually fetch 
and build the following mandatory dependencies.
* TileDB

        git clone https://github.com/Intel-HLS/TileDB.git

* [Rapidjson library](https://github.com/miloyip/rapidjson): Parameters are passed to TileDB tools/examples through a JSON file - Rapidjson is used to parse this JSON file. The library is a header-only library - no compilation needed.

        git clone https://github.com/miloyip/rapidjson

* [Htslib](https://github.com/samtools/htslib) for parsing and exporting VCFs. We maintain a [fork of htslib](https://github.com/Intel-HLS/htslib) with some modifications that are needed for use with GenomicsDB.

        git clone https://github.com/Intel-HLS/htslib
        cd htslib
        git checkout intel_mods
        make -j 8

###Optional pre-requisites
* _OpenMPv4_: We use directives from OpenMP specification v4. This is supported on gcc versions >= 4.9.0. If you enable OpenMP by setting OPENMP=1 during the build process using an older compiler, you will see compilation errors around the line listed below

        #pragma omp parallel for default(shared) num_threads(m_num_parallel_vcf_files) reduction(l0_sum_up : combined_histogram)

    You can disable OpenMP by omitting the flag OPENMP=1 during compilation (see below). You may lose some performance during loading without OpenMP.
    
    On MacOSX systems, OpenMP is disabled by default during compilation.

* _For executables_:  If you wish to produce any of the executables provided by GenomicsDB, an MPI compiler, library and runtime are required. We have tested
with reasonably new versions of OpenMPI, MPICH and MVAPICH2.
    If you wish to only build the combined TileDB/GenomicsDB shared library and the Java jar (see below), an MPI compiler is not needed.
    * On CentOS/RedHat systems:

            sudo yum install mpich-devel

    * On Ubuntu systems:

            sudo apt-get install mpich 

    * On MacOSX:

            brew install mpich

* _For importing CSV files_: If you wish to import [[CSV data into TileDB|Importing-CSVs-into-GenomicsDB]], then you need 
[libcsv](https://sourceforge.net/projects/libcsv/). You also need to pass special flags while invoking make (see below).

    * On CentOS/RedHat systems: if you have the [EPEL repo](https://fedoraproject.org/wiki/EPEL) installed and enabled, you 
can install the libcsv packages using yum:

            sudo yum install libcsv libcsv-devel

    * Ubuntu systems: on systems with Vivid(15.04) or newer:

            sudo apt-get install libcsv3 libcsv-dev
    
    * Build from source for older Ubuntu systems:

            wget -O libcsv.tar.gz http://downloads.sourceforge.net/project/libcsv/libcsv/libcsv-3.0.3/libcsv-3.0.3.tar.gz
            tar xzf libcsv.tar.gz
            cd libcsv-<version> && ./configure && make

    * MacOSX:

            brew install libcsv

* _For the Java/JNI interface_
  * Java SDK version 8.
  * Apache Maven 3
    * The other Java dependencies are pulled in by Maven as needed. 

## Building
* Get the right branch based on what you wish to do - see the other pages for which branch to get. If you do not know which branch to use, the *master* branch is your best bet.
* To get dependencies using git submodule, run:

        git clone --recursive https://github.com/Intel-HLS/GenomicsDB.git

* If you have an existing git repository and wish to pull in the latest changes:

        git pull origin master
        git submodule update --recursive --init

* Make sure you have the required gcc version in your PATH.
* Assuming you want to use the dependencies pulled in by git and you have the MPI compiler (mpicxx) in your PATH
        
        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make BUILD=release OPENMP=1 -j 8

* Compiling in debug mode:

        #debug mode - assertions enabled, can use gdb for stepping, no OPENMP (can enable with the OPENMP=1 flag)
        make BUILD=debug -j 8

* If you do not have the MPI compiler in your PATH (note that the trailing slash is required in the following command):
        
        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release OPENMP=1 -j 8

* If you wish to import CSV data and libcsv is installed at a location where the compiler can find the header file and 
library (for example under /usr), then enable libcsv usage

        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release USE_LIBCSV=1 OPENMP=1 -j 8

    If you have downloaded libcsv [from sourceforge](https://sourceforge.net/projects/libcsv/) and compiled it at a 
custom location, then pass the directory to the make command

        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release LIBCSV_DIR=<libcsv_directory> OPENMP=1 -j 8

    The build process assumes that the library file is located in _\<libcsv_directory\>/.libs_ or _\<libcsv_directory\>/lib_  (the default location in the 
build process of libcsv).

* On a MacOSX system, assuming you installed the pre-requisites using Homebrew under /usr/local/opt, the following command can be used:

        make MPIPATH=/usr/local/opt/mpich/bin/ LIBCSV_DIR=/usr/local/opt/libcsv/ OPENSSL_PREFIX_DIR=/usr/local/opt/openssl BUILD=release

If you have downloaded and compiled the dependencies manually, use the following commands:

* Compiling in release mode

        #release mode - O3, NDEBUG - assertions disabled, OpenMP enabled
        make MPIPATH=<mpi_package_dir>/bin/ TILEDB_DIR=<TileDB_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8

* If you have the MPI compilers (mpicc, mpicxx) in your PATH, you can drop the MPIPATH argument

        #release mode
        make TILEDB_DIR=<TileDB_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8

* Compiling in debug mode (for developers)

        #debug mode - assertions enabled, can use gdb for stepping, no OPENMP (can enable with the OPENMP=1 flag)
        make MPIPATH=<mpi_package_dir>/bin/ TILEDB_DIR=<TileDB_dir> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=debug -j 8

* Compiling with a custom htslib source directory:

        make MPIPATH=<mpi_package_dir>/bin/ TILEDB_DIR=<TileDB_dir> HTSDIR=<htslib_directory> RAPIDJSON_INCLUDE_DIR=<rapidjson_dir>/include BUILD=release OPENMP=1 -j 8

## Java and Apache Spark interface for TileDB/GenomicsDB
* With the BUILD_JAVA flag enabled, the build environment compiles both Java and Apache Spark interfaces of GenomicsDB.
* Remember to use Java SDK version 8.
* For GenomicsDB versions >= 0.4.0 (commit id [860400623ed44a15](https://github.com/Intel-HLS/GenomicsDB/commit/860400623ed44a155018d691c3427352b166701d) or newer), use the following steps. Otherwise use the steps described in [[Building GenomicsDB Version 0.3.0]].

* To build the jar:
        
         #On GNU/Linux        
        make MPIPATH=<mpi_package_dir>/bin/ BUILD=release LIBCSV_DIR=<libcsv_directory> OPENMP=1 BUILD_JAVA=1 JNI_FLAGS="-I<java_SDK_dir>/include -I<java_SDK_dir>/include/linux"
        
        #On MacOSX
        make MPIPATH=/usr/local/opt/mpich/bin/ LIBCSV_DIR=/usr/local/opt/libcsv/ OPENSSL_PREFIX_DIR=/usr/local/opt/openssl BUILD=release BUILD_JAVA=1 JNI_FLAGS="-I/System/Library/Frameworks/JavaVM.framework/Headers"

    You don't need an MPI compiler and library to only build the jar and the shared TileDB/GenomicsDB library (no executables will be built).

        make BUILD=release DISABLE_MPI=1 BUILD_JAVA=1 JNI_FLAGS="-I<java_SDK_dir>/include -I<java_SDK_dir>/include/linux"

    The jar file genomicsdb-<version>.jar will be created in the bin/ directory. You can 
[install](https://maven.apache.org/guides/mini/guide-3rd-party-jars-local.html) this jar file into your local Maven 
repository for use in downstream Maven/Gradle build systems using:

        mvn install:install-file -Dfile=bin/genomicsdb-<version>.jar -DpomFile=pom.xml

Caveats:
* The shared library (libtiledbgenomicsdb.so) that is packaged in the jar depends on GNU libc (glibc). If you 
compile the library on one system and run it on another system with a newer version of glibc, the library should work 
since glibc is backward compatible (for example, you can compile the library on CentOS-6 and run it on CentOS-7).
However, if you do the reverse, then very likely you will see errors about missing symbols when loading the library. A 
quick check is to run _ldd bin/libtiledbgenomicsdb.so_; you should *NOT* see errors about missing symbols in a 
correctly functioning configuration.

## Building a distributable jar

Note: For most users this section is not applicable. If you are interested in packaging and distributing a jar file that 
must not contain any distribution specific dependencies (MPI shared libraries for example), follow the steps in this 
section:

* The following libraries should be statically linked in:
    * libgcc: We use the option "-static-libgcc"
    * stdc++: We use the option "-static-libstdc++" while building TileDB/GenomicsDB to create portable binaries. Please 
ensure that your build system has the static version of this library installed.
        * On CentOS/RedHat systems:

                sudo yum -y install libstdc++-static

    * OpenSSL

* The following libraries are dynamically linked - however, they are backward compatible. Hence, you should build your 
jar on an 'older' system (I build on CentOS-6).
    * zlib
    * glibc

* Command

        make BUILD_JAVA=1 JNI_FLAGS="-I<java_SDK_dir>/include -I<java_SDK_dir>/include/linux" BUILD=release DISABLE_MPI=1 DISABLE_OPENMP=1 MAXIMIZE_STATIC_LINKING=1

