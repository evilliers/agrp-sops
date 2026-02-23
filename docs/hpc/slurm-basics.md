# SLURM Basics on the HPC

**SOP ID:** HPC-001
**Version:** 2.0
**Date:** 2026-02-23
**Author:** AGRP

---

## Overview

SLURM is the job scheduling system on `lab417.saiab.ac.za`. It manages access to compute nodes, queues jobs, and ensures resources are shared fairly among users. All heavy computations must be submitted through SLURM — never run them directly on the login node.

!!! tip "Full Course Available"
    This SOP covers the essentials for day-to-day use. For a complete walkthrough with exercises, see:

    **[Job Scheduling with SLURM — Shell for Bioinformatics Course](https://lab417.saiab.ac.za/unix_course/Shell-for-bioinformatics/lessons/09_Job_scheduling_with_SLURM.html)**

---

## 1. Key Commands

| Command | What it does |
|---------|-------------|
| `sbatch script.sh` | Submit a batch job script |
| `srun` | Run a command interactively on a compute node |
| `squeue -u $USER` | Check the status of your jobs |
| `sq` | Shortcut alias for the queue (pre-configured on your account) |
| `si` | Shortcut alias for node information |
| `scancel <jobid>` | Cancel a job |
| `sinfo` | Show available partitions and node states |

Job states in the queue: **R** = Running, **PD** = Pending (waiting)

---

## 2. Interactive Jobs

Use an interactive job when you want to run commands directly on a compute node — useful for testing or short exploratory work.

The `slogin` alias is pre-configured on your account:

```bash
slogin
```

This is equivalent to:

```bash
srun -p agrp --cpus-per-task=1 --nodes=1 --mem=4G --pty bash -i
```

Your prompt will change to show the SLURM job ID when you are on a compute node:

```
[SLURM:12345] your_username@lab417:~$
```

Type `exit` to return to the login node.

---

## 3. Batch Jobs

For longer analyses, write a SLURM script and submit it with `sbatch`. The job will run in the background — you can log out and it will continue.

### Required Attributes

Every job submitted on lab417 must include these directives:

| Attribute | Flag | Example |
|-----------|------|---------|
| Partition | `-p` | `#SBATCH -p agrp` |
| Time limit | `-t` | `#SBATCH -t 02:00:00` |
| Number of nodes | `-N` | `#SBATCH -N 1` |
| Number of tasks | `-n` | `#SBATCH -n 1` |

### Optional but Recommended

| Attribute | Flag | Example |
|-----------|------|---------|
| Job name | `--job-name` | `#SBATCH --job-name=my_job` |
| CPU cores | `--cpus-per-task` | `#SBATCH --cpus-per-task=8` |
| Memory | `--mem` | `#SBATCH --mem=16G` |
| Stdout log | `-o` | `#SBATCH -o logs/%j.out` |
| Stderr log | `-e` | `#SBATCH -e logs/%j.err` |
| Email alerts | `--mail-user` / `--mail-type` | `#SBATCH --mail-type=ALL` |

The `%j` variable is automatically replaced by the job ID in filenames.

---

## 4. Example Scripts

### Minimal Script

```bash
#!/bin/bash
#SBATCH -p agrp
#SBATCH -t 00:30:00
#SBATCH -N 1
#SBATCH -n 1

echo "Job started: $(date)"
# your commands here
echo "Job finished: $(date)"
```

### Full Example with Conda (e.g., BUSCO)

```bash
#!/bin/bash
#SBATCH --job-name=busco
#SBATCH --partition=agrp
#SBATCH --time=14-00:00:00
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --cpus-per-task=10
#SBATCH --mem=100G
#SBATCH --output=logs/busco.%j.out
#SBATCH --error=logs/busco.%j.err
#SBATCH --mail-user=your_email@saiab.ac.za
#SBATCH --mail-type=ALL

source ~/miniconda3/etc/profile.d/conda.sh
conda activate busco

INPUT_FASTA="/path/to/assembly.fasta"
OUTPUT_DIR="/path/to/output"
LINEAGE="actinopterygii_odb10"

busco -i ${INPUT_FASTA} \
      -m genome \
      -l ${LINEAGE} \
      --download_path ~/busco_downloads \
      -c ${SLURM_CPUS_PER_TASK} \
      -o ${OUTPUT_DIR}
```

!!! note
    Use `${SLURM_CPUS_PER_TASK}` in your script instead of a hard-coded number — it will always match what you requested.

---

## 5. Submitting and Monitoring Jobs

**Submit:**
```bash
sbatch my_script.sh
```

**Check your jobs:**
```bash
sq
# or
squeue -u $USER
```

**Check when a pending job is estimated to start:**
```bash
squeue --start -j <jobid>
```

**Cancel a job:**
```bash
scancel <jobid>
```

---

## 6. Job Dependencies

You can hold a job until a previous one finishes successfully:

```bash
sbatch -d afterok:<jobid> next_script.sh
```

If the first job fails, the dependent job enters `DependencyNeverSatisfied` state. To release it manually:

```bash
scontrol release <jobid>
```

---

## 7. Troubleshooting

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| Job stuck in queue (PD) | Fairshare priority or resource limits | Check `sq`; wait or reduce resource request |
| No output file produced | Job failed early | Check the `.err` log file for errors |
| `command not found` in job | Conda env not activated | Add `source ~/miniconda3/etc/profile.d/conda.sh` and `conda activate myenv` to your script |
| Job runs but gives wrong results | Script path issues | Use full absolute paths for input/output files |
| Job runs out of time | Time limit too short | Resubmit with a longer `-t` value |

!!! tip "Debugging tip"
    Test your script interactively first using `slogin`, then submit as a batch job once it works.

---

## Further Reading

- **[Full SLURM lesson with exercises](https://lab417.saiab.ac.za/unix_course/Shell-for-bioinformatics/lessons/09_Job_scheduling_with_SLURM.html)**
- **[SLURM Cheat Sheet (PDF)](https://lab417.saiab.ac.za/unix_course/Shell-for-bioinformatics/lessons/Slurm_Cheat_Sheet.pdf)**
- [Official SLURM documentation](https://slurm.schedmd.com/documentation.html)
