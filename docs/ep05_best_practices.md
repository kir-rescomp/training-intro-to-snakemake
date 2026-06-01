# Episode 5: Robustness, Best Practices & Going Further

## Beyond "it ran"

A Snakemake pipeline that reaches completion is not automatically a good pipeline. In a research computing context, you also need to know:

- *How long did each rule actually take, and how much memory did it use?* — for resource tuning and future estimates.
- *Which rule produced a corrupt output, and why?* — for debugging and reproducibility.
- *How do I rerun only the jobs that failed without wasting cluster hours on the ones that succeeded?*
- *Are the tool versions pinned somewhere, or is this pipeline quietly non-reproducible?*

This episode covers the directives and invocation flags that close these gaps.

## `benchmark:` — timing and memory profiling

The `benchmark:` directive writes a tab-separated file after every job, recording wall time, CPU time, and peak memory usage:

<div class="dracula" markdown="1">
```python
rule hisat2:
    input: ...
    output: ...
    log:       "logs/hisat2/{sample}.log"
    benchmark: "benchmarks/hisat2/{sample}.tsv"
    threads: 8
    resources: ...
    shell: ...
```
</div>


The benchmark file has a header row followed by one row of measurements:

```
s       h:m:s   max_rss  max_vms  max_uss  max_pss  io_in   io_out  mean_load  cpu_time
137.42  0:02:17 28654.2  38120.4  27304.1  27680.3  12180.2 4311.6  713.22     1018.4
```

The key column is `max_rss` — peak resident set size in megabytes. This is the number to use when setting `resources.mem_mb`. A good rule of thumb: set `mem_mb` to 1.5× the observed `max_rss`. This provides a safety margin while avoiding gross over-allocation.

!!! tip "Run benchmarks on a representative subset first"
    On the first run of a new pipeline, use 2–3 samples rather than the full cohort. Collect benchmarks, tune resources, then run at scale. This avoids submitting 30 HISAT2 jobs with `mem_mb=64000` when 32000 would suffice.

Add `benchmark:` to every slow rule — alignment, quantification, trimming. Lightweight rules (FastQC, MultiQC) are less critical, but it costs nothing to include them.

## `log:` — a brief but important addendum

You have been using `log:` throughout. A few patterns worth standardising before running production pipelines:

**Capture stderr only (for tools that stream results to stdout):**

<div class="dracula" markdown="1">
```python
shell:
    "hisat2 ... 2> {log} | samtools sort -o {output.bam}"
```


**Capture both stdout and stderr (for tools that write to files):**

```python
shell:
    "fastqc {input} --outdir results/fastqc/ &> {log}"
```

**Common mistake — redirecting the result stream to the log:**

```python
# Wrong: {output.bam} will be empty; the SAM stream goes to the log
shell:
    "hisat2 ... | samtools sort -o {output.bam} &> {log}"
```

</div>
Test redirect behaviour on a small file before scaling up. A zero-byte output with a log full of SAM data is a reliable sign of this mistake.

## Handling failures and reruns

### How Snakemake decides what to rerun

This is the part that has changed the most across Snakemake versions, so it is worth being precise about how Snakemake 9 behaves.

In older Snakemake (≤ 7), rerun decisions were based almost entirely on file modification times. A change to a rule's `params:` or `shell:` command would *not* be detected, and you had to force reruns manually.

Since Snakemake 8 — and unchanged in 9.16+ — rerun decisions are governed by `--rerun-triggers`, which **defaults to all of**:

```
mtime params input software-env code
```

In other words, out of the box Snakemake 9 *does* detect changes to a rule's parameters, its input set, its shell/script code, and its conda/container environment, and reruns the affected jobs accordingly. The old assumption that "Snakemake only looks at timestamps" no longer holds.

!!! note-sticky "Implication for this workshop"
    If you edit a `params:` value, a `shell:` command, or a value in `config.yaml` that feeds into `params:`, a default Snakemake 9 run will already plan to rerun the affected rules. You generally no longer need `-R rule_name` purely to pick up a parameter change.

You can inspect *why* Snakemake wants to rerun something:

<div class="dracula" markdown="1">
```py
# List jobs whose code changed since the last run
snakemake --list-changes code [other flags]

# Also available: params, input
snakemake --list-changes params [other flags]
```

If you want the old timestamp-only behaviour (for example to avoid a large rerun after a trivial edit), restrict the triggers explicitly:

```py
snakemake --rerun-triggers mtime [other flags]
```


### `--rerun-incomplete`

If a job is killed mid-run — out of memory, walltime exceeded, node failure — the output file may be partially written. Snakemake marks such outputs as *incomplete* in its metadata, but it will not act on that unless told to. On the next run, without this flag, it may treat the file as present and the rule as done, which is wrong.

```py
snakemake --rerun-incomplete [other flags]
```

This flag forces Snakemake to rerun any jobs whose outputs are recorded as incomplete. Make it a default in your workflow profile.

### `--keep-going` / `-k`

By default, Snakemake stops submitting new jobs when any job fails. With `--keep-going`, it continues submitting independent branches of the DAG even when one branch has failed:

```py
snakemake --keep-going [other flags]
```

This is particularly useful when failures are likely to be sample-specific — a corrupt FASTQ for one sample should not halt all alignment jobs for the remaining 29. You can investigate and fix the failed sample while the others complete.

### `--forcerun` / `-R`

Even though Snakemake 9 detects most meaningful changes automatically, you sometimes want to force a rerun deliberately — for example after an upstream reference file changed outside Snakemake's view, or to reproduce a result from scratch.

Force a rerun of a specific rule and all downstream rules:

```py
snakemake -R hisat2 [other flags]
```
</div>

!!! tip "When you still need `-R` in Snakemake 9"
    The default `--rerun-triggers` cover params, input, code, and software-env. Cases where you may still want `-R` explicitly: a change to a file Snakemake doesn't track as an input (e.g. a shared reference outside the workflow), forcing a clean reproduction, or when you have deliberately restricted triggers to `mtime`.

## `wildcard_constraints:` — preventing ambiguous matches

Consider a rule with the output pattern `results/{sample}.bam`. If a filename like `results/SRR014335_trimmed.bam` exists, Snakemake might match `sample=SRR014335_trimmed` when you intended only `SRR014335`. Ambiguous wildcard matching can produce subtle, hard-to-diagnose bugs.

Constrain wildcards at the top of the Snakefile using regex:

<div class="snakefile" markdown="1">
```python title="Snakefile"
wildcard_constraints:
    sample="[A-Za-z0-9]+",   # alphanumeric only — no underscores or dots
    read="R[12]"              # exactly R1 or R2
```


Or constrain per-rule:

```python
rule hisat2:
    wildcard_constraints:
        sample="[A-Za-z0-9]+"
    ...
```



For the RNA-seq pipeline in this workshop, adding global constraints for `sample` and `read` prevents any ambiguous matching between the `{sample}_{read}` pattern in `fastqc` and the `{sample}` pattern in `fastp`.

## Software deployment per rule

Snakemake can automatically create and activate a per-rule conda environment, pinning exact tool versions alongside the code:

```python
rule fastp:
    conda: "envs/fastp.yaml"
    ...
```
</div>

<div class="github-dark" markdown="1">
```yaml title="envs/fastp.yaml"
channels:
  - bioconda
  - conda-forge
dependencies:
  - fastp=0.23.4
```
</div>

Activate it with the software-deployment-method flag:

<div class="dracula" markdown="1">
```py
snakemake --software-deployment-method conda [other flags]
# short form:
snakemake --sdm conda [other flags]
```
</div>

!!! warning "`--use-conda` is the old spelling"
    Pre-8 Snakemake used `--use-conda` and `--use-singularity`. In Snakemake 9 these are superseded by `--software-deployment-method` (alias `--sdm`), which takes one or more of `conda`, `apptainer`, and `env-modules`. The old flags may still alias through but should not be taught for 9.16+; use `--sdm`.

Snakemake creates each environment on first use and caches it. The YAML file lives in your repository — a reviewer or collaborator can recreate the exact software environment years later.

!!! note-sticky "Lmod on BMRC vs `pixi`/`apptainer`"
    On BMRC, most tools are deployed via Lmod modules. You can load a module inside the shell command (`module load HISAT2/...`) or use the `envmodules:` directive with `--sdm env-modules`. For pipelines you intend to share beyond BMRC, `pixi` or `apptainer` provides stronger portability.

## The pre-flight checklist

Before any cluster run, work through this list:


- [ ] rule all lists every desired final output
- [ ] Every computational rule has: input, output, log, threads, resources
- [ ] benchmark: added to slow rules (alignment, trimming, quantification)
- [ ] wildcard_constraints: defined for all wildcards used across rules
- [ ] localrules: all (plus any other trivial rules)
- [ ] log directories exist (e.g. `logs/`, and `logs/drmaa/` if using the DRMAA executor)
- [ ] `--dry-run` passes cleanly with no errors or warnings
- [ ] `--rulegraph` renders the expected structure
- [ ] Running inside tmux or screen
- [ ] `--rerun-incomplete` and `--keep-going` included in the run command


## Common pitfalls

| Problem                                   | Symptom                                                    | Fix                                                          |
| ----------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------ |
| Missing output                            | Job exits 0 but Snakemake reports `MissingOutputException` | Verify the shell command actually writes to the exact `{output}` path |
| Ambiguous wildcards                       | `WorkflowError: Ambiguous rule`                            | Add `wildcard_constraints:`                                  |
| Partial output from OOM/timeout           | Rule appears done but output is empty or corrupt           | `--rerun-incomplete`                                         |
| GPFS propagation delay                    | `MissingOutputException` immediately after job             | Increase `--latency-wait` (try 60 s)                         |
| Unexpected large rerun after a small edit | Many jobs rerun because `params`/`code` triggers fired     | Restrict with `--rerun-triggers mtime` if intended           |
| Protected file blocks rerun               | `ProtectedOutputException`                                 | `chmod +w results/bam/*.bam` then rerun                      |
| DRMAA log dir missing                     | Submission fails or logs are lost                          | `mkdir -p logs/drmaa` before running                         |
| `&> {log}` swallows the output stream     | Output file is empty                                       | Switch to `2> {log}` for tools writing to stdout             |

## Where to go next

**Snakemake wrappers** — [snakemake-wrappers.readthedocs.io](https://snakemake-wrappers.readthedocs.io/) provides pre-written, tested rule wrappers for hundreds of bioinformatics tools (FastQC, STAR, DESeq2, samtools, and many more). A wrapper replaces the `shell:` block with a single `wrapper:` directive and a version tag:

```python
rule fastqc:
    input: "data/{sample}_{read}.fastq.gz"
    output:
        html="results/fastqc/{sample}_{read}_fastqc.html",
        zip="results/fastqc/{sample}_{read}_fastqc.zip"
    wrapper:
        "v5.5.0/bio/fastqc"
```

Pin a recent wrapper tag from the wrappers repository (the `v5.x` series at time of writing) rather than an older one. The wrapper pulls the correct conda environment and shell command automatically, which substantially reduces the boilerplate you maintain.

**`script:` directive** — run an R or Python script as the rule action, with the `snakemake` object automatically injected:

```python
rule plot_pca:
    input: "results/counts/all_samples.counts.txt"
    output: "results/pca.pdf"
    script: "scripts/plot_pca.R"
```

Inside `plot_pca.R`, access `snakemake@input[["counts"]]`, `snakemake@output[["pdf"]]`, and so on.

**`checkpoint` rules** — for workflows where the set of outputs is not known until a rule has run (for example, splitting by chromosome after peak calling, or processing a variable number of clusters). Checkpoints allow Snakemake to re-evaluate the DAG mid-run.

**`--report`** — generate a self-contained HTML run report summarising job timing, resource usage, and workflow structure. Useful for sharing with collaborators or including in supplementary materials.

---

!!! circle-question "Exercise"

    ## Final exercise
    
    Take the complete Snakefile from Episode 3 and:
    
    1. Add `benchmark: "benchmarks/{rule}/{sample}.tsv"` to every per-sample rule (you can use the special `{rule}` wildcard, which Snakemake fills with the rule name).
    2. Add global `wildcard_constraints:` for `sample` and `read` at the top of the Snakefile.
    3. Add `--rerun-incomplete` and `--keep-going` to your workflow profile.
    4. Run a final dry-run with `--workflow-profile profiles/drmaa -n -p` and confirm everything looks correct.
    
    ??? success "benchmark with {rule} wildcard"
        <div class="snakefile" markdown="1">
        ```python
        rule fastqc:
            benchmark: "benchmarks/{rule}/{sample}_{read}.tsv"
            ...
    
        rule fastp:
            benchmark: "benchmarks/{rule}/{sample}.tsv"
            ...
    
        rule hisat2:
            benchmark: "benchmarks/{rule}/{sample}.tsv"
            ...
        ```
        Note that `{rule}` is a special keyword — it is not a user-defined wildcard and does not need to appear in `output:`.
        </div>

---

## Key takeaways

!!! clipboard-list "Episode 5 summary"
    - **`benchmark:`** records timing and peak memory per job — use `max_rss` to tune `resources.mem_mb`.
    - **`--rerun-triggers`** defaults to `mtime params input software-env code` in Snakemake 9 — params and code changes are detected automatically; restrict to `mtime` if you want timestamp-only behaviour.
    - **`--rerun-incomplete`** reruns jobs whose outputs were recorded as incomplete after a killed job.
    - **`--keep-going`** allows independent branches to continue when one branch fails.
    - **`-R rule_name`** forces reruns for changes Snakemake cannot see (e.g. untracked reference files) or to reproduce from scratch.
    - **`wildcard_constraints:`** prevents ambiguous wildcard matching — define them globally for all wildcards used across multiple rules.
    - **`conda:`** + `--sdm conda` pins per-rule software versions in a version-controlled YAML — the reproducibility baseline.
    - **Snakemake wrappers** eliminate most boilerplate for standard bioinformatics tools.
    - The pre-flight checklist is not pedantry — it is how you avoid debugging a 30-sample run at midnight.