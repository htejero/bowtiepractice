---
title: "Short read aligners: Bowtie "
author: "Hector Tejero"
date: "November 27, 2017"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Bowtie 

In order to use this tutorial, you have to download Bowtie from [SourceForge](https://sourceforge.net/projects/bowtie-bio/files/bowtie/1.2.1.1). Then unzip the file in a folder you choose. Now, open a terminal and go to the folder. The folder contains the following executables and directories. 

```{bash cars}
ls  
```


### Indexing the reference

The first thing to do when using a Burrows-Wheeler based aligner such as bowtie is to index the genome reference. For long genomes, as those of mammals, this is a highly time-consuming process. So, pre-built indexed references can be downloaded from the [Bowtie Web](http://bowtie-bio.sourceforge.net/index.shtml) (right column, _Pre-built indexes_ section) or from the [iGenomes](https://support.illumina.com/sequencing/sequencing_software/igenome.html) web. 

Now, download the _S. cerevisiae_ prebuilt reference and unzip it in our working directory. 
```{bash, eval = FALSE}
wget ftp://ftp.ccb.jhu.edu/pub/data/bowtie_indexes/s_cerevisiae.ebwt.zip
unzip s_cerevisiae_c.ebwt.zip
```

In order to index a reference, the `bowtie-build` command is used. 

```{bash}
./bowtie-build -h
```

The `bowtie-build` command need two parameters: `<reference_in>` is the reference genome as a fasta file. `<ebwt_outfile_base>` is the name of the index that are going to be generated, which will have `ebwt` extension


```{bash}
./bowtie-build ./genomes/NC_008253.fna e_coli_index
```

When the indexing ends, several new files are generated. 

```{bash}
ls *.ebwt
```

We can inspect the reference index using the `bowtie-inspect` command. We will use the `-s` option, which prints only a summary of the index. By deafult, prints the FASTA record of the indexed nucleotide sequence. (see `bowtie-inspect -h` for other options)

```{bash}
./bowtie-inspect -s e_coli_index
```

### Aligning reads

Once the reference has been indexed we can now align the reads to the genome using the `bowtie` command. This command has a lot of options: 

```{bash}
./bowtie -h
```

An interesting option to play with bowtie is the `-c` option, which allows to introduce the sequence to align using the command line. 


```{bash}
./bowtie -c e_coli_index ATAA
```

Let's try a more complex read, for example this _ATTGTAGTTCGAGTAAGTAATGTGGGTTTG_

```{bash}
./bowtie -c e_coli_index ATTGTAGTTCGAGTAAGTAATGTGGGTTTG
```

In the output we can see that the read fails to be aligned. 

```{bash}
./bowtie -c  s_cerevisiae ATTGTAGTTCGAGTAAGTAATGTGGGTTTG
```

and that's because this sequence comes from _S. cerevisiae_. 

Bowtie outputs one alignment per line. The  [bowtie default output](http://bowtie-bio.sourceforge.net/manual.shtml#default-bowtie-output) of the aligner is a collection of 8 fields separated by tabs. The most important column is the fourth, which is the 0-based leftomost position of the alignment. This format is usually stores with .map extension.

However, the most used alignment format is the SAM/BAM format. The `-S` or `--sam` option outputs the hits in SAM format. 


```{bash}
./bowtie -cS  s_cerevisiae ATTGTAGTTCGAGTAAGTAATGTGGGTTTG
```

We can also redirect the output to a file: 

```{bash}
./bowtie -c  s_cerevisiae ATTGTAGTTCGAGTAAGTAATGTGGGTTTG > alignment.sam
```

To store as a more efficient BAM file, use samtools: 

```{bash, eval = FALSE}
samtools view -Sb alignment.sam > alignment.bam
```

Let's see the differences in size: 

```{bash}
ls -h alignment.*
```

### Using reads from files 

Altought the `-c` option is quite useful to play and learn, we usually don't want to align single reads from the command line, but reads coming from big FASTQ files. Actually, the FASTQ format is the default bowtie input format. 

Bowtie comes with some example files in the `reads` directory

```{bash}
ls ./reads/
```

Let's explore the files. 

First, let's see reads in a multifasta (`.fa`) file. As you see, reads have no quality

```{bash}
cat reads/e_coli_1000.fa | head
```

FASTQ files, however, come with qualities associated. The most important thing to remember is that in a FASTQ file, there are 4 lines per read: 

- \@Read Information
- sequence of the read
- +Read Information (usually a blank line)
- qualities in ASCII codification 

```{bash}
cat reads/e_coli_1000.fq | head -n 8
```

If you look at the options of bowtie, you'll see that you can change the codification of the qualities: the SANGER codification ASCII+33 (`--phred33-quals`) is the default option, but other qualities as `--phred64-quals` or `--solexa-quals` are allowed.


#### Single-end experiment

If the reads come from a single-end experiment they are stored in a single fastq file `e_coli_1000.fq`. To align these reads the command is: 

```{bash}
./bowtie -S e_coli reads/e_coli_1000.fq > single_end_alignment.sam
```

A lot of reads (about a 30%) are not aligned. Let's take a look to the SAM file generated


```{bash}
cat single_end_alignment.sam | head
```


```{bash}
cat single_end_alignment.sam | wc
```

It has 1003 lines. If means that all the reads (aligned or not are stored in the SAM file). See, for example the `r3` read. In case we don't want to store the unaligned reads, we just use the `--no-unal` option. 

#### Paired-end experiments

Now, we usually have reads coming from paired-end experiments. When this is the case, we usually have two FASTQ files, with filenames ending in "_1.fq" and "_2.fq". In order to tell bowtie this is the case, we have to use the `-1` and `-2` options. 

```{bash}
./bowtie e_coli -1 reads/e_coli_1000_1.fq -2 reads/e_coli_1000_2.fq paired_end_alignment.map
```

or, with SAM output

```{bash}
./bowtie --sam e_coli -1 reads/e_coli_1000_1.fq -2 reads/e_coli_1000_2.fq paired_end_alignment.sam
```

If you look into the map and SAM files, you'll see that they have 2000 and 2003 linesas each read alignment is written in one line. But take into account that now consecutive lines are related. 

#### Some other options

Two intersting options are the `-t` and `-p` options. The first one prints the execution times. The second allows to state the number of aligment threads to launch:


```{bash}
./bowtie -t -p 2 e_coli reads/e_coli_1000.fq > single_end_alignment.map
```







