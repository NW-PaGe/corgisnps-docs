---
title: Quick Start
layout: page
nav_order: 1
parent: v1.0
permalink: /docs/v1.0/pages/quickstart/
---

# {{ page.title }}
{: .no_toc}

{: .important}
CorgiSNPs requires [Nextflow](https://www.nextflow.io/docs/latest/install.html) (version 25.10.0+) and <ins>one</ins> of the following container engines: [Docker](https://docs.docker.com/engine/install/), [Podman](https://podman.io/docs/installation), [Apptainer](https://apptainer.org/docs/admin/main/installation.html), [Singularity](https://docs.sylabs.io/guides/3.0/user-guide/installation.html).

---

## 1. Create your samplesheet

{: .note}
Only the `sample` column is required, plus reads supplied as either `fastq_1` / `fastq_2` or an `sra` accession. The `species` and `subtype` columns are optional — leave them empty and CorgiSNPs will determine them automatically.

`samplesheet.csv`:
```csv
sample,fastq_1,fastq_2,species,subtype,sra
sample1,/path/to/sample1_R1.fastq.gz,/path/to/sample1_R2.fastq.gz,Candidozyma auris,Clade I,
sample2,/path/to/sample2_R1.fastq.gz,/path/to/sample2_R2.fastq.gz,,,
sample3,,,,,SRR23958488
```

---

## 2. Run CorgiSNPs

{: .note}
CorgiSNPs ships with a reference set for *Candidozyma auris* (*Candida auris*). Other species require a reference set - see [Creating a Reference Set](../reference_sets/).

```bash
nextflow run NW-PaGe/CorgiSNPs \
    -r main \
    -profile docker \
    --input $PWD/samplesheet.csv \
    --outdir $PWD/results \
    --db $PWD/corgisnps_db
```

---

## 3. Review your results

Results are saved to `--outdir`. See the [outputs](../outputs/) page for details.

---

## 4. Push your results

Use `--push true` to save each sample's consensus genome to your CorgiSNPs database (set with `--db`) so it is included in future runs. Use `-resume` to avoid recomputing results from step 2.
```bash
nextflow run NW-PaGe/CorgiSNPs \
    -r main \
    -profile docker \
    --input $PWD/samplesheet.csv \
    --outdir $PWD/results \
    --db $PWD/corgisnps_db \
    --push true \
    -resume
```

---

{: .tip}
🔍 Want to learn more? Check out the [Getting Started](../getting_started/) and [Overview](../overview/) pages.
