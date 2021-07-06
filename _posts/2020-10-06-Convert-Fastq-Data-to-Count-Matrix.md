---
layout: post
title: Convert Fastq Data to Count Matrix
date: 2020-10-06
author: Dongyuan Wu
tags: [linux, RNAseq, genome]
comments: true
---

**Step 1:** Download the `.gtf` file and `.fa` file for reference genome.

For example, the fastq data are from human genes, then we need to download the human reference genome: GRCh38. I downloaded them from the page of NCBI: https://www.ncbi.nlm.nih.gov/assembly/GCF_000001405.26/

There is no `.fa` file but we can use `.fna` file. And I met a problem that `.gtf` file on this page was empty. So I just downloaded `.gff`file and then convert it to `.gtf`. 

Thus, in this step, I went to the NCBI page, chose source database as `RefSeq` and then downloaded two files:
- Genomic FASTA (.fna)
- Genomic GFF (.gff)

**Step 2:** Convert GFF to GTF (can skip if already have the GTF file)

I used `gffread` from `Cufflinks` package to convert `.gff` to `.gtf`.

```
gffread -T /PATH/GFF_FILENAME.gff -o /PATH/GTF_FILENAME.gtf
```

`GFF_FILENAME.gff` is the GFF file that has been existed, while `GTF_FILENAME.gtf` is the file name you would like to get.

**Step 3:** Index the reference genome with STAR

```
STAR --runMode genomeGenerate --genomeDir /PATH/genome \
--genomeFastaFiles /PATH/FASTA_FILENAME.fna --runThreadN 8 \
--sjdbGTFfile /PATH/GTF_FILENAME.gtf --genomeChrBinNbits 14 \
--genomeSAsparseD 3 --genomeSAindexNbases 12
```

`--runThreadN`, `--genomeChrBinNbits`, `--genomeSAsparseD`, and `--genomeSAindexNbases` are all the parameters can improve the speed or reduce the RAM used.

**Step 4:** Align sequencing reads to the indexed genome

```
STAR --quantMode GeneCounts --genomeDir /PATH/genome --runThreadN 8 \
--readFilesIn /PATH/L001_R1.fastq.gz,/PATH/L002_R1.fastq.gz,/PATH/L003_R1.fastq.gz,/PATH/L004_R1.fastq.gz \
/PATH/L001_R2.fastq.gz,/PATH/L002_R2.fastq.gz,/PATH/L003_R2.fastq.gz,/PATH/L004_R2.fastq.gz \
--readFilesCommand zcat --outFileNamePrefix star_ --outFilterMultimapNmax 1 --outFilterMismatchNmax 2 \
--outSAMtype BAM SortedByCoordinate
```

In my example, there are multiple lanes and paired-end reads. So notice the usage of `--readFilesIn`, comma separates different lanes of the same mate (1st or 2nd), while space separates the mates.

**Step 5:** Finally get count matrix using featureCounts

`featureCounts` is from the `Subread` package.

```
featureCounts -T 4 -s 2 -p -g gene_name -a /PATH/GTF_FILENAME.gtf \
-o /PATH/MATRIX_FILENAME.txt /PATH/star_Aligned.sortedByCoord.out.bam
```

`star_Aligned.sortedByCoord.out.bam` is a file we can get from step 4. `MATRIX_FILENAME.txt` is the count matrix you would like to get. `-p` represents it is paired-end. Just remove it if it is single-end. `-g gene_name` is what I want to display in the count matrix. It is the variable name denoted in the `.gtf` file.

The `.txt` file we get will be a little massive and we can use the command below to clean it and just get two columns: one for gene id and another for the counts.

```
cut -f1,7,8,9,10,11,12 /PATH/MATRIX_FILENAME.txt > /PATH/MATRIX_NEW.txt
```


**We have done!**