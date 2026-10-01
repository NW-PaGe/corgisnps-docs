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
- **Variant calling** - reads are aligned to the reference assembly for the sample's subtype, and variants are called using the species' `ploidy`
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

If it is not listed, supply the `species` column in the samplesheet for those samples.

## Automated QC (NCBI genome statistics)

Automated QC estimates sequencing depth using the mean genome length of the species from the bundled NCBI statistics file (`--ncbi_stats`). Check whether your species is included:

```bash
grep -o '"species_name": "<Genus> <species>"' CorgiSNPs/assets/ncbi_stats/2026-02-10_ncbi-fungal-sp.json
```

{: .important}
If the species is not in the NCBI statistics file, estimated depth is undetermined and samples will fail automated QC. Assembly length and GC z-scores also require at least 3 NCBI genomes for the species.


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

`subtype_ani` is the minimum ANI (as a fraction, e.g., `0.997`) between a sample and its closest subtype reference for the subtype to be assigned. Samples below the threshold are `undefined`, which stops the pipeline. If no value is set, `0.997` is used. If subtypes of the same species have different values, the highest (strictest) value is used for the species.

{: .todo}
Add guidance on how to choose `subtype_ani` for a new species (e.g., by comparing ANI within and between subtypes for a set of known genomes).

---

# Step 4: Set the Ploidy

`ploidy` is passed to FreeBayes for variant calling and to polycore for core genome analysis. CorgiSNPs supports organisms from haploid through triploid.

{: .important}
[`--min_allele_fraction`]({{ site.baseurl }}/docs/v1.0/pages/inputs/#--min_allele_fraction) is set per run, not per species. The suggested value is `0.8` for haploid organisms and `0.25` for diploid / triploid organisms, so adjust it when running a non-haploid species.

{: .todo}
Add any additional guidance for diploid / triploid species.

---

# Step 5: Add Antifungal Resistance Targets (Optional)

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

# Step 6: Write the Manifest

Add one entry per species to `manifest.yml`.

## Species fields

These apply to every subtype of the species; a subtype can override them.

|Field|Required|Description|
|:-|:-|:-|
|`name`|Yes|Species ID and directory name. Letters, numbers, `.`, `_`, and `-` only; must be unique.|
|`species`|Yes|List of species names. The first is preferred; the rest are aliases (e.g., older names). Samplesheet and GAMBIT species names are matched against all of them.|
|`ploidy`|No|Ploidy used for variant calling.|
|`subtype_ani`|No|ANI threshold for subtyping, as a fraction (e.g., `0.997`).|
|`subtypes`|Yes|One entry per subtype (below).|

## Subtype fields

|Field|Required|Description|
|:-|:-|:-|
|`subtype`|Yes|List of subtype names; must be unique within the species.|
|`assembly`|Yes|Reference assembly (FASTA, may be gzipped). File in `<name>/assembly/`, or a path.|
|`annotation`|No|Annotation for the assembly (GFF). File in `<name>/annotation/`, or a path. Required when `amr` is set.|
|`amr`|No|Resistance targets (see [Step 5](#step-5-add-antifungal-resistance-targets-optional)).|
|`primary`|No|`true` for the subtype whose resistance targets are used for samples on subtypes without `amr`. Only needed when more than one subtype has `amr`.|

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

The bundled *C. auris* entry defines five clades, with resistance targets for the *FKS1* hot spot regions on Clade I:

```yaml
## Candidozyma auris
- name: candidozyma_auris
  species:
    - Candidozyma auris
    - Candida auris
  ploidy: 1
  subtype_ani: 0.997
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
Add the rationale for the *C. auris* reference assemblies and `subtype_ani` value.

---

# Step 7: Validate & Test

The reference set is validated at the start of every run (unless `--validate_refs false`). Validation checks required fields, allowed `name` characters, unique names and subtypes, that files exist, that `amr` subtypes have an `annotation` and a `gene` for every target, and that a single primary subtype is set when needed. All problems are reported in a single error.

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
