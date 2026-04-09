<div class="catalog-return">
  <a href="https://kir-rescomp.github.io/kir-training-home/">← Return to KIR Training Catalogue</a>
</div>

<h1></h1>

<div class="admonition spinner" style="text-align: center;">
  <p class="admonition-title">
    <span style="display: inline-block; animation: pulse 2s ease-in-out infinite;">🚧</span>
    Work in Progress
  </p>
  <p>This repository is under active development.<br>Expected completion: <strong>20th of April 2026</strong></p>
</div>

<style>
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}
</style>

<p align="center">
    <img src="./images/snakemake_new_logo.png" alt="drawing" width="300">
</p>

  
!!! people-group "Who this workshop is for"

    Researchers and research software engineers comfortable with Python and bash who want to move beyond shell scripts and for loops. No prior Snakemake experience is assumed.


!!! chart-diagram "What you will build"

    By the end of Episode 5, you will have a complete, cluster-ready RNA-seq quantification pipeline:
    <div class="nord" markdown="1">
    ```py
    Paired-end FASTQs
           │
           ▼
       FastQC          ← quality assessment (per sample, per read)
           │
           ▼
        fastp           ← adapter trimming (intermediate outputs auto-deleted)
           │
           ▼
       HISAT2           ← splice-aware alignment (BAMs write-protected)
           │
           ▼
     featureCounts      ← quantification across all samples in one call
           │
           ▼
      counts matrix
    ```
    </div>

The pipeline is driven by a single `config.yaml` — adding a new sample is a one-line edit.

## Episodes

| Episode                               | Topic                                    | Key concepts                                                 |
| ------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| [Episode 1](ep01_motivation.md)       | From shell scripts to Snakemake          | Rules, inputs/outputs, `rule all`, dry runs                  |
| [Episode 2](ep02_wildcards_dag.md)    | Wildcards, `expand()`, and the DAG       | `{wildcards}`, `expand()`, `--dag`, `--rulegraph`            |
| [Episode 3](ep03_rnaseq_snakefile.md) | A real Snakefile — RNA-seq from scratch  | `configfile:`, `params:`, `log:`, `temp()`, `protected()`    |
| [Episode 4](ep04_slurm_drmaa.md)      | Scaling to the cluster — SLURM via DRMAA | `threads:`, `resources:`, `--executor drmaa`, `--drmaa-args`, profiles |
| [Episode 5](ep05_best_practices.md)   | Robustness and best practices            | `benchmark:`, `--rerun-incomplete`, `wildcard_constraints:`, conda envs |

## Before you start

This workshop assumes Snakemake 9, `snakemake-executor-plugin-drmaa`, and Python DRMAA bindings are already installed. See the [installation guide](../installation.md) for instructions specific to this cluster.

!!! clipboard-list "A note on the examples"

    All exercises use toy data (text files, word counts) in Episodes 1–2, then switch to a realistic but deliberately simplified RNA-seq skeleton in Episodes 3–5. The pipeline is designed to illustrate Snakemake concepts cleanly. For a production-grade RNA-seq workflow ready to run out of the box, see the [Snakemake wrappers](https://snakemake-wrappers.readthedocs.io/) and community workflow catalogues.
