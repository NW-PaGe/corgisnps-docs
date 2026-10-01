---
title: Overview
layout: page
nav_order: 3
parent: v1.0
permalink: /docs/v1.0/pages/overview/
---

# {{ page.title }}
{: .no_toc }

1. TOC
{:toc}

---

# What is CorgiSNPs?

CorgiSNPs (Core Genome Investigation SNPs) is a Nextflow pipeline for fungal core-genome SNP analysis in public health genomic surveillance. It accepts Illumina reads or SRA accessions, determines the species and subtype of each sample, calls variants against a subtype-specific reference genome, detects antifungal resistance markers, and produces phylogenetic trees and pairwise SNP distance matrices.

CorgiSNPs is currently tested with *Candidozyma auris* (*Candida auris*). It builds on the foundation of [MycoSNP](https://github.com/CDCgov/mycosnp-nf) but improves workflow automation, higher-ploidy handling, and phylogenetic interpretation.

## Key Features

<div style="padding: 1em; margin: 1em 0;">

🧬 <strong>Fungal SNP detection</strong> - calls SNPs directly from fungal whole-genome sequencing data using a reference-based workflow<br>
🧬 <strong>Flexible ploidy</strong> - handles organisms with up to three genome copies, from haploid through triploid<br>
🧬 <strong>Phylogenetics and distances</strong> - builds core SNP phylogenies and pairwise distance matrices in a single pass<br>
🧬 <strong>Layered summaries</strong> - reports at both the sample and cluster level, ready for downstream visualization<br>
🧬 <strong>Standard outputs</strong> - exports results as VCF, FASTA, Newick, CSV, and Microreact files<br>

</div>

## Flowchart
![]({{ site.baseurl }}/docs/v1.0/media/corgisnps-v1.0.png)

---

# Inputs

## Read Processing

Reads can be supplied as FASTQ files (`fastq_1` / `fastq_2` columns) or downloaded automatically from NCBI using an SRA accession (`sra` column) with `fasterq-dump`. Samples exceeding the maximum read count set by `max_reads` (default: `10_000_000`) are randomly subsampled using [seqtk](https://github.com/lh3/seqtk). Reads are then checked with [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) and quality filtered using [fastp](https://github.com/opengene/fastp).

## Species & Subtype

Species and subtype can be supplied via the `species` and `subtype` columns. Samples with **both** values supplied skip the classification steps below. Supplied values must match a species name (or alias) and subtype name in the [reference set](../reference_sets/) or be provided with a reference genome using the `reference` column in the samplesheet.

---

# Classification

Samples missing a species or subtype are classified automatically.

## De Novo Assembly

Each sample being classified is assembled using [Shovill](https://github.com/tseemann/shovill) with [SKESA](https://github.com/ncbi/SKESA) as the default assembler. Contigs below a minimum coverage (default: `10`) or length (default: `300` bp) are removed. The assembly is used for species and subtype assignment and for assembly-based QC metrics.

## Species Assignment

Samples without a species are classified using [GAMBIT](https://github.com/jlumpe/gambit) with a fungal GAMBIT database bundled with the pipeline (`gambit-fungal-1.0.0-20241213`). Only species-level calls are accepted.

## Subtyping

Samples without a subtype are compared to the reference assemblies for their species. Each reference assembly is sketched with [sourmash](https://sourmash.readthedocs.io/en/latest/) (`ksize=31`, `scaled=100`), and the sample is assigned the subtype of the reference with the highest average nucleotide identity (ANI), provided the ANI meets the species' `subtype_ani` threshold (default: `0.997`). Otherwise, the subtype is `undefined`.

{: .important}
Any sample that cannot be matched to a reference - because no species-level call was made, no reference set exists for the species, or the subtype could not be determined - stops the pipeline. All affected samples are listed in a single error message.

---

# Automated Quality Control

Each sample is summarized and evaluated against the following criteria. Samples that fail are reported in the summary but are **not** passed to variant calling or saved to the database.

|Check|Criterion|
|:-|:-|
|Read quality|Q30 rate after filtering ≥ `min_q30_rate_qc` (default: `0.8`)|
|Species|A species was supplied or assigned|
|Subtype|A subtype was supplied or assigned|
|Estimated depth|Total bases after filtering ÷ NCBI mean genome length for the species ≥ `min_depth_qc` (default: `30`)|
|Assembly length|\|z-score\| of assembly length vs. NCBI genomes for the species < `max_z_score_qc` (default: `2.58`)|
|Assembly GC|\|z-score\| of assembly GC content vs. NCBI genomes for the species < `max_z_score_qc` (default: `2.58`)|

Genome length and GC statistics come from a bundled summary of fungal genomes hosted on NCBI (`ncbi_stats`). Z-scores are only calculated when at least 3 NCBI genomes are available for the species; otherwise the assembly checks are skipped.

{: .important}
Estimated depth requires the species to be present in the NCBI statistics file. If it is not, the sample's depth is undetermined and the sample fails QC. You can bypass this using `--ignore_qc true` - use with caution!

---

# Variant Calling

## Read Alignment

Reads are aligned to the reference assembly for the sample's species and subtype using [BWA-MEM2](https://github.com/bwa-mem2/bwa-mem2) and sorted with [SAMtools](https://www.htslib.org/).

## Calling & Filtering Variants

Variants are called using [FreeBayes](https://github.com/freebayes/freebayes) with the ploidy set in the reference set and a coverage cap of `limit_coverage` (default: `100`). Calls are then tagged with [bcftools](https://samtools.github.io/bcftools/) filters:

|Filter tag|Condition (default)|
|:-|:-|
|`non_gt_alt`|Allele not present in the called genotype|
|`low_af`|Variant allele fraction < `min_allele_fraction` (`0.8`)|
|`low_depth`|Allele depth < `min_base_depth` (`10`)|
|`low_base_qual`|Mean allele base quality ≤ `min_base_quality` (`30`)|
|`low_map_qual`|Mean allele mapping quality < `min_mapping_quality` (`40`)|
|`strand_bias`|Strand bias score ≥ `max_strand_bias` (`15`) or forward-strand fraction < `min_fwd_strand_fraction` (`0.3`)|
|`read_pos_bias`|Allele seen only on one side of reads or read position bias score > `max_read_pos_bias` (`30`)|

{: .important}
`min_allele_fraction` applies to all samples in a run. The pipeline's suggested values are `0.8` for haploid organisms and `0.25` for diploid / triploid organisms.

## Consensus Genome

A consensus genome is built for each sample using `bcftools consensus`, applying **SNVs** that passed all filters. Sites are masked when they have low depth or low base quality (based on `samtools mpileup`), or when they carry a variant that is not a passing SNV (indels, MNPs and complex variants are masked). Heterozygous calls are written using IUPAC codes.

---

# Antifungal Resistance

When the reference set defines resistance targets (`amr`) for a species, CorgiSNPs annotates variants with [SnpEff](https://pcingola.github.io/SnpEff/) and reports changes within the target genes and regions (e.g., the *FKS1* hot spot regions for *C. auris*). A SnpEff database is built from the reference assembly and annotation.

Samples are handled in one of three ways:

- **Direct** - the sample's own reference has resistance targets. The sample's filtered VCF is annotated directly.
- **Extract** - the sample's reference has no resistance targets, but another subtype of the same species does (the *primary* subtype). The target regions are extracted, variants are re-called against the primary reference, and the new VCF is annotated.
- **None** - no subtype of the species has resistance targets. Resistance detection is skipped with a warning.

---

# Phylogenetic Analysis

Samples are grouped by species and subtype. If a CorgiSNPs database (`--db`) exists, consensus genomes saved from previous runs for the same species and subtype are added to the analysis.

## Defining the Core Genome

The core genome is defined using [polycore](https://github.com/DOH-JDJ0303/polycore). Samples are added progressively, and samples below the minimum genome fraction (default: `0.9`) are excluded. A site is included in the core genome if data are present in at least the minimum core fraction of samples (default: `0.9`). Polycore produces the SNP alignment, constant site counts, and pairwise SNP distances used downstream.

{: .todo}
Add an example core genome plot (`*_cg-plot.html`) and description, as in the BigBacter overview.

<!-- <iframe
  src="{{ site.baseurl }}/assets/example_output/ecoli/1780330154-Escherichia_coli-3_cg-plot.html"
  width="100%"
  height="600px"
  frameborder="0"
  style="border: 1px solid #ccc; border-radius: 4px;"
  allowfullscreen>
</iframe> -->

## Core SNP Tree

A maximum likelihood tree is inferred using [IQ-TREE 3](http://www.iqtree.org/) under the **GTR+I+G** model. Ascertainment bias correction is applied via `-fconst` using the constant site counts from polycore. Trees are only built when a species / subtype group has more than 2 unique sequences, and 1,000 ultrafast bootstrap replicates are run when there are more than 4 unique sequences.

Trees are midpoint rooted and branch lengths are rescaled by the reference genome length. Trees are then **partitioned** into groups using DBSCAN on the tree's pairwise (patristic) distances (default threshold: `25`).

{: .todo}
Add an example tree image.

## Pairwise Distances & Linkage

Pairwise SNP distances are computed by polycore from the core genome alignment. Sample pairs are classified as:

- **Strongly linked** - SNP distance ≤ `strong_link_threshold` (default: `5`)
- **Intermediately linked** - SNP distance > `strong_link_threshold` and ≤ `inter_link_threshold` (default: `10`)

---

# Summary Report

Results are summarized at two levels:

- A run-level summary (`CorgiSNPs-summary.csv`) with QC, classification, and resistance results for every sample.
- A species / subtype level [Microreact](https://microreact.org) report combining the phylogenetic tree, SNP distance matrix, and per-sample summary.

{: .todo}
Add an example Microreact report to `assets/example_output/` and a download link, as in the BigBacter overview.

{: .note}
For a full description of report contents, see the [Outputs]({{ site.baseurl }}/docs/v1.0/pages/outputs/results) page.
