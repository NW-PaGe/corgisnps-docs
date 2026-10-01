---
title: CorgiSNPs Database
layout: page
nav_order: 2
grandparent: v1.0
parent: Outputs
permalink: /docs/v1.0/pages/outputs/database
---

# {{ page.title }}
{: .no_toc}

1. TOC
{:toc}

## Overview

CorgiSNPs maintains a simple database of consensus genomes for routine fungal surveillance analysis. Each execution compares new samples from the input samplesheet (`--input`) against historical samples stored in the database (`--db`) that share the same species and subtype.

{: .important}
Samples are only saved to the database when you run CorgiSNPs using the `--push true` parameter.

{: .note}
The CorgiSNPs database is separate from the reference set (`--reference_db`). The reference set defines the species, subtypes, and reference genomes used by the pipeline - see [Creating a Reference Set]({{ site.baseurl }}/docs/v1.0/pages/reference_sets/).

This page details the database structure and the purpose of each component within a CorgiSNPs database.

## Database Structure

The CorgiSNPs database is organized by species and subtype:

```bash
${db}/
└── ${species}/
    └── ${subtype}/
        └── ${sample}.fa.gz
```

### Directory Structure Explanation

- **`${db}`**: Root database directory
- **`${species}`**: Species name, lowercase with spaces and special characters replaced by underscores (e.g., `candidozyma_auris`)
- **`${subtype}`**: Subtype name, lowercase with spaces and special characters replaced by underscores (e.g., `clade_i`)

---

## Subtype-Level Files

### Consensus Genomes

#### Purpose
Per-sample consensus genomes, all built against the same reference genome for the species / subtype. These are combined with the consensus genomes of new samples during core genome and phylogenetic analysis.

#### Contents

|File|Description|
|:-|:-|
|`${sample}.fa.gz`|Consensus genome produced by `bcftools consensus` with low quality sites masked (see [Overview]({{ site.baseurl }}/docs/v1.0/pages/overview/#consensus-genome))|

{: .note}
Consensus genomes reflect the variant calling settings in effect when they were created. Changing a species' or subtype's [analysis settings]({{ site.baseurl }}/docs/v1.0/pages/reference_sets/#step-5-set-analysis-settings-recommended) does not update genomes already in the database. Re-run those samples if they need to reflect the new settings.

{: .todo}
Describe what happens to existing database entries when a subtype's reference assembly is changed in the reference set, and the recommended procedure (e.g., start a new database or re-run historical samples).

{: .todo}
Document how to remove a sample from the database (e.g., deleting its `${sample}.fa.gz` file) and any other database maintenance guidance.

---
