# 🧬 MetaFlowX – Metagenomics Bash Pipeline

![Status](https://img.shields.io/badge/status-active-success)
![Level](https://img.shields.io/badge/level-beginner%20to%20advanced-blue)
![Focus](https://img.shields.io/badge/focus-hands--on-brightgreen)
![Platform](https://img.shields.io/badge/platform-linux-lightgrey)

---

### Beginner → Advanced | Code-Driven Metagenomics

This repository provides a **step-by-step, hands-on metagenomics workflow**  
designed for **absolute beginners** and biology students transitioning into computational analysis.

All steps are written as **terminal commands**, following the style of real bioinformatics workshops.

> ⚠️ This is **not a lecture-based course**.  
> This is a **run-the-code, understand-the-output** roadmap.

---

## Table of Contents

1. [Setup & Installation](#1-setup--installation)
2. [NGS Data Overview](#2-ngs-data-overview)
3. [Quality Control](#3-quality-control)
4. [Read Merging & Trimming](#4-read-merging--trimming)
5. [rRNA Masking](#5-rrna-masking)
6. [CDS Prediction](#6-cds-prediction)
7. [Optional: Taxonomic Classification](#7-optional-taxonomic-classification)
8. [Final Notes](#8-final-notes)

---

## 1. Setup & Installation

### 1.1 Linux Basic Commands

```bash
  pwd
  ls
  mkdir metagenomics_pipeline
  cd metagenomics_pipeline

----

#1.2 Install Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
1.3 Configure Conda Channels
conda config --add channels defaults
conda config --add channels conda-forge
conda config --add channels bioconda
conda config --set channel_priority strict
2. NGS Data Overview
2.1 Inspect FASTQ File
zcat sample_R1.fastq.gz | head

Observe:

Read identifier
DNA sequence
Phred quality scores
3. Quality Control
3.1 Install QC Tools
conda create -n qc_env fastqc multiqc -y
conda activate qc_env
3.2 Run FastQC
fastqc *.fastq.gz
3.3 Generate MultiQC Report
multiqc .
4. Read Merging & Trimming
4.1 Merge Paired-End Reads
SeqPrep -f sample_1.fastq -r sample_2.fastq -s merged.fastq.gz
4.2 Trim Reads
trimmomatic SE -phred33 merged.fastq.gz \
    trimmed_output.fq.gz \
    ILLUMINACLIP:adaptor.fa:2:30:10 \
    LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36
4.3 Convert to FASTA
seqkit fq2fa trimmed_output.fq.gz > sample.fa
5. rRNA Masking
5.1 Download rRNA Models
mkdir ribsome
wget ftp://ftp.ebi.ac.uk/pub/databases/metagenomics/pipeline-5.0/ref-dbs/rfam_models/ribosomal_models/ribo.cm -P ribsome
5.2 Run cmsearch
cmsearch --tblout SRR_ribo.tbl ribsome/ribo.cm sample.fa
5.3 Convert to BED & Mask rRNA
awk '/^[^#]/ {if ($8>$9){print $1 "\t"$9-1 "\t"$8}else{print $1 "\t"$8-1 "\t"$9}}' SRR_ribo.tbl > bed_extracted.bed
bedtools maskfasta -fi sample.fa -bed bed_extracted.bed -fo masked_output.fasta
6. CDS Prediction
./FragGeneScan-master/FragGeneScan -s masked_output.fasta -o cds.fa -w 0 -t complete
7. Optional: Taxonomic Classification
Using Kraken2
kraken2 --paired sample_1.fastq sample_2.fastq \
        --db /path/to/kraken2_db \
        --output kraken_output.txt \
        --report kraken_report.txt
