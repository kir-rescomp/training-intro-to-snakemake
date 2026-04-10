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