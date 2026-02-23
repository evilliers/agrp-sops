# Standard Operating Procedure: ONT COI Barcoding Pipeline for Environmental Samples

**Document ID:** SAIAB-SOP-ONT-001
**Version:** 1.0
**Date:** 2026-02-17
**Author:** SAIAB Genomics
**Status:** Active

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Scope](#2-scope)
3. [Background](#3-background)
4. [Prerequisites](#4-prerequisites)
5. [Pipeline Overview](#5-pipeline-overview)
6. [Detailed Procedure](#6-detailed-procedure)
   - 6.1 [Prepare Input Data](#61-prepare-input-data)
   - 6.2 [Configure the Pipeline](#62-configure-the-pipeline)
   - 6.3 [Create the Conda Environment](#63-create-the-conda-environment)
   - 6.4 [Run the Pipeline](#64-run-the-pipeline)
   - 6.5 [Monitor Jobs](#65-monitor-jobs)
7. [Output Description](#7-output-description)
8. [Quality Control Criteria](#8-quality-control-criteria)
9. [Interpreting Results](#9-interpreting-results)
10. [Troubleshooting](#10-troubleshooting)
11. [Parameter Reference](#11-parameter-reference)
12. [References](#12-references)
13. [Revision History](#13-revision-history)

---

## 1. Purpose

This SOP describes how to run the SAIAB ONT COI barcoding pipeline to process Oxford Nanopore Technologies (ONT) amplicon sequencing data from multiplexed environmental specimens. The pipeline takes basecalled FASTQ reads as input and produces quality-controlled COI barcode sequences with taxonomic identifications against both NCBI and BOLD reference databases.

## 2. Scope

This procedure applies to:

- COI gene (658 bp) amplicon data generated on PromethION (FLO-PRO114M, R10.4.1 chemistry) or MinION/Flongle flow cells.
- Specimens multiplexed using a 2-step PCR strategy with M13-tagged FishF1/FishR1 primers and unique molecular identifier (UMI) barcodes (16 bp UMI + GGTAG pad).
- Pools of up to 100 specimens per sequencing run.

The pipeline is designed to run on the SAIAB SLURM cluster (partition: `agrp`). Users must have an active cluster account with access to this partition.

## 3. Background

DNA barcoding using the mitochondrial cytochrome c oxidase subunit I (COI) gene is the standard approach for species identification in animals (Hebert et al., 2003). Oxford Nanopore sequencing has recently been demonstrated as a cost-effective platform for high-throughput DNA barcoding, capable of processing tens of thousands of specimens in a single run (Hebert et al., 2025; Srivathsan et al., 2021).

Our protocol uses a 2-step PCR approach adapted from the Aguirre Lab protocol (DePaul University):

1. **PCR 1:** COI is amplified using FishF1/FishR1 primers (Ward et al., 2005) carrying M13 universal tag sequences at their 5' ends.
2. **PCR 2:** Specimen-specific UMI barcodes are added via a second PCR using barcode-M13 fusion primers. Each specimen receives a unique combination of forward and reverse UMI barcodes, enabling computational demultiplexing after pooled sequencing.

The final amplicon structure is:

```
5'-[FWD UMI (16bp)]-[GGTAG pad]-[M13 fwd]-[FishF1]--- COI (658bp) ---[FishR1]-[M13 rev]-[GGTAG pad]-[REV UMI (16bp)]-3'
```

This pipeline automates all bioinformatic steps from raw reads to taxonomic assignment.

## 4. Prerequisites

### 4.1 Cluster Access

- Active account on the SAIAB SLURM cluster
- Access to the `agrp` partition
- Conda or Mamba installed (module load or user installation)

### 4.2 Reference Databases

Before first use, ensure the following databases are available on the cluster:

| Database | Purpose | Obtain from |
|----------|---------|-------------|
| NCBI nt | BLAST taxonomy | `update_blastdb.pl nt` or download from [NCBI FTP](https://ftp.ncbi.nlm.nih.gov/blast/db/) |
| BOLD COI (SINTAX-formatted) | SINTAX taxonomy | Download from [BOLD Systems](https://www.boldsystems.org/) and format headers for VSEARCH SINTAX (see Section 6.2.3) |

### 4.3 Required Input Files

| File | Description |
|------|-------------|
| Basecalled FASTQ file(s) | From PromethION/MinION, gzipped or uncompressed |
| UMI barcode sequences | TSV file mapping barcode names to 16 bp sequences |
| Sample sheet | CSV mapping each specimen to its forward + reverse UMI pair |

### 4.4 Wetlab Information Needed

You will need the following information from the wetlab team:

- Which UMI barcode pair was used for each specimen
- The primer pair used (default: FishF1/FishR1; update in config if different)
- Sequencing platform and flow cell type (for QC interpretation)

## 5. Pipeline Overview

The pipeline consists of 10 steps, submitted as SLURM jobs with automatic dependency management:

```
Step 0: Setup (local)       --- Create directories, validate inputs
Step 1: Raw QC              --- NanoPlot quality assessment
Step 2: Filtering           --- Length (550-950 bp) & quality (Q10) filtering
Step 3: Demultiplexing      --- UMI-based specimen assignment (Cutadapt)
Step 4: Clustering          --- VSEARCH clustering at 95% identity
Step 5: Consensus           --- Multiple alignment & consensus calling
Step 6: QC Filtering        --- Translation, length, stop codon checks
Step 7: BLAST Taxonomy      --- NCBI nt search
Step 8: SINTAX Taxonomy     --- BOLD database classification
Step 9: Report              --- Merged results & HTML report
```

**Job dependency graph:**

```
          Step 0 (local)
          /           \
     Step 1 (QC)    Step 2 (filter)
                      |
                    Step 3 (demux)
                      |
                    Step 4 (cluster) [array]
                      |
                    Step 5 (consensus) [array]
                      |
                    Step 6 (QC filter)
                    /           \
              Step 7 (BLAST)  Step 8 (SINTAX)
                    \           /
                    Step 9 (report)
```

Steps 1 and 2 run in parallel. Steps 7 and 8 run in parallel. All other steps are sequential.

## 6. Detailed Procedure

### 6.1 Prepare Input Data

**6.1.1** Copy or symlink your basecalled FASTQ file(s) into the `data/raw/` directory:

```bash
cd /path/to/ont_barcoding

# Option A: Symlink (recommended to avoid duplicating large files)
ln -s /path/to/your/basecalled_reads.fastq.gz data/raw/

# Option B: Copy
cp /path/to/your/basecalled_reads.fastq.gz data/raw/
```

> **Note:** The pipeline accepts both `.fastq` and `.fastq.gz` files. Multiple files in `data/raw/` will be processed together.

**6.1.2** Verify your data is in place:

```bash
ls -lh data/raw/
```

### 6.2 Configure the Pipeline

#### 6.2.1 Edit the Sample Sheet

Open `config/sample_sheet.csv` and enter one row per specimen, mapping each specimen ID to its forward and reverse UMI barcode names:

```csv
specimen_id,forward_umi_name,reverse_umi_name
FISH001,bc01F,bc97R
FISH002,bc02F,bc97R
ARTH003,bc01F,bc98R
```

- **specimen_id**: Your unique identifier for the specimen. Use alphanumeric characters and underscores only (no spaces or special characters).
- **forward_umi_name** / **reverse_umi_name**: Must match names in the UMI sequences file (see 6.2.2).

> **Tip:** The barcode index combinations are documented in `articles/Proposed_barocding_indices.xlsx`. Consult this file for the valid F/R UMI pairings used in your experiment.

#### 6.2.2 Create the UMI Sequences File

Create `config/umi_sequences.tsv` with the actual 16 bp UMI barcode sequences. This is a tab-separated file:

```
umi_name	sequence
bc01F	AACTGACTAACCGTAG
bc02F	AAGGTCGATCAATTGC
bc97R	CGTCGATGTCTTCAAG
bc98R	GACTTAGCACGTCAAT
```

> **Important:** Replace the example sequences above with your actual UMI sequences. These must exactly match the oligos used in the PCR 2 step (16 bp portion only, without the pad or M13 tag).

#### 6.2.3 Edit the Configuration File

Open `config/config.sh` and update the following parameters:

**Required changes (you MUST update these):**

```bash
# Set paths to your local reference databases
BLAST_DB="/path/to/nt"                    # Path to NCBI nt BLAST database
BOLD_DB="/path/to/bold_sintax.fasta"      # Path to BOLD SINTAX-formatted DB
```

**Optional changes (adjust only if needed):**

| Parameter | Default | When to change |
|-----------|---------|----------------|
| `PARTITION` | `agrp` | If your SLURM partition is different |
| `CPUS` | `8` | Adjust based on cluster allocation policy |
| `MEM` | `32G` | Increase for very large datasets |
| `TIME` | `04:00:00` | Increase if jobs hit the time limit |
| `MIN_LENGTH` | `550` | If targeting a different amplicon size |
| `MAX_LENGTH` | `950` | If targeting a different amplicon size |
| `MIN_QUALITY` | `10` | Lower for older chemistry; R10.4.1 supports Q20+ |
| `CLUSTER_ID` | `0.95` | Standard for COI intraspecific variation (Hebert et al., 2025) |
| `MIN_CLUSTER_SIZE` | `5` | Minimum reads to form a valid cluster |
| `GENETIC_CODE` | `5` | Code 5 = invertebrate mitochondrial; use 2 for vertebrate mitochondrial |
| `MIN_READ_DEPTH` | `10` | Minimum reads in dominant cluster for a specimen to pass QC |
| `SINTAX_CUTOFF` | `0.8` | Bootstrap confidence threshold for SINTAX classification |

> **Note on genetic code:** If your samples are exclusively fish, you may use genetic code 2 (vertebrate mitochondrial). For mixed fish + arthropod samples, use code 5 (invertebrate mitochondrial), which is compatible with both groups for COI. See NCBI genetic codes: https://www.ncbi.nlm.nih.gov/Taxonomy/Utils/wprintgc.cgi

#### 6.2.4 Formatting the BOLD Database for SINTAX

If you do not already have a SINTAX-formatted BOLD database, prepare one as follows:

1. Download COI sequences from [BOLD Systems](https://www.boldsystems.org/index.php/resources/api) or use the BOLD data releases.
2. Format FASTA headers to include taxonomy in SINTAX format:

```
>BOLD:AAA0001;tax=d:Animalia,p:Arthropoda,c:Insecta,o:Coleoptera,f:Carabidae,g:Pterostichus,s:Pterostichus_melanarius
ATGCATGC...
```

3. The key requirement is that headers contain `tax=` followed by comma-separated rank:name pairs using single-letter rank codes: `d` (domain), `k` (kingdom), `p` (phylum), `c` (class), `o` (order), `f` (family), `g` (genus), `s` (species).

### 6.3 Create the Conda Environment

**First-time setup only.** This step installs all required software.

```bash
cd /path/to/ont_barcoding

# Create the environment (this may take 10-20 minutes)
conda env create -f envs/environment.yml

# Or with mamba (faster)
mamba env create -f envs/environment.yml
```

To verify the installation:

```bash
conda activate ont_barcoding

# Check key tools
NanoPlot --version
cutadapt --version
vsearch --version
blastn -version
Rscript --version
```

> **For subsequent runs:** You only need to activate the environment:
> ```bash
> conda activate ont_barcoding
> ```

### 6.4 Run the Pipeline

#### 6.4.1 Dry Run (Recommended First Step)

Always perform a dry run before submitting jobs. This validates your configuration and prints the SLURM commands without executing them:

```bash
conda activate ont_barcoding
bash scripts/run_pipeline.sh --dry-run
```

Review the output carefully. Check that:
- All input files are found
- The correct number of specimens is detected
- SLURM parameters look correct
- No errors are reported during setup

#### 6.4.2 Full Pipeline Run

Once the dry run succeeds:

```bash
conda activate ont_barcoding
bash scripts/run_pipeline.sh
```

The script will:
1. Run Step 0 (setup) locally
2. Submit Steps 1-9 as SLURM jobs with dependency chaining
3. Print all job IDs for monitoring

**Save the terminal output** — it contains the SLURM job IDs you will need for monitoring.

Example output:
```
=== ONT Barcoding Pipeline ===
Project: /home/user/ont_barcoding
Specimens: 48

--- Step 0: Setup ---
Output directories created...

--- Step 1: Raw QC ---
  Job ID: 123451
--- Step 2: Filtering ---
  Job ID: 123452
--- Step 3: Demultiplexing ---
  Job ID: 123453
...
=== All jobs submitted ===
```

### 6.5 Monitor Jobs

#### 6.5.1 Check Job Status

```bash
# View all your running/pending jobs
squeue -u $USER

# View a specific job
squeue -j <JOB_ID>

# View detailed job info
scontrol show job <JOB_ID>
```

SLURM job states:
| State | Meaning |
|-------|---------|
| `PD` | Pending (waiting for resources or dependencies) |
| `R` | Running |
| `CD` | Completed |
| `F` | Failed |
| `CA` | Cancelled |

#### 6.5.2 Check Job Logs

All SLURM logs are written to `results/logs/`:

```bash
# View log for a specific step (e.g., filtering)
cat results/logs/02_filter_<JOB_ID>.out

# Follow a running job's output in real time
tail -f results/logs/03_demux_<JOB_ID>.out
```

#### 6.5.3 Check for Failures

```bash
# List any failed jobs
sacct -u $USER --state=FAILED --format=JobID,JobName,State,ExitCode,Elapsed

# Check the error log for a failed job
cat results/logs/<step>_<JOB_ID>.err
```

#### 6.5.4 Cancel Jobs

```bash
# Cancel a single job
scancel <JOB_ID>

# Cancel all your jobs
scancel -u $USER
```

## 7. Output Description

When the pipeline completes successfully, the `results/` directory contains:

```
results/
+-- 01_qc/                      # NanoPlot HTML report and plots
|   +-- raw_NanoPlot-report.html    # Open in browser for QC overview
|   +-- raw_LengthvsQualityScatterPlot_dot.png
|   +-- raw_NanoStats.txt           # Summary statistics
+-- 02_filtered/
|   +-- filtered.fastq.gz          # Length- and quality-filtered reads
+-- 03_demux/
|   +-- <SPECIMEN_ID>.fastq         # One file per specimen
|   +-- demux_stats.tsv             # Read counts per specimen
|   +-- adapters/                   # Generated adapter sequences
+-- 04_clusters/
|   +-- <SPECIMEN_ID>_centroids.fasta
|   +-- <SPECIMEN_ID>_clusters.uc
|   +-- <SPECIMEN_ID>_dominant_reads.fasta
|   +-- cluster_stats.tsv
+-- 05_consensus/
|   +-- <SPECIMEN_ID>_consensus.fasta
|   +-- all_consensus.fasta         # All consensus sequences combined
+-- 06_qc_passed/
|   +-- passed.fasta                # Final QC-passed barcode sequences
|   +-- failed.tsv                  # Failed specimens with reasons
|   +-- qc_summary.tsv             # QC pass/fail counts
+-- 07_blast/
|   +-- blast_results.tsv          # BLAST hits (tabular format)
+-- 08_sintax/
|   +-- sintax_results.tsv         # SINTAX classification results
+-- 09_report/
|   +-- taxonomy_merged.tsv        # *** MAIN RESULT: Merged taxonomy table ***
|   +-- report.html                # *** MAIN RESULT: Visual HTML report ***
+-- logs/                           # SLURM job logs
```

### Key output files

| File | Description | Use |
|------|-------------|-----|
| `09_report/report.html` | Interactive HTML report | Open in web browser; primary deliverable |
| `09_report/taxonomy_merged.tsv` | Tab-separated taxonomy table | Import into R/Excel for downstream analysis |
| `06_qc_passed/passed.fasta` | QC-passed COI sequences | Submit to BOLD/GenBank; use in phylogenetic analyses |
| `03_demux/demux_stats.tsv` | Reads per specimen | Assess sequencing depth and demux success |
| `06_qc_passed/failed.tsv` | QC failures with reasons | Troubleshoot failed specimens |

## 8. Quality Control Criteria

Consensus sequences must pass **all five** QC criteria to be included in the final results (following Hebert et al., 2025):

| # | Criterion | Threshold | Rationale |
|---|-----------|-----------|-----------|
| 1 | Sequence length | 600-700 bp | COI barcode region is 658 bp; allows tolerance for primer trimming variation |
| 2 | Correct reading frame | Translatable in frame 1, 2, or 3 | Verifies the sequence is a genuine coding region |
| 3 | No internal stop codons | 0 stops (genetic code 5) | Stop codons indicate pseudogenes (NUMTs) or frameshifts |
| 4 | No ambiguous bases | 0 N's | Ambiguities indicate low consensus support |
| 5 | Minimum read depth | >= 10 reads in dominant cluster | Ensures sufficient data for reliable consensus |

> **NUMTs warning:** Nuclear copies of mitochondrial genes (NUMTs) are a known issue in COI barcoding. The stop codon and reading frame checks (criteria 2-3) are specifically designed to detect NUMTs (Bensasson et al., 2001). Specimens flagged with stop codons should be manually reviewed.

## 9. Interpreting Results

### 9.1 The HTML Report

Open `results/09_report/report.html` in a web browser. The report contains:

1. **Read count summary** — Tracks read attrition through the pipeline (raw -> filtered -> demuxed -> clustered -> QC passed). Expect 70-90% of reads to survive filtering.

2. **Per-specimen read depth barplot** — Specimens with very low depth (< 50 reads) may yield unreliable consensus sequences. Specimens with zero reads indicate a demultiplexing failure (check UMI sequences).

3. **Demultiplexing success rate** — Target: > 70% of reads assigned to specimens. Low assignment rates may indicate:
   - Incorrect UMI sequences in the config
   - High adapter dimer content
   - Off-target amplification

4. **Taxonomic composition** — Barplots at order and family level. Review for unexpected taxa that may indicate contamination or mis-assignment.

5. **BLAST vs SINTAX agreement** — Target: > 80% genus-level agreement. Disagreements may indicate:
   - Incomplete reference databases
   - Closely related species not resolved at genus level
   - Specimens at the boundary of taxonomic groups

6. **Neighbor-joining tree** — Visual check for clustering of related specimens and potential outliers.

7. **QC failure breakdown** — Identifies the most common reasons for specimen failure, guiding troubleshooting of the wetlab protocol.

### 9.2 The Merged Taxonomy Table

The file `results/09_report/taxonomy_merged.tsv` contains one row per specimen with columns:

| Column | Description |
|--------|-------------|
| `specimen_id` | Your specimen identifier |
| `blast_species` | Top BLAST hit species name |
| `blast_pident` | Percent identity to top BLAST hit |
| `blast_evalue` | E-value of top BLAST hit |
| `blast_genus` | Genus from BLAST top hit |
| `sintax_phylum` through `sintax_species` | SINTAX classifications at each rank |
| `genus_agree` | `agree`, `conflict`, or `one_missing` |

**Interpreting percent identity (BLAST):**

| % Identity | Interpretation | Reference |
|------------|---------------|-----------|
| >= 99% | Species-level match | Hebert et al. (2003) |
| 95-99% | Likely correct genus, possibly different species | |
| 90-95% | Likely correct family | |
| < 90% | Higher-level match only; possible novel lineage | |

### 9.3 Expected Performance Benchmarks

Based on Hebert et al. (2025) and Srivathsan et al. (2021):

| Metric | Expected Range |
|--------|---------------|
| Reads passing length/quality filter | 70-90% |
| Demultiplexing assignment rate | > 70% |
| Specimens passing all QC checks | 80-95% |
| BLAST vs SINTAX genus agreement | > 80% |
| Consensus accuracy (vs Sanger) | > 99.9% |

## 10. Troubleshooting

### 10.1 Common Issues

| Problem | Possible Cause | Solution |
|---------|---------------|----------|
| Setup fails: "No FASTQ files found" | Wrong path or missing data | Verify files exist in `data/raw/` with `ls -la data/raw/` |
| Setup warns: "UMI sequence not found" | Mismatch between sample sheet and UMI file | Check that barcode names in `sample_sheet.csv` match `umi_sequences.tsv` exactly |
| Low demux assignment rate (< 50%) | Incorrect UMI sequences | Verify UMI sequences match the oligos used in PCR 2 |
| Low demux assignment rate (< 50%) | Wrong primer sequences | Update `PRIMER_FWD`/`PRIMER_REV` in config if using non-standard primers |
| Many specimens with 0 reads | Contamination or failed PCR | Check wetlab gel images; re-extract/re-amplify failed specimens |
| High QC failure — stop codons | NUMTs co-amplified | Increase `MIN_CLUSTER_SIZE`; consider redesigning primers |
| High QC failure — length | Chimeric or truncated amplicons | Tighten length filter; check gel for non-specific bands |
| High QC failure — ambiguous bases | Low read depth | Pool fewer specimens per run to increase per-specimen depth |
| BLAST job times out | Large dataset + remote DB | Use local BLAST database; increase `TIME` in config |
| SINTAX: "database not found" | Wrong path in config | Verify `BOLD_DB` path exists and is readable |
| Conda environment fails to solve | Package conflicts | Try `mamba env create -f envs/environment.yml` or create in stages |
| SLURM: "dependency never satisfied" | A prerequisite job failed | Check logs for the failed dependency job; fix and resubmit from that step |

### 10.2 Rerunning Individual Steps

If a step fails, you do not need to rerun the entire pipeline. Fix the issue, then submit the failed step and all subsequent steps manually:

```bash
conda activate ont_barcoding
cd /path/to/ont_barcoding

# Example: Rerun from Step 6 onward
JOB6=$(sbatch -p agrp scripts/06_qc_filter.sh | grep -oP '\d+')
JOB7=$(sbatch -p agrp --dependency=afterok:$JOB6 scripts/07_taxonomy_blast.sh | grep -oP '\d+')
JOB8=$(sbatch -p agrp --dependency=afterok:$JOB6 scripts/08_taxonomy_sintax.sh | grep -oP '\d+')
sbatch -p agrp --dependency=afterok:$JOB7:$JOB8 scripts/09_report.sh
```

### 10.3 Rerunning a Single Specimen (Steps 4-5)

For array job steps, you can rerun a single specimen by specifying its array index:

```bash
# Find the specimen's line number in the specimen list
grep -n "SPEC042" results/specimen_list.txt
# Output: 42:SPEC042

# Rerun clustering for just that specimen
sbatch -p agrp --array=42 scripts/04_cluster.sh
```

## 11. Parameter Reference

All parameters are defined in `config/config.sh`. This table provides a complete reference:

### Paths

| Parameter | Description |
|-----------|-------------|
| `RAW_DATA` | Directory containing raw FASTQ files |
| `RESULTS` | Base output directory |
| `SAMPLE_SHEET` | Path to the specimen-UMI mapping CSV |

### SLURM

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PARTITION` | `agrp` | SLURM partition for job submission |
| `CPUS` | `8` | CPUs per job |
| `MEM` | `32G` | Memory per job |
| `TIME` | `04:00:00` | Wall time limit per job |

### Filtering

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MIN_LENGTH` | `550` | Minimum read length in bp |
| `MAX_LENGTH` | `950` | Maximum read length in bp |
| `MIN_QUALITY` | `10` | Minimum Phred quality score |

### Demultiplexing

| Parameter | Default | Description |
|-----------|---------|-------------|
| `UMI_MISMATCHES` | `2` | Allowed mismatches in 16 bp UMI (no indels) |
| `PRIMER_FWD` | FishF1 sequence | Forward COI primer |
| `PRIMER_REV` | FishR1 sequence | Reverse COI primer |
| `M13_FWD` | `TGTAAAACGACGGCCAGT` | M13 forward universal tag |
| `M13_REV` | `CAGGAAACAGCTATGAC` | M13 reverse universal tag |
| `PAD` | `GGTAG` | Pad sequence between UMI and M13 tag |

### Clustering

| Parameter | Default | Description |
|-----------|---------|-------------|
| `CLUSTER_ID` | `0.95` | Sequence identity threshold (95%) |
| `MIN_CLUSTER_SIZE` | `5` | Minimum reads to form a cluster |

### Quality Control

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MIN_SEQ_LENGTH` | `600` | Minimum consensus length (bp) |
| `MAX_SEQ_LENGTH` | `700` | Maximum consensus length (bp) |
| `GENETIC_CODE` | `5` | NCBI translation table (5 = invertebrate mito) |
| `MIN_READ_DEPTH` | `10` | Minimum reads in dominant cluster |

### Taxonomy

| Parameter | Default | Description |
|-----------|---------|-------------|
| `BLAST_DB` | `/path/to/nt` | Path to NCBI nt BLAST database |
| `BOLD_DB` | `/path/to/bold_sintax.fasta` | Path to BOLD SINTAX database |
| `BLAST_EVALUE` | `1e-5` | BLAST E-value threshold |
| `BLAST_PIDENT` | `80` | Minimum percent identity for BLAST hits |
| `SINTAX_CUTOFF` | `0.8` | SINTAX bootstrap confidence cutoff |

## 12. References

Bensasson, D., Zhang, D.-X., Hartl, D. L., & Hewitt, G. M. (2001). Mitochondrial pseudogenes: evolution's misplaced witnesses. *Trends in Ecology & Evolution*, 16(6), 314-321. https://doi.org/10.1016/S0169-5347(01)02151-6

Hebert, P. D. N., Cywinska, A., Ball, S. L., & deWaard, J. R. (2003). Biological identifications through DNA barcodes. *Proceedings of the Royal Society of London B*, 270(1512), 313-321. https://doi.org/10.1098/rspb.2002.2218

Hebert, P. D. N., Floyd, R., Jafarpour, S., & Prosser, S. W. J. (2025). Barcode 100K specimens: In a single nanopore run. *Molecular Ecology Resources*, 25, e14028. https://doi.org/10.1111/1755-0998.14028

Martin, M. (2011). Cutadapt removes adapter sequences from high-throughput sequencing reads. *EMBnet.journal*, 17(1), 10-12. https://doi.org/10.14806/ej.17.1.200

Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH: a versatile open source tool for metagenomics. *PeerJ*, 4, e2584. https://doi.org/10.7717/peerj.2584

Srivathsan, A., Lee, L., Katoh, K., Hartop, E., Kutty, S. N., Wong, J., Yeo, D., & Meier, R. (2021). ONTbarcoder and MinION barcodes aid biodiversity discovery and identification by everyone, for everyone. *BMC Biology*, 19, 217. https://doi.org/10.1186/s12915-021-01141-x

Ward, R. D., Zemlak, T. S., Innes, B. H., Last, P. R., & Hebert, P. D. N. (2005). DNA barcoding Australia's fish species. *Philosophical Transactions of the Royal Society B*, 360(1462), 1847-1857. https://doi.org/10.1098/rstb.2005.1716

Wright, E. S. (2016). Using DECIPHER v2.0 to analyze big biological sequence data in R. *The R Journal*, 8(1), 352-359. https://doi.org/10.32614/RJ-2016-025

### Software Versions

| Tool | Version | Purpose |
|------|---------|---------|
| NanoPlot | 1.42 | Read quality assessment (De Coster & Rademakers, 2023) |
| Cutadapt | 4.6 | Read filtering and demultiplexing (Martin, 2011) |
| VSEARCH | 2.28 | Clustering and SINTAX classification (Rognes et al., 2016) |
| BLAST+ | 2.16 | Sequence similarity search (Camacho et al., 2009) |
| MUSCLE | 5.1 | Multiple sequence alignment (Edgar, 2022) |
| seqkit | 2.8 | Sequence file manipulation (Shen et al., 2016) |
| DECIPHER | (R package) | Consensus sequence calling (Wright, 2016) |
| Biostrings | (R package) | Sequence handling in R (Pagès et al., 2024) |

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-17 | SAIAB Genomics | Initial release |
