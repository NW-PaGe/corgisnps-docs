---
title: Inputs
layout: page
nav_order: 4
parent: v1.0
permalink: /docs/v1.0/pages/inputs/
---

# {{ page.title }}
{: .no_toc}

1. TOC
{:toc}

---

# Overview
Pipeline parameters can be adjusted using the following methods:

1. At the command line using `--{parameter_name}` (e.g., `--input`)
2. In the `nextflow.config` file
3. In a JSON file via the `-params-file` parameter

It is also possible to pass arguments directly to a pipeline process using the `ext.args` variable in `conf/modules.config` (see example below):
```
    withName: 'IQTREE' {
        container   = "public.ecr.aws/o8h2f0o1/iqtree3:3.1.3"
        ext.args    = '-m GTR+I+G --seqtype DNA'
        stageInMode = 'copy'
        publishDir  = [ enabled: false, path: { params.outdir } ]
    }
```

---

# Input / Output Options

## `--input`
Path to the samplesheet.

### Example samplesheet
`samplesheet.csv`:
```
sample,fastq_1,fastq_2
sample01,/path/to/sample01_R1_001.fastq.gz,/path/to/sample01_R2_001.fastq.gz
sample02,/path/to/sample02_R1_001.fastq.gz,/path/to/sample02_R2_001.fastq.gz
```

### Samplesheet columns

{: .note}
- Required columns: `sample`, and `fastq_1` + `fastq_2` or `sra`
- Sample names cannot contain spaces.

|Column Name|Description|
|:-|:-|
|`sample`|Sample name. Cannot contain spaces.|
|`fastq_1`|Path to the forward (R1) Illumina read file (`.fq.gz` or `.fastq.gz`).|
|`fastq_2`|Path to the reverse (R2) Illumina read file (`.fq.gz` or `.fastq.gz`).|
|`sra`|SRA accession (e.g., `SRR12345678`). Must start with `SRR`, `ERR`, or `DRR`. Reads are downloaded from NCBI.|
|`species`|Species name (e.g., `Candidozyma auris`). Must match a species name or alias in the reference set. If provided with `subtype`, classification is skipped for this sample.|
|`subtype`|Subtype name (e.g., `Clade I`). Must match a subtype name for the species in the reference set. If provided with `species`, classification is skipped for this sample.|
|`reference`|Path to a reference genome (`.fa.gz`, `.fasta.gz`, or `.fna.gz`) to use for this sample instead of the reference set. Must be supplied with `ploidy`.|
|`ploidy`|Ploidy of the organism (integer). Must be supplied with `reference`.|

## `--outdir`
Path to the output directory.

## `--db`
Path to the CorgiSNPs surveillance database.

- Default: `null`

> Consensus genomes stored in this database for a species and subtype are included in the phylogenetic analysis of new samples with the same species and subtype. See [CorgiSNPs Database](../outputs/database).


## `--push`
Whether to save consensus genomes to the CorgiSNPs database.

- Options: `true`, `false`
- Default: `false`

> When enabled, each sample's consensus genome is written to `--db` so it is included in future runs.

## `--microreact_template`
Path to the Microreact template JSON file used for visualization.

- Default: `${projectDir}/assets/template.microreact`

---

# Workflow Toggles

## `--classify`
Whether to run de novo assembly, species assignment, and subtyping for samples missing a species or subtype.

- Options: `true`, `false`
- Default: `true`

## `--variants`
Whether to run read alignment, variant calling, and consensus generation.

- Options: `true`, `false`
- Default: `true`

> Resistance detection and phylogenetic analysis require variant calling.

## `--amr`
Whether to run antifungal resistance detection.

- Options: `true`, `false`
- Default: `true`

## `--phylo`
Whether to run core genome and phylogenetic analysis.

- Options: `true`, `false`
- Default: `true`

---

# Automated QC Options

## `--ignore_qc`
Whether to ignore automated QC results.

- Options: `true`, `false`
- Default: `false`

{: .warning}
this allows low quality samples to pass to variant calling.

## `--min_depth_qc`
Minimum estimated mean read depth across the full genome.

- Options: `0...Inf`
- Default: `30`

> Depth is estimated as the total bases after filtering divided by the NCBI mean genome length for the species.

## `--min_q30_rate_qc`
Minimum read Q30 rate after filtering.

- Options: `0...1`
- Default: `0.8`

## `--max_z_score_qc`
Maximum absolute z-score for genome assembly size and % GC, relative to NCBI genomes for the species.

- Options: `0...Inf`
- Default: `2.58`

---

# Assembly Options

## `--assembler`
Assembler used by Shovill.

- Options: `skesa`, `spades`, `megahit`, `velvet`
- Default: `skesa`

## `--shovill_depth`
Target depth for Shovill downsampling.

- Options: `0...Inf`
- Default: `70`

## `--genome_size`
Expected genome size (bp).

- Options: `0...Inf`
- Default: `0`

> Use `0` to let Shovill estimate the genome size.

## `--min_contig_cov`
Minimum read depth to retain a contig in the de novo assembly.

- Options: `0...Inf`
- Default: `10`

## `--min_contig_len`
Minimum contig length for it to be retained in the de novo assembly.

- Options: `0...Inf`
- Default: `300`

---

# Database & Asset Options

## `--reference_db`
Path to the reference set directory (must contain `manifest.yml`).

- Default: `${projectDir}/assets/reference_db/`

> See [Creating a Reference Set](../reference_sets/).

## `--validate_refs`
Whether to validate the reference set before running.

- Options: `true`, `false`
- Default: `true`

## `--gambit_db`
Path to the GAMBIT fungal metadata database used for species assignment.

- Default: `${projectDir}/assets/gambit_db/gambit-fungal-metadata-1.0.0-20241213.gdb`

## `--gambit_h5_dir`
Path to the directory of GAMBIT signature files used for species assignment.

- Default: `${projectDir}/assets/gambit_db/signatures/`

## `--ncbi_stats`
Genome statistics for fungal species hosted on NCBI, used for automated QC.

- Default: `${projectDir}/assets/ncbi_stats/2026-02-10_ncbi-fungal-sp.json`

---

# Variant Calling Options

## `--max_reads`
The maximum number of reads to include in the analysis per sample.

- Options: `0...Inf`
- Default: `10_000_000`

> Samples with more than this number of reads will be randomly down-sampled using `seqtk sample`. Read counts are based on the sum of the forward and reverse reads.

## `--limit_coverage`
Maximum site coverage used during variant calling and pileup.

- Options: `0...Inf`
- Default: `100`

## `--min_base_depth`
Minimum read depth per allele / site.

- Options: `0...Inf`
- Default: `10`

## `--min_base_quality`
Minimum mean base quality (Phred) per allele / site.

- Options: `0...Inf`
- Default: `30`

## `--min_mapping_quality`
Minimum mean mapping quality (Phred) per allele.

- Options: `0...Inf`
- Default: `40`

## `--min_allele_fraction`
Minimum fraction of reads supporting an allele.

- Options: `0...1`
- Default: `0.8`

> Suggested values: `0.8` for haploid organisms, `0.25` for diploid / triploid organisms. Applies to all samples in the run.

## `--min_fwd_strand_fraction`
Minimum forward-strand fraction for strand balance at each allele.

- Options: `0...1`
- Default: `0.3`

## `--max_strand_bias`
Maximum strand bias score at each allele.

- Options: `0...Inf`
- Default: `15`

## `--max_read_pos_bias`
Maximum read position bias score at each allele.

- Options: `0...Inf`
- Default: `30`

---

# Phylogenetic Analysis Options

## `--min_genome_fraction`
Minimum fraction of the reference genome that must be covered for a sample to be included in core genome analysis.

- Options: `0...1`
- Default: `0.9`

## `--min_core_fraction`
Minimum fraction of samples that must have coverage at a reference site for it to be included in the core genome.

- Options: `0...1`
- Default: `0.9`

## `--strong_link_threshold`
Upper SNP distance threshold for defining a strong linkage between two samples.

- Options: `0...Inf`
- Default: `5`

## `--inter_link_threshold`
Upper SNP distance threshold for defining an intermediate linkage between two samples.

- Options: `0...Inf`
- Default: `10`

> Sample pairs with SNP distances above `--strong_link_threshold` and up to this value are classified as intermediately linked.

## `--partition_distance`
Distance threshold for grouping samples into tree partitions.

- Options: `0...Inf`
- Default: `25`
