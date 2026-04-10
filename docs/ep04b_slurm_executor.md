# Episode 4b:Running on the Cluster — Slurm Executor



|                             | DRMAA                                 | Slurm executor                         |
| ----------------------------| ------------------------------------- | -------------------------------------- |
| **Mechanism**               | C API (DRMAA library)                 | Direct `sbatch` calls                  |
| **Installation dependency** | `python-drmaa` + cluster DRMAA lib    | `snakemake-executor-plugin-Slurm` only |
| **Resource mapping**        | Manual template string                | Automatic from `resources:`            |
| **Portability**             | Requires DRMAA support on the cluster | Works on any Slurm cluster             |


!!! circle-info "When to chose DRMAA vs Slurm executor"

    If your cluster supports DRMAA and you want fine-grained control over the submission string, use DRMAA. If you 
    want something simpler that works out of the box on any Slurm system, use the Slurm executor.


## Resources in the Snakefile

The Slurm executor reads directly from the `resources:` block. The key difference from the DRMAA setup is the partition key — it must be `slurm_partition` rather than `partition`:

<div class="snakefile" markdown="1">

```py
rule fastp:
    threads: config["threads"]["fastp"]
    resources:
        mem_mb=8000,
        runtime=60,
        slurm_partition="short"

rule hisat2:
    threads: config["threads"]["hisat2"]
    resources:
        mem_mb=32000,
        runtime=120,
        slurm_partition="short"

rule featurecounts:
    threads: config["threads"]["featurecounts"]
    resources:
        mem_mb=8000,
        runtime=60,
        slurm_partition="short"
```
</div>

`threads` is mapped to `--cpus-per-task` automatically — you don't need to wire it up manually as you do in the DRMAA args string.

### Ths submission command 

<div class="dracula" markdown="1">

```py
snakemake \
  --executor slurm \
  --jobs 6 \
  --default-resources mem_mb=1000 runtime=5 slurm_partition=short \
  --latency-wait 30 \
  -p
```

That's it. No `--drmaa-args` template to maintain.
