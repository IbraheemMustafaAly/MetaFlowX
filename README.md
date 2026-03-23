# 🧬 Metagenomics Analysis Pipeline

A complete, end-to-end metagenomics pipeline for preprocessing, rRNA removal, and gene prediction from raw sequencing data.

---

## 📂 Workflow Overview

This pipeline performs:

1. Read merging (paired-end)
2. Quality control (FastQC & MultiQC)
3. Adapter trimming (Trimmomatic)
4. FASTQ → FASTA conversion
5. rRNA detection & masking (Infernal + Rfam)
6. Extraction of non-coding regions
7. Gene prediction (FragGeneScan)

---

## ⚙️ Requirements

Make sure the following tools are installed:

* SeqPrep
* FastQC
* MultiQC
* Trimmomatic
* SeqKit
* Infernal (cmsearch)
* Bedtools
* FragGeneScan
* wget

---

## 📁 Input Files

* Paired-end reads:

  * `*_1.fastq`
  * `*_2.fastq`

Example:
subset_SRR5903366_1.fastq
subset_SRR5903366_2.fastq

---

## 🚀 Full Pipeline Script


# 1. Merge Paired-End Reads
```bash
for R1 in *1.fastq; do
  R2="${R1%_1.fastq}_2.fastq"
  SeqPrep -f $R1 -r $R2 \
    -1 "unmerged_${R1}" \
    -2 "unmerged_${R2}" \
    -s "${R1%_1.fastq}_merged.gz"
done
```


# 2. Quality Control (Before Trimming)
```bash
for i in *merged.gz; do
  fastqc $i
done

multiqc ./
```


# 3. Adapter Trimming
```bash
for i in *merged.gz; do
  trimmomatic SE -phred33 $i ${i%.gz}_trimmed.fq.gz \
    ILLUMINACLIP:./adaptor.fa:2:30:10 \
    LEADING:3 TRAILING:3 \
    SLIDINGWINDOW:4:15 MINLEN:36
done
```



# 4. Quality Control (After Trimming)
```bash
for i in *trimmed.fq.gz; do
  fastqc $i
done
```



# 5. Convert FASTQ → FASTA
```bash
gunzip ./*trimmed.fq.gz

for i in *trimmed.fq; do
  seqkit fq2fa $i > ${i%.fq}.fa
done
```


# 6. Download rRNA Models (Rfam)
```bash
mkdir ribsome

wget "ftp://ftp.ebi.ac.uk/pub/databases/metagenomics/pipeline-5.0/ref-dbs/rfam_models/other_models/*.cm" \
  -P ribsome

cd ribsome

wget ftp://ftp.ebi.ac.uk/pub/databases/metagenomics/pipeline-5.0/ref-dbs/rfam_models/ribosomal_models/ribo.cm
```


# 7. Prepare Sample Directory
```bash
mkdir ./../sample_SRR5903366

mv ./../*_trimmed.fa ./../sample_SRR5903366/
```


# 8. rRNA Detection using cmsearch
```bash
cmsearch --tblout ./../sample_SRR5903366/ribo.cm.tbl \
  ./ribo.cm \
  ./../sample_SRR5903366/*.fa

for i in *.cm; do
  cmsearch --tblout ./../sample_SRR5903366/$i.tbl \
    $i \
    ./../sample_SRR5903366/*.fa
done
```


# 9. Convert cmsearch Output → BED
```bash
cd ..
for tbl_file in sample_SRR5903366/*.tbl; do
  awk '/^[^#]/ {
    if ($8 > $9) {
      print $1 "\t" $9-1 "\t" $8
    } else {
      print $1 "\t" $8-1 "\t" $9
    }
  }' "$tbl_file" >> sample_SRR5903366/combined.bed
done
```


# 10. Mask rRNA Regions
```bash
bedtools maskfasta \
  -fi sample_SRR5903366/*.fa \
  -bed sample_SRR5903366/combined.bed \
  -fo sample_SRR5903366/masked_output.fasta
```


# 11. Extract rRNA (Non-coding regions)
```bash
bedtools getfasta \
  -fi sample_SRR5903366/*.fa \
  -bed sample_SRR5903366/combined.bed \
  -fo sample_SRR5903366/noncoding_output.fasta
```


# 12. Gene Prediction (FragGeneScan)
```bash
./FragGeneScan -s sample_SRR5903366/masked_output.fasta \
  -o sample_SRR5903366/predicted_genes \
  -w 0 -t complete
```


# ✅ Pipeline Finished



---

## 📊 Output Files

| Step            | Output                 |
| --------------- | ---------------------- |
| QC Reports      | FastQC + MultiQC       |
| Trimmed Reads   | *_trimmed.fq           |
| FASTA Files     | *.fa                   |
| rRNA Regions    | combined.bed           |
| Masked Genome   | masked_output.fasta    |
| Non-coding RNA  | noncoding_output.fasta |
| Predicted Genes | predicted_genes        |

---

## 🧠 Notes

* This pipeline removes rRNA contamination using Rfam models.
* FragGeneScan is optimized for error-prone reads (ideal for metagenomics).
* You can extend this pipeline with:

  * Taxonomic classification (Kraken2 / MetaPhlAn)
  * Functional annotation (PROKKA / eggNOG)

---

## 📌 Author

**Ibraheem Mustafa Aly**
Bioinformatics & Metagenomics Researcher

---
