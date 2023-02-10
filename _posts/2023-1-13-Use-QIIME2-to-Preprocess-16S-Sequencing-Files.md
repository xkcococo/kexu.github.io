---
layout: post
title: Preprocessing Microbiome Sequences with QIIME 2: A Guide to Importing, Trimming, Denoising, and Classifying 16S Suquencing Data
date: 2023-1-12
author: Ke Xu
tags: [QIIME2, 16S, sequencing, fastq]
comments: true
---

**The latest QIIME 2 tutorials version [2022.11](https://docs.qiime2.org/2022.11/).** 
This tutorial is based on the latest version of QIIME 2, 2022.11. Before starting, it's recommended to review the QIIME 2 tutorials to understand the different functions and plugins available.


<br>
**Step 1: Importing Data in QIIME2.**

Before importing your data, it is important to identify the correct sequencing file format. The format of your sequencing files can be compared to the formats listed on [this page](https://docs.qiime2.org/2022.11/tutorials/importing/). 

For the purpose of this tutorial, let's use `Casava 1.8 paired-end demultiplexed fastq` as an example. This format consists of two `fastq.gz` files for each sample, with names such as `L2S357_15_L001_R1_001.fastq.gz` and `L2S357_15_L001_R2_001.fastq.gz`. The `R1` and `R2` stand for one forward and one reverse sequencing file, respectively.

The fields in the file name, separated by underscores, are: `sample identifier`,  `barcode sequence or barcode identifier`, `lane number`, `read direction (i.e. R1 or R2)`, and `set number`.

To import your data, run the following code:
```
qiime tools import  \
  --type 'SampleData[PairedEndSequencesWithQuality]'  \
  --input-path /path/you/saved/sequencing/files/afog16s-all-modified/seq  \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt  \
  --output-path /path/you/saved/your/output/demux-paired-end_demodata.qza
```

This will import your demultiplexed paired-end sequences, ready for adapter trimming.


<br>
**Step 2: Searching Demultiplexed Paired-end Sequences For Adapters and Remove Them.**

If adapters were used during PCR procedures, they need to be searched for and removed from all reads using Cutadapt. For the purposes of this tutorial, let's suppose the adapter in all forward sequences is `GACTACCAGGGTATCTAATCCTGTTTGCTCCCC`, and the adapter in all reverse sequences is `CCTACGGGAGGCAGCAGTGGGGAATATTGCACAAT`.

To trim the adapters, run the following code:

```
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences demux-paired-end_demodata.qza \
  --p-front-f GACTACCAGGGTATCTAATCCTGTTTGCTCCCC \
  --p-front-r CCTACGGGAGGCAGCAGTGGGGAATATTGCACAAT \
  --p-error-rate 0 \
  --o-trimmed-sequences /path/you/saved/your/output/trimmed-seq.qza \
  --verbose
```

Let's move on to the next step of denoising our trimmed sequence data.

<br>
**Step 3: Sequence Denoising with DADA2.**

Before we denoise our sequence data, we need to visualize it using the [QIIME 2 view](https://view.qiime2.org/). This allows us to determine the parameters for denoising. To do this, we will use the following command:

```
qiime demux summarize \
  --i-data trimmed-seq.qza \
  --o-visualization trimmed-seq.qzv
```

We can then drop the trimmed-seq.qzv file into the view window. The parameters of `p-trim-left-f`, `p-trim-left-r`, `p-trunc-len-f`, and `p-trunc-len-r` will be determined based on the overall quality of the reads.

To perform the denoising, we will use the following command:

```
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs 'trimmed-seq.qza' \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 220 \
  --p-trunc-len-r 200 \
  --o-representative-sequences rep-seqs-dada2.qza \
  --o-table table-dada2.qza \
  --o-denoising-stats satats-dada2.qza \
  --p-n-threads 24
```

With this command, we have generated `table-dada2.qza` and `rep-seqs-dada2.qza` for creating the taxonomy table and feature table.

<br>
**Step 4: Generating the Taxonomy Table.**

There are two options for generating the taxonomy table: 
1. [Train your own feature classifier](https://docs.qiime2.org/2022.11/tutorials/feature-classifier/), or 2. Use a [pre-traiend feature classifier](https://docs.qiime2.org/2022.11/data-resources/) provided by QIIME 2.

In this tutorial, we will use the pre-trained classifier "Silva 138 99% OTUs full-length sequences."

```
#download classifier
wget -O "silva-138-99-nb-classifier.qza" "https://data.qiime2.org/2020.8/common/silva-138-99-nb-classifier.qza"

qiime feature-classifier classify-sklearn \
  --i-classifier silva-138-99-nb-classifier.qza \
  --i-reads rep-seqs-dada2.qza \
  --o-classification taxonomy.qza

#Export taxonomy info in .tsv format
qiime tools export 
  --input-path taxonomy.qza \
  --output-path exported-feature-table
```

With this, we have generated the taxonomy table in `.tsv` format.

<br>
**Step 5: Generating Feature Table for Each Sample.**

In this step, we will use `table-dada2.qza` to generate our feature table.

```
#Creating a TSV BIOM table
#first, export your data as a .biom
qiime tools export --input-path table-dada2.qza --output-path exported-feature-table


#Covert .biom to .tsv
biom convert -i exported-feature-table/feature-table.biom -o exported-feature-table/feature-table.tsv --to-tsv
biom head -i feature-table.tsv
```

<br>
**Step 6: Combining the Feature Table and Taxonomy Table.**

At this point, you should have two output files: `taxonomy.tsv` and `feature-table.tsv`. These two tables can be combined by using the sequencing id as the key.

<br>
**You're all set! You now have everything you need to perform your downstream statistical analysis. Enjoy!**

