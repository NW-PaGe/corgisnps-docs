---
title: Creating a Reference Set
layout: page
nav_order: 6
parent: v1.0
permalink: /docs/v1.0/pages/reference_sets/
---

# {{ page.title }}
{: .no_toc}

1. TOC
{:toc}

---

# Overview

A **reference set** tells CorgiSNPs which species it can analyze, how each species is divided into subtypes, and which reference genome to use for each subtype. CorgiSNPs ships with a reference set for *Candidozyma auris* (*Candida auris*). To analyze another species, a reference set must be created for it.

Each species in the reference set is used for:

- **Subtyping** - samples are assigned the subtype of the reference assembly with the highest ANI, provided it meets the species' `subtype_ani` threshold
- **Variant calling** - reads are aligned to the reference assembly for the sample's subtype, and variants are called and filtered using the species' `ploidy` and [analysis settings](#step-5-set-analysis-settings-recommended)
- **Phylogenetic analysis** - core genome, linkage and tree partitioning thresholds come from the species' or subtype's [analysis settings](#step-5-set-analysis-settings-recommended)
- **Antifungal resistance** (optional) - variants are annotated using the reference annotation and reported for the resistance targets defined in `amr`

The reference set is a directory supplied via [`--reference_db`]({{ site.baseurl }}/docs/v1.0/pages/inputs/#--reference_db) (default: `${projectDir}/assets/reference_db/`).

{: .important}
`--reference_db` replaces the bundled reference set. If you supply your own directory and still want to analyze *C. auris*, copy the *C. auris* entry and files into your directory.

---

# Before You Begin

## Species assignment (GAMBIT)

Samples without a `species` in the samplesheet are classified using the bundled GAMBIT fungal database. Check whether your species is included in the database's taxa list:

```bash
grep -i "<Genus> <species>" CorgiSNPs/assets/gambit_db/gambit-1.0.0-20241213-taxa-list.txt
```

If it is not listed, supply the `species` column in the samplesheet for those samples. Otherwise they will not get a species-level call, and they will fail QC and be excluded from downstream analysis.

## Automated QC (NCBI genome statistics)

Automated QC estimates sequencing depth from the species' expected genome length, and checks that each de novo assembly's length and GC content fall within the species' expected ranges. These come from the bundled NCBI statistics file (`--ncbi_stats`) unless they are set in the manifest ([Step 6](#step-6-set-automated-qc-ranges-optional)). Check whether your species is included, and what its ranges are:

```bash
grep -A 25 '"species_name": "<Genus> <species>"' CorgiSNPs/assets/ncbi_stats/2026-10-01_ncbi-fungal-sp.json
```

The file only lists species with at least 2 NCBI genomes, and its ranges are used when the species has at least 3. Names are matched against the record's `names`, so a species listed under another name is found through the aliases in the manifest's `species` field.

{: .important}
If the species is not in the NCBI statistics file and its manifest entry has no `length_range`, estimated depth is undetermined and samples will fail automated QC. Set `length_range` and `gc_range` in the manifest for these species.


---

# Step 1: Create the Directory

The reference set directory holds a `manifest.yml` file plus one sub-directory per species, named after the species entry's `name`, with one folder per file type:

```bash
reference_db/
├── manifest.yml
└── ${name}/
    ├── assembly/
    │   ├── ${subtype_1_assembly}.fna.gz
    │   └── ${subtype_2_assembly}.fna.gz
    └── annotation/
        └── ${subtype_1_annotation}.gff.gz
```

File fields in the manifest are looked up first as written (an absolute path, or a path relative to the launch directory), then at `<reference_db>/<name>/<assembly|annotation>/<file>`.

---

# Step 2: Select Reference Genomes

Each subtype needs one reference assembly (FASTA, may be gzipped). Assemblies are also used to build the sourmash signatures for subtyping, so every subtype you want samples assigned to must have its own entry.

{: .todo}
Add guidance on selecting reference assemblies for a species: recommended sources (e.g., NCBI RefSeq / GenBank), quality criteria (completeness, contiguity), how many subtypes to define, and how subtypes should be named.

---

# Step 3: Set the Subtype ANI Threshold

`subtype_ani` is the minimum ANI (as a fraction, e.g., `0.997`) between a sample and its closest subtype reference for the subtype to be assigned. Samples below the threshold are `undefined`; they fail QC and are excluded from downstream analysis, but the rest of the run continues. If no value is set, `0.997` is used. If subtypes of the same species have different values, the highest (strictest) value is used for the species.

{: .todo}
Add guidance on how to choose `subtype_ani` for a new species (e.g., by comparing ANI within and between subtypes for a set of known genomes).

---

# Step 4: Set the Ploidy

`ploidy` is passed to FreeBayes for variant calling and to polycore for core genome analysis. CorgiSNPs supports organisms from haploid through triploid.

{: .important}
Set `min_allele_fraction` to match the ploidy in the species' [analysis settings](#step-5-set-analysis-settings-recommended): the suggested value is `0.8` for haploid organisms and `0.25` for diploid / triploid organisms. Because the value is set per species, haploid and non-haploid species can be analyzed in the same run.

{: .todo}
Add any additional guidance for diploid / triploid species.

---

# Step 5: Set Analysis Settings (Recommended)

The variant calling and phylogenetic analysis thresholds can be set in the manifest, on the species entry, on a subtype, or both. Each setting uses the same name as its pipeline parameter. **Setting them in the manifest is the recommended approach**: the values that suit a species are stored with its reference genomes, every run uses them, and species with different needs (e.g., haploid and diploid) can be analyzed together in one run.

For each sample, the most specific value wins:

|Level|Where it is set|Applies to|
|:-|:-|:-|
|Subtype|On a subtype in `manifest.yml`|Samples on that subtype|
|Species|On a species entry in `manifest.yml`|Samples on every subtype of the species, unless the subtype sets its own value|
|Run|`--{parameter}`, `-params-file`, or `nextflow.config` (see [Inputs]({{ site.baseurl }}/docs/v1.0/pages/inputs/))|Samples whose species and subtype don't set the value|

{: .important}
A run-level parameter does **not** override a value set in the manifest. For example, `--min_allele_fraction 0.5` has no effect on a species whose manifest entry sets `min_allele_fraction`. To change a value for such a species, edit the manifest (or use a copy of the reference set via `--reference_db`).

## Available settings

|Setting|Type|Default (run level)|Used for|
|:-|:-|:-|:-|
|`limit_coverage`|Whole number, ≥ 1|`100`|Coverage cap for FreeBayes and `samtools mpileup`|
|`min_base_depth`|Whole number, ≥ 0|`10`|`low_depth` filter and consensus masking|
|`min_base_quality`|Whole number, ≥ 0|`30`|`low_base_qual` filter and consensus masking|
|`min_mapping_quality`|Whole number, ≥ 0|`40`|`low_map_qual` filter|
|`min_allele_fraction`|Number, 0-1|`0.8`|`low_af` filter|
|`min_fwd_strand_fraction`|Number, 0-1|`0.3`|`strand_bias` filter|
|`max_strand_bias`|Whole number, ≥ 0|`15`|`strand_bias` filter|
|`max_read_pos_bias`|Whole number, ≥ 0|`30`|`read_pos_bias` filter|
|`min_genome_fraction`|Number, 0-1|`0.9`|Minimum genome fraction for a sample to enter the core genome|
|`min_core_fraction`|Number, 0-1|`0.9`|Minimum fraction of samples with data for a site to enter the core genome|
|`strong_link_threshold`|Whole number, ≥ 0|`5`|Upper SNP distance for strong linkage|
|`inter_link_threshold`|Whole number, ≥ 0|`10`|Upper SNP distance for intermediate linkage|
|`partition_distance`|Whole number, ≥ 0|`25`|Distance threshold for tree partitions|

See [Overview]({{ site.baseurl }}/docs/v1.0/pages/overview/#calling--filtering-variants) for how each filter is applied. We recommend setting **all** of them at the species level, even where the value matches the default. This records the full configuration for the species in one place and keeps its results the same if the pipeline defaults change. Use subtype-level values only where a subtype needs something different.

```yaml
- name: genus_species
  species:
    - Genus species
  ploidy: 2
  min_allele_fraction: 0.25     # species level: all subtypes
  partition_distance: 25
  subtypes:
    - subtype: [Subtype A]
      assembly: GCA_000000000.1.fna.gz
      min_core_fraction: 0.95   # subtype level: Subtype A only
    - subtype: [Subtype B]
      assembly: GCA_000000001.1.fna.gz
```

## Things to know

- **The startup log lists overrides.** At the start of each run, CorgiSNPs lists every subtype whose settings differ from the run-level values, so you can confirm what was applied.
- **Samples with a supplied reference use run-level values.** Samples with a `reference` in the samplesheet don't use the manifest, so they use the run-level parameters.
- **Each tree has one set of phylogenetic settings.** All samples in a species / subtype tree share its phylogenetic settings. If they disagree (e.g., a sample with a supplied reference alongside manifest samples), the run-level values are used for that tree and a warning is logged.
- **Existing database genomes are not re-processed.** Changing variant calling settings does not change consensus genomes already saved in a [CorgiSNPs database]({{ site.baseurl }}/docs/v1.0/pages/outputs/db/), which are still included in later trees. Re-run those samples if they need to reflect the new settings.

{: .todo}
Add guidance on choosing values for a new species (e.g., `min_allele_fraction` for diploid / triploid organisms, and linkage thresholds based on the species' mutation rate and outbreak history).

---

# Step 6: Set Automated QC Ranges (Optional)

A sample fails automated QC if its de novo assembly length or GC content falls outside its species' range. The ranges can be set on the species entry:

|Field|Units|Description|
|:-|:-|:-|
|`length_range`|bp|`[min, max]` acceptable assembly length|
|`gc_range`|%|`[min, max]` acceptable assembly GC content (0-100)|

```yaml
- name: genus_species
  species:
    - Genus species
  length_range: [11000000, 13000000]
  gc_range: [40.0, 49.5]
```

Each range is chosen separately:

1. **Manifest** - used when the species entry sets it.
2. **NCBI statistics** - otherwise, the range from `--ncbi_stats` is used if the species has at least 3 NCBI genomes. These ranges are the mean ± 2.58 standard deviations of the NCBI genomes for the species.
3. **Neither** - the check is reported as undetermined (in `qc_reason`) and does not fail the sample.

Set ranges in the manifest when the species has few or no NCBI genomes, or when the NCBI genomes don't represent the samples you sequence. The species' `length_range` and `gc_range` in the NCBI statistics file are a good starting point. For a species with no NCBI genomes, `length_range` is also used to estimate sequencing depth (from its midpoint), so set it to allow those samples to pass QC.

{: .note}
QC ranges can only be set on the species entry, not on a subtype. The source of each sample's ranges is reported in the `qc_range_source` [summary column]({B}/outputs/reports/#summary-columns).

{: .todo}
Add guidance on choosing ranges for a new species (e.g., how wide to make them relative to the NCBI values).

---

# Step 7: Add Antifungal Resistance Targets (Optional)

Resistance targets are defined on a subtype with the `amr` field. A subtype with `amr` must also have an `annotation` (GFF), and each target's `gene` must match a gene name in that annotation. Coordinates are in the assembly's coordinates.

When more than one subtype of a species has `amr`, mark one with `primary: true`. Samples on subtypes without `amr` have the primary subtype's target genes extracted and re-called against the primary reference. A single `amr` subtype is primary automatically.

```yaml
      amr:
        - chrom: CONTIG_ID          # contig containing the gene
          gene: GENE                # gene name, as it appears in the annotation
          coords: [1, 1000]         # gene start / end
          regions:                  # optional named regions within the gene
            - name: REGION
              coords: [100, 200]
```

{: .todo}
Add guidance on how to identify resistance genes, regions, and coordinates for a new species, and how to verify them against the annotation.

---

# Step 8: Write the Manifest

Add one entry per species to `manifest.yml`.

## Species fields

These apply to every subtype of the species; a subtype can override them.

|Field|Required|Description|
|:-|:-|:-|
|`name`|Yes|Species ID and directory name. Letters, numbers, `.`, `_`, and `-` only; must be unique.|
|`species`|Yes|List of species names. The first is preferred; the rest are aliases (e.g., older names). Samplesheet and GAMBIT species names are matched against all of them.|
|`ploidy`|No|Ploidy used for variant calling.|
|`subtype_ani`|No|ANI threshold for subtyping, as a fraction (e.g., `0.997`).|
|`length_range`|No|Acceptable de novo assembly length for automated QC, in bp: `[min, max]` (see [Step 6](#step-6-set-automated-qc-ranges-optional)). Species level only.|
|`gc_range`|No|Acceptable de novo assembly GC content for automated QC, in %: `[min, max]` (see [Step 6](#step-6-set-automated-qc-ranges-optional)). Species level only.|
|Analysis settings|No|Variant calling and phylogenetic thresholds for every subtype (see [Step 5](#step-5-set-analysis-settings-recommended)).|
|`subtypes`|Yes|One entry per subtype (below).|

## Subtype fields

|Field|Required|Description|
|:-|:-|:-|
|`subtype`|Yes|List of subtype names; must be unique within the species.|
|`assembly`|Yes|Reference assembly (FASTA, may be gzipped). File in `<name>/assembly/`, or a path.|
|`annotation`|No|Annotation for the assembly (GFF). File in `<name>/annotation/`, or a path. Required when `amr` is set.|
|`amr`|No|Resistance targets (see [Step 7](#step-7-add-antifungal-resistance-targets-optional)).|
|`primary`|No|`true` for the subtype whose resistance targets are used for samples on subtypes without `amr`. Only needed when more than one subtype has `amr`.|
|Analysis settings|No|Variant calling and phylogenetic thresholds for this subtype only, overriding the species' values (see [Step 5](#step-5-set-analysis-settings-recommended)). QC ranges can't be set here.|

Any other field is passed through to the pipeline unchanged.

{: .note}
Species and subtype names are compared after converting to lowercase and replacing spaces and special characters with underscores, so `Clade I` and `clade_i` match.

## Template

```yaml
## Genus species
- name: genus_species
  species:
    - Genus species
  ploidy: 1
  subtype_ani: 0.997
  # Automated QC
  length_range: [11000000, 13000000]
  gc_range: [40.0, 49.5]
  # Variant calling
  limit_coverage: 100
  min_base_depth: 10
  min_base_quality: 30
  min_mapping_quality: 40
  min_allele_fraction: 0.8
  min_fwd_strand_fraction: 0.3
  max_strand_bias: 15
  max_read_pos_bias: 30
  # Phylogenetics
  min_genome_fraction: 0.9
  min_core_fraction: 0.9
  strong_link_threshold: 5
  inter_link_threshold: 10
  partition_distance: 25
  subtypes:
    - subtype: [Subtype A]
      assembly: GCA_000000000.1.fna.gz
      annotation: GCA_000000000.1.gff.gz
      primary: true
      amr:
        - chrom: CONTIG_ID
          gene: GENE
          coords: [1, 1000]
          regions:
            - name: REGION
              coords: [100, 200]
    - subtype: [Subtype B]
      assembly: GCA_000000001.1.fna.gz
```

## Example: *Candidozyma auris*

The bundled *C. auris* entry defines five clades, with resistance targets for the *FKS1* hot spot regions on Clade I. It sets every analysis setting at the species level, so run-level values for these parameters do not apply to *C. auris* samples. It does not set QC ranges, so *C. auris* samples are checked against the NCBI ranges:

```yaml
## Candidozyma auris
- name: candidozyma_auris
  species:
    - Candidozyma auris
    - Candida auris
  ploidy: 1
  subtype_ani: 0.997
  # Variant calling
  limit_coverage: 100
  min_base_depth: 10
  min_base_quality: 30
  min_mapping_quality: 40
  min_allele_fraction: 0.8
  min_fwd_strand_fraction: 0.3
  max_strand_bias: 15
  max_read_pos_bias: 30
  # Phylogenetics
  min_genome_fraction: 0.9
  min_core_fraction: 0.9
  strong_link_threshold: 5
  inter_link_threshold: 10
  partition_distance: 25
  subtypes:
    - subtype: [Clade I]
      assembly: GCA_016772135.1.fna.gz
      annotation: GCA_016772135.1.gff.gz
      amr:
        - chrom: CP060340.1
          gene: FKS1
          coords: [219735, 225401]
          regions:
            - name: HS1
              coords: [221637, 221663]
            - name: HS2
              coords: [223782, 223805]
            - name: HS3
              coords: [221805, 221807]
    - subtype: [Clade II]
      assembly: GCF_003013715.1.fna.gz
    - subtype: [Clade III]
      assembly: GCF_002775015.1.fna.gz
    - subtype: [Clade IV]
      assembly: GCA_003014415.1.fna.gz
    - subtype: [Clade V]
      assembly: GCA_016809505.1.fna.gz
```

{: .todo}
Add the rationale for the *C. auris* reference assemblies, `subtype_ani` value, and analysis settings, and whether its QC ranges should be fixed in the manifest.

---

# Step 9: Validate & Test

The reference set is validated at the start of every run (unless `--validate_refs false`). Validation checks required fields, allowed `name` characters, unique names and subtypes, that files exist, that `amr` subtypes have an `annotation` and a `gene` for every target, that a single primary subtype is set when needed, that analysis settings are numbers of the right type and range (e.g., `min_allele_fraction` from 0 to 1), and that `length_range` and `gc_range` are `[min, max]` pairs with min ≤ max, set only on species entries. All problems are reported in a single error.

Run CorgiSNPs with your reference set:

```bash
nextflow run NW-PaGe/CorgiSNPs \
    -r main \
    -profile docker \
    --input samplesheet.csv \
    --outdir results \
    --db corgisnps_db \
    --reference_db /path/to/reference_db
```

{: .todo}
Add a recommended validation procedure for a new reference set (e.g., a panel of samples with known subtypes and resistance profiles, and the expected results).

---

# Additional Species Notes

{: .todo}
Space for species-specific instructions and lessons learned when creating reference sets (one section per species).

---

# Sharing a Reference Set

{: .todo}
Describe how to contribute a new reference set to the CorgiSNPs repository (e.g., via a pull request to `assets/reference_db/`) and any review requirements.
