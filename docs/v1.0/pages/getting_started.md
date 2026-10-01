---
title: Getting Started
layout: page
nav_order: 2
parent: v1.0
permalink: /docs/v1.0/pages/getting_started/
---

# {{ page.title }}
{: .no_toc}

1. TOC
{:toc}

# Dependencies
CorgiSNPs is built using [Nextflow](https://www.nextflow.io/). Nextflow simplifies the development and execution of complex, scalable data analysis workflows by enabling reproducibility, portability across computing environments, and seamless integration with container technologies like Docker and Singularity.

The following are required to run CorgiSNPs:
- [Nextflow](https://www.nextflow.io/docs/latest/install.html) (version 25.10.0+)
- One of the following container engines:
    - [Podman](https://podman.io/docs/installation)
    - [Docker](https://docs.docker.com/engine/install/)
    - [Apptainer](https://apptainer.org/docs/admin/main/installation.html)
    - [Singularity](https://docs.sylabs.io/guides/3.0/user-guide/installation.html)

{: .important}
BigBacter does not support Conda / Mamba. Please submit a [feature request](https://github.com/NW-PaGe/CorgiSNPs/issues) if this is essential for your lab.


# Nextflow Basics
Below are some general pointers for how to run Nextflow workflows.
## Specifying the workflow version
There are two general ways you can specify which version of CorgiSNPs you want to run:
1. Tell Nextflow which version you want to use

    ```bash
    nextflow run NW-PaGe/CorgiSNPs \
        -r main \
        -profile docker \
        --input samplesheet.csv \
        --outdir results
    ```

2. Clone the workflow version manually
    ```bash
    git clone https://github.com/NW-PaGe/CorgiSNPs.git -b main
    ```

    ```bash
    nextflow run CorgiSNPs/main.nf \
        -profile docker \
        --input samplesheet.csv \
        --outdir results
    ```

{: .tip}
Nextflow caches repos in ~/.nextflow/assets/ by default. Removing this cache can be helpful when running into version-related issues.

## Specifying resource limits
We recommend adjusting the maximum resource limits that Nextflow can use. Setting these limits too high will cause the workflow to fail. This can be accomplished in the config file supplied via the `-c` parameter (see below).

{: .important}
Nextflow no longer allows resource limits to be specified from command line (e.g., `--max_cpus` or `--max_memory`).

`custom.config`
```
process {
    resourceLimits = [
        cpus: 8,
        memory: 14.GB
    ]
}
```

```bash
nextflow run NW-PaGe/CorgiSNPs \
    -r main \
    -c custom.config \
    -profile docker \
    --input samplesheet.csv \
    --outdir results
```

---

## Resuming a run
Nextflow can resume a run. This comes in handy when a workflow fails or when you need to make small parameter adjustments. Below is an example of how you can resume a workflow run:
```bash
nextflow run NW-PaGe/CorgiSNPs \
    -r main \
    -profile docker \
    --input samplesheet.csv \
    --outdir results \
    -resume
```

# Testing CorgiSNPs
Verify that CorgiSNPs is running properly using the command below. Update `-profile` to your preferred container engine.
```bash
nextflow run NW-PaGe/CorgiSNPs \
    -r main \
    -profile (docker|podman|apptainer|singularity),test \
    --outdir CorgiSNPs_test \
    --db CorgiSNPs_test_db
```
The test profile runs a set of public *Candidozyma auris* samples that are downloaded from NCBI SRA, so an internet connection is required. You can learn more about the test configuration [here](https://github.com/NW-PaGe/CorgiSNPs/blob/main/conf/test.config).

# Basic Usage
In-depth overviews of [inputs](../inputs/), [outputs](../outputs/), and [creating a reference set](../reference_sets/) are available. All other questions / issues should be submitted via the CorgiSNPs [GitHub issues](https://github.com/NW-PaGe/CorgiSNPs/issues) page or by [email](mailto:waphl-bioinformatics@doh.wa.gov).
