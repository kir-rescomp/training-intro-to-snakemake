# Episode 3: A Real Snakefile — RNA-seq from Scratch

## The pipeline

Over this episode you will build a complete, realistic RNA-seq quantification pipeline:

1. **FastQC** — quality assessment of raw reads
2. **fastp** — adapter trimming and quality filtering (paired-end)
3. **HISAT2** — splice-aware alignment to the reference genome
4. **samtools** — sort and index BAM files
5. **featureCounts** — read counting against a gene annotation (all samples in one call)

The inputs are paired-end FASTQ files. The final output is a single counts matrix covering all samples. Every step produces a log file. Intermediate trimmed FASTQs are automatically cleaned up when no longer needed.

## Project layout

Before writing a single rule, establish a clean directory structure:

<div class="dracula" markdown="1">
```rust
rnaseq_pipeline/
├── Snakefile
├── config.yaml
├── data/
│   ├── SRR014335.fastq
│   ├── SRR014336.fastq
│   ├── SRR014337.fastq
│   ├── SRR014339.fastq
│   ├── SRR014340.fastq
│   └── SRR014341.fastq'
├── logs/
└── results/
```
</div>

Keep `logs/` and `results/` separate from `data/`. This means you can safely delete all intermediate results without touching raw inputs, and re-run from scratch is simply `rm -rf results/ logs/`.

## The config file

Rather than hardcoding sample names or genome paths inside the Snakefile, put them in a YAML config file. This is the single file a collaborator — or your future self — should need to edit when reusing the pipeline on a new dataset.

!!! tip "Extracting filename pre-fixes can be done with `ls -1 | awk -F'[-.]' '{print $1}'`"

<div class="dracula" markdown="1">
```yaml title="config.yaml"
samples:
  - SRR014335
  - SRR014336
  - SRR014337
  - SRR014339
  - SRR014340
  - SRR014341

genome:
  hisat2_index: /well/kir/references/hg38/hisat2/genome
  gtf: /well/kir/references/hg38/gencode.v44.annotation.gtf

threads:
  fastqc: 2
  fastp: 4
  hisat2: 4
  featurecounts: 4
```
</div>

Load it at the top of the Snakefile with the `configfile:` directive:

<div class="snakefile" markdown="1">
```python title="Snakefile"
configfile: "config.yaml"

SAMPLES = config["samples"]
```
</div>

After this, `config` is a standard Python dictionary. Changing the sample list in `config.yaml` is the only edit needed to run the same pipeline on a new experiment.

## `rule all`

Define all desired final outputs upfront. This forces you to be explicit about what the pipeline produces, and gives Snakemake an unambiguous target to plan backwards from.

```python title="Snakefile"
rule all:
    input:
        # FastQC reports — one HTML per sample per read direction
        expand("results/fastqc/{sample}_fastqc.html",
               sample=SAMPLES ),
        # fastp QC reports
        expand("results/fastp/{sample}_fastp.html", sample=SAMPLES),
        # Final counts matrix
        "results/counts/all_samples.counts.txt"
```

## Rule by rule

### FastQC

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule fastqc:
    input:
        "data/{sample}.fastq"
    output:
        html="results/fastqc/{sample}_fastqc.html",
        zip="results/fastqc/{sample}_fastqc.zip"
    log:
        "logs/fastqc/{sample}.log"
    threads: config["threads"]["fastqc"]
    shell:
        "fastqc {input} --outdir results/fastqc/ --threads {threads} &> {log}"
```
</div>

Two new directives appear here.

**`log:`** declares a file to capture the rule's output. It behaves like any other Snakemake file path — the parent directory is created automatically, and the file is *never* deleted even if the rule fails. Critically, Snakemake does **not** redirect output to this file automatically; you must do it yourself in the shell command. Use `&> {log}` to capture both stdout and stderr, or `2> {log}` for tools that write results to stdout and diagnostics to stderr.

**`threads:`** declares how many CPU threads this rule needs. Snakemake uses this locally to avoid overcommitting cores across concurrent jobs. On the cluster, it maps to `--cpus-per-task` in the SLURM submission (wired up in Episode 4). Reference it inside the shell command as `{threads}`.

### fastp (trimming)

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule fastp:
    input:
        "data/{sample}.fastq",
    output:
        fastq=temp("results/trimmed/{sample}.fastq"),
        html="results/fastp/{sample}_fastp.html",
        json="results/fastp/{sample}_fastp.json"
    log:
        "logs/fastp/{sample}.log"
    threads: config["threads"]["fastp"]
    shell:
        """
        fastp \
            -i {input} \
            -o {output} \
            --html {output.html} --json {output.json} \
            --thread {threads} \
            2> {log}
        """
```
</div>

The trimmed FASTQs are wrapped in **`temp()`**. This marks them as intermediate files — Snakemake will automatically delete them once every downstream rule that consumes them has finished. For a 30-sample experiment with 20 GB FASTQs per sample, this saves hundreds of gigabytes of scratch space without any manual cleanup.

The HTML and JSON reports are *not* marked `temp()` — you want to keep those for QC review and MultiQC.

### HISAT2 (alignment, sort, index)

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule hisat2:
    input:
        "results/trimmed/{sample}.fastq",

    output:
        bam=protected("results/bam/{sample}.sorted.bam"),
        bai="results/bam/{sample}.sorted.bam.bai"
    log:
        "logs/hisat2/{sample}.log"
    params:
        index=config["genome"]["hisat2_index"],
        extra="--dta"
    threads: config["threads"]["hisat2"]
    shell:
        """
        hisat2 \
            -x {params.index} \
            -U {input} \
            --threads {threads} \
            {params.extra} \
            2> {log} \
        | samtools sort -@ {threads} -o {output.bam}
        samtools index {output.bam}
        """
```
</div>

Two new things here.

**`params:`** holds values that are *not* input or output files — tool parameters, index prefixes, or flags. Why not embed these directly in the shell string? Because Snakemake tracks `input:` and `output:` files for dependency resolution and reruns. Putting the genome index prefix in `params:` rather than `input:` means Snakemake won't try to check whether the index "exists" as a single file (it's a directory with multiple files), and won't trigger reruns if the index timestamp changes. It also makes the config-driven values visible in one place, separate from the mechanics of the shell command.

**`protected()`** write-protects the sorted BAM once created. Snakemake will `chmod -w` the file after the rule completes. This prevents accidental deletion or overwrite of an expensive-to-regenerate alignment. If you ever need to rerun, you will need to `chmod +w` the file first — which is a deliberate friction, not a bug.

The shell command pipes HISAT2's SAM output directly into `samtools sort`. No intermediate uncompressed SAM file ever touches disk, saving both time and space.

### featureCounts

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule featurecounts:
    input:
        bams=expand("results/bam/{sample}.sorted.bam", sample=SAMPLES),
        bais=expand("results/bam/{sample}.sorted.bam.bai", sample=SAMPLES),
        gtf=config["genome"]["gtf"]
    output:
        "results/counts/all_samples.counts.txt"
    log:
        "logs/featurecounts/featurecounts.log"
    threads: config["threads"]["featurecounts"]
    shell:
        """
        featureCounts \
            -a {input.gtf} \
            -o {output} \
            -T {threads} \
            -p --countReadPairs \
            {input.bams} \
            2> {log}
        """
```
</div>

This is a **many-to-one rule**: it takes all BAMs as input and produces a single output. Notice that `expand()` is used *inside the rule's input block*, not in `rule all`. This resolves the wildcards immediately, producing a concrete list. Snakemake calls this an aggregate step.

!!! note-sticky "`expand()` inside a rule removes wildcards"
    Using `expand()` inside `input:` generates a fixed list — there are no wildcards left to resolve. Snakemake will not run this rule once per sample; it will run it once, receiving all BAMs as input simultaneously. The upstream `hisat2` rules still run per-sample normally; `featurecounts` simply waits until all of them have completed.

## The complete Snakefile

<div class="snakefile" markdown="1">

```python title="Snakefile"
configfile: "config.yaml"

SAMPLES = config["samples"]

localrules: all


rule all:
    input:
        expand("results/fastqc/{sample}_fastqc.html",
               sample=SAMPLES,
        expand("results/fastp/{sample}_fastp.html", sample=SAMPLES),
        "results/counts/all_samples.counts.txt"


rule fastqc:
    input:
        "data/{sample}_{read}.fastq"
    output:
        html="results/fastqc/{sample}_fastqc.html",
        zip="results/fastqc/{sample}_fastqc.zip"
    log:
        "logs/fastqc/{sample}.log"
    threads: config["threads"]["fastqc"]
    shell:
        "fastqc {input} --outdir results/fastqc/ --threads {threads} &> {log}"


rule fastp:
    input:
        "data/{sample}.fastq",
    output:
        fastq=temp("results/trimmed/{sample}_R1.fastq"),
        html="results/fastp/{sample}_fastp.html",
        json="results/fastp/{sample}_fastp.json"
    log:
        "logs/fastp/{sample}.log"
    threads: config["threads"]["fastp"]
    shell:
        """
        fastp \
            -i {input} \
            -o {output} \
            --html {output.html} --json {output.json} \
            --thread {threads} \
            2> {log}
        """


rule hisat2:
    input:
        "results/trimmed/{sample}.fastq",

    output:
        bam=protected("results/bam/{sample}.sorted.bam"),
        bai="results/bam/{sample}.sorted.bam.bai"
    log:
        "logs/hisat2/{sample}.log"
    params:
        index=config["genome"]["hisat2_index"],
        extra="--dta"
    threads: config["threads"]["hisat2"]
    shell:
        """
        hisat2 \
            -x {params.index} \
            -U {input} \
            --threads {threads} \
            {params.extra} \
            2> {log} \
        | samtools sort -@ {threads} -o {output.bam}
        samtools index {output.bam}
        """


rule featurecounts:
    input:
        bams=expand("results/bam/{sample}.sorted.bam", sample=SAMPLES),
        bais=expand("results/bam/{sample}.sorted.bam.bai", sample=SAMPLES),
        gtf=config["genome"]["gtf"]
    output:
        "results/counts/all_samples.counts.txt"
    log:
        "logs/featurecounts/featurecounts.log"
    threads: config["threads"]["featurecounts"]
    shell:
        """
        featureCounts \
            -a {input.gtf} \
            -o {output} \
            -T {threads} \
            -p --countReadPairs \
            {input.bams} \
            2> {log}
        """
```
</div>


## Running locally

For a small test dataset, or to verify the DAG before submitting to the cluster:

<div class="dracula" markdown="1">
```py
# Always dry-run first
snakemake --cores 8 -n -p

# Production run 
snakemake --cores 8
```
</div>

`--cores` sets the total CPU budget Snakemake may use concurrently. It will schedule multiple independent jobs in parallel, respecting each rule's `threads:` declaration. For example, with `--cores 8`, two four-thread fastp jobs could run simultaneously.

!!! warning "Do not run real data on the login node"
    On the BMRC cluster, the login node is shared. Use `--cores 1` with tiny test files to verify the pipeline structure, then move to cluster submission (Episode 4) for real workloads.

---

!!! circle-question "Exercise"

    ## Exercise 3

    1. Add a fourth sample `SRR004` to `config.yaml`. Run `snakemake --cores 1 -n` and confirm that only jobs for `SRR004` appear in the execution plan.

    2. Add a `multiqc` rule that takes all FastQC zip files and fastp JSON files as input and produces `results/multiqc/multiqc_report.html`. Add this output to `rule all`.

    3. Render the rule graph: `snakemake --rulegraph | dot -Tsvg > rulegraph.svg`.

    ??? success "MultiQC rule"
        ```python
        rule multiqc:
            input:
                fastqc=expand("results/fastqc/{sample}_{read}_fastqc.zip",
                              sample=SAMPLES, read=["R1", "R2"]),
                fastp=expand("results/fastp/{sample}_fastp.json", sample=SAMPLES)
            output:
                "results/multiqc/multiqc_report.html"
            log:
                "logs/multiqc/multiqc.log"
            shell:
                """
                multiqc results/fastqc/ results/fastp/ \
                    --outdir results/multiqc/ \
                    2> {log}
                """
        ```
        Add `"results/multiqc/multiqc_report.html"` to the `input:` of `rule all`.

---

## Key takeaways

!!! clipboard-list "Episode 3 summary"
    - **`configfile:`** loads a YAML file into the `config` dictionary — sample lists and paths belong there, not hardcoded in the Snakefile.
    - **`log:`** declares a capture file for stderr/stdout; redirect to it manually with `2> {log}` or `&> {log}`.
    - **`params:`** holds non-file values (tool flags, index paths) that should not be dependency-tracked.
    - **`temp()`** marks intermediates for automatic deletion after all downstream consumers have completed.
    - **`protected()`** write-protects expensive outputs once generated — a deliberate safeguard.
    - **Aggregate rules** use `expand()` inside `input:` to collect all samples into a single many-to-one job.
    - `localrules: all` keeps the target-collection rule on the login node, avoiding a pointless cluster submission.
