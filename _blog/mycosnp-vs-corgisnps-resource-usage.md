---
title: "MycoSNP vs. CorgiSNPs: Resource Usage"
layout: page
date: 2026-10-08
author: Jared Johnson
description: How much CPU, memory, time, and disk each pipeline needs to analyze the same Candidozyma auris dataset.
nav_order: -20261008
---

# {{ page.title }}
{: .no_toc }

<small>{{ page.date | date: "%B %-d, %Y" }} · {{ page.author }}</small>

1. TOC
{:toc}

---

# Overview

Resource usage was compared between CorgiSNPs and the CDC MycoSNP pipeline using the MycoSNP [full test dataset](https://github.com/CDCgov/mycosnp-nf/blob/master/assets/sra_large.csv). **CorgiSNPs was found to be 2.95× faster, use 1.72× less CPU time, and cost 1.67× less than MycoSNP.**

---

# Test setup

Both workflows were run on **Seqera Cloud** using **AWS** compute, with the same input reads and the same samplesheet.

|Item|MycoSNP|CorgiSNPs|
|:-|:-|:-|
|Version / commit|`CDCgov/mycosnp-nf` v1.6.3 @ [`39eafa6`](https://github.com/CDCgov/mycosnp-nf/commit/39eafa650ab439665fe61b1d44d754eba700d981)|`NW-PaGe/CorgiSNPs` main @ [`02aa425`](https://github.com/NW-PaGe/CorgiSNPs/commit/02aa42594523a80456da548aa82db1aaecd189f9)|
|Nextflow version|26.04.6 build 12646|same|
|Platform|Seqera Cloud, AWS (compute environment TBD)|same|
|Runs|Three runs: `PRE_MYCOSNP` over all samples, `NFCORE_MYCOSNP` over all samples (clade I reference), and `NFCORE_MYCOSNP` over the single clade IV sample (clade IV reference)|One run (`NWPAGE_CORGISNPS`) over all samples|
|Reference|Clade I, [GCA_016772135.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_016772135.1/), for all samples (needed for *FKS1* detection); clade IV, [GCA_003014415.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_003014415.1/), for B12847|Reference set (subtype-matched), selected automatically; clade I samples use GCA_016772135.1 and B12847 uses GCA_003014415.1|

**Dataset**
- The 24 *C. auris* samples in the CDC MycoSNP full test dataset, [`assets/sra_large.csv`](https://github.com/CDCgov/mycosnp-nf/blob/master/assets/sra_large.csv) in the mycosnp-nf repository.
- The SRA reads were downloaded separately, staged in an S3 bucket, and supplied to both pipelines through a single samplesheet pointing at those S3 paths. The **same samplesheet** was used for both pipelines.
- One sample, **B12847**, is clade IV. In MycoSNP, B12847 is analyzed twice: once against the clade I reference with the rest of the batch (so *FKS1* mutations can be detected with the clade I SnpEff database), and once on its own against the clade IV reference. CorgiSNPs handles both in its single run.

---

# What we measured

All metrics come from the **Seqera Cloud run metrics** for each run.

|Metric|Why it matters|
|:-|:-|
|Wall-clock time|How long until results are ready|
|CPU time|Cost on shared / cloud hardware|
|Memory|Whether it fits on a workstation|
|Data read / written|Storage and I/O load per run|
|Estimated cost|Direct cloud spend per run|

---

# Results

|Pipeline|Run|Wall time (h)|CPU time (h)|Memory (GB)|Data read (GB)|Data written (GB)|Est. cost (USD)|
|:-|:-|-:|-:|-:|-:|-:|-:|
|MycoSNP|`PRE_MYCOSNP`|1.02|46.9|341.90|376.46|232.45|1.58|
|MycoSNP|`NFCORE_MYCOSNP` (clade I ref, all samples)|1.88|57.4|792.29|454.27|307.66|1.94|
|MycoSNP|`NFCORE_MYCOSNP` (clade IV ref, B12847)|0.88|2.6|37.56|27.03|19.63|0.06|
|**MycoSNP**|**Total**|**3.78**|**106.9**|**1,171.75**|**857.76**|**559.74**|**3.58**|
|**CorgiSNPs**|`NWPAGE_CORGISNPS`|**1.28**|**62.3**|**527.30**|**554.40**|**284.15**|**2.14**|

**Improvement with CorgiSNPs**

|Metric|MycoSNP total|CorgiSNPs|Times better|Reduction|
|:-|-:|-:|-:|-:|
|Wall time (h)|3.78|1.28|**2.95×** faster|66.1%|
|CPU time (h)|106.9|62.3|**1.72×** less|41.7%|
|Memory (GB)|1,171.75|527.30|**2.22×** less|55.0%|
|Data read (GB)|857.76|554.40|**1.55×** less|35.4%|
|Data written (GB)|559.74|284.15|**1.97×** less|49.2%|
|Est. cost (USD)|3.58|2.14|**1.67×** less|40.2%|

How these are calculated:

- **Times better** = MycoSNP total ÷ CorgiSNPs. For wall time, 3.78 h ÷ 1.28 h = 2.95, so CorgiSNPs is 2.95× faster.
- **Reduction** = (MycoSNP total − CorgiSNPs) ÷ MycoSNP total × 100. For wall time, (3.78 − 1.28) ÷ 3.78 = 66.1% less time.

---

# Takeaway

On the same 24 samples and the same AWS infrastructure, CorgiSNPs completed in a single run while using less time, CPU, memory, I/O, and money than MycoSNP's three runs, and without the manual steps of picking a reference and splitting out the clade IV sample.

---

# Reproduce it yourself

Both pipelines were launched from Seqera Cloud against an AWS compute environment. The equivalent command-line runs are:

```bash
# S3 location where the reads are staged
BUCKET=s3://your-bucket/reads

# Create samplesheet
wget -O sra_large.csv https://github.com/CDCgov/mycosnp-nf/raw/refs/heads/master/assets/sra_large.csv

mkdir -p reads

echo "sample,fastq_1,fastq_2" > samplesheet.csv

# Each row is: sample,SRR accession (header row, if any, is skipped)
tr -d '\r' < sra_large.csv | grep -E ',[SED]RR[0-9]+' | while IFS=, read -r sample srr; do
    echo "Downloading ${sample} (${srr})"
    fasterq-dump "${srr}" --split-files --outdir reads --threads 8 --progress
    gzip -f "reads/${srr}_1.fastq" "reads/${srr}_2.fastq"

    # Rename to sample IDs so both pipelines report the same names
    mv "reads/${srr}_1.fastq.gz" "reads/${sample}_R1.fastq.gz"
    mv "reads/${srr}_2.fastq.gz" "reads/${sample}_R2.fastq.gz"

    echo "${sample},${BUCKET}/${sample}_R1.fastq.gz,${BUCKET}/${sample}_R2.fastq.gz" >> samplesheet.csv
done

# Download the clade I and clade IV references from NCBI (datasets CLI)
datasets download genome accession GCA_016772135.1 GCA_003014415.1 \
    --include genome \
    --filename references.zip
unzip -o references.zip -d references
for acc in GCA_016772135.1 GCA_003014415.1; do
    cp references/ncbi_dataset/data/${acc}/${acc}_*_genomic.fna ${acc}.fna
done

# MycoSNP, stage 1: PRE_MYCOSNP (all samples)
nextflow run CDCgov/mycosnp-nf -r 39eafa650ab439665fe61b1d44d754eba700d981 \
    -profile docker \
    --workflow PRE_MYCOSNP \
    --input samplesheet.csv \
    --outdir results_mycosnp_pre

# MycoSNP, stage 2: NFCORE_MYCOSNP (all samples, clade I reference)
nextflow run CDCgov/mycosnp-nf -r 39eafa650ab439665fe61b1d44d754eba700d981 \
    -profile docker \
    --workflow NFCORE_MYCOSNP \
    --input samplesheet.csv \
    --fasta GCA_016772135.1.fna \
    --outdir results_mycosnp \
    --snpeff true

# Create a samplesheet with only the clade IV sample (B12847).
# Clade is not known until PRE_MYCOSNP finishes, so this subset is made by hand
# from the full samplesheet: keep the header row plus the rows for clade IV samples.
head -n 1 samplesheet.csv > samplesheet_clade-IV.csv
grep '^B12847,' samplesheet.csv >> samplesheet_clade-IV.csv

# MycoSNP, stage 3: NFCORE_MYCOSNP (B12847, clade IV reference)
nextflow run CDCgov/mycosnp-nf -r 39eafa650ab439665fe61b1d44d754eba700d981 \
    -profile docker \
    --workflow NFCORE_MYCOSNP \
    --input samplesheet_clade-IV.csv \
    --fasta GCA_003014415.1.fna \
    --outdir results_mycosnp_clade-IV \
    --snpeff true

# CorgiSNPs (all samples, single run, same samplesheet)
nextflow run NW-PaGe/CorgiSNPs -r 02aa42594523a80456da548aa82db1aaecd189f9 \
    -profile docker \
    --input samplesheet.csv \
    --outdir results_corgisnps
```