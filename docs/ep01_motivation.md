# Episode 1: From Shell Scripts to Snakemake

## The shell-script trap

Picture a familiar scene. You have three paired-end RNA-seq samples — `SRR001`, `SRR002`, `SRR003` — and a clean bash script that runs FastQC, trims adapters, aligns to the genome, and counts reads. It works. You're happy.

Then reality arrives.

- Your PI sends two more samples. You edit the loop, rerun everything — including the three samples that were already done.
- You realise you used the wrong HISAT2 index. Which steps need to rerun? You're not sure, so you delete everything and start again.
- A compute node fails mid-alignment. The BAM file is half-written. Your script doesn't notice, and continues downstream with a corrupt file.
- Three months later, a reviewer asks exactly how the counts table was produced. You have a script, but it doesn't capture which version of trimmer was used, which genome annotation, or whether you re-ran anything manually.

None of these problems are unique to you. They are the *universal shell-script trap*: a loop that runs top-to-bottom is simple to write, but brittle, opaque, and entirely unaware of its own history.

## What is a workflow manager?

A workflow manager lets you describe your analysis as a **graph of dependencies** rather than a linear sequence of steps. You declare:

> *"This sorted BAM comes from running HISAT2 on these trimmed FASTQs."*

The workflow manager then works out what needs to run, in what order, and whether it has already run. If a FASTQ is newer than its BAM, the alignment is rerun. If the BAM already exists and is up to date, it is skipped. If you have eight samples, independent jobs run in parallel automatically.

## Why Snakemake?

Several workflow managers exist — Nextflow, WDL, CWL, Luigi. Snakemake has a particular fit for academic bioinformatics and HPC environments:

- **It is Python.** Rules embed Python syntax directly. There is no new language to learn beyond what you already use.
- **It is file-centric.** Dependencies are expressed through input and output files, which maps naturally onto how bioinformatics tools actually work.
- **It integrates natively with HPC schedulers.** Via executor plugins, the same Snakefile that runs on your laptop submits jobs to SLURM with a single extra flag.
- **It is the community standard.** The majority of published bioinformatics pipelines in the past five years are written in Snakemake or Nextflow.

This workshop covers **Snakemake 9**, which uses a clean executor plugin system for cluster submission. All syntax shown here is compatible with v9.

## Anatomy of a rule

The basic unit of a Snakemake workflow is a **rule**. Here is the simplest possible one:

<div class="snakefile" markdown="1">

```python title="Snakefile"
rule say_hello:
    output:
        "hello.txt"
    shell:
        "echo 'Hello, Snakemake' > {output}"
```
</div>

| Component | Meaning |
|-----------|---------|
| `rule say_hello:` | Names the rule |
| `output:` | Files this rule is responsible for producing |
| `shell:` | The shell command to execute |
| `{output}` | Placeholder — Snakemake substitutes the output filename at runtime |

Run it:

<div class="dracula" markdown="1">
```py
snakemake --cores 1 hello.txt
```
</div>

Snakemake sees that `hello.txt` doesn't exist, finds the rule that produces it, and runs the shell command. Run it again — it prints "Nothing to be done" because `hello.txt` already exists and is up to date.

!!! circle-info "Snakemake works backwards"
    You tell Snakemake *what you want*, not *what to run*. It finds the rule that produces your target, then traces backwards through its inputs to build a full execution plan. This reversal — from imperative to declarative — is the core shift in thinking.

## Adding an input

Rules become useful when they declare both inputs and outputs, establishing a dependency:

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule count_lines:
    input:
        "hello.txt"
    output:
        "line_count.txt"
    shell:
        "wc -l {input} > {output}"
```
</div>

Now Snakemake knows: to produce `line_count.txt`, it first needs `hello.txt`. Requesting `line_count.txt` therefore triggers `say_hello` automatically. This two-rule chain is already a DAG — a Directed Acyclic Graph of dependencies.

## `rule all` — the entry point

When you run `snakemake --cores 1` with no explicit target, Snakemake executes the **first rule in the file**. By convention, this is always called `rule all`, and it lists all the final outputs you want from the entire workflow:

<div class="snakefile" markdown="1">
```python title="Snakefile"
rule all:
    input:
        "line_count.txt"
```
</div>

`rule all` has no `output:` or `shell:` — its only job is to name the targets. Snakemake traces backwards from these to build the full execution plan.

!!! tip "Always put `rule all` first"
    `rule all` should be the first rule in your Snakefile. Snakemake runs the first rule by default, and `rule all` acts as the single entry point that describes the complete desired state of your analysis. Reviewers and collaborators will look here first to understand what the pipeline produces.

## Dry runs — your first habit to form

Before running anything for real, ask Snakemake to show you what *would* happen:

```bash
snakemake --cores 1 --dry-run
# equivalently:
snakemake --cores 1 -n
```

This prints every job that would be submitted, with its inputs and outputs, but executes nothing. Add `-p` to also print the resolved shell command for each job:

```bash
snakemake --cores 1 -n -p
```

On a cluster where jobs cost real CPU hours, running a dry-run first is non-negotiable.

---

## Exercise 1

Create a directory called `ep01/` and write a `Snakefile` inside it that:

1. Creates a file `numbers.txt` containing the integers 1 to 10, one per line (`seq 1 10`).
2. Creates a file `sorted_numbers.txt` by sorting `numbers.txt` in reverse numerical order (`sort -rn`).
3. Has a `rule all` that requests `sorted_numbers.txt` as its final output.

Test it:

```bash
cd ep01/
snakemake --cores 1 -n -p   # dry run first
snakemake --cores 1         # then run for real
```

Delete `sorted_numbers.txt` and re-run. Does Snakemake regenerate it? Delete `numbers.txt` too — what does Snakemake do now?

??? success "Solution"

    <div class="snakefile" markdown="1">
    ```python title="Snakefile"
    rule all:
        input:
            "sorted_numbers.txt"


    rule generate_numbers:
        output:
            "numbers.txt"
        shell:
            "seq 1 10 > {output}"


    rule sort_numbers:
        input:
            "numbers.txt"
        output:
            "sorted_numbers.txt"
        shell:
            "sort -rn {input} > {output}"
    ```
    </div>
---

## Key takeaways

!!! success "Episode 1 summary"
    - Snakemake describes *outputs and their dependencies*, not a sequence of steps to execute.
    - A **rule** connects `input:` files, `output:` files, and an action (`shell:`, `run:`, or `script:`).
    - `{input}` and `{output}` are placeholders that Snakemake substitutes at runtime.
    - Snakemake **works backwards** from a target to plan the minimum set of jobs needed.
    - `rule all` is the conventional entry point — always the first rule in the file.
    - Always use `--dry-run -n` to preview execution before committing.
