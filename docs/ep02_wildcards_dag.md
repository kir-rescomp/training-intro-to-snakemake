# Episode 2: Wildcards, `expand()`, and the DAG

## The scaling problem

In Episode 1 you wrote rules with hardcoded filenames. That works for one file. But RNA-seq experiments rarely involve one sample — you have `SRR001`, `SRR002`, `SRR003`, and next month, potentially six more.

Writing one rule per sample is not the answer:

<div class="dracula" markdown="1">
```python
# Don't do this
rule fastqc_SRR001:
    input: "data/SRR014335-chr1.fastq"
    output: "results/fastqc/SRR001_R1_fastqc.html"
    shell: "fastqc {input} -o results/fastqc/"

rule fastqc_SRR002:
    input: "data/SRR014337-chr1.fastq"
    output: "results/fastqc/SRR002_R1_fastqc.html"
    shell: "fastqc {input} -o results/fastqc/"
    # ... and so on
```
</div>

Instead, you write *one rule* that works for any sample, using a **wildcard**.

## Wildcards

A wildcard is a named placeholder in curly braces — `{sample}`, `{read}`, `{chromosome}` — that Snakemake fills in at runtime by matching against file paths it is trying to produce.

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule fastqc:
    input:
        "data/{sample}_{read}.fastq"
    output:
        html="results/fastqc/{sample}_{read}_fastqc.html",
        zip="results/fastqc/{sample}_{read}_fastqc.zip"
    shell:
        "fastqc {input} --outdir results/fastqc/"
```
</div>

When Snakemake needs to produce `results/fastqc/SRR001_R1_fastqc.html`, it pattern-matches that path against the output template `results/fastqc/{sample}_{read}_fastqc.html` and extracts `sample=SRR001`, `read=R1`. It then substitutes these values into the input template, expecting to find `data/SRR014335-chr1.fastq`.

!!! circle-info "Wildcards are resolved from outputs, not inputs"
    Snakemake always works backwards from a *requested output* to determine wildcard values. This means every wildcard that appears in `input:` must also appear in `output:` — otherwise Snakemake has no way to resolve its value.

## Named inputs and outputs

When a rule produces or consumes multiple files, use **named** inputs and outputs for clarity:

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule fastp:
    input:
        r1="data/{sample}_R1.fastq",
        r2="data/{sample}_R2.fastq"
    output:
        r1="results/trimmed/{sample}_R1.fastq",
        r2="results/trimmed/{sample}_R2.fastq",
        html="results/fastp/{sample}_fastp.html"
    shell:
        """
        fastp \
            -i {input.r1} -I {input.r2} \
            -o {output.r1} -O {output.r2} \
            --html {output.html}
        """
```
</div>

Named files are accessed with dot notation: `{input.r1}`, `{output.html}`. This is far less error-prone than relying on positional ordering, especially as rules grow.

## `expand()` — generating lists of targets

A wildcard rule can produce outputs for any sample, but `rule all` needs to name all of them explicitly. Writing them by hand defeats the purpose. `expand()` generates a concrete list of filenames by substituting values into a template:

<div class="dracula" markdown="1">
```python
expand("results/fastqc/{sample}_{read}_fastqc.html",
       sample=["SRR001", "SRR002", "SRR003"],
       read=["R1", "R2"])
```


This produces:

```python
[
    "results/fastqc/SRR001_R1_fastqc.html",
    "results/fastqc/SRR001_R2_fastqc.html",
    "results/fastqc/SRR002_R1_fastqc.html",
    "results/fastqc/SRR002_R2_fastqc.html",
    "results/fastqc/SRR003_R1_fastqc.html",
    "results/fastqc/SRR003_R2_fastqc.html",
]
```
</div>

By default, `expand()` produces the **Cartesian product** — every combination of `sample` and `read`. This is exactly what you want for FastQC reports.

Used in `rule all`:

<div class="snakefile" markdown="1">
```python title="Snakefile"
SAMPLES = ["SRR001", "SRR002", "SRR003"]

rule all:
    input:
        expand("results/fastqc/{sample}_{read}_fastqc.html",
               sample=SAMPLES,
               read=["R1", "R2"])
```
</div>

!!! tip "Capitalise global constants"
    Sample lists and other global values are written in `UPPER_CASE` at the top of the Snakefile by convention. This visually distinguishes them from wildcards (which are lowercase, resolved per-job) and Python variables local to a rule.

## The Directed Acyclic Graph (DAG)

Snakemake's internal model of your workflow is a **Directed Acyclic Graph**: each node is a concrete job (a rule applied to specific wildcard values), and each directed edge represents a dependency. Snakemake builds this graph automatically from the input/output declarations in your rules, then executes it in topological order — respecting all dependencies, running independent jobs in parallel.

Render it to inspect the structure:

<div class="dracula" markdown="1">
```py
snakemake --dag | dot -Tsvg > dag.svg
```


For a three-sample, four-rule pipeline, you will see a clean tree: `rule all` at the top, `featurecounts` aggregating BAMs from all three alignment branches, each branch independently running FastQC, fastp, and HISAT2.

For large workflows with many samples, the full DAG becomes unreadably dense. Use the rule-level view instead:

```py
snakemake --rulegraph | dot -Tsvg > rulegraph.svg
```



The rule graph shows one node per rule — the logical structure without per-sample repetition. This is the view to put in papers and README files.

!!! note "Graphviz must be installed"
    `dot` is part of the Graphviz package. On BMRC, load it with `module load Graphviz/...` before running these commands.

## Dry-run revisited

With wildcards and `expand()` in place, the dry-run output becomes genuinely informative:

```py
snakemake --cores 1 -n -p
```


Each job appears with its resolved wildcard values, inputs, outputs, and the exact shell command that would execute:

```rust
rule fastqc:
    input: data/SRR014335-chr1.fastq
    output: results/fastqc/SRR001_R1_fastqc.html, results/fastqc/SRR001_R1_fastqc.zip
    wildcards: sample=SRR001, read=R1
    shell: fastqc data/SRR014335-chr1.fastq --outdir results/fastqc/
```

Scan this output carefully before running on the cluster. Wildcards that resolve unexpectedly are much cheaper to catch here than after an eight-hour alignment run.

## Useful inspection commands

```py
# List all rules defined in the Snakefile
snakemake --list

# Show all expected files and whether they currently exist
snakemake --summary
```

`--summary` is particularly useful after a partial run: it shows which outputs exist, their timestamps, and whether they are up to date relative to their inputs.

</div>
---
!!! circle-question "Exercise"

    ## Exercise 2

    Create a directory `ep02/` with three subdirectories: `input/a/`, `input/b/`, `input/c/`. In each, create a file `reads.txt` containing a short list of words (use `printf` or `echo -e`).

    Write a Snakefile that:

    1. Uses a `{dataset}` wildcard to count the number of lines in `input/{dataset}/reads.txt`, writing the result to `results/{dataset}_linecount.txt`.
    2. Has a `rule all` using `expand()` to request counts for all three datasets.
    3. Passes `-n -p` dry-run: verify the wildcard values shown are correct.
    4. Runs for real.
    5. **Bonus:** Render the DAG with `--dag | dot -Tsvg > dag.svg` and inspect it.

    ??? success "Solution"
        <div class="snakefile" markdown="1">
        ```python title="Snakefile"
        DATASETS = ["a", "b", "c"]

        rule all:
            input:
                expand("results/{dataset}_linecount.txt", dataset=DATASETS)


        rule count_lines:
            input:
                "input/{dataset}/reads.txt"
            output:
                "results/{dataset}_linecount.txt"
            shell:
                "wc -l {input} > {output}"
        ```
        </div>
---

## Key takeaways

!!! clipboard-list "Episode 2 summary"
    - **Wildcards** (`{sample}`, `{read}`) generalise a single rule across many files. They are resolved from output patterns, never from inputs.
    - **Named inputs/outputs** (`input.r1`, `output.html`) keep multi-file rules readable.
    - **`expand()`** generates concrete filename lists from templates — used in `rule all` to enumerate all desired outputs.
    - The **DAG** is Snakemake's dependency graph, automatically computed from your rules. Inspect it with `--dag | dot -Tsvg`.
    - `--rulegraph` shows the logical rule structure; use this for documentation.
    - `--dry-run -n -p` previews every job with resolved wildcards and shell commands before any execution.
