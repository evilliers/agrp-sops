# Managing Conda Environments on the HPC

**SOP ID:** HPC-002
**Version:** 1.0
**Date:** 2026-02-23
**Author:** AGRP

---

## Overview

Conda is a package and environment manager that lets you install bioinformatics software without needing administrator access. Each **environment** is an isolated space with its own set of tools and versions — so different projects never conflict with each other.

!!! tip "Full Course Available"
    This SOP covers the essentials. For a comprehensive walkthrough including advanced workflows and best practices, see the full course:

    **[Conda for Bioinformatics](https://lab417.saiab.ac.za/conda_course/conda_website.html)**

---

## 1. Installation

Each user installs conda in their own home directory. Download and run the Miniconda installer:

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

Accept the defaults and allow the installer to initialise conda in your `.bashrc`. Then reload your shell:

```bash
source ~/.bashrc
```

Confirm it works:

```bash
conda --version
```

---

## 2. First-Time Setup

Add the bioinformatics channels so that tools like GATK, BWA, and QIIME2 are available:

```bash
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
```

---

## 3. Creating an Environment

Always create a dedicated environment for each project or tool rather than installing everything into `base`.

```bash
conda create -n my_project python=3.11
```

Activate it:

```bash
conda activate my_project
```

Your prompt will update to show the active environment:

```
(my_project) your_username@lab417:~$
```

---

## 4. Installing Software

With your environment active, install tools using `conda install`:

```bash
conda install -c bioconda samtools bwa
```

Search for a package:

```bash
conda search -c bioconda fastqc
```

---

## 5. Essential Commands

| Command | What it does |
|---------|-------------|
| `conda activate my_env` | Switch into an environment |
| `conda deactivate` | Return to base |
| `conda env list` | List all your environments |
| `conda list` | List packages in the active environment |
| `conda install package` | Install a package |
| `conda remove package` | Remove a package |
| `conda env remove -n my_env` | Delete an environment entirely |

---

## 6. Sharing and Reproducing Environments

Export your environment so collaborators (or your future self) can recreate it exactly:

```bash
conda env export > environment.yml
```

Recreate it from the file:

```bash
conda env create -f environment.yml
```

---

## 7. Using Conda in SLURM Jobs

Activate your environment inside your SLURM batch script:

```bash
#!/bin/bash
#SBATCH --job-name=my_analysis
#SBATCH --partition=agrp
#SBATCH --cpus-per-task=4
#SBATCH --mem=8G

source ~/miniconda3/etc/profile.d/conda.sh
conda activate my_project

# Your commands here
python my_script.py
```

!!! warning
    Do not rely on your `.bashrc` to activate conda inside batch jobs — always source it explicitly as shown above.

---

## Further Reading

- **[Full Conda for Bioinformatics Course](https://lab417.saiab.ac.za/conda_course/conda_website.html)** — covers advanced workflows, Snakemake/Nextflow integration, and best practices
- [Official Conda documentation](https://docs.conda.io)
- [Bioconda package list](https://bioconda.github.io/conda-package_index.html)
