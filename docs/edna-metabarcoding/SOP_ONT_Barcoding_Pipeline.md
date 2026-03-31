# Standard Operating Procedure: ONT COI Barcoding Pipeline for Environmental Samples

**Document ID:** SAIAB-SOP-ONT-001
**Version:** 2.1
**Date:** 2026-03-31
**Author:** SAIAB Genomics
**Status:** Active

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Scope](#2-scope)
3. [Background](#3-background)
4. [Prerequisites](#4-prerequisites)
   - 4.5 [Obtain the Pipeline Scripts](#45-obtain-the-pipeline-scripts)
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
- Specimens multiplexed using a 2-step PCR strategy with M13-tagged FishF1/FishF2/FishR1/FR1d-t1 primers (4-primer cocktail) and unique molecular identifier (UMI) barcodes (GGTAG pad + 16 bp UMI).
- Pools of up to 100 specimens per sequencing run.

The pipeline is designed to run on the SAIAB SLURM cluster (partition: `agrp`). Users must have an active cluster account with access to this partition.

## 3. Background

DNA barcoding using the mitochondrial cytochrome c oxidase subunit I (COI) gene is the standard approach for species identification in animals (Hebert et al., 2003). Oxford Nanopore sequencing has recently been demonstrated as a cost-effective platform for high-throughput DNA barcoding, capable of processing tens of thousands of specimens in a single run (Hebert et al., 2025; Srivathsan et al., 2021).

Our protocol uses a 2-step PCR approach adapted from the Aguirre Lab protocol (DePaul University):

1. **PCR 1:** COI is amplified using FishF1/FishR1 primers (Ward et al., 2005) carrying M13 universal tag sequences at their 5' ends.
2. **PCR 2:** Specimen-specific UMI barcodes are added via a second PCR using barcode-M13 fusion primers. Each specimen receives a unique combination of forward and reverse UMI barcodes, enabling computational demultiplexing after pooled sequencing.

The final amplicon structure is:

```
5'-[GGTAG pad]-[FWD UMI (16bp)]-[M13 fwd]-[FishF1 or FishF2]--- COI (658bp) ---[FishR1 or FR1d-t1]-[M13 rev]-[REV UMI (16bp)]-[GGTAG pad]-3'
```

> **PCR primer cocktail:** PCR 1 uses a 4-primer cocktail — two forward primers (M13-FishF1 and M13-FishF2) and two reverse primers (M13-FishR1 and M13-FR1d-t1) — to improve amplification success across a range of fish taxa. Any combination of one forward and one reverse primer may amplify a given specimen. The pipeline handles all four variants during primer trimming (Step 3).

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
| MIDORI2 COI (BLAST-formatted) | BLAST taxonomy | Download from [MIDORI](https://www.reference-midori.info/download.php); use the `MIDORI2_LONGEST_NUC_*_CO1_BLAST` release and run `makeblastdb` |
| BOLD COI (SINTAX-formatted) | SINTAX taxonomy | Download from [BOLD Systems](https://www.boldsystems.org/) and format headers for VSEARCH SINTAX (see Section 6.2.3) |

> **Why MIDORI2 instead of NCBI nt?** MIDORI2 is a curated subset of GenBank containing only COI sequences from metazoans. For COI barcoding it is preferred over NCBI nt because: (1) searches are much faster due to the smaller database size; (2) only COI-relevant hits are returned, avoiding false positives from non-COI sequences; (3) it is taxonomically curated, reducing erroneous hits. Taxonomy is encoded directly in the sequence ID field (`sseqid`) in the format `accession###...;Genus_species_taxid` — the pipeline parses this automatically.

### 4.3 Required Input Files

| File | Description |
|------|-------------|
| Basecalled FASTQ file(s) | From PromethION/MinION, gzipped or uncompressed |
| UMI barcode sequences | TSV file mapping barcode names to 16 bp sequences |
| Sample sheet | CSV mapping each specimen to its forward + reverse UMI pair |

### 4.4 Wetlab Information Needed

You will need the following information from the wetlab team:

- Which UMI barcode pair was used for each specimen
- Which primer cocktail was used (default: FishF1+FishF2 forward, FishR1+FR1d-t1 reverse; update `PRIMER_FWD`/`PRIMER_FWD2`/`PRIMER_REV`/`PRIMER_REV2` in config if different)
- Sequencing platform and flow cell type (for QC interpretation)

### 4.5 Obtain the Pipeline Scripts

Clone the pipeline repository from the SAIAB internal Gitea server:

```bash
git clone http://172.20.142.126:3000/evilliers/ont_barcoding
cd ont_barcoding
```

> **Note:** All subsequent steps in this SOP assume you are working from the cloned `ont_barcoding/` directory. Replace any reference to `/path/to/ont_barcoding` with the actual path to your clone (e.g., `~/ont_barcoding`).

If you have already cloned the repository previously and want to update to the latest version:

```bash
cd /path/to/ont_barcoding
git pull
```

## 5. Pipeline Overview

The pipeline consists of 10 steps, submitted as SLURM jobs with automatic dependency management:

```
Step 0: Setup (local)       --- Create directories, validate inputs
Step 1: Raw QC              --- NanoPlot quality assessment
Step 2: Filtering           --- Length (550-950 bp) & quality (Q10) filtering
Step 3: Demultiplexing      --- UMI-based specimen assignment (Cutadapt)
Step 4: Clustering          --- VSEARCH clustering at 95% identity
Step 5: Consensus           --- Multiple alignment & consensus calling
Step 6: QC Filtering        --- Primer stripping (cutadapt), then translation, length, and stop codon checks
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

Open `config/sample_sheet.csv` and enter one row per specimen, mapping each specimen ID to its forward and reverse UMI barcode names. The file **must** use exactly these three column headers:

```csv
specimen_id,fwd_umi,rev_umi
SPR22_008,bc1001,bc1097_rc
SPR22_054,bc1002,bc1097_rc
SPR22_264,bc1003,bc1097_rc
SPR22_033,bc1001,bc1098_rc
SPR22_080,bc1002,bc1098_rc
```

**Column descriptions:**

| Column | Required | Description |
|--------|----------|-------------|
| `specimen_id` | Yes | Unique identifier for the specimen. Use alphanumeric characters and underscores only — **no spaces or special characters**. Must be unique within the file. |
| `fwd_umi` | Yes | Name of the forward UMI barcode assigned to this specimen during PCR 2. Must exactly match a `umi_name` in `config/umi_sequences.tsv`. Valid values: `bc1001` – `bc1096`. |
| `rev_umi` | Yes | Name of the reverse UMI barcode. Must exactly match a `umi_name` in `config/umi_sequences.tsv`. Valid values: `bc1097_rc` – `bc1192_rc` (note the `_rc` suffix — these are reverse-complement barcodes). |

**Combinatorial indexing:** Each specimen is uniquely identified by the **combination** of its forward and reverse barcode. Multiple specimens can share the same forward barcode (e.g., `bc1001`) as long as they have different reverse barcodes, and vice versa. The same barcode pair must **never** be used for two different specimens in the same run.

> **Tip:** The barcode index combinations are documented in `articles/Proposed_barcoding_indices.xlsx`. Consult this file together with the wetlab team to confirm which UMI pair was assigned to each specimen before PCR 2.

**Example layout for a 3×3 pool (9 specimens):**

```
          bc1097_rc  bc1098_rc  bc1099_rc
bc1001    SPEC_A     SPEC_D     SPEC_G
bc1002    SPEC_B     SPEC_E     SPEC_H
bc1003    SPEC_C     SPEC_F     SPEC_I
```

#### 6.2.2 Verify the UMI Sequences File

The file `config/umi_sequences.tsv` is **pre-populated** with the full set of 16 bp UMI barcode sequences used in the SAIAB barcoding protocol. You do **not** need to create or edit this file for standard runs.

The file is tab-separated with two columns:

```
umi_name	sequence
bc1001	CACATATCAGAGTGCG
bc1002	ACACACAGACTGTGAG
...
bc1096	TGTGCTCTCTACACAG
bc1097_rc	TAGAGAGATAGAGACG
bc1098_rc	TGATGTGACACTGCGC
...
bc1192_rc	GTCTCAGCACGAGACA
```

- **Forward barcodes** (`bc1001`–`bc1096`): 96 barcodes used on the forward primer arm (PCR 2 forward primer).
- **Reverse barcodes** (`bc1097_rc`–`bc1192_rc`): 96 barcodes used on the reverse primer arm, pre-computed as the reverse complement (hence the `_rc` suffix). The `_rc` suffix is required — these names must match exactly what is written in `sample_sheet.csv`.

> **Only edit this file** if you are adding new barcodes beyond the existing 192-barcode set. New entries must follow the same tab-separated format with a unique `umi_name` and a 16 bp DNA sequence.

#### 6.2.3 Edit the Configuration File

Open `config/config.sh` and update the following parameters:

**Required changes (you MUST update these):**

```bash
# Set paths to your local reference databases
BLAST_DB="/path/to/MIDORI2_LONGEST_NUC_*_CO1_BLAST"  # Path to MIDORI2 COI BLAST database
BOLD_DB="/path/to/bold_COI_sintax_derep.fasta"        # Path to BOLD SINTAX-formatted DB
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
| `GENETIC_CODE` | `2` | NCBI translation table (2 = vertebrate mitochondrial; use 5 for invertebrate mitochondrial) |
| `MIN_READ_DEPTH` | `10` | Minimum reads in dominant cluster for a specimen to pass QC |
| `SINTAX_CUTOFF` | `0.6` | Bootstrap confidence threshold for SINTAX classification |

> **Note on genetic code:** Use code 2 (vertebrate mitochondrial) for fish samples. Use code 5 (invertebrate mitochondrial) for arthropod samples or mixed vertebrate + invertebrate pools. See NCBI genetic codes: https://www.ncbi.nlm.nih.gov/Taxonomy/Utils/wprintgc.cgi

> **Note on SINTAX cutoff:** A cutoff of 0.6 balances sensitivity and specificity. For many Indo-Pacific and deep-sea fish species that are under-represented in BOLD, genus- and species-level SINTAX bootstrap values rarely exceed 0.8, so a cutoff of 0.8 silently discards valid higher-rank classifications (order, family). If your taxa are very well represented in BOLD (e.g., common Atlantic fish, insects), you may raise this to 0.8 for stricter classifications.

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

> **Important:** Steps 4 and 5 are SLURM array jobs and must be submitted with `--array=1-N` where N is the number of specimens (rows in `config/sample_sheet.csv`, typically 40 for a full run). All other steps are single jobs and must NOT include `--array`. Submitting Step 3 as an array job will cause multiple processes to write to the same output files simultaneously.

| Step | Script | Submission command | Notes |
|------|--------|--------------------|-------|
| 3 | `03_demux.sh` | `sbatch scripts/03_demux.sh` | **NOT an array job.** Loops over all specimens internally. |
| 4 | `04_cluster.sh` | `sbatch --array=1-N scripts/04_cluster.sh` | **Array job.** Uses `SLURM_ARRAY_TASK_ID` to select specimen. |
| 5 | `05_consensus.sh` | `sbatch --array=1-N scripts/05_consensus.sh` | **Array job.** Same as Step 4. |
| 6–9 | All others | `sbatch scripts/0X_name.sh` | **NOT array jobs.** |

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
|   +-- length_distribution.png    # Histogram of consensus lengths (passed vs failed)
|   +-- all_consensus_trimmed.fasta # Primer-stripped consensus (input to QC)
|   +-- primer_trim.log             # Cutadapt primer-stripping log
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
| `06_qc_passed/length_distribution.png` | Consensus length histogram (passed vs failed) | Detect off-target amplicons (e.g., upper band on PCR gel); sequences outside the 600–700 bp window appear in red |

## 8. Quality Control Criteria

> **Pre-QC primer stripping:** Before QC is applied, all consensus sequences are stripped of COI primer sequences (and their reverse complements) using cutadapt. This is necessary because the Step 3 demultiplexing trim removes whichever primer end it first encounters but may leave the other end intact; consensus calling then preserves these partial primer sequences. The stripping step uses `--match-read-wildcards` to handle IUPAC ambiguity codes and `--overlap 15` to ensure only genuine primer matches are trimmed.

> **Expected amplicon length for this primer set:** The FishF1/FishF2/FishR1/FR1d-t1 cocktail amplifies a product of approximately 680–686 bp (after primer stripping), not the standard 658 bp COI barcode. This is normal and within the 600–700 bp QC window. Do not narrow the QC window below 680 bp for this primer set.

Consensus sequences must pass **all five** QC criteria to be included in the final results (following Hebert et al., 2025):

| # | Criterion | Threshold | Rationale |
|---|-----------|-----------|-----------|
| 1 | Sequence length | 600-700 bp | COI barcode region is 658 bp; allows tolerance for primer trimming variation |
| 2 | Correct reading frame | Translatable in frame 1, 2, or 3 | Verifies the sequence is a genuine coding region |
| 3 | No internal stop codons | 0 stops (genetic code 2 for fish) | Stop codons indicate pseudogenes (NUMTs) or frameshifts |
| 4 | No ambiguous bases | 0 N's | Ambiguities indicate low consensus support |
| 5 | Minimum read depth | >= 10 reads in dominant cluster | Ensures sufficient data for reliable consensus |

> **NUMTs warning:** Nuclear copies of mitochondrial genes (NUMTs) are a known issue in COI barcoding. The stop codon and reading frame checks (criteria 2-3) are specifically designed to detect NUMTs (Bensasson et al., 2001). Specimens flagged with stop codons should be manually reviewed.

## 9. Interpreting Results

### 9.1 The HTML Report

Open `results/09_report/report.html` in a web browser. The report contains:

1. **Read count summary** — Tracks read attrition through the pipeline (raw -> filtered -> demuxed -> clustered -> QC passed). Expect 70-90% of reads to survive filtering.

2. **Per-specimen read depth barplot** — Specimens with very low depth (< 50 reads) may yield unreliable consensus sequences. Specimens with zero reads indicate a demultiplexing failure (check UMI sequences).

3. **Demultiplexing success rate** — Target: 50–80% of reads assigned to specimens, depending on PCR2 efficiency. Rates as low as 46% may be acceptable if all specimens are assigned and read depths are sufficient. Rates below 50% indicate incomplete PCR2 conversion — see Section 4 lab recommendations. Low assignment rates may also indicate:
   - Incorrect UMI sequences in the config
   - High adapter dimer content
   - Off-target amplification

4. **Taxonomic composition** — Barplots at order and family level. Review for unexpected taxa that may indicate contamination or mis-assignment.

5. **BLAST vs SINTAX agreement** — Target: > 80% genus-level agreement. Disagreements may indicate:
   - Incomplete reference databases (see note below)
   - Closely related species not resolved at genus level
   - Specimens at the boundary of taxonomic groups

   > **BOLD coverage note:** BOLD has uneven coverage across taxa. Indo-Pacific and deep-sea fish species (e.g., *Monocentris japonica*, *Hoplostethus* spp.) are frequently under-represented compared to commercially important Atlantic or temperate species. For such taxa, SINTAX bootstrap values at genus and species level may remain below the classification cutoff even with a correct identification, causing SINTAX to report only order- or family-level assignments (or nothing at all). In these cases, **BLAST against MIDORI2 is the more reliable identification method** and should be used as the primary result.

6. **Mixed-well detection:** When a specimen's primary and b consensus sequences match different species (or highly divergent genera), this is strong evidence of a mixed well, cross-contamination, or a labelling error. The pipeline does not automatically flag this — compare BLAST top hits for primary and b sequences manually. In confirmed mixed wells, neither sequence should be used as a definitive identification without re-extraction.

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

> **When SINTAX assigns a wrong kingdom (e.g., Fungi, Plantae) with very low confidence (< 0.1):** This indicates the sequence has no close match in the BOLD database. It is NOT a sign of contamination — BLAST will correctly identify the specimen if it is a real fish COI sequence. Always check the BLAST result when SINTAX confidence is below 0.1 at kingdom level. Use BLAST as the definitive identification.

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
| Demultiplexing assignment rate | 50–80%, depending on PCR2 efficiency. Rates as low as 46% may be acceptable if all specimens are assigned and read depths are sufficient. |
| Specimens passing all QC checks | 80-95% |
| BLAST vs SINTAX genus agreement | > 80% |
| Consensus accuracy (vs Sanger) | > 99.9% |

## 10. Troubleshooting

### 10.1 Common Issues

| Problem | Possible Cause | Solution |
|---------|---------------|----------|
| Setup fails: "No FASTQ files found" | Wrong path or missing data | Verify files exist in `data/raw/` with `ls -la data/raw/` |
| Setup warns: "UMI sequence not found" | Mismatch between sample sheet and UMI file | Check that barcode names in `sample_sheet.csv` match `umi_sequences.tsv` exactly. Common mistake: using `bc1097` instead of `bc1097_rc` for reverse barcodes |
| Low demux assignment rate (< 50%) | Incorrect UMI sequences | Verify UMI sequences match the oligos used in PCR 2 |
| Low demux assignment rate (< 50%) | Wrong primer sequences | Update `PRIMER_FWD`/`PRIMER_REV` in config if using non-standard primers |
| Demux rate 40–55% but all specimens assigned | Incomplete PCR2 UMI-tagging | Expected when PCR1 products are not fully consumed by PCR2. Run is valid. Improve PCR2 efficiency in future runs (bead cleanup after PCR1, verify PCR2 by gel, increase PCR2 cycles). |
| Many specimens with 0 reads | Contamination or failed PCR | Check wetlab gel images; re-extract/re-amplify failed specimens |
| High QC failure — stop codons | NUMTs co-amplified | Increase `MIN_CLUSTER_SIZE`; consider redesigning primers |
| QC failure: stops=16–18 on otherwise normal-looking consensus | 1–2 IUPAC bases immediately before the primer start caused cutadapt to miss the primer; remaining bases shift the reading frame | Expected for ~15% of sequences. These represent real COI sequences. Manual trimming of the IUPAC prefix is possible but not currently automated. |
| QC failure: not_translatable on "b" sequence; starts with PRIMER_REV | Consensus assembled in minus orientation; `-g PRIMER_REV` in cutadapt failed to strip it due to IUPAC prefix | Same as above. These "b" sequences are typically redundant — if the primary sequence passed, the specimen still has a valid barcode. |
| QC failure: non-COI sequence; sequence does not resemble fish COI | Non-specific PCR amplification in that well | Re-extract and re-amplify the specimen. Check PCR gel for anomalous band size. |
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

> **If Step 6 produces 0 sequences passing:** Check whether consensus sequences contain primer sequence. Inspect the 5′ end of a few sequences in `results/05_consensus/all_consensus.fasta` using `grep -A1 "^>" results/05_consensus/all_consensus.fasta | head -20`. If primer sequences are visible, verify that the primer-stripping cutadapt block in `scripts/06_qc_filter.sh` is present and that the primer sequences in `config/config.sh` are correct.

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
| `PRIMER_FWD` | FishF1 sequence | Forward COI primer 1 (4-primer cocktail) |
| `PRIMER_FWD2` | FishF2 sequence | Forward COI primer 2 (4-primer cocktail) |
| `PRIMER_REV` | FishR1 sequence | Reverse COI primer 1 (4-primer cocktail) |
| `PRIMER_REV2` | FR1d-t1 sequence | Reverse COI primer 2 (4-primer cocktail) |
| `M13_FWD` | `TGTAAAACGACGGCCAGT` | M13 forward universal tag (18 bp; absorbs leading T of FishF1/FishF2) |
| `M13_REV` | `CAGGAAACAGCTATGAC` | M13 reverse universal tag |
| `PAD` | `GGTAG` | Pad sequence preceding the UMI in PCR 2 primers (order: PAD → UMI → M13) |

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
| `GENETIC_CODE` | `2` | NCBI translation table (2 = vertebrate mitochondrial; use 5 for invertebrate mitochondrial) |
| `MIN_READ_DEPTH` | `10` | Minimum reads in dominant cluster |

### Taxonomy

| Parameter | Default | Description |
|-----------|---------|-------------|
| `BLAST_DB` | — | Path to MIDORI2 COI BLAST database (MIDORI2_LONGEST_NUC_*_CO1_BLAST) |
| `BOLD_DB` | — | Path to BOLD COI SINTAX-formatted database |
| `BLAST_EVALUE` | `1e-5` | BLAST E-value threshold |
| `BLAST_PIDENT` | `80` | Minimum percent identity for BLAST hits |
| `SINTAX_CUTOFF` | `0.6` | SINTAX bootstrap confidence cutoff; lower values recover more classifications for under-represented taxa |

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
| 1.1 | 2026-03-13 | SAIAB Genomics | Corrected sample sheet column names to `fwd_umi`/`rev_umi` (matching actual code); updated UMI naming convention to `bc1001`–`bc1096` (forward) and `bc1097_rc`–`bc1192_rc` (reverse); clarified that `umi_sequences.tsv` is pre-populated; expanded sample sheet field descriptions and added combinatorial indexing layout example |
| 1.2 | 2026-03-17 | SAIAB Genomics | Updated BLAST database from NCBI nt to MIDORI2 COI (preferred for COI barcoding: smaller, faster, COI-specific); updated `GENETIC_CODE` default from 5 (invertebrate) to 2 (vertebrate mitochondrial) for fish samples; lowered `SINTAX_CUTOFF` from 0.8 to 0.6 to recover order/family-level BOLD classifications for Indo-Pacific and deep-sea fish that are under-represented in BOLD at species level; added guidance on BOLD coverage limitations and when BLAST should be treated as the primary identification; added consensus sequence length distribution plot to Step 6 QC |
| 1.3 | 2026-03-17 | SAIAB Genomics | Corrected PCR 2 adapter construction order from UMI→PAD→M13 to PAD→UMI→M13 (matching actual primer design confirmed by wetlab); updated amplicon structure diagram; added second forward primer (FishF2) and second reverse primer (FR1d-t1) to config and demux step to handle the full 4-primer cocktail used in PCR 1; demux now correctly identifies reads regardless of which primer combination amplified the specimen |
| 2.0 | 2026-03-18 | SAIAB Genomics | Incorporated lessons from first full production run (SPR22). Added primer-stripping cutadapt pass to Step 6 (consensus sequences retain partial primers requiring pre-QC stripping); updated demux assignment rate benchmark from >70% to 50–80% (low rates due to PCR2 UMI-tagging efficiency are acceptable); clarified array vs non-array job submission (Steps 4–5 are array jobs; Step 3 is not); added mixed-well detection guidance (divergent primary/b BLAST hits); added SINTAX wrong-kingdom guidance for deep-sea taxa absent from BOLD; documented expected amplicon length of 680–686 bp for FishF1/F2/R1/FR1d-t1 primer set; added three new QC failure patterns to troubleshooting; added Step 6 zero-output diagnostic note |
| 2.1 | 2026-03-31 | SAIAB Genomics | Added Section 4.5: instructions for obtaining the pipeline scripts by cloning from the SAIAB Gitea repository (http://172.20.142.126:3000/evilliers/ont_barcoding) |
