# Getting Started on the SAIAB HPC Server

**SOP ID:** HPC-GS-001
**Version:** 1.0
**Date:** 2026-02-23
**Author:** AGRP

---

## Overview

This guide will help you log in to the SAIAB HPC server (`lab417.saiab.ac.za`) for the first time and run your first SLURM jobs. No prior HPC experience is needed.

!!! tip "Recommended Training"
    Before diving in, we strongly recommend working through the full course:

    **[Introduction to the Unix Shell and High Performance Computing](https://lab417.saiab.ac.za/unix_course/Shell-for-bioinformatics/index.html)**

    It covers everything from basic shell commands to SLURM job scheduling on this very server — and will make everything below much easier.

---

## 1. Logging In

You connect to the server using **SSH** from a terminal.

### Mac / Linux

Open the **Terminal** app and run:

```bash
ssh your_username@lab417.saiab.ac.za
```

### Windows

Use [MobaXterm](https://mobaxterm.mobatek.net/) or [PuTTY](https://www.putty.org/). Set the hostname to `lab417.saiab.ac.za` and your username.

### First Login — Change Your Password

On your very first login you will be prompted to change your temporary password:

```
Changing password for your_username.
Current password: (enter the temporary password sent to you)
New password:     (choose a strong password)
Retype new password:
```

Choose a password that is at least 8 characters and includes uppercase, lowercase, numbers, and a special character.

---

## 2. Understanding the Server Layout

The server has two types of nodes:

| Node | Purpose |
|------|---------|
| **Login node** (`lab417`) | Where you land after SSH. Use for file management, editing scripts, and submitting jobs. **Do not run heavy computations here.** |
| **Compute nodes** | Where your actual analyses run, via SLURM. |

---

## 3. Useful Shortcuts

Your account comes pre-configured with the following shortcuts:

| Alias | What it does |
|-------|-------------|
| `sq` | Show the current SLURM job queue |
| `si` | Show available compute node resources |
| `slogin` | Start an interactive session on a compute node |
| `ll` | List files in detail (`ls -la`) |

---

## 4. Running Jobs with SLURM

SLURM is the job scheduler — it manages who gets to use the compute nodes and when.

### Interactive Session

Use `slogin` when you want to run commands interactively on a compute node (e.g., for testing):

```bash
slogin
```

Your prompt will change to show the SLURM job ID, confirming you are now on a compute node:

```
[SLURM:12345] your_username@lab417:~$
```

Type `exit` to leave the interactive session and return to the login node.

### Batch Job

For longer analyses, write a SLURM script and submit it so it runs in the background:

**Example script (`my_job.sh`):**

```bash
#!/bin/bash
#SBATCH --job-name=my_analysis
#SBATCH --partition=agrp
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G
#SBATCH --time=02:00:00
#SBATCH --output=logs/my_job_%j.out

echo "Job started on: $(date)"

# Your commands go here
# e.g. python my_script.py

echo "Job finished on: $(date)"
```

**Submit the job:**

```bash
sbatch my_job.sh
```

**Check its status:**

```bash
sq
```

**Cancel a job** (if needed):

```bash
scancel <JOBID>
```

---

## 5. Transferring Files

### Copy a file from your computer to the server

```bash
scp my_file.txt your_username@lab417.saiab.ac.za:~/
```

### Copy a file from the server to your computer

```bash
scp your_username@lab417.saiab.ac.za:~/results/output.txt ./
```

---

## 6. Next Steps

Once you are comfortable with the basics:

- **[Conda Environments](conda-environments.md)** — Set up and manage software environments
- **[SLURM Basics (advanced)](slurm-basics.md)** — More detailed SLURM options and best practices
- **[Storage Management](storage_management_sop.md)** — Understand disk quotas and where to store data

!!! question "Need Help?"
    Contact the HPC administrator at **evilliers@saiab.ac.za**
