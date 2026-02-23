---
title: "eDNA Metabarcoding Data Analysis SOP"
subtitle: "mothur + BLAST Pipeline for Illumina Paired-End Sequencing Data"
version: "3.0"
date: "January 2026"
authors: "Updated from v2 (October 2021)"
pipeline: "mothur → BLAST → R Integration"
---

# eDNA Metabarcoding Data Analysis
## mothur + BLAST Pipeline for Illumina Paired-End Data

**Version 3.0** | **January 2026**

---

## Table of Contents

1. [Introduction & Overview](#1-introduction--overview)
2. [Prerequisites & Requirements](#2-prerequisites--requirements)
3. [Software Installation & Setup](#3-software-installation--setup)
4. [Data Preparation](#4-data-preparation)
5. [Mothur Analysis Pipeline](#5-mothur-analysis-pipeline)
6. [BLAST Taxonomic Assignment](#6-blast-taxonomic-assignment)
7. [R Data Integration](#7-r-data-integration)
8. [Results Interpretation](#8-results-interpretation)
9. [Troubleshooting](#9-troubleshooting)
10. [Glossary](#10-glossary)
11. [References & Citations](#11-references--citations)
12. [Appendices](#12-appendices)

---

## 1. Introduction & Overview

### 1.1 Purpose

This Standard Operating Procedure (SOP) provides a comprehensive workflow for analyzing environmental DNA (eDNA) metabarcoding data from Illumina paired-end sequencing. The pipeline integrates three powerful bioinformatics tools:

- **mothur**: Microbial community analysis and sequence quality control
- **BLAST**: Taxonomic assignment via sequence similarity
- **R**: Data integration, summarization, and export

### 1.2 Workflow Summary

```
Raw FASTQ files
    ↓
mothur: Quality filtering & chimera removal
    ↓
Curated FASTA sequences
    ↓
BLAST: Taxonomic assignment
    ↓
R: Data integration & summarization
    ↓
Final species/taxa table
```

### 1.3 Expected Timeline

| Phase | Estimated Time |
|-------|---------------|
| Data preparation | 30 min - 1 hour |
| mothur analysis | 2-6 hours (dataset dependent) |
| BLAST analysis | 1-4 hours (dataset dependent) |
| R integration | 30 min - 1 hour |
| **Total** | **4-12 hours** |

⚠️ **Note**: Processing time varies significantly based on dataset size, sequence depth, and computational resources.

---

## 2. Prerequisites & Requirements

### 2.1 Required Software

| Software | Version (2026) | Download Link |
|----------|---------------|---------------|
| **mothur** | Latest release | [GitHub Releases](https://github.com/mothur/mothur/releases) |
| **BLAST+** | v2.17.0+ | [NCBI FTP](https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/) |
| **R** | v4.3.0+ | [CRAN](https://cran.r-project.org/) |
| **R packages** | tidyverse 2.0+, dplyr 1.1.4+ | Install via `install.packages()` |

**File Transfer Tools** (choose based on your platform):
- **Windows**: WinSCP, FileZilla, or native `scp`
- **Mac/Linux**: Terminal with `scp`, `rsync`, or FileZilla

**SSH Clients** (choose based on your platform):
- **Windows**: PuTTY, Windows Terminal, or PowerShell SSH
- **Mac/Linux**: Native Terminal or iTerm2

**Text Editors** (for creating configuration files):
- **Windows**: Notepad++, VS Code, Sublime Text
- **Mac/Linux**: TextEdit, nano, vim, VS Code

### 2.2 System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| **RAM** | 8 GB | 16-32 GB |
| **Disk Space** | 50 GB free | 100+ GB free |
| **CPU Cores** | 4 cores | 8+ cores (for parallel processing) |
| **Network** | Required for server access | High-speed for large file transfers |

### 2.3 Required Skills

- Basic command line navigation (Linux/Unix)
- Understanding of file paths and directory structures
- Basic knowledge of molecular biology and sequencing
- Familiarity with R or willingness to learn scripting basics

### 2.4 Input Data Requirements

- **Illumina paired-end FASTQ files** (`.fastq` or `.fastq.gz`)
- **Primer sequences** (forward and reverse)
- **Reference database** (FASTA format for BLAST)
- **Metadata** about samples (sample names, experimental design)

---

## 3. Software Installation & Setup

### 3.1 mothur Installation

#### Option 1: Precompiled Executable (Recommended)

1. Visit the [mothur GitHub releases page](https://github.com/mothur/mothur/releases)
2. Download the latest version for your platform:
   - **Mac**: Universal binary (Intel + ARM support)
   - **Windows**: Windows executable
   - **Linux**: Linux executable

3. Extract the archive:

```bash
# On Linux/Mac
unzip mothur_*.zip
cd mothur
```

4. Test the installation:

```bash
./mothur
# You should see the mothur welcome message
```

💡 **TIP**: Add mothur to your PATH for easier access, or work within the mothur directory.

#### Option 2: Compile from Source

See the [mothur installation wiki](https://mothur.org/wiki/installation/) for detailed compilation instructions.

### 3.2 BLAST+ Installation

#### Linux (Ubuntu/Debian)

```bash
# Download latest version
wget https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/ncbi-blast-*-x64-linux.tar.gz

# Extract
tar -xzvf ncbi-blast-*-x64-linux.tar.gz

# Add to PATH or navigate to bin directory
cd ncbi-blast-*/bin
```

#### Mac

```bash
# Download macOS version
curl -O https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/ncbi-blast-*-universal-macosx.tar.gz

# Extract and navigate
tar -xzf ncbi-blast-*-universal-macosx.tar.gz
cd ncbi-blast-*/bin
```

#### Windows

1. Download the Windows installer from [NCBI FTP](https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/)
2. Run the `.exe` installer
3. Follow the installation wizard
4. Verify installation by opening Command Prompt and typing `blastn -version`

### 3.3 R and Package Installation

#### Install R

Download and install R from [CRAN](https://cran.r-project.org/) for your platform.

#### Install Required Packages

Open R or RStudio and run:

```r
# Install tidyverse (includes dplyr and many other useful packages)
install.packages("tidyverse")

# Install xlsx package for Excel export
install.packages("xlsx")

# Verify installation
library(tidyverse)
library(xlsx)
```

✅ **CHECKPOINT**: All software should now be installed and accessible.

---

## 4. Data Preparation

### 4.1 Organizing Your Project Directory

Create a well-organized directory structure:

```
PROJECT/
├── raw_data/
│   ├── Sample_A_R1_001.fastq.gz
│   ├── Sample_A_R2_001.fastq.gz
│   ├── Sample_B_R1_001.fastq.gz
│   └── Sample_B_R2_001.fastq.gz
├── mothur_analysis/
│   ├── PROJECT.files
│   └── PROJECT.oligos
├── blast_analysis/
│   └── reference_database.fasta
├── r_analysis/
└── results/
```

💡 **TIP**: Use descriptive project names without spaces, dashes, or special characters. Underscores are acceptable.

### 4.2 Creating the `.files` File

The `.files` file tells mothur which FASTQ files belong to which samples.

#### Format

The file has **three tab-separated columns**:
1. **Sample name** (no spaces, dashes, or special characters; underscores OK)
2. **Forward read** (R1 file)
3. **Reverse read** (R2 file)

#### Example: `PROJECT.files`

```text
Sample_A	Sample_A_L001_R1_001.fastq	Sample_A_L001_R2_001.fastq
Sample_B	Sample_B_L001_R1_001.fastq	Sample_B_L001_R2_001.fastq
Sample_C	Sample_C_L001_R1_001.fastq	Sample_C_L001_R2_001.fastq
Negative_Control	NegCtrl_L001_R1_001.fastq	NegCtrl_L001_R2_001.fastq
```

#### Creating the File

**Windows (Notepad++):**
1. Open Notepad++
2. Type your sample information with TAB characters between columns
3. Save As → Set "Save as type" to "All Files (*.*)"
4. Name the file `PROJECT.files`

**Mac (TextEdit):**
1. Open TextEdit → Format → Make Plain Text
2. Type your sample information with TAB characters
3. Save as `PROJECT.files`

**Linux (nano/vim):**
```bash
nano PROJECT.files
# Type content, use TAB key between columns
# Save: Ctrl+O, Enter, Ctrl+X
```

⚠️ **CRITICAL**:
- Use TAB characters (not spaces) between columns
- Save with `.files` extension
- The filename prefix (e.g., "PROJECT") will be used for all output files

### 4.3 Creating the `.oligos` File

The `.oligos` file contains your PCR primer sequences for trimming.

#### Format

```text
forward	PRIMER_SEQUENCE_5_to_3
reverse	REVERSE_PRIMER_SEQUENCE_5_to_3
```

#### Example: `PROJECT.oligos`

```text
forward	GACCCTATGGAGCTTTAGAC
reverse	CGCTGTTATCCCTADRGTAACT
```

🔍 **NOTE**:
- Use degenerate base codes if needed (R, Y, M, K, S, W, etc.)
- The reverse primer should be in the orientation as it appears in your amplicon (mothur will handle reverse complement)
- If your reads don't extend to the reverse primer, comment it out with `#` to avoid losing sequences

#### Optional: Commenting Out Reverse Primer

If many sequences don't reach the reverse primer:

```text
forward	GACCCTATGGAGCTTTAGAC
# reverse	CGCTGTTATCCCTADRGTAACT
```

### 4.4 Uploading Data to Server

#### Using WinSCP (Windows)

1. Launch WinSCP
2. Enter server credentials and connect
3. Navigate to your data folder (left panel = local computer)
4. Navigate to destination on server (right panel = server)
5. Drag and drop files:
   - All FASTQ files
   - `PROJECT.files`
   - `PROJECT.oligos`

#### Using scp (Mac/Linux)

```bash
# Upload all FASTQ files
scp *.fastq username@server:/path/to/mothur/

# Upload configuration files
scp PROJECT.files PROJECT.oligos username@server:/path/to/mothur/
```

#### Using rsync (Mac/Linux)

```bash
# Sync entire directory
rsync -avz --progress raw_data/ username@server:/path/to/mothur/
```

✅ **CHECKPOINT**: All data and configuration files should be on the server in the mothur directory.

---

## 5. Mothur Analysis Pipeline

**Estimated time**: 2-6 hours depending on dataset size

### 5.1 Launching mothur

Connect to your server via SSH and navigate to the mothur directory:

```bash
# Navigate to mothur directory
cd /path/to/mothur

# Launch mothur
./mothur
```

**Expected output:**
```text
mothur v.1.48.0
Last updated: [date]
By Patrick D. Schloss
Department of Microbiology & Immunology
University of Michigan
http://www.mothur.org

When using, please cite:
Schloss, P.D., et al., Introducing mothur: Open-source, platform-independent,
community-supported software for describing and comparing microbial communities.
Appl Environ Microbiol, 2009. 75(23):7537-41.

mothur >
```

### 5.2 Step 1: Merging Paired-End Reads

The `make.contigs` command merges forward (R1) and reverse (R2) reads into contigs.

#### Command

```bash
make.contigs(file=PROJECT.files, trimoverlap=T, processors=64)
```

#### Parameters Explained

| Parameter | Description | When to Use |
|-----------|-------------|-------------|
| `file` | Your `.files` file | Always required |
| `trimoverlap` | Trim to overlapping region only | Set to `T` for 2x300bp reads longer than amplicon; `F` otherwise |
| `processors` | Number of CPU cores | Adjust based on available cores (check with `htop` or `top`) |

💡 **TIP**: Check your sequencing chemistry! For 2x150bp or 2x250bp where reads fully overlap, use `trimoverlap=F`. For 2x300bp where reads are longer than your amplicon, use `trimoverlap=T`.

#### Algorithm Details

mothur merges reads using quality-aware alignment:
1. Aligns forward and reverse complement reads
2. Where reads disagree:
   - **Gap vs. base**: Requires base quality score > 25
   - **Base vs. base**: Requires quality score difference > 6
   - If neither condition is met → assigns "N" (ambiguous base)

#### Expected Output

```text
Group count:
Sample_A    54051
Sample_B    51359
Sample_C    55079
Negative_Control    12

Output File Names:
PROJECT.trim.contigs.fasta
PROJECT.contigs.groups
PROJECT.contigs.qual
PROJECT.contigs.report
```

#### Output Files

- `PROJECT.trim.contigs.fasta`: All merged sequences
- `PROJECT.contigs.groups`: Links each sequence to its sample
- `PROJECT.contigs.qual`: Quality scores
- `PROJECT.contigs.report`: Alignment statistics

### 5.3 Step 2: Sequence Summary

Examine sequence length distribution and quality metrics.

#### Command

```bash
summary.seqs(fasta=PROJECT.trim.contigs.fasta)
```

#### Expected Output

```text
            Start    End    NBases    Ambigs    Polymer    NumSeqs
Minimum:    1        1      1         0         1          1
2.5%-tile:  1        35     35        0         4          90269
25%-tile:   1        205    205       0         4          902681
Median:     1        206    206       0         4          1805361
75%-tile:   1        240    240       0         5          2708041
97.5%-tile: 1        246    246       35        35         3520453
Maximum:    1        306    306       98        35         3610720
Mean:       1        206    206       1         5
# of Seqs:  3610720
```

#### Interpreting the Summary

| Column | Meaning | What to Look For |
|--------|---------|------------------|
| **Start** | Starting position | Should be 1 for all sequences |
| **End** | Length of sequence | Should match expected amplicon size ±50bp |
| **NBases** | Number of bases | Same as End |
| **Ambigs** | Ambiguous bases (N) | Lower is better; we'll remove these next |
| **Polymer** | Longest homopolymer | Long homopolymers (>8-10) may indicate sequencing errors |
| **NumSeqs** | Total sequences | Tracking dataset size |

🔍 **NOTE**: You can run `summary.seqs()` after every step to monitor data quality and quantity.

⚠️ **WARNING**: Sequences much longer than expected amplicon size may indicate non-specific amplification.

### 5.4 Step 3: Primer Trimming and Quality Filtering

Remove primers and filter for quality.

#### Command

```bash
trim.seqs(fasta=PROJECT.trim.contigs.fasta, oligos=PROJECT.oligos, pdiffs=2, maxambig=0, minlength=100, maxlength=300)
```

#### Parameters Explained

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `fasta` | `PROJECT.trim.contigs.fasta` | Input sequences from make.contigs |
| `oligos` | `PROJECT.oligos` | Primer sequences to trim |
| `pdiffs` | `2` | Allow 2 mismatches in primers (adjust for primer quality) |
| `maxambig` | `0` | Remove any sequence with ambiguous bases |
| `minlength` | `100` | Minimum sequence length (adjust for your amplicon) |
| `maxlength` | `300` | Maximum sequence length (adjust for your amplicon) |

#### Adjusting Parameters for Your Data

**`pdiffs` (primer differences):**
- High-quality primers: `pdiffs=1`
- Standard primers: `pdiffs=2` (recommended)
- Lower-quality primers: `pdiffs=3`

Test on a subset and manually check if primers are being removed properly.

**`maxambig`:**
- Strict quality: `maxambig=0` (recommended)
- More permissive: `maxambig=1-2` (retains more sequences but lower quality)

**`minlength` and `maxlength`:**
Set based on your expected amplicon size:
- 12S MiFish: ~150-200bp
- COI: ~300-650bp
- 16S V4: ~250-280bp

#### Expected Output

```text
Output File Names:
PROJECT.trim.contigs.trim.fasta
PROJECT.trim.contigs.scrap.fasta
PROJECT.trim.contigs.trim.groups
```

✅ **CHECKPOINT**: Check sequence summary to confirm quality improvement.

#### Verify Results

```bash
summary.seqs(fasta=PROJECT.trim.contigs.trim.fasta)
```

```text
            Start    End    NBases    Ambigs    Polymer    NumSeqs
Minimum:    1        100    100       0         3          1
2.5%-tile:  1        179    179       0         5          27976
25%-tile:   1        198    198       0         5          279759
Median:     1        198    198       0         5          559517
75%-tile:   1        203    203       0         5          839275
97.5%-tile: 1        205    205       0         6          1091058
Maximum:    1        256    256       0         16         1119033
Mean:       1        198    198       0         5
# of Seqs:  1119033
```

**Key improvements:**
- ✅ No ambiguous bases (Ambigs = 0)
- ✅ Sequence lengths within expected range
- ✅ Reduction in total sequences is normal (quality > quantity)

🔍 **NOTE**: It's normal to lose 30-70% of sequences during quality filtering. A high-quality dataset is more valuable than a large, noisy dataset.

### 5.5 Step 4: Identifying Unique Sequences

Remove duplicate sequences while tracking abundance.

#### Command

```bash
unique.seqs(fasta=PROJECT.trim.contigs.trim.fasta)
```

#### What This Does

- Identifies sequences with identical nucleotide composition
- Keeps ONE copy of each unique sequence
- Creates a `.names` file linking duplicates to their representative sequence

#### Example

If sequences X, Y, and Z are identical:
- **FASTA file**: Contains only sequence X
- **Names file**: Lists X, Y, Z all linked to X

#### Expected Output

```text
Output File Names:
PROJECT.trim.contigs.trim.unique.fasta
PROJECT.trim.contigs.trim.names
```

#### Verify Results

```bash
summary.seqs(fasta=PROJECT.trim.contigs.trim.unique.fasta, name=PROJECT.trim.contigs.trim.names)
```

```text
            Start    End    NBases    Ambigs    Polymer    NumSeqs
Minimum:    1        100    100       0         3          1
2.5%-tile:  1        179    179       0         5          27976
25%-tile:   1        198    198       0         5          279759
Median:     1        198    198       0         5          559517
75%-tile:   1        203    203       0         5          839275
97.5%-tile: 1        205    205       0         6          1091058
Maximum:    1        256    256       0         16         1119033
Mean:       1        198    198       0         5
# of unique seqs:    24695
total # of seqs:     1119033
```

**Key observation:**
- 1,119,033 total sequences collapsed into 24,695 unique sequences
- ~95% reduction is typical for high-depth sequencing

### 5.6 Step 5: Creating Count Table

Combine names and groups into a single count table.

#### Command

```bash
count.seqs(name=PROJECT.trim.contigs.trim.names, group=PROJECT.contigs.groups, compress=f)
```

#### What This Does

Creates a table where:
- **Rows** = unique sequences
- **Columns** = samples
- **Values** = count of each unique sequence in each sample

#### Expected Output

```text
Output File Names:
PROJECT.trim.contigs.trim.count_table
```

#### Verify Results

```bash
summary.seqs(count=PROJECT.trim.contigs.trim.count_table)
```

You should see identical results to the previous summary with the `.names` file.

💡 **TIP**: The count table format is more efficient and easier to work with in downstream analyses.

### 5.7 Step 6: Chimera Removal

Identify and remove chimeric sequences using VSEARCH.

#### What are Chimeras?

Chimeras are artificial sequences formed when PCR amplification joins fragments from different templates. They must be removed to avoid false diversity estimates.

#### Command

```bash
chimera.vsearch(fasta=PROJECT.trim.contigs.trim.unique.fasta, count=PROJECT.trim.contigs.trim.count_table, dereplicate=t)
```

#### Parameters Explained

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `fasta` | Unique sequences | Input from unique.seqs |
| `count` | Count table | Sample abundances |
| `dereplicate` | `t` | Only remove chimeras from samples where they're flagged |

#### dereplicate=t vs. dereplicate=f

- **`dereplicate=t`** (recommended): A sequence flagged as chimeric in one sample is only removed from that sample. If it's the most abundant sequence in another sample, it's kept there.
- **`dereplicate=f`**: If flagged as chimeric anywhere, remove from all samples (more aggressive)

#### Expected Output

```text
[VSEARCH output - lots of processing text]

Output File Names:
PROJECT.trim.contigs.trim.unique.denovo.vsearch.chimeras
PROJECT.trim.contigs.trim.unique.denovo.vsearch.accnos
PROJECT.trim.contigs.trim.denovo.vsearch.pick.count_table
```

🔍 **NOTE**: Don't be alarmed by the verbose VSEARCH output. Wait for mothur to return to the command prompt.

#### Remove Chimeras from FASTA

```bash
remove.seqs(fasta=PROJECT.trim.contigs.trim.unique.fasta, accnos=PROJECT.trim.contigs.trim.unique.denovo.vsearch.accnos)
```

#### Expected Output

```text
Output File Names:
PROJECT.trim.contigs.trim.unique.pick.fasta
```

✅ **CHECKPOINT**: Verify chimera removal results.

#### Verify Results

```bash
summary.seqs(fasta=PROJECT.trim.contigs.trim.unique.pick.fasta, count=PROJECT.trim.contigs.trim.denovo.vsearch.pick.count_table)
```

```text
            Start    End    NBases    Ambigs    Polymer    NumSeqs
Minimum:    1        100    100       0         3          1
2.5%-tile:  1        179    179       0         5          27659
25%-tile:   1        198    198       0         5          276581
Median:     1        198    198       0         5          553162
75%-tile:   1        203    203       0         5          829742
97.5%-tile: 1        205    205       0         6          1078664
Maximum:    1        256    256       0         16         1106322
Mean:       1        198    198       0         5
# of unique seqs:    21701
total # of seqs:     1106322
```

**Key observations:**
- Unique sequences: 24,695 → 21,701 (removed ~3,000 chimeras)
- Total sequences: 1,119,033 → 1,106,322 (lost ~1% of sequences)

### 5.8 Step 7: Sample Sequence Counts

Check how many sequences remain in each sample.

#### Command

```bash
count.groups(count=PROJECT.trim.contigs.trim.denovo.vsearch.pick.count_table)
```

#### Expected Output

```text
Sample_A contains 478651 sequences.
Sample_B contains 405339 sequences.
Sample_C contains 221520 sequences.
Negative_Control contains 812 sequences.

Total sequences: 1106322
```

#### Interpreting Results

- **Environmental samples**: Should have thousands to hundreds of thousands of sequences
- **Negative controls**: Should have very few sequences (<1000 ideally)
- **Failed samples**: <10,000 sequences may indicate PCR or sequencing failure

⚠️ **WARNING**: High sequence counts in negative controls may indicate contamination.

### 5.9 Step 8: Exit mothur

Your curated sequence file is ready for BLAST analysis.

```bash
quit
```

📁 **Key output file for BLAST**: `PROJECT.trim.contigs.trim.unique.pick.fasta`

✅ **CHECKPOINT**: mothur analysis complete. You now have high-quality, chimera-free sequences ready for taxonomic assignment.

---

## 6. BLAST Taxonomic Assignment

**Estimated time**: 1-4 hours depending on dataset size and database

### 6.1 Preparing for BLAST Analysis

#### Navigate to BLAST Directory

```bash
cd ..
cd BLAST
cd ncbi-blast-2.17.0+
```

🔍 **NOTE**: Directory name may vary based on BLAST version. Use `ls` to confirm.

#### Transfer Files to BLAST Directory

Using WinSCP, FileZilla, or command line, transfer:
1. **Query sequences**: `PROJECT.trim.contigs.trim.unique.pick.fasta` (from mothur directory)
2. **Reference database**: Your custom reference database FASTA file

💡 **TIP**: Reference databases should contain sequences from your target taxonomic group with proper formatting:
```
>SequenceID:Species_name:Family:Order:etc
ATCGATCGATCG...
```

### 6.2 Creating BLAST Database

#### Create Database Directory (First Time Only)

```bash
mkdir -p blast/db
```

#### Format Reference Database

```bash
makeblastdb -in reference_database.fasta -out reference_db -dbtype nucl
```

#### Parameters Explained

| Parameter | Value | Description |
|-----------|-------|-------------|
| `-in` | `reference_database.fasta` | Your reference FASTA file |
| `-out` | `reference_db` | Database name (no extension needed) |
| `-dbtype` | `nucl` | Nucleotide database |

#### Expected Output

You should see three new files created:
- `reference_db.nsq`
- `reference_db.nin`
- `reference_db.nhr`

✅ **CHECKPOINT**: BLAST database files created successfully.

### 6.3 Running BLAST Search

#### Command

```bash
blastn -query PROJECT.trim.contigs.trim.unique.pick.fasta -db reference_db -out PROJECT_blast_results.txt -outfmt 6 -max_target_seqs 1
```

#### Parameters Explained

| Parameter | Value | Description |
|-----------|-------|-------------|
| `-query` | Your FASTA file | Sequences to search |
| `-db` | `reference_db` | Database created in previous step |
| `-out` | Output filename | Results file |
| `-outfmt` | `6` | Tabular output (recommended) |
| `-max_target_seqs` | `1` | Return only top hit per query |

#### Alternative Parameters

**Return multiple hits per query:**
```bash
blastn -query PROJECT.trim.contigs.trim.unique.pick.fasta -db reference_db -out PROJECT_blast_results.txt -outfmt 6 -max_target_seqs 5
```

💡 **TIP**: Use `-max_target_seqs 1` for initial analysis. If you get unexpected results (e.g., elephant sequences from seawater), re-run with `-max_target_seqs 5-10` for those specific sequences to investigate further.

#### Expected Runtime

- Small datasets (<10,000 sequences): 5-30 minutes
- Medium datasets (10,000-50,000): 30 minutes - 2 hours
- Large datasets (>50,000): 2-4+ hours

### 6.4 Understanding BLAST Output

The output is a tab-delimited file with 12 columns:

| Column | Name | Description |
|--------|------|-------------|
| 1 | qseqid | Query sequence ID (your sequence) |
| 2 | sseqid | Subject sequence ID (reference database match) |
| 3 | pident | Percentage identity |
| 4 | length | Alignment length |
| 5 | mismatch | Number of mismatches |
| 6 | gapopen | Number of gap openings |
| 7 | qstart | Start position in query |
| 8 | qend | End position in query |
| 9 | sstart | Start position in subject |
| 10 | send | End position in subject |
| 11 | evalue | E-value (expected number of hits by chance) |
| 12 | bitscore | Bit score (higher is better) |

#### Key Columns for Taxonomic Assignment

- **Column 1 (qseqid)**: Your sequence name (links back to count table)
- **Column 2 (sseqid)**: Best match in database (contains taxonomic information)
- **Column 3 (pident)**: Percentage identity (use for confidence thresholds)
- **Column 4 (length)**: Should be close to your amplicon length

#### Download Results

Using WinSCP or scp, download `PROJECT_blast_results.txt` to your local computer.

---

## 7. R Data Integration

**Estimated time**: 30 minutes - 1 hour

### 7.1 BLASTMatch Processing

The BLASTMatch Java tool processes raw BLAST results and applies identity thresholds.

#### Identity Thresholds

- **≥99% identity**: Species-level assignment
- **95-98.9% identity**: Family-level assignment
- **<95% identity**: Marked as "NO MATCH"

#### Running BLASTMatch

**On your local computer (Windows):**

1. Right-click on the folder containing `BLASTMatch.jar` → Properties
2. Copy the file path
3. Open Command Prompt
4. Navigate to the folder:

```bash
cd C:\path\to\BLASTMatch
```

5. Run BLASTMatch:

```bash
java -jar BLASTMatch.jar PROJECT_blast_results.txt
```

**On Mac/Linux:**

```bash
cd /path/to/BLASTMatch
java -jar BLASTMatch.jar PROJECT_blast_results.txt
```

#### Expected Output

Two files will be created:
- `outfile` - Classifications based on identity thresholds
- `checkfile` - Quality control information

### 7.2 Curating BLASTMatch Output

#### Open outfile in Excel/Spreadsheet

1. Open `outfile` in Excel or LibreOffice Calc
2. Review classifications for accuracy
3. Manually curate ambiguous results if needed

#### Add Header Row

Insert a new row at the top and add the header:
```
results
```

⚠️ **CRITICAL**: Keep the text formatting exactly as it appears. Do not add extra columns or change delimiters.

#### Save as CSV

File → Save As → Select "CSV (Comma delimited) (*.csv)"

Name it: `PROJECT_BLAST_curated.csv`

### 7.3 R Script for Data Integration

#### Setup

1. Create a new folder for R analysis
2. Place these files in the folder:
   - `PROJECT_BLAST_curated.csv` (from BLASTMatch)
   - `PROJECT.trim.contigs.trim.denovo.vsearch.pick.count_table` (from mothur)

3. Open RStudio
4. File → New Project → Existing Directory → Select your R analysis folder

#### R Script

Create a new R script (`analysis.R`) with the following code:

```r
# ============================================
# eDNA Data Integration Script
# ============================================

# Install packages (first time only)
# Uncomment these lines if packages are not installed
# install.packages("tidyverse")
# install.packages("xlsx")

# Load packages
library(tidyverse)
library(dplyr)
library(xlsx)

# ============================================
# 1. Load and Process BLAST Results
# ============================================

# Read BLAST output file
curateClassification <- read_tsv("PROJECT_BLAST_curated.csv")

# View data structure
View(curateClassification)
dim(curateClassification)
head(curateClassification)

# Split the results column to separate sequence ID from taxonomic classification
# The separator is typically a colon (:) or period (.)
curateDATA <- curateClassification %>%
  separate(results, c("Query", "match"), sep = "([.?:])")

# Verify split
head(curateDATA)
View(curateDATA)
dim(curateDATA)

# ============================================
# 2. Load mothur Count Table
# ============================================

# Read count table (adjust filename to match yours)
CountTable <- read_tsv("PROJECT.trim.contigs.trim.denovo.vsearch.pick.count_table")

# Rename first column to match BLAST data
CountTable <- rename(CountTable, "Query" = "Representative_Sequence")

# View count table structure
head(CountTable)
View(CountTable)

# ============================================
# 3. Merge BLAST and Count Data
# ============================================

# Join BLAST classifications with sequence counts
MergedDATA <- left_join(curateDATA, CountTable, by = c("Query"))

# Verify merge
head(MergedDATA)
View(MergedDATA)

# ============================================
# 4. Create Final Summary Table
# ============================================

# Remove Query and total columns (keep only match and sample counts)
MyData <- select(MergedDATA, -Query, -total)
View(MyData)

# Group by taxonomic classification and sum counts
MyData2 <- MyData %>%
  group_by(match) %>%
  summarise(across(everything(), sum))

View(MyData2)

# ============================================
# 5. Export Results
# ============================================

# Export to Excel
write.xlsx(MyData2,
           file = "PROJECT_final_results.xlsx",
           sheetName = "Species_Counts",
           col.names = TRUE,
           row.names = FALSE,
           append = FALSE)

# Also export as CSV for compatibility
write.csv(MyData2,
          file = "PROJECT_final_results.csv",
          row.names = FALSE)

# ============================================
# 6. Basic Summary Statistics
# ============================================

# Total unique taxa detected
cat("Total unique taxa detected:", nrow(MyData2), "\n")

# Total sequences assigned
total_seqs <- MyData2 %>%
  select(-match) %>%
  summarise(across(everything(), sum))
print(total_seqs)

# Top 10 most abundant taxa
top10 <- MyData2 %>%
  mutate(total = rowSums(select(., -match))) %>%
  arrange(desc(total)) %>%
  select(match, total) %>%
  head(10)

print("Top 10 most abundant taxa:")
print(top10)
```

#### Running the Script

1. Update filenames in the script to match your actual files
2. Run line by line (Ctrl+Enter or Cmd+Enter) or source the entire script
3. Check output files: `PROJECT_final_results.xlsx` and `PROJECT_final_results.csv`

✅ **CHECKPOINT**: Final results table created with species/taxa and counts per sample.

---

## 8. Results Interpretation

### 8.1 Understanding Your Results Table

Your final results table (`PROJECT_final_results.xlsx`) contains:

| Column | Description |
|--------|-------------|
| **match** | Taxonomic classification (Species, Family, or NO MATCH) |
| **Sample_A, Sample_B, etc.** | Read counts for each sample |

### 8.2 Quality Metrics to Check

#### 1. Negative Control Sequences

- **Good**: <100 sequences total, mostly "NO MATCH"
- **Concerning**: >1000 sequences, real species detected
- **Action**: If contamination is high, consider re-running PCR or adjusting filtering

#### 2. Species Detection Rates

- **Species-level (≥99%)**: Gold standard for identification
- **Family-level (95-98.9%)**: Indicates genus/species not in database
- **NO MATCH (<95%)**: Novel sequences or database gaps

#### 3. Read Depth per Sample

- **Excellent**: >50,000 sequences
- **Good**: 10,000-50,000 sequences
- **Poor**: <10,000 sequences (may need re-sequencing)

#### 4. Diversity Patterns

- High-quality samples should show taxonomically coherent communities
- Unexpected taxa (e.g., terrestrial species in marine samples) warrant investigation

### 8.3 Common Result Patterns

#### Pattern 1: High NO MATCH Rate (>30%)

**Possible causes:**
- Incomplete reference database
- Novel/uncharacterized biodiversity
- Poor primer specificity (amplifying non-target DNA)

**Action**:
- Expand reference database
- BLAST a subset against NCBI GenBank to identify potential matches
- Review primer design

#### Pattern 2: Dominant Single Species

**Possible causes:**
- Genuine biological dominance
- Primer bias favoring that species
- Contamination

**Action**:
- Cross-reference with ecological expectations
- Check negative controls for same species
- Review field collection protocols

#### Pattern 3: Negative Control Contamination

**Possible causes:**
- Lab contamination
- Index hopping (Illumina sequencing artifact)
- Reagent contamination

**Action**:
- Remove taxa found in negative controls from all samples (if counts are similar)
- Re-run samples with fresh reagents
- Implement stricter lab practices

---

## 9. Troubleshooting

### 9.1 mothur Issues

#### Error: "Unable to open file PROJECT.files"

**Cause**: File path incorrect or file doesn't exist

**Solution**:
```bash
# Check if file exists
ls -l PROJECT.files

# Check current directory
pwd

# Make sure you're in the correct directory
cd /path/to/mothur
```

---

#### Error: "Your file contains only Ns"

**Cause**: Quality filtering removed all bases, or file encoding issue

**Solution**:
- Check your FASTQ files are not corrupted
- Reduce stringency: `maxambig=1` instead of `maxambig=0`
- Verify `pdiffs` setting isn't too strict

---

#### Error: "Sequence X is too long/short"

**Cause**: Sequences outside `minlength`/`maxlength` range

**Solution**:
- Review sequence length distribution from `summary.seqs()`
- Adjust `minlength` and `maxlength` parameters appropriately

---

#### Very Low Sequence Retention (<10%)

**Cause**: Overly strict filtering or poor sequencing quality

**Solution**:
1. Run `summary.seqs()` after each step to identify where sequences are lost
2. Relax parameters incrementally:
   - `pdiffs=3`
   - `maxambig=1`
   - Widen `minlength`/`maxlength` range
3. Check raw FASTQ quality scores (use FastQC)

---

### 9.2 BLAST Issues

#### Error: "BLAST database not found"

**Cause**: Database path incorrect or not formatted

**Solution**:
```bash
# Check database files exist
ls -l reference_db.*

# Should see .nsq, .nin, .nhr files

# If missing, rerun makeblastdb
makeblastdb -in reference_database.fasta -out reference_db -dbtype nucl
```

---

#### Error: "Out of memory"

**Cause**: Database or query too large for available RAM

**Solution**:
- Split query FASTA into smaller chunks
- Use a machine with more RAM
- Reduce `-max_target_seqs` parameter

---

#### BLAST Taking Too Long (>8 hours)

**Cause**: Large dataset or database

**Solution**:
```bash
# Use more threads (if available)
blastn -query input.fasta -db reference_db -out results.txt -outfmt 6 -max_target_seqs 1 -num_threads 16

# Reduce search space (if appropriate)
blastn -query input.fasta -db reference_db -out results.txt -outfmt 6 -max_target_seqs 1 -task blastn-short
```

---

### 9.3 R Integration Issues

#### Error: "Could not find file"

**Cause**: File path incorrect or working directory not set

**Solution**:
```r
# Check current working directory
getwd()

# Set working directory to project folder
setwd("/path/to/project/folder")

# Or use RStudio Projects (File → New Project)
```

---

#### Error: Package installation failed

**Cause**: Internet connection, package dependencies, or permissions

**Solution**:
```r
# Try installing from different repository
install.packages("tidyverse", repos = "https://cloud.r-project.org/")

# Install dependencies separately
install.packages("dplyr")
install.packages("readr")
install.packages("tidyr")

# For xlsx package issues, try openxlsx instead
install.packages("openxlsx")
library(openxlsx)
write.xlsx(MyData2, file = "results.xlsx")
```

---

#### Error: "Column 'Query' not found"

**Cause**: Column naming mismatch between BLAST and count table

**Solution**:
```r
# Check column names
colnames(curateDATA)
colnames(CountTable)

# Adjust rename() to match your actual column name
CountTable <- rename(CountTable, "Query" = "Representative_Sequence")
```

---

### 9.4 Performance Optimization

#### Speed Up mothur Processing

```bash
# Use maximum available processors
make.contigs(file=PROJECT.files, processors=64)

# Check available cores
nproc  # on Linux
sysctl -n hw.ncpu  # on Mac
```

#### Speed Up BLAST Processing

```bash
# Use multiple threads
blastn -query input.fasta -db ref_db -out results.txt -outfmt 6 -num_threads 32

# Pre-filter sequences by length (if many are too short/long)
# Use seqkit or mothur to filter before BLAST
```

#### Reduce Memory Usage

```bash
# In mothur: reduce processor count if memory limited
processors=4

# In BLAST: reduce max_target_seqs
-max_target_seqs 1
```

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **Amplicon** | DNA fragment amplified by PCR using specific primers |
| **ASV (Amplicon Sequence Variant)** | Unique sequence recovered from high-throughput sequencing |
| **BLAST** | Basic Local Alignment Search Tool; finds similar sequences in databases |
| **Chimera** | Artificial sequence formed from multiple parent sequences during PCR |
| **Contig** | Continuous sequence formed by merging overlapping reads |
| **Count table** | Matrix showing abundance of each unique sequence in each sample |
| **Coverage** | Proportion of a sequence that aligns to another sequence |
| **eDNA (Environmental DNA)** | DNA extracted from environmental samples (water, soil) without isolating organisms |
| **FASTA** | Text file format for nucleotide or protein sequences |
| **FASTQ** | Text file format for sequences with quality scores |
| **Homopolymer** | Stretch of identical nucleotides (e.g., AAAAA) |
| **Identity (%)** | Percentage of aligned bases that are identical |
| **Metabarcoding** | High-throughput sequencing of taxonomic markers to identify species in mixed samples |
| **mothur** | Software for analyzing microbial community sequencing data |
| **OTU (Operational Taxonomic Unit)** | Group of similar sequences, often used as proxy for species |
| **Paired-end sequencing** | Sequencing both ends of DNA fragments (forward R1 and reverse R2) |
| **Phred score** | Measure of base calling accuracy (Q30 = 99.9% accuracy) |
| **Primer** | Short DNA sequence used to initiate PCR amplification |
| **Quality score** | Measure of confidence in base calling |
| **Read** | A single sequencing output (R1 = forward, R2 = reverse) |
| **Reference database** | Collection of known sequences used for taxonomic assignment |
| **Trimming** | Removing low-quality bases or adapter sequences |
| **VSEARCH** | Open-source tool for sequence analysis, including chimera detection |

---

## 11. References & Citations

### Primary Citations

1. **mothur**: Schloss, P.D., et al. (2009). Introducing mothur: Open-source, platform-independent, community-supported software for describing and comparing microbial communities. *Applied and Environmental Microbiology*, 75(23):7537-7541.
   - Website: https://mothur.org

2. **BLAST**: Camacho, C., et al. (2009). BLAST+: architecture and applications. *BMC Bioinformatics*, 10:421.
   - Website: https://blast.ncbi.nlm.nih.gov

3. **VSEARCH**: Rognes, T., et al. (2016). VSEARCH: a versatile open source tool for metagenomics. *PeerJ*, 4:e2584.
   - GitHub: https://github.com/torognes/vsearch

### R Packages

4. **tidyverse**: Wickham, H., et al. (2019). Welcome to the tidyverse. *Journal of Open Source Software*, 4(43):1686.
   - Website: https://www.tidyverse.org

5. **dplyr**: Wickham, H., et al. (2023). dplyr: A Grammar of Data Manipulation. R package.
   - Website: https://dplyr.tidyverse.org

### Additional Reading

6. **eDNA Metabarcoding Reviews**:
   - Taberlet, P., et al. (2012). Towards next-generation biodiversity assessment using DNA metabarcoding. *Molecular Ecology*, 21(8):2045-2050.
   - Deiner, K., et al. (2017). Environmental DNA metabarcoding: Transforming how we survey animal and plant communities. *Molecular Ecology*, 26(21):5872-5895.

7. **Bioinformatics Best Practices**:
   - Callahan, B.J., et al. (2016). DADA2: High-resolution sample inference from Illumina amplicon data. *Nature Methods*, 13(7):581-583.
   - Edgar, R.C. (2016). UNOISE2: improved error-correction for Illumina 16S and ITS amplicon sequencing. *bioRxiv*.

### Online Resources

- mothur MiSeq SOP: https://mothur.org/wiki/miseq_sop/
- BLAST Command Line User Manual: https://www.ncbi.nlm.nih.gov/books/NBK279690/
- R for Data Science (free online book): https://r4ds.had.co.nz/

---

## 12. Appendices

### Appendix A: Parameter Quick Reference

#### mothur make.contigs Parameters

| Parameter | Default | Recommended Range | Notes |
|-----------|---------|-------------------|-------|
| `trimoverlap` | F | F or T | T for 2x300bp reads > amplicon |
| `processors` | 1 | 8-64 | Match available CPU cores |

#### mothur trim.seqs Parameters

| Parameter | Default | Recommended Range | Notes |
|-----------|---------|-------------------|-------|
| `pdiffs` | 0 | 1-3 | 2 is standard; adjust for primer quality |
| `maxambig` | - | 0-2 | 0 for strict quality |
| `minlength` | - | 80-150 | Set based on amplicon |
| `maxlength` | - | 200-400 | Set based on amplicon |

#### BLAST Parameters

| Parameter | Default | Recommended | Notes |
|-----------|---------|-------------|-------|
| `-outfmt` | 0 | 6 | Tabular is easiest to parse |
| `-max_target_seqs` | 500 | 1-5 | 1 for initial analysis |
| `-num_threads` | 1 | 8-32 | Match available CPU cores |
| `-evalue` | 10 | 1e-5 | Stricter = fewer false positives |

### Appendix B: Recommended Directory Structure

```
PROJECT_NAME/
│
├── 00_raw_data/
│   ├── Sample_A_R1_001.fastq.gz
│   ├── Sample_A_R2_001.fastq.gz
│   └── ...
│
├── 01_mothur_analysis/
│   ├── PROJECT.files
│   ├── PROJECT.oligos
│   └── [mothur output files]
│
├── 02_blast_analysis/
│   ├── reference_database.fasta
│   ├── reference_db.n* [database files]
│   └── PROJECT_blast_results.txt
│
├── 03_r_analysis/
│   ├── analysis.R
│   ├── PROJECT_BLAST_curated.csv
│   ├── PROJECT.count_table
│   └── PROJECT_final_results.xlsx
│
├── 04_results/
│   ├── final_species_table.xlsx
│   ├── figures/
│   └── reports/
│
└── README.md [project documentation]
```

### Appendix C: File Naming Conventions

**Best Practices:**
- ✅ Use descriptive, consistent names
- ✅ Include dates in YYYY-MM-DD format
- ✅ Use underscores (not spaces or hyphens)
- ✅ Keep names concise but informative

**Examples:**
```
Good:
  2026-01-13_PROJECT_mothur_analysis
  Sample_A_R1_001.fastq
  reference_database_v2.fasta

Avoid:
  My Analysis-Jan 13 (spaces and ambiguous date)
  sampleA.R1.fastq (inconsistent separators)
  refdb.fasta (not descriptive enough)
```

### Appendix D: Data Backup Recommendations

1. **Raw Data**: Keep original FASTQ files in multiple locations (server + external drive)
2. **Intermediate Files**: Keep key mothur outputs (final .fasta, .count_table)
3. **Reference Databases**: Version control; document sources and dates
4. **Scripts**: Use Git for version control of R scripts and analysis code
5. **Final Results**: Export to multiple formats (Excel, CSV, PDF reports)

**Backup Schedule:**
- Daily: Active analysis files to server backup
- Weekly: Copy to external drive
- Monthly: Archive completed projects

### Appendix E: Version History

#### Version 3.0 (January 2026)
- Complete rewrite in Markdown format
- Updated software versions (BLAST+ 2.17.0, current R packages)
- Added comprehensive troubleshooting section
- Added glossary and expanded references
- Included cross-platform instructions
- Added quality checkpoints throughout
- Created visual enhancements (callout boxes, tables)
- Standardized placeholder naming (PROJECT)
- Fixed all character encoding issues
- Removed unfinished editor comments

#### Version 2.0 (October 2021)
- Original plain text version
- mothur v1.44.3, BLAST 2.12.0+
- Basic pipeline with Windows-specific instructions

### Appendix F: Workflow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    eDNA ANALYSIS WORKFLOW                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  Raw FASTQ Data │
│  (R1 & R2 files)│
└────────┬────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│                  MOTHUR ANALYSIS PIPELINE                  │
├────────────────────────────────────────────────────────────┤
│  1. make.contigs()     → Merge paired-end reads            │
│  2. summary.seqs()     → Check sequence quality            │
│  3. trim.seqs()        → Remove primers & filter quality   │
│  4. unique.seqs()      → Identify unique sequences         │
│  5. count.seqs()       → Create count table                │
│  6. chimera.vsearch()  → Detect chimeras                   │
│  7. remove.seqs()      → Remove chimeric sequences         │
│  8. count.groups()     → Verify sample counts              │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────┐
│  Curated FASTA     │
│  + Count Table     │
└────────┬───────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│                  BLAST TAXONOMIC ASSIGNMENT                │
├────────────────────────────────────────────────────────────┤
│  1. makeblastdb      → Format reference database           │
│  2. blastn           → Search sequences vs. database       │
│  3. BLASTMatch       → Apply identity thresholds           │
│     • ≥99%  = Species level                               │
│     • 95-99% = Family level                               │
│     • <95%   = NO MATCH                                   │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────┐
│  BLAST Results     │
│  (taxonomic IDs)   │
└────────┬───────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│                    R DATA INTEGRATION                      │
├────────────────────────────────────────────────────────────┤
│  1. Load BLAST results + Count table                       │
│  2. Merge by sequence ID                                   │
│  3. Group by taxon, sum counts                             │
│  4. Export final species × samples table                   │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────┐
│  Final Results     │
│  Excel/CSV Table   │
│  (Taxa × Samples)  │
└────────────────────┘
```

---

## Document End

For questions, issues, or contributions to this SOP, please contact the original authors or maintain a version-controlled copy in your lab's documentation repository.

**Last updated**: January 13, 2026
**Pipeline tested on**: mothur v1.48+, BLAST+ v2.17.0, R 4.3+
