---
layout: post
title: Some Linux Commands Notes
date: 2021-05-18
author: Dongyuan Wu
tags: [linux, genome]
comments: true
---

**Copy all files from folder 1 to folder 2:**
```
cp -R /path/to/folder1/ /path/to/folder2/
```

**Remove unexpected part of the file names (those substrings begin with "-" to the end of the strings) in one folder:**
```
for file in /path/to/folder/*; do mv "$file" "${file%-*}"; done
```
Note: `%` removes substring from the end, `#` removes substring from the beginning.

**Change the folder name as the file name in the folder without extention (one folder only contains one file):**
```
find "/path/to/folder/" -type f -exec bash -c ' DIR=$( dirname "{}" ); mv "{}" "$DIR"/"${DIR##*/}".fastq.gz  ' \;
```

**Check the strandness type of RNA-seq data:**\\
Method 1: Salmon
```
module load salmon
salmon quant -t GRCh38.fa -l A -a onesample.bam -o salmon_quant
```
Method 2: RSeQC
```
module load python
module load rseqc
infer_experiment.py -r GRCh38.bed12 -i onesample.bam
```
Note: How to convert GTF to bed12 https://gist.github.com/dongyuanwu/4ff6467eecf2ef04d4a3851c336b876b