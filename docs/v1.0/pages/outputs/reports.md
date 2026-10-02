---
title: Reports & Summaries
layout: page
nav_order: 1
grandparent: v1.0
parent: Outputs
permalink: /docs/v1.0/pages/outputs/results
---

# {{ page.title }}
{: .no_toc}

1. TOC
{:toc}

{: .note}
This page describes results saved to `--outdir`. Files written to the CorgiSNPs database (`--db`) are described on the [CorgiSNPs Database]({{ site.baseurl }}/docs/v1.0/pages/outputs/database) page.

# Overview
Outputs are organized into sample-level results (`sample/`), species / subtype level results (`species/`), and run-level summaries.

Below is an overview of the standard outputs produced by CorgiSNPs. `${species}` and `${subtype}` are lowercase with spaces and special characters replaced by underscores (e.g., `candidozyma_auris` and `clade_i`), and `${prefix}` is `${species}-${subtype}`.
```bash
${outdir}/
├── sample
│   └── ${sample}
│       ├── reads
│       │   ├── ${sra}_1.fastq.gz
│       │   ├── ${sra}_2.fastq.gz
│       │   ├── *.sample.fastq.gz
│       │   ├── ${sample}_1.fastp.fastq.gz
│       │   └── ${sample}_2.fastp.fastq.gz
│       ├── qc
│       │   └── ${sample}.fastp.json
│       ├── classify
│       │   ├── ${sample}.denovo.fa
│       │   ├── ${sample}_gambit.csv
│       │   └── ${sample}_subtype.csv
│       ├── variants
│       │   ├── aln
│       │   │   ├── ${sample}.mapped.bam
│       │   │   ├── ${sample}.mapped.bam.bai
│       │   │   ├── ${sample}.unmapped.bam
│       │   │   └── ${sample}.unmapped.bam.bai
│       │   ├── lowsites
│       │   │   ├── ${sample}.mpileup.gz
│       │   │   ├── ${sample}.mask.bed
│       │   │   └── lowsites.csv
│       │   ├── vcf
│       │   │   ├── ${sample}.raw.vcf.gz
│       │   │   ├── ${sample}.tagged.vcf.gz
│       │   │   ├── ${sample}.filt.vcf.gz
│       │   │   ├── ${sample}.snvs.vcf.gz
│       │   │   └── ${sample}.final-mask.bed
│       │   └── consensus
│       │       └── ${sample}.fa.gz
│       ├── amr
│       │   ├── aln
│       │   ├── vcf
│       │   │   └── ${sample}.ann.vcf
│       │   ├── ${sample}.amr-full.csv
│       │   └── ${sample}.amr-target.csv
│       └── ${sample}_summary.csv
├── species
│   ├── ${species}
│   │   ├── subtype_model
│   │   │   └── ${species}.sig.zip
│   │   ├── snpeff
│   │   │   └── ${species}_${subtype}.snpeff_db.tar.gz
│   │   └── ${subtype}
│   │       ├── aln
│   │       │   ├── ${prefix}.aln.gz
│   │       │   ├── ${prefix}_full.aln.gz
│   │       │   └── ${prefix}_full.csv.gz
│   │       ├── qc
│   │       │   └── ${prefix}_cg-plot.html
│   │       ├── dist
│   │       │   └── ${prefix}_dist.csv
│   │       ├── summary
│   │       │   └── ${prefix}_summary.csv
│   │       ├── tree
│   │       │   └── ${prefix}.nwk
│   │       └── reports
│   │           └── ${prefix}.microreact
├── multiqc
│   └── multiqc_report.html
├── CorgiSNPs-summary.csv
└── pipeline_info
    └── ...
```

{: .todo}
Verify this tree against a real run and update any file names that differ.

# Sample Reads & QC
Reads downloaded from NCBI SRA, down-sampled reads (when `--max_reads` is exceeded), and quality filtered reads are published in sample-specific subdirectories, along with fastp QC statistics.
```bash
│   └── ${sample}
│       ├── reads
│       │   ├── ${sra}_1.fastq.gz
│       │   ├── ${sra}_2.fastq.gz
│       │   ├── *.sample.fastq.gz
│       │   ├── ${sample}_1.fastp.fastq.gz
│       │   └── ${sample}_2.fastp.fastq.gz
│       └── qc
│           └── ${sample}.fastp.json
```

| File | Description |
|------|-------------|
| `${sra}_1.fastq.gz` | Forward reads downloaded from NCBI SRA |
| `${sra}_2.fastq.gz` | Reverse reads downloaded from NCBI SRA |
| `*.sample.fastq.gz` | Reads randomly down-sampled with seqtk |
| `*.fastp.fastq.gz` | Quality filtered reads from fastp |
| `${sample}.fastp.json` | fastp read statistics |

# Classification
De novo assemblies created via Shovill, species assignments from GAMBIT, and subtype assignments are published per sample. Only produced for samples missing a species or subtype in the samplesheet.
```bash
│       ├── classify
│       │   ├── ${sample}.denovo.fa
│       │   ├── ${sample}_gambit.csv
│       │   └── ${sample}_subtype.csv
```

| File | Description |
|------|-------------|
| `${sample}.denovo.fa` | De novo genome assembly in FASTA format |
| `${sample}_gambit.csv` | GAMBIT species classification results |
| `${sample}_subtype.csv` | Subtype assignment, including the closest subtype, its ANI, the threshold used, and whether the threshold was met |

The sourmash signatures of the reference assemblies used for subtyping are published at the species level.
```bash
│   ├── ${species}
│   │   ├── subtype_model
│   │   │   └── ${species}.sig.zip
```

# Variants
Read alignments, variant calls, masks, and consensus genomes are published per sample.
```bash
│       ├── variants
│       │   ├── aln
│       │   ├── lowsites
│       │   ├── vcf
│       │   └── consensus
```

| File | Description |
|------|-------------|
| `${sample}.mapped.bam` | Reads aligned to the reference (with index) |
| `${sample}.unmapped.bam` | Reads that did not align to the reference (with index) |
| `${sample}.mpileup.gz` | SAMtools pileup used to identify low depth / quality sites |
| `${sample}.mask.bed` | Low depth / low quality sites |
| `lowsites.csv` | Summary of low depth / quality sites |
| `${sample}.raw.vcf.gz` | Unfiltered FreeBayes variant calls |
| `${sample}.tagged.vcf.gz` | All variant calls with filter tags applied (see [Overview]({{ site.baseurl }}/docs/v1.0/pages/overview/#calling--filtering-variants)) |
| `${sample}.filt.vcf.gz` | Variant calls that passed all filters |
| `${sample}.snvs.vcf.gz` | SNPs that passed all filters (used to build the consensus genome) |
| `${sample}.final-mask.bed` | Final set of masked sites applied to the consensus genome |
| `${sample}.fa.gz` | Consensus genome. These files are also published to the CorgiSNPs database (`--db`) when using `--push true` |

# Antifungal Resistance
Resistance results are published per sample when the species' reference set includes resistance targets. The SnpEff database built for each reference is published at the species level.
```bash
│       ├── amr
│       │   ├── aln
│       │   ├── vcf
│       │   │   └── ${sample}.ann.vcf
│       │   ├── ${sample}.amr-full.csv
│       │   └── ${sample}.amr-target.csv
```

| File | Description |
|------|-------------|
| `${sample}.ann.vcf` | Variant calls annotated by SnpEff |
| `${sample}.amr-full.csv` | All annotated variants |
| `${sample}.amr-target.csv` | Annotated variants within the resistance target genes / regions |
| `*.snpeff_db.tar.gz` | SnpEff database built from the reference assembly and annotation |

{: .note}
The `amr/aln` and `amr/vcf` directories also contain the re-called alignments and variants for samples whose resistance targets were extracted against the species' primary reference.

{: .todo}
Add the column descriptions for the `*.amr-full.csv` and `*.amr-target.csv` files.

# Core Genome Alignments
Core genome alignments produced by polycore are published per species / subtype.
```bash
│   │   └── ${subtype}
│   │       ├── aln
│   │       │   ├── ${prefix}.aln.gz
│   │       │   ├── ${prefix}_full.aln.gz
│   │       │   └── ${prefix}_full.csv.gz
```

| File | Description |
|------|-------------|
| `${prefix}.aln.gz` | Core genome SNP alignment in FASTA format |
| `${prefix}_full.aln.gz` | Full core genome alignment including invariant sites |
| `${prefix}_full.csv.gz` | Per-site summary of the full alignment |

# Quality Control
A core genome plot is produced by polycore for each species / subtype and a MultiQC report aggregating per-sample QC metrics is produced at the run level.
```bash
│   │       ├── qc
│   │       │   └── ${prefix}_cg-plot.html
└── multiqc
    └── multiqc_report.html
```

| File | Description |
|------|-------------|
| `${prefix}_cg-plot.html` | Interactive plot of how the core genome changes as samples are added |
| `multiqc_report.html` | Aggregated QC report including FastQC and fastp metrics for all samples |

# Distances
Pairwise core SNP distances are published per species / subtype.
```bash
│   │       ├── dist
│   │       │   └── ${prefix}_dist.csv
```

| File | Description |
|------|-------------|
| `${prefix}_dist.csv` | Pairwise core SNP distance matrix |

# Phylogeny
A maximum likelihood phylogenetic tree is produced per species / subtype when there are more than 2 unique sequences.
```bash
│   │       ├── tree
│   │       │   └── ${prefix}.nwk
```

| File | Description |
|------|-------------|
| `${prefix}.nwk` | Midpoint rooted maximum likelihood tree in Newick format produced by IQ-TREE, with branch lengths rescaled by reference genome length |

# Summary
Summary tables are produced at two levels: per sample / run and per species / subtype.

## Run Summary
A run-level summary table is published at the top of the output directory. It combines the sample summaries for every sample in the run, including samples that failed QC or could not be matched to a reference.
```bash
${outdir}/
├── sample
│   └── ${sample}
│       └── ${sample}_summary.csv
└── CorgiSNPs-summary.csv
```

| File | Description |
|------|-------------|
| `${sample}_summary.csv` | Summary for a single sample |
| `CorgiSNPs-summary.csv` | Summary for all samples in the run |

## Species / Subtype Summary
A per species / subtype summary table is published within each species / subtype subdirectory. It adds core genome, linkage, and partition information to the sample summaries.
```bash
│   │       ├── summary
│   │       │   └── ${prefix}_summary.csv
```

## Summary Columns
The sample and run summary files contain the following columns.

| Column | Description |
|--------|-------------|
| `sample` | Sample identifier (same as supplied in samplesheet) |
| `status` | Whether the sample was added in the current run (`new`) |
| `qc_status` | Automated QC result (`PASS` / `FAIL`) |
| `qc_reason` | Reasons for QC failure, grouped as classification (sample could not be matched to a reference), undetermined, failed, or errored checks |
| `species` | Species (from samplesheet or GAMBIT) |
| `subtype` | Subtype (from samplesheet or subtyping) |
| `subtype_ani` | ANI (%) to the closest subtype reference |
| `estimated_depth` | Total bases after filtering divided by the species' expected genome length (NCBI mean, or the midpoint of the reference set `length_range`) |
| `denovo_contigs` | Number of contigs in the de novo assembly |
| `denovo_length` | Length of the de novo assembly (bp) |
| `denovo_length_range` | Acceptable assembly length for the species (`min-max`, bp); blank if undetermined |
| `denovo_gc` | GC content (%) of the de novo assembly |
| `denovo_gc_range` | Acceptable assembly GC content for the species (`min-max`, %); blank if undetermined |
| `qc_range_source` | Where the ranges came from: `manifest` (the [reference set]({{ site.baseurl }}/docs/v1.0/pages/reference_sets/#step-6-set-automated-qc-ranges-optional)), `ncbi` (`--ncbi_stats`), or one per metric when they differ (e.g., `length: manifest; gc: ncbi`) |
| `*_after_filtering` / `*_before_filtering` | fastp read statistics: total reads, total bases, Q30 bases, Q30 rate, mean read 1 / read 2 length, and GC content |
| `amr_variants_target` | Moderate / high impact changes within a named resistance target region, formatted as `gene(region):mutation` |
| `amr_variants_other` | Moderate / high impact changes elsewhere in a resistance target gene, formatted as `gene:mutation` |

The species / subtype summary adds the following columns:

| Column | Description |
|--------|-------------|
| `strong_links` | Samples within [`strong_link_threshold`]({{ site.baseurl }}/docs/v1.0/pages/reference_sets/#step-5-set-analysis-settings-recommended) SNPs of this sample |
| `inter_links` | Samples within [`inter_link_threshold`]({{ site.baseurl }}/docs/v1.0/pages/reference_sets/#step-5-set-analysis-settings-recommended) SNPs of this sample (and not strongly linked) |
| `partition` | Tree partition assigned to the sample. _NOTE: partitions are subject to change depending on which samples are included in the analysis!_ |

{: .todo}
Add descriptions for the core genome statistics columns added from polycore (e.g., genome fraction, core fraction, missing / mixed sites).

# Reports
A Microreact report is produced per species / subtype, combining the phylogenetic tree, SNP distance matrix, and species / subtype summary.
```bash
│   │       └── reports
│   │           └── ${prefix}.microreact
```

| File | Description |
|------|-------------|
| `${prefix}.microreact` | Microreact project file for interactive visualization at [microreact.org](https://microreact.org) |
