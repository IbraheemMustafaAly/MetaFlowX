# 🧬 MetaFlowX - Metagenomics Analysis Pipeline

A production-ready, modular metagenomics pipeline for preprocessing, rRNA removal, gene prediction, and taxonomic visualization using Krona.

---

## 📂 Workflow Overview

This pipeline performs:

1. Paired-end read merging
2. Quality control (FastQC & MultiQC)
3. Adapter and quality trimming
4. FASTQ → FASTA conversion
5. rRNA detection & masking (Infernal + Rfam)
6. Gene prediction (FragGeneScan)
7. Taxonomic classification (Kraken2)
8. Interactive visualization (Krona)

---

## ⚙️ Requirements

Install the following tools:

* SeqPrep
* FastQC
* MultiQC
* Trimmomatic
* SeqKit
* Infernal (cmsearch)
* Bedtools
* FragGeneScan
* Kraken2
* KronaTools
* wget

---

## 📁 Input

Paired-end FASTQ files:

```
*_1.fastq
*_2.fastq
```

---

## 📁 Recommended Project Structure

```
metagenomics-pipeline/
│
├── data/
├── results/
│   ├── qc/
│   ├── trimmed/
│   ├── fasta/
│   ├── rrna/
│   ├── genes/
│   ├── taxonomy/
│   │   ├── kraken/
│   │   └── krona/
│
├── databases/
│   ├── kraken_db/
│   └── rfam/
│
├── scripts/
│   └── pipeline.sh
│
└── README.md
```

---

## 🚀 Full Pipeline Script



# CONFIG
```
THREADS=8
SAMPLE="sample_SRR5903366"

mkdir -p results/{qc,trimmed,fasta,rrna,genes,taxonomy/kraken,taxonomy/krona}
```

# 1. Merge Reads
```
for R1 in *_1.fastq; do
  R2="${R1%_1.fastq}_2.fastq"

  SeqPrep -f $R1 -r $R2 \
    -1 results/trimmed/unmerged_${R1} \
    -2 results/trimmed/unmerged_${R2} \
    -s results/trimmed/${R1%_1.fastq}_merged.gz
done
```


# 2. QC (Pre-trimming)
```
fastqc results/trimmed/*merged.gz -o results/qc/
multiqc results/qc/ -o results/qc/
```

# 3. Trimming
```
for i in results/trimmed/*merged.gz; do
  trimmomatic SE -phred33 $i ${i%.gz}_trimmed.fq.gz \
    ILLUMINACLIP:adaptor.fa:2:30:10 \
    LEADING:3 TRAILING:3 \
    SLIDINGWINDOW:4:15 MINLEN:36
done
```

# 4. QC (Post-trimming)
```
fastqc results/trimmed/*trimmed.fq.gz -o results/qc/
```

# 5. FASTQ → FASTA
```
gunzip results/trimmed/*trimmed.fq.gz

for i in results/trimmed/*trimmed.fq; do
  seqkit fq2fa $i > results/fasta/$(basename ${i%.fq}.fa)
done
```

# 6. Download Rfam (once)
```
mkdir -p databases/rfam

wget -P databases/rfam \
ftp://ftp.ebi.ac.uk/pub/databases/metagenomics/pipeline-5.0/ref-dbs/rfam_models/ribosomal_models/ribo.cm
```

# 7. rRNA Detection
```
for fa in results/fasta/*.fa; do
  base=$(basename $fa .fa)

  cmsearch --tblout results/rrna/${base}.tbl \
    databases/rfam/ribo.cm \
    $fa
done
```

# 8. Convert to BED
```
for tbl in results/rrna/*.tbl; do
  awk '/^[^#]/ {
    if ($8 > $9) {
      print $1 "\t" $9-1 "\t" $8
    } else {
      print $1 "\t" $8-1 "\t" $9
    }
  }' $tbl >> results/rrna/combined.bed
done
```

# 9. Mask rRNA
```
for fa in results/fasta/*.fa; do
  base=$(basename $fa .fa)

  bedtools maskfasta \
    -fi $fa \
    -bed results/rrna/combined.bed \
    -fo results/rrna/${base}_masked.fasta
done
```

# 10. Gene Prediction
```
for fa in results/rrna/*_masked.fasta; do
  base=$(basename $fa _masked.fasta)

  FragGeneScan -s $fa \
    -o results/genes/${base} \
    -w 0 -t complete
done
```

# 11. Taxonomy (Kraken2)
```
# Run (database should be pre-built)
for fa in results/rrna/*_masked.fasta; do
  base=$(basename $fa _masked.fasta)

  kraken2 \
    --db databases/kraken_db \
    --threads $THREADS \
    --report results/taxonomy/kraken/${base}.report \
    --output results/taxonomy/kraken/${base}.out \
    $fa
done
```

# 12. Krona Visualization
```
for file in results/taxonomy/kraken/*.out; do
  base=$(basename $file .out)

  cut -f2,3 $file > results/taxonomy/krona/${base}.txt

  ktImportTaxonomy \
    results/taxonomy/krona/${base}.txt \
    -o results/taxonomy/krona/${base}.html
done
```

# ✅ DONE



---

## 📊 Outputs

| Category      | Output              |
| ------------- | ------------------- |
| QC            | FastQC + MultiQC    |
| Trimmed reads | *.fq                |
| FASTA         | *.fa                |
| rRNA masked   | *_masked.fasta      |
| Genes         | FragGeneScan output |
| Taxonomy      | Kraken reports      |
| Visualization | Krona HTML          |

---

## 🧠 Notes

* rRNA removal is based on Rfam models (Infernal).
* Kraken2 provides fast taxonomic classification.
* Krona generates interactive hierarchical visualizations.
* Pipeline structure is inspired by the EMBL-EBI MGnify pipeline.

---

## 🔥 Future Extensions

* Assembly (MEGAHIT / metaSPAdes)
* Functional annotation (eggNOG / PROKKA)
* Abundance estimation (Bracken)
* Pathway analysis (HUMAnN)

---

## 📌 Author

**Ibraheem Mustafa Aly**
Bioinformatics & Metagenomics Researcher

---
