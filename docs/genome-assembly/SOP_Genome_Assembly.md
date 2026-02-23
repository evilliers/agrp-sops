# Standard Operating Procedure: De Novo Genome Assembly
## 1KSA Genome Assembly Pipeline — SLURM / Conda

**Version:** 1.0
**Date:** 2026-02-19
**Queue:** `agrp`
**Conda environment:** `1ksa_assembly`
**Pipeline tools:** KMC · NanoPlot · Chopper · Kraken2 · KrakenTools · Flye · Hifiasm · Racon · minimap2 · BUSCO · QUAST · samtools

---

## Overview

This SOP describes how to assemble a draft genome from Oxford Nanopore long-read sequencing data using the 1KSA Nextflow pipeline, adapted for a SLURM cluster using a conda environment.

The pipeline has six stages:

```
[0] K-mer analysis        → estimate genome size and coverage
[1] QC + trimming         → NanoPlot + Chopper
[2] Decontamination       → Kraken2 + extract_kraken_reads.py
[3] Assembly              → Flye (< 3 Gb) or Hifiasm (≥ 3 Gb)
[4] Mapping               → minimap2
[5] Polishing             → Racon (Flye only)
[6] Evaluation            → BUSCO + QUAST
[7] Report generation     → generate_report.sh
```

Stages 1, 2, 4, 5, and 6 are managed by Nextflow and submitted automatically to SLURM. Stages 0, 3, and 7 are submitted as standalone SLURM jobs or run directly.

---

## Prerequisites

- Access to the SLURM cluster with queue `agrp`
- Conda environment `1ksa_assembly` created from `environment.yml` (see step 0.3)
- BUSCO lineage database downloaded (see step 1.3)
- Raw FASTQ file (concatenated, basecalling already done)

---

## Step 0: Prepare Your Environment

### 0.1 Log in to the server

```bash
ssh username@your-server.ac.za
```

### 0.2 Clone the pipeline repository

```bash
cd /path/to/your/working/directory
git clone https://github.com/evilliers/1ksa-genome-assembly-pipeline.git
cd 1ksa-genome-assembly-pipeline
```

Or copy your adapted pipeline files into the working directory.

### 0.3 Create the conda environment

The repository includes an `environment.yml` file that installs all required tools into a conda environment named `1ksa_assembly`.

```bash
conda env create -f environment.yml
```

This installs: KMC · NanoPlot · Chopper · Flye · Hifiasm · Racon · minimap2 · BUSCO · QUAST · samtools · Nextflow · Kraken2 · KrakenTools

Verify the environment was created:

```bash
conda activate 1ksa_assembly
nextflow -version
busco --version
```

!!! note
    This only needs to be done **once per server**. If the environment already exists, activate it with `conda activate 1ksa_assembly`.

### 0.4 Download the BUSCO lineage database (once per server)

Choose your lineage based on your study organism (see table below), then download it:

```bash
conda activate 1ksa_assembly

# Broad eukaryote lineage (use for any eukaryote, or as a first-pass check)
busco --download eukaryota_odb10

# Ray-finned fish lineage (use for Actinopterygii — more informative for fish)
busco --download actinopterygii_odb10

# Verify downloads:
ls ./busco_downloads/lineages/
```

**Choosing a BUSCO lineage:**

| Lineage | Genes | Use when |
|---------|-------|----------|
| `eukaryota_odb10` | ~255 | Your species is any eukaryote, or you want a quick broad check. Lower resolution but universally applicable. |
| `actinopterygii_odb10` | ~3,640 | Your species is a **ray-finned fish** (Actinopterygii — e.g. teleosts, sharks are *not* included). Far more genes assessed, giving a much more sensitive and meaningful completeness score. |

!!! tip
    For fish genome projects, always run **both** lineages. `eukaryota_odb10` allows cross-phylum comparison; `actinopterygii_odb10` gives the most biologically meaningful completeness score for your assembly.

Other lineage options: `viridiplantae_odb10`, `insecta_odb10`, `vertebrata_odb10`

### 0.5 Prepare your FASTQ file

If your files are split across multiple files or compressed:

```bash
# Concatenate multiple FASTQ files
cat *.fastq > species_name_fastq_pass_con.fastq

# Or, if gzipped:
gunzip -c *.fastq.gz > species_name_fastq_pass_con.fastq
```

---

## Step 1: K-mer Analysis

This is **mandatory before assembly**. It estimates genome size and read coverage — the two key parameters for Flye assembly.

### 1.1 Submit the k-mer job

```bash
sbatch submit_kmer.sh /path/to/species_name_fastq_pass_con.fastq species_name
```

### 1.2 Monitor the job

```bash
squeue -u $USER
# or watch the log:
tail -f logs/kmer_<JOBID>.out
```

### 1.3 Read the results

```bash
cat k_mers_Stats_species_name.txt
```

Example output:
```
Species_name: k-mer= 17
Total input bases 153410333621
Peaks or Plateaus detected=2
Ploidy= 2n =2n
2n Genome Length=0.87 Gb at 176 X Coverage
Expected Assembly Length if fully collapsed=0.87 Gb at 176 X Coverage
```

View the k-mer plot (download to local machine):
```bash
# On your local machine:
scp username@your-server.ac.za:/path/to/pipeline/species_name_k_mers.png .
```

### 1.4 Record: genome size and coverage

Note the **genome length** (e.g. `0.87g`) and **coverage** (e.g. `176`) — you will need these in the next step.

---

## Step 2: Configure Assembly Parameters

```bash
nano params.config
```

Edit the following values:

```bash
species_name='species_name'    # Your species name (no spaces)
assembler='flye'               # 'flye' for < 3 Gb; 'hifiasm' for ≥ 3 Gb

threads=15
LINEAGE='actinopterygii_odb10' # Use 'eukaryota_odb10' for non-fish or broad check

# Flye only (from k-mer analysis):
genome_size='0.87g'            # From k_mers_Stats file
flye_coverage=176              # From k_mers_Stats file
flye_read_type='nano-raw'      # Use 'nano-hq' for Q15+ reads
```

### Choosing the assembler

| Genome size | Assembler | Notes |
|---|---|---|
| < 3 Gb | `flye` | Runs through full pipeline including Racon polishing |
| ≥ 3 Gb | `hifiasm` | No polishing step; BUSCO + QUAST run directly on assembly |

### Choosing read type for Flye

| Read quality | Parameter |
|---|---|
| Q > 10 (standard Nanopore) | `nano-raw` |
| Q > 15 (high-accuracy Nanopore) | `nano-hq` |

---

## Step 3: Run the Assembly Pipeline

### 3.1 Submit the main pipeline

```bash
sbatch submit_slurm.sh /path/to/species_name_fastq_pass_con.fastq
```

This single job runs all pipeline stages in order:
1. QC + trimming (Nextflow → SLURM)
2. Assembly — submitted as its own SLURM job (`submit_flye.sh` or `submit_hifiasm.sh`), master.sh waits for it
3. Mapping (Nextflow → SLURM)
4. Polishing, if Flye (Nextflow → SLURM)
5. BUSCO + QUAST evaluation (Nextflow → SLURM)

### 3.2 Monitor progress

```bash
# Top-level job
squeue -u $USER

# All SLURM jobs including Nextflow-submitted sub-jobs
watch squeue -u $USER

# Live log of the main job
tail -f genome_assembly_<JOBID>.out
```

### 3.3 Resume if interrupted

The pipeline uses Nextflow's `-resume` flag automatically. If the top-level job is killed, just resubmit:

```bash
sbatch submit_slurm.sh /path/to/reads.fastq
```

Assembly steps (Flye/Hifiasm) also have built-in resume logic.

---

## Step 4: Check Assembly Outputs

After the pipeline finishes, check the results:

```bash
# BUSCO summary
cat results/Busco_results/Busco_output/short_summary.specific.*.txt

# QUAST report
cat results/quast_report/Quast_result/report.txt

# Final assembly
ls results/Racon_results/     # Flye path
ls results/Hifiasm_results/   # Hifiasm path
```

**BUSCO completeness > 80%** is generally acceptable for a draft assembly.

---

## Step 5: Generate the Final Report

Once you are satisfied with the assembly quality, generate the consolidated report:

```bash
bash generate_report.sh
```

This script (run from the pipeline root directory, no job submission needed):
1. Sorts the SAM file and calculates mapping coverage
2. Collects all key output files
3. Compiles a single report text file with all QC metrics
4. Appends software version information
5. Organises outputs into two folders:
   - `results/<species_name>/` — primary outputs (assemblies, BUSCO, QUAST, NanoPlot, report)
   - `results/<species_name>_other_results_outputs/` — raw pipeline directories

### Primary output files in `results/<species_name>/`

| File | Description |
|---|---|
| `<species>_report.txt` | Consolidated assembly report |
| `<species>_flye_assembly.fasta` | Raw Flye assembly |
| `<species>_racon_polished.fasta` | Polished assembly (Flye only) |
| `<species>_hifiasm_assembly.fasta` | Hifiasm assembly |
| `<species>_busco_summary.txt` | BUSCO completeness summary |
| `<species>_quast_report.txt` | QUAST assembly statistics |
| `<species>_NanoStats_before_trim.txt` | Raw read QC |
| `<species>_NanoStats_after_trim.txt` | Trimmed read QC |
| `<species>_NanoPlot_*.html` | Interactive QC reports |

---

## Parameter Reference

### Assembly parameters (params.config)

| Parameter | Description | Example |
|---|---|---|
| `species_name` | Species identifier (no spaces) | `Acacia_karroo` |
| `assembler` | `flye` or `hifiasm` | `flye` |
| `threads` | CPUs for Nextflow-managed steps | `15` |
| `kraken_db` | Path to Kraken2 database directory | `/data/kraken2_db` |
| `LINEAGE` | BUSCO lineage database | `actinopterygii_odb10` (fish) or `eukaryota_odb10` (broad) |
| `genome_size` | From k-mer analysis (Flye only) | `0.87g` |
| `flye_coverage` | From k-mer analysis (Flye only) | `176` |
| `flye_read_type` | `nano-raw` or `nano-hq` | `nano-raw` |

### SLURM resource allocations (nextflow.config)

| Process | CPUs | Memory | Walltime |
|---|---|---|---|
| NANOCHECK1/2 | 8 | 32 GB | 4h |
| TRIM | 15 | 32 GB | 8h |
| DECONTAMINATE | 15 | 64 GB | 12h |
| MAPPINGS | 15 | 64 GB | 12h |
| POLISH1 | 15 | 64 GB | 12h |
| BUSCOstat_final | 15 | 64 GB | 24h |
| QUAST_final | 8 | 32 GB | 8h |
| Flye (submit_flye.sh) | 40 | 300 GB | 72h |
| Hifiasm (submit_hifiasm.sh) | 40 | 200 GB | 48h |

Adjust these in `nextflow.config` and `submit_flye.sh` / `submit_hifiasm.sh` based on your actual genome size and cluster limits.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| K-mer job finds no peaks | Check read quality; try increasing k-mer size in `kmer-Analysis.sh` |
| Flye job runs out of memory | Increase `--mem` in `submit_flye.sh` |
| Flye job hits walltime | Resubmit `submit_slurm.sh` — Flye will resume automatically |
| BUSCO completeness < 50% | Check read depth and quality; consider re-basecalling with a newer model |
| Nextflow cannot find conda env | Verify `conda activate 1ksa_assembly` works interactively on the cluster |
| `sacct` shows FAILED state | Check `logs/` files for the failing step |

---

## Quick-Start Checklist

```
[ ] 0. Log in to server and navigate to pipeline directory
[ ] 1. Concatenate/unzip FASTQ files
[ ] 2. sbatch submit_kmer.sh reads.fastq species_name
[ ] 3. Check k_mers_Stats_<species>.txt — note genome size and coverage
[ ] 4. Edit params.config with correct values (including kraken_db path)
[ ] 5. sbatch submit_slurm.sh reads.fastq
[ ] 6. Monitor with: watch squeue -u $USER
[ ] 7. Check BUSCO and QUAST results
[ ] 8. bash generate_report.sh
[ ] 9. Review results/<species_name>/<species>_report.txt
```
