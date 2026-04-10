# Episode 4b:Running on the Cluster — SLURM Executor



|                             | DRMAA                                 | Slurm executor                         |
| ----------------------------| ------------------------------------- | -------------------------------------- |
| **Mechanism**               | C API (DRMAA library)                 | Direct `sbatch` calls                  |
| **Installation dependency** | `python-drmaa` + cluster DRMAA lib    | `snakemake-executor-plugin-slurm` only |
| **Resource mapping**        | Manual template string                | Automatic from `resources:`            |
| **Portability**             | Requires DRMAA support on the cluster | Works on any Slurm cluster             |