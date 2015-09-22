### Why we want to revert bam files back to fastq file?

Sometimes, you get a processed bam file from other resources, but you want to map it with another aligner. For example, the old
BWA aln does not keep the information for soft-cliped reads which are useful for structural variant detection, the new BWA mem is 
more suitable for this. So, I want to dump bam to fastq first and then realign it with BWA MEM.

After googleling around, I found:  
* [bedtools](http://bedtools.readthedocs.org/en/latest/content/tools/bamtofastq.html) `bedtools bamtofastq [OPTIONS] -i <BAM> -fq <FASTQ>`
* [picard](https://broadinstitute.github.io/picard/command-line-overview.html) `samtofastq`
* [samtools](http://www.htslib.org/doc/samtools-1.1.html) `samtools bam2fq`

It turns out that it is not that straightforward.

###Use samtools 
Read this [(howto) Revert a BAM file to FastQ format](http://gatkforums.broadinstitute.org/discussion/2908/howto-revert-a-bam-file-to-fastq-format) in the GATK forum back to 2013.

####Shuffle the reads in the bam file

Action
Shuffle the reads in the bam file so they are not in a biased order before alignment by running the following HTSlib command:

`htscmd bamshuf -uOn 128 aln_reads.bam tmp > shuffled_reads.bam` 
Expected Result
This creates a new BAM file containing the original reads, which still retain their mapping information, but now they are no longer sorted.

The aligner uses blocks of paired reads to estimate the insert size. If you don’t shuffle your original bam, the blocks of insert size will not be randomly distributed across the genome, rather they will all come from the same region, biasing the insert size calculation. This is a very important step which is unfortunately often overlooked.

####Revert the BAM file to FastQ

Revert the BAM file to FastQ format by running the following HTSlib command:  
`htscmd bam2fq -a shuffled_reads.bam > interleaved_reads.fq` 
Expected Result
This creates an interleaved FastQ file called interleaved_reads.fq containing the now-unmapped paired reads.

**Interleaved simply means that for each pair of reads in your paired-end data set, both the forward and the reverse reads are in the same file, as opposed to having them in separate files.**

####Compress the FastQ file

Action
Compress the FastQ file to reduce its size using the gzip utility:

`gzip interleaved_reads.fq`
Expected Result
This creates a gzipped FastQ file called interleaved_reads.fq.gz. This file is ready to be used as input for the Best Practices workflow.

BWA handles gzipped fastq files natively, so you don’t need to unzip the file to use it later on.

####Note for advanced users

If you’re feeling adventurous, you can do all of the above with this beautiful one-liner, which will save you a heap of time that the program would otherwise spend performing I/O (loading in and writing out data to/from disk):

`htscmd bamshuf -uOn 128 aln_reads.bam tmp | htscmd bam2fq -a - | gzip > interleaved_reads.fq.gz` 

the `htscmd bamshuf` and `htscmd bam2fq` are legacy code, now they are under `samtools`.  

But the point is that **you need to shuffle the reads in the bam file**.  
from [Hengli](https://github.com/samtools/htslib/issues/26), the author of samtools and bwa:  
>The right way to create paired fastq for bwa is:  
`htscmd bamshuf -uO in.bam tmp-prefix | htscmd bam2fq -as se.fq.gz - | bwa mem -p ref.fa -`
**If your bam is coordinate sorted, it is important to use "bamshuf" to change the ordering; otherwise bwa will fail to infer >insert size for a batch of reads coming from centromeres.**

BWA `-p` flag
>"If mates.fq file is absent and option -p is not set, this command regards input reads are single-end. If mates.fq is present, this command assumes the i-th read in reads.fq and the i-th read in mates.fq constitute a read pair. If -p is used, the command assumes the 2i-th and the (2i+1)-th read in reads.fq constitute a read pair (such input file is said to be interleaved). In this case, mates.fq is ignored. In the paired-end mode, the mem command will infer the read orientation and >the insert size distribution from a batch of reads.

**Updated version**   
>Bam2fq -s only helps when your bam contains singletons. If your bam contains both ends, that option has no effect

`samtools bamshuf -uon 128 in.bam tmp-prefix | samtools bam2fq -s se.fq.gz - | bwa mem -p ref.fa -`

### Use bedtools

From the [bcbio source code](https://github.com/chapmanb/bcbio-nextgen/blob/01a6d99c7a8bb7a73ee35313c8af4c6b4d8c66fe/bcbio/ngsalign/bwa.py#L41-L43):

```python
cmd = ("{samtools} sort -n -o -l 1 -@ {num_cores} -m {max_mem} {in_bam} {prefix1} "
                       "| {bedtools} bamtofastq -i /dev/stdin -fq /dev/stdout -fq2 /dev/stdout "
                       "| {bwa} mem -p -M -t {num_cores} -R '{rg_info}' -v 1 {ref_file} - | ")
```
from the bedtools mannual page:  
> BAM should be sorted by query name (samtools sort -n aln.bam aln.qsort) if creating paired FASTQ with this option.

That's why the command above uses `samtools sort -n` to sort by name. My bam files are huge (300Gb). This might be very time consuming.  

### Speedseq realign

In the [speedseq pipeline](https://github.com/hall-lab/speedseq), there is a script called `bamtofastq.py` 
```
bamtofastq.py
author: Ira Hall (ihall@genome.wustl.edu) and Colby Chiang (cc2qe@virginia.edu)
version: $Revision: 0.0.1 $
description: Convert a coordinate sorted BAM file to FASTQ
             (ignores unpaired reads)

positional arguments:
  BAM                   Input BAM file(s). If absent then defaults to stdin.

optional arguments:
  -h, --help            show this help message and exit
  -r STR, --readgroup STR
                        Read group(s) to extract (comma separated)
  -n, --rename          Rename reads
  -S, --is_sam          Input is SAM format
  -H FILE, --header FILE
                        Write BAM header to file
```


