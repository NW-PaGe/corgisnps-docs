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

Resource usage was compared between CorgiSNPs and the CDC MycoSNP pipeline using the MycoSNP [full test dataset](https://github.com/CDCgov/mycosnp-nf/blob/master/assets/sra_large.csv). **CorgiSNPs was found to be 2.27× faster, use 1.67× less CPU time, and cost 1.64× less than MycoSNP.**

---

# Test setup

Both workflows were run on **Seqera Cloud** using **AWS** compute, with the same input reads and the same samplesheet.

|Item|MycoSNP|CorgiSNPs|
|:-|:-|:-|
|Version / commit|`CDCgov/mycosnp-nf` v1.6.3 @ [`39eafa6`](https://github.com/CDCgov/mycosnp-nf/commit/39eafa650ab439665fe61b1d44d754eba700d981)|`NW-PaGe/CorgiSNPs` main @ [`02aa425`](https://github.com/NW-PaGe/CorgiSNPs/commit/02aa42594523a80456da548aa82db1aaecd189f9)|
|Nextflow version|26.04.6 build 12646|same|
|Platform|Seqera Cloud, AWS (compute environment TBD)|same|
|Runs|Two stages: `PRE_MYCOSNP`, then `NFCORE_MYCOSNP`, each over all samples|One run (`NWPAGE_CORGISNPS`) over all samples|
|Reference|Clade I, [GCA_016772135.1](https://www.ncbi.nlm.nih.gov/datasets/genome/GCA_016772135.1/), for all samples|Reference set (subtype-matched); clade I samples use the same GCA_016772135.1 reference|

**Dataset**
- The 24 *C. auris* samples in the CDC MycoSNP full test dataset, [`assets/sra_large.csv`](https://github.com/CDCgov/mycosnp-nf/blob/master/assets/sra_large.csv) in the mycosnp-nf repository.
- The SRA reads were downloaded separately, staged in an S3 bucket, and supplied to both pipelines through a single samplesheet pointing at those S3 paths. The **same samplesheet** was used for both pipelines.

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
|MycoSNP|`NFCORE_MYCOSNP`|1.88|57.4|792.29|454.27|307.66|1.94|
|**MycoSNP**|**Total**|**2.90**|**104.3**|**1,134.19**|**830.73**|**540.11**|**3.52**|
|**CorgiSNPs**|`NWPAGE_CORGISNPS`|**1.28**|**62.3**|**527.30**|**554.40**|**284.15**|**2.14**|

**Improvement with CorgiSNPs**

|Metric|MycoSNP total|CorgiSNPs|Times better|Reduction|
|:-|-:|-:|-:|-:|
|Wall time (h)|2.90|1.28|**2.27×** faster|55.9%|
|CPU time (h)|104.3|62.3|**1.67×** less|40.3%|
|Memory (GB)|1,134.19|527.30|**2.15×** less|53.5%|
|Data read (GB)|830.73|554.40|**1.50×** less|33.3%|
|Data written (GB)|540.11|284.15|**1.90×** less|47.4%|
|Est. cost (USD)|3.52|2.14|**1.64×** less|39.2%|

How these are calculated:

- **Times better** = MycoSNP total ÷ CorgiSNPs. For wall time, 2.90 h ÷ 1.28 h = 2.27, so CorgiSNPs is 2.27× faster.
- **Reduction** = (MycoSNP total − CorgiSNPs) ÷ MycoSNP total × 100. For wall time, (2.90 − 1.28) ÷ 2.90 = 55.9% less time.

---

# Takeaway

- On the same 24 samples and the same AWS infrastructure, CorgiSNPs completed in a single run while using less time, CPU, memory, I/O, and money than MycoSNP's two-stage run.

---

# Reproduce it yourself

Both pipelines were launched from Seqera Cloud against an AWS compute environment. The equivalent command-line runs are:

```bash
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
    --fasta GCA_016772135.1.fasta \
    --outdir results_mycosnp \
    --snpeff true

# CorgiSNPs (all samples, single run, same samplesheet)
nextflow run NW-PaGe/CorgiSNPs -r 02aa42594523a80456da548aa82db1aaecd189f9 \
    -profile docker \
    --input samplesheet.csv \
    --outdir results_corgisnps
```