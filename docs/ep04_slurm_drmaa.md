# Episode 4: Scaling to the Cluster — SLURM via DRMAA

## From laptop to HPC

Your Snakefile now describes a complete pipeline. Running it locally is useful for testing on a small dataset, but for real data — 30 samples, 8-thread HISAT2 alignments, 30 GB of FASTQs — you need the cluster.

You could wrap each rule in an `sbatch` call. But that reintroduces every problem Snakemake was solving: manual dependency tracking, no reruns logic, no parallel scheduling, no awareness of which jobs failed. Instead, Snakemake can **submit each job to SLURM itself**, using the resource declarations in your rules to construct each submission, respecting dependencies across the graph, and managing the full execution from a single process on the login node.

## The `threads:` directive — revisited

`threads:` does double duty. Locally, it limits how many CPU cores a rule may claim from Snakemake's shared pool. On the cluster, it maps directly to `--cpus-per-task` in the SLURM submission.

<div class="dracula" markdown="1">
```python
rule hisat2:
    threads: config["threads"]["hisat2"]   # 8
    shell:
        "hisat2 ... --threads {threads} ..."
```
</div>

Always declare `threads:` on any rule that benefits from parallelism, and pass `{threads}` to the tool — that way the threads value is consistent between what SLURM allocates and what the tool uses.

## The `resources:` directive

`resources:` declares the per-job requirements that the executor plugin reads when constructing the SLURM submission:

```python
rule hisat2:
    threads: 8
    resources:
        mem_mb=32000,    # 32 GB — passed as --mem
        runtime=120,     # wall time in minutes — passed as --time
        partition="short"
```

These values are not used in the shell command. They exist solely for the scheduler.

| Resource key | Unit | Maps to SLURM flag |
|---|---|---|
| `mem_mb` | Megabytes | `--mem` |
| `runtime` | Minutes | `--time` |
| `partition` | String | `-p` |

!!! tip "Be honest about resources"
    Under-requesting memory gets your job killed by the OOM killer with no useful error message. Over-requesting wastes queue slots and reduces throughput for everyone. Start with generous estimates, then use the `benchmark:` directive (Episode 5) to collect real peak-memory figures and tune accordingly.

## Adding `resources:` to every rule

Update the Snakefile from Episode 3:

<div class="snakefile" markdown="1">
```python title="Snakefile (additions)"
rule fastqc:
    threads: config["threads"]["fastqc"]
    resources:
        mem_mb=4000,
        runtime=30,
        partition="short"

rule fastp:
    threads: config["threads"]["fastp"]
    resources:
        mem_mb=8000,
        runtime=60,
        partition="short"

rule hisat2:
    threads: config["threads"]["hisat2"]
    resources:
        mem_mb=32000,
        runtime=120,
        partition="short"

rule featurecounts:
    threads: config["threads"]["featurecounts"]
    resources:
        mem_mb=8000,
        runtime=60,
        partition="short"
```
</div>

## The DRMAA executor

DRMAA (Distributed Resource Management Application API) is a standardised C API for submitting, monitoring, and cancelling jobs on cluster schedulers. Snakemake's DRMAA executor plugin uses this API to submit each job as a proper SLURM job — not via a shell call to `sbatch`, but through the DRMAA library that SLURM exposes.

The plugin is `snakemake-executor-plugin-drmaa`. Once installed (see the installation guide), activate it with:

<div class="dracula" markdown="1">
```py
snakemake --executor drmaa [options]
```

## The submission command — explained

```py
snakemake \
  --executor drmaa \
  --drmaa-args " -p {resources.partition} --mem={resources.mem_mb} --cpus-per-task={threads} --time=00:{resources.runtime}:00 --output=logs/drmaa/job_%j.out --error=logs/drmaa/job_%j.err" \
  --drmaa-log-dir logs/drmaa \
  --jobs 14 \
  --default-resources mem_mb=1000 runtime=5 partition=short \
  --latency-wait 30 \
  -p
```
</div>

| Flag | Purpose |
|---|---|
| `--executor drmaa` | Activate the DRMAA executor plugin |
| `--drmaa-args "..."` | Native SLURM options passed per job; placeholders filled at runtime |
| `--drmaa-log-dir logs/drmaa` | Directory for DRMAA-level per-job stdout/stderr |
| `--jobs 14` | Maximum number of jobs running simultaneously on the cluster |
| `--default-resources ...` | Fallback resource values for any rule that does not declare them |
| `--latency-wait 30` | Wait up to 30 s for output files to appear after a job exits |
| `-p` | Print the resolved shell command for each submitted job |

### Inside `--drmaa-args`

The value is a template string. For every job Snakemake submits, it substitutes the rule's actual values before passing the string to the DRMAA library:

| Placeholder | Resolved to |
|---|---|
| `{resources.partition}` | e.g. `short` |
| `{resources.mem_mb}` | e.g. `32000` |
| `{threads}` | e.g. `8` |
| `{resources.runtime}` | e.g. `120`, formatted as `00:120:00` |

The result is a valid SLURM native specification — equivalent to the arguments you would pass to `sbatch`.

!!! warning "The leading space in `--drmaa-args` is intentional"
    The string starts with a space: `" -p ..."`. The DRMAA library internally prepends a job script name before this string. Without the leading space, the first SLURM flag runs directly into that name, producing a malformed submission. Do not remove it.

### `--latency-wait` and GPFS

The BMRC cluster uses GPFS, a distributed parallel file system. After a compute node writes a file, there can be a short propagation delay before the file is visible from the login node where Snakemake is running. Without `--latency-wait`, Snakemake may declare a job failed immediately after it exits, because the output file has not yet appeared on the login node. Setting `--latency-wait 30` tells Snakemake to poll for up to 30 seconds before giving up. This is almost always sufficient for GPFS.

### `--default-resources`

Any rule without explicit `resources:` will fall back to these values. This prevents a submission error if you add a new rule and forget to declare resources:

```bash
--default-resources mem_mb=1000 runtime=5 partition=short
```

Note that the line was commented out in the example — uncomment it for production use. Rules with their own `resources:` block are not affected by `--default-resources`.

## `localrules` — rules that should not be submitted

Some rules are so fast and lightweight that submitting them to SLURM is wasteful — the queuing overhead would dwarf the actual computation. Declare them as local at the top of the Snakefile:

```python title="Snakefile"
localrules: all
```

`rule all` has no shell command and should always be local. If you have a rule that only creates a directory, writes a small config file, or copies a tiny file, make it local too.

!!! note-sticky "Local rules run on the login node"
    Local rules execute directly in the Snakemake process on the login node, with no SLURM submission. Keep them genuinely lightweight — no alignment, no sorting, no compression.

## Log directory setup

Create the DRMAA log directory before running:

<div class="dracula" markdown="1">
```py
mkdir -p logs/drmaa
```

SLURM writes per-job stdout and stderr to `logs/drmaa/job_%j.out` and `logs/drmaa/job_%j.err` (where `%j` is the SLURM job ID). These are distinct from your rule-level `log:` files, which capture the tool's own output. When debugging a failure, check both:

- **Rule log** (`logs/hisat2/SRR014335.log`) — what the tool printed
- **DRMAA job log** (`logs/drmaa/job_12345678.err`) — what SLURM saw (out of memory, time limit, etc.)

## Running the pipeline

```py
# Step 1 — dry run to verify the plan and resource assignments
snakemake \
  --executor drmaa \
  --drmaa-args " -p {resources.partition} --mem={resources.mem_mb} --cpus-per-task={threads} --time=00:{resources.runtime}:00 --output=logs/drmaa/job_%j.out --error=logs/drmaa/job_%j.err" \
  --drmaa-log-dir logs/drmaa \
  --jobs 14 \
  --default-resources mem_mb=1000 runtime=5 partition=short \
  --latency-wait 30 \
  -n -p
```

Scan the dry-run output. Confirm that each job shows the memory, threads, runtime, and partition you expect. Then:

```py
# Step 2 — run for real from inside a tmux session
tmux new -s snakemake

snakemake \
  --executor drmaa \
  --drmaa-args " -p {resources.partition} --mem={resources.mem_mb} --cpus-per-task={threads} --time=00:{resources.runtime}:00 --output=logs/drmaa/job_%j.out --error=logs/drmaa/job_%j.err" \
  --drmaa-log-dir logs/drmaa \
  --jobs 14 \
  --default-resources mem_mb=1000 runtime=5 partition=short \
  --latency-wait 30 \
  -p
```
</div>

!!! warning "Always run inside tmux or screen"
    The Snakemake process on the login node is the conductor — it submits jobs, monitors them, and triggers downstream steps when dependencies complete. If this process is killed by an SSH disconnect, submitted jobs may finish but Snakemake will not know, leaving the workflow in an inconsistent state. Always use **tmux** or **screen**.

## Workflow profiles — the clean way

Typing the full `--drmaa-args` string on every invocation is error-prone and not version-controllable in a meaningful way. A **workflow profile** stores all execution options in a YAML file alongside your Snakefile:

<div class="dracula" markdown="1">
```yaml title="profiles/drmaa/config.yaml"
executor: drmaa
drmaa-args: " -p {resources.partition} --mem={resources.mem_mb} --cpus-per-task={threads} --time=00:{resources.runtime}:00 --output=logs/drmaa/job_%j.out --error=logs/drmaa/job_%j.err"
drmaa-log-dir: logs/drmaa
jobs: 14
default-resources:
  - mem_mb=1000
  - runtime=5
  - partition=short
latency-wait: 30
printshellcmds: true
```


Run with:

```py
snakemake --workflow-profile profiles/drmaa
```
</div>

Commit `profiles/drmaa/config.yaml` to your repository. Collaborators running the same pipeline on BMRC use the same profile with no flags to remember.

---

!!! circle-question "Exercise"

    ## Exercise 4

    1. Add `resources:` to every rule in your Episode 3 Snakefile (use the values from this episode as a starting point).
    2. Add `localrules: all` at the top.
    3. Create `logs/drmaa/`.
    4. Run a full dry-run with the DRMAA command. Verify that each job line shows the correct memory, threads, runtime, and partition values.
    5. **Bonus:** Create `profiles/drmaa/config.yaml` and confirm that `snakemake --workflow-profile profiles/drmaa -n -p` produces the same output as the long-form command.

---

## Key takeaways

!!! success "Episode 4 summary"
    - **`threads:`** maps to `--cpus-per-task`; **`resources:`** (mem_mb, runtime, partition) maps to the corresponding SLURM flags.
    - The `--drmaa-args` string is a per-job template filled with `{resources.X}` and `{threads}` at submission time.
    - **`--jobs`** caps the number of concurrently running cluster jobs.
    - **`--latency-wait`** compensates for GPFS file propagation delay — 30 seconds is appropriate on BMRC.
    - **`--default-resources`** provides fallback values for rules without explicit resource declarations.
    - **`localrules:`** keeps trivial rules on the login node, avoiding unnecessary queue submissions.
    - Always run the Snakemake conductor process inside **tmux** or **screen**.
    - **Workflow profiles** (`--workflow-profile`) store all cluster options in a version-controlled file — use them.
