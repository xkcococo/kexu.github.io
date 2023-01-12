---
layout: post
title: Use QIIME 2 to preprocess 16S microbiome gene sequencing files in fastq.gz fromat
date: 2023-1-12
author: Ke Xu
tags: [QIIME2, 16S, sequencing, fastq]
comments: true
---

**The latest QIIME 2 tutorials version [2022.11](https://docs.qiime2.org/2022.11/).
** Please read the tutorials for any questions. There are lots of different functions and plugins provided by QIIME2. Here I only put the bits and pieces of the codes for different procedures together, then use my own files to run it as an simple example.

<br>
**Step 1: Before importing data, first you need to identify the corresponding sequencing file format**

First, we need to compare our sequencing files to the formats listed on [this page](https://docs.qiime2.org/2022.11/tutorials/importing/). 

Here we will use `Casava 1.8 paired-end demultiplexed fastq` as an example. The format of `Casava 1.8 paired-end demultiplexed fastq` is two `fastq.gz` files for each sample L2S357_15_L001_R1_001.fastq.gz and L2S357_15_L001_R2_001.fastq.gz. The R1 and R2 stand for one forward and one reverse sequencing file. 

The underscore-separated fields in this file name are: `the sample identifier`_`the barcode sequence or a barcode identifier`_`the lane number`_`the direction of the read (i.e. R1 or R2)`_`the set number`.


```
qiime tools import  \
  --type 'SampleData[PairedEndSequencesWithQuality]'  \
  --input-path /path/you/saved/sequencing/files/afog16s-all-modified/seq  \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt  \
  --output-path /path/you/saved/your/output/demux-paired-end_demodata.qza
```

Now we have our demultiplexed paired-end sequences output, and ready for trimming adapters.

<br>
**Step 2: Search demultiplexed paired-end sequences for adapters and remove them.**

If adapters were used during PCR procedures, we need to search them in all reads and remove them by Cutadapt. Let's suppose the adapter in all forward sequences is `GACTACCAGGGTATCTAATCCTGTTTGCTCCCC`, and the adapter in all reverse sequences is a `CCTACGGGAGGCAGCAGTGGGGAATATTGCACAAT`.

```
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences demux-paired-end_demodata.qza \
  --p-front-f GACTACCAGGGTATCTAATCCTGTTTGCTCCCC \
  --p-front-r CCTACGGGAGGCAGCAGTGGGGAATATTGCACAAT \
  --p-error-rate 0 \
  --o-trimmed-sequences /path/you/saved/your/output/trimmed-seq.qza \
  --verbose
```

Now we have our trimmed sequence data for denoising.

<br>
**Step 3: Denoising trimmed sequence data with DADA2**

Before perform sequence denoising, we need to use [QIIME 2 view](https://view.qiime2.org/) to visualize our trimmed sequence data, then decide our parameters for denoising. This tutorials 

```
qiime demux summarize \
  --i-data trimmed-seq.qza \
  --o-visualization trimmed-seq.qzv
```

We can drop our trimmed-seq.qzv file to the view window. The parameters of `p-trim-left-f`, `p-trim-left-r`, `p-trunc-len-f`, and `p-trunc-len-r` will be determined by the overall quality of the reads.

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

Now we have our `table-dada2.qza` and `rep-seqs-dada2.qza` for next step.

<br>
**Step 4: Use classifier to generate our taxonomy table**

Here we have two options: 1. [Train your own feature classifier](https://docs.qiime2.org/2022.11/tutorials/feature-classifier/), or 2. Use [pre-traiend feature classifier](https://docs.qiime2.org/2022.11/data-resources/) provided by QIIME 2.

Here we used pre-trained classifier Silva 138 99% OTUs full-length sequences.

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

Now we have our taxonomy table in tsv format.

<br>
**Step 5: Generate feature table for each sample**

We will use `table-dada2.qza` to generate our feature table.

```
#Creating a TSV BIOM table
#first, export your data as a .biom
qiime tools export --input-path table-dada2.qza --output-path exported-feature-table


#Covert .biom to .tsv
biom convert -i exported-feature-table/feature-table.biom -o exported-feature-table/feature-table.tsv --to-tsv
biom head -i feature-table.tsv
```

<br>
**Step 6: Combine your feature table and taxonomy table**

We have two output files `taxonomy.tsv` and `feature-table.tsv`. We can combine them by sequencing id. 

<br>
**Now you have everything you have. Please feel free to perform your downstream statistical analysis.**

