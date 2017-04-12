# Objective

Sorting big NGS data file in the context of distributed cluster and high performance computing, Version 1.0.

## Sections:

1) Release notes <br />
2) Installation <br />
3) Algorithm <br />
4) Architectures <br />
5) Memory usage <br />
6) CPU usage <br />
7) Inputs <br />
8) Outputs <br />
9) Compiler <br />
10) Sample test  <br />
11) Configuration <br />
12) Parallel file system configuration <br />
13) Lustre optimization
14) Network file system configuration <br />
15) Results <br />
16) Example of code <br />
17) Options <br />
18) Improvements and future work <br />
19) Authors and contacts  <br />

### 1) Release notes 

Release 1.0 from 28/03/2017

1) We have redesign the communications during the sorting. <br />
Communication are optimized for power of 2 in processor number.  <br />
This has been done to scale gigabit ethernet network and for a better load balancing of the computing and memory pressure. <br />
With processors in power of 2 we avoid extra communication. <br />
All Results are the same than previous release. <br />

New tests and benchmarking will follow.

2) Remove memory leaks. <br />
Comments added. <br />

Release 1.0 from 25/11/2016

1) Bug fix at the beginning of the programm during the parsing of the SAM.<br />
2) Need to test if the MPI sctructures scale very large buffer (above 5Gb per job).<br />

Release 1.0 from 23/11/2016

1) Remove the limit in  MPI_file_read_at  of 1 Gb per read buffer.<br />
SAM data are first stored in intermediate buffer and copy to the main SAM buffer see in mpiSORT.c (line 344)<br />
2) Replace MPI_Create_struct with a local copy (in write.c line 2712) <br />
3) Cleaning of the code and add comments <br />
4) Tested with a beegfs scratch (4 MDS) and 10gb ethernet, it scales to 200Gb sam file and 80 cpu and takes 10mn for reading and 10mn sorting). <br />

Release 1.0 from 18/11/2016

1) Trimming of the memory after big or multiple free() with malloc_trim().<br /> 
Efficient with Linux but not tested on BSD of IOX. <br />
2) Remove memory leaks.<br />
3) Add/remove  comments.<br />

Release 1.0 from 04/11/2016

1) Due to MPI packing overhead we replace with a local copy (write.c). See in future if MPI_Unpack is stable.  <br />
2) File extension becomes  .gz (not bam) <br />
3) Discordant and unmapped are computed first.  <br />
4) Cleaning of memory.  <br />

Release 1.0 from 11/10/2016

1) In this release we use the local data buffer for re-reading instead of the linux client cache. <br />
Because for slow network the time to get the data back from the client or server cache can be long. <br />
Now re-reading the data is intantaneous. <br />
2) Add documentation and free some variables. <br />

Release 1.0 from 06/10/2016

1) The previous version didn't sort the offset destination before the shuffle. This bug is fixed. <br />
2) New packaging with autools for distribution. <br />

Release 0.9 from 29/07/2016

1) Bug fix when data sample is too small and some jobs have nothing to sort <br />
2) Bug fix when jobs under 6 <br />
3) We add new option -n for sorting by name <br />
4) the parallel merge sort has been replaced with bitonic sorting (25% percent gain in speed-up) <br />
5) Refactoring and cleaning of the code <br />

### 2) Installation

You need to install zlib librairies <br />
You need automake 1.15 for the installation. <br />
You can install automake and autoconf in differents directories and export the path like this: <br />
export PATH=path_to_automake/automake-1.15/bin:path_to_autoconf/autoconf-2.69/bin:$PATH <br />
(The best is to add it in your .bashrc and to source it).

Download from git. In the folder mpiSORT type: <br />
./configure && make install && make <br />

or for distribution: <br />
make dist  <br />
tar xzf .tar.gz  <br />
cd mpisort-1.0  <br />
./configure && make install && make<br />

for passing mpi path: <br />
./configure CC=mpi_bin_path  <br />
add --prefix in configure if you need  <br />

example of command line in a terminal:  <br />

mpirun -n 5 psort INPUT_FILE OUTPUT_DIR -q 0  <br />

-q is for quality filtering of the reads.

### 3) Algorithm

Sorting a file is all about IO's and shuffling (or movements) of data. 

We have developed a real parallel and distributed file system aware program to overcome some issues encounter with traditionnal tools like Samtools, Sambamba, Picard. We propose a novel approach based on message passing interface paradigm (MPI) and distributed memory computer. <br />

There are several aspects in this sorting algorithm that are important: the bitonic-sort, the shuffling of the data and the distributed cache. <br />

The parallel merge-sort has been replaced with a bitonic merge-sort. The bitonic sort is a real parallel sorting algorithm and work on parallel architectures. The complexity of the bitonic is of (log(n))^2 instead of nlog(n) with the parallel merge-sort. The bitonic sorter has been developped using MPI message passing primitives and is inspired from the book of Peter S. Pacheco "Parallel programming with MPI". <br />

The shuffing of the data is done through the Bruck method. This method has the advantage of avoiding the shuffle bottleneck (The All2all). Bruck is a log(N) method and scale very well for distributed architectures. <br /> 

As the programs use MPI fonctions for reading and writing you can take advantage of a parallel file system. To speed-up reading and writing you can set the striping of your data the striping tells the number of file servers you want to use and the size of data blocks on each server. You can compare the striping of the file with the mapping process use in Hadoop. This is the way your data are distributed among the servers of your file system. This kind of optimizations accelerate drastically the IO operations.<br />

Ordinary softwares (Samtools, Sambamba, Picard,... ) doesn't take into account the underlying distributed file system and low latency interconnexion when MPI does. <br />

Here are some links fo further reading and MPI technics. <br />

Description of Bruck algorithm:<br />
http://www.mcs.anl.gov/~thakur/papers/ijhpca-coll.pdf<br />
http://authors.library.caltech.edu/12348/1/BRUieeetpds97.pdf<br />
http://hunoldscience.net/paper/classical_sahu_2014.pdf<br />

Description of Bitonic sorting:<br />
https://en.wikipedia.org/wiki/Bitonic_sorter<br />

### 4) Architectures:

We have tested the programm on different architectures.

In the Institut Curie we have a NFS and a 10Gb network. The France Genomic cluster of TGCC (Très Grand Centre de Calcul) of CEA (Bruyeres le Chatel, France) is equiped with Lustre. <br />

Because of the development design the programm is optimized for HPC architecture. This programm runs better on low latency network and parallel file system. <br />

Contact us if you need information.

### 5) Memory:

The total memory used during the sorting is around one and a half the size of the SAM file. <br />
For instance to sort 100Gb sam file (10X WGS 100pb, paired) memory usage is around 150 gb.  <br /> 

### 6) CPU usage:

Due to the bitonic sorting the algorithm is optimized for power of 2 number of CPU. <br />
So chose 2,4,8,16,32... for better performances.  <br />

It works also for other number of CPU but the algorithm add extra communications to fit power of 2 in bitonic part. <br />
For no power 2 extra memory is needed for the rank 0. <br />
This rank is responsible for collecting and dispatching data before and after bitonic. <br />

### 7) Input Data:

A SAM file produced by an aligner with paired reads (BWA, Bowtie) and compliant with the SAM format. The reads must be paired. <br />

### 8) Outputs: 

Output are bam files per chromosome, a bam file for discordant reads and a bam file for unmapped reads. <br />

To index the bam file use tabix like this: <br />
tabix -p sam chrN.bam <br />
(samtools index doesn't work yet)<br />

To uncompress the results:<br />
samtools view -Sh chrN.bam > chrN.sam <br />

Or rename the file with suffix .gz and 
bgzip -d chrN.gz > chrN.sam

### 9) MPI version:

A MPI version should be installed first. We have tested the program with different MPI flavour: OpenMPI 1.10.0 and 1.8.3. <br /> 

### 10) Compiler: 

A C compiler must be present. We have tested the programm with GCC and Intel Compiler. <br /> 

### 11) Sample test:

We furnish a sample sam to sort for testing your installation. <br /> 
We test it with from 1 to 8 jobs and 2 nodes with a normal network and file system. <br /> 

### 12) Parallel file system configuration:

MPI improves IOs by means of parallelization. 

On parallel file system like Lustre, GPFS one way of accelerating IO is to stripe them across servers. 
You chose the number of servers with the striping factor and the size of chunks with the striping size. 

Before running and compile you must tell the programm how the input data is striped on Lustre and how to stripe the output data.
The parallel writing and reading are done via the MPI finfo structure in the first line of mpiSort.c.

For reading and set the Lustre buffer size.  <br />
To do that edit the code of mpiSort.c and change the parameters in the header part.  <br />
After tuning parameters recompile the application.  <br />

If you are familiar with MPI IO operation you can also test different commands collective, double buffered, data sieving.  <br />
In file write.c in the function writeSam, writeSam_unmapped, writeSam_discordant. <br />

The default parameters of the programm are unharmed for other filesystem. <br />

At TGCC the striping factor is 128 (number of OSSs servers) and the striping size is maximum 2.5 GB a SAM file of 1.2TB could be loaded in memory in less than 1 minute. <br /> 

### 13) Lustre optimization:

The section of the code you can modify in MPI info according to your Lustre configuration: <br /> 

For reading part (mpiSORT.c):<br /> 
line 92 => 96 and 260 => 274 <br /> 

For writing part (in write.c): <br /> 
line 1382 => 1388 <br /> 
line 1483 => 1488 <br /> 
line 1769 => 1776 <br /> 
line 1801 => 1809 <br /> 

The parameters are harmeless for other filesystem. <br /> 

!!!We recommand to test different parameters before setting them once for all. <br /> 

### 14) Network File system configuration:

We don't recommend to use MPI with NFS (it works but it does not scale very well). <br />

If you run the programm on NFS take a power of 2 number of jobs. <br />

For NFS users there is a bug in openMPI. <br />
https://www.open-mpi.org/community/lists/users/2016/06/29434.php

a patch is available here:<br />
https://trac.mpich.org/projects/mpich/attachment/ticket/2338/ADIOI_NFS_ReadStrided.patch

To improve TCP communications over openMPI : <br />
https://www.open-mpi.org/faq/?category=tcp

### 15) Results:

To see results and speed-ups please check out those links

2015: This link was the first presentation of the tools 
http://devlog.cnrs.fr/_media/jdev2015/poster_jdev2015_institut_curie_hpc_sequencage_jarlier.pdf?id=jdev2015%3Aposters&cache=cache

2016: OpenSFS conference Lustre LAD 2016 
http://www.eofs.eu/_media/events/lad16/03_speedup_whole_genome_analysis_jarlier.pdf

### 16) Example of code:

How to launch the program with Torque:

!/bin/bash
MSUB -r mpiSORT_HCC1187_20X_380cpu <br />
MSUB -@ frederic.jarlier@curie.fr:begin,end <br />
MSUB -n 380 <br />
MSUB -T 6000 <br />
MSUB -q large <br />
MSUB -o $SCRATCHDIR/ngs_data/ERROR/output_%I.o <br />
MSUB -e $SCRATCHDIR/ngs_data/ERROR/erreur_%I.e <br />
MSUB -w <br />

mpiSORT_BIN_DIR=$SCRATCHDIR/script_files/mpi_SORT <br />
BIN_NAME=psort <br />
FILE_TO_SORT=$SCRATCHDIR/ngs_data/HCC1187C_20X.sam <br />
OUTPUT_DIR=$SCRATCHDIR/ngs_data/sort_result/20X/ <br />
FILE=$SCRATCHDIR/ngs_data/sort_result/psort_time_380cpu_HCC1187_20X.txt <br />
mprun $mpiSORT_BIN_DIR/$BIN_NAME $FILE_TO_SORT $OUTPUT_DIR -q 0 <br />

### 17) Options 

the -q option is for quality filtering. <br />
the -n for sorting by name <br />

### 18) Improvements

* Make a pile up of the reads to produce VCF.
* Merge aligner and sorting to avoid writing and reading part. 
* Manage Bam file in input, output. 
* Optimize communication for non power of 2 cpu number. 
	     The gather and bradcast should be replace with Bruck. 
* Generate index with the gz output as tabix does. 
* Manage single reads. 
* Mark or remove duplicates. 
* Write SAM files per chromosom. 
* Propose an option to write a big SAM file. 
* Test malloc_trim on BSD or OSX. 
	
### 19) Authors and contacts

This program has been developed by<br />

Frederic Jarlier from Institut Curie and Nicolas Joly from Institut Pasteur<br />

With the help of students from Paris Descartes University <br /> 

and supervised by <br />

Philippe Hupe from Institut Curie <br />

Contacts:

frederic.jarlier@curie.fr <br />
njoly@pasteur.fr <br />
philippe.hupe@curie.fr <br />


