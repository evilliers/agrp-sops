# eDNA Reference Database Creation Pipeline

**SOP ID:** EDNA-005
**Version:** 2.0
**Date:** October 2025
**Author:** AGRP
**CRABS Version:** 1.12.0+

---

## Overview

This pipeline creates curated multi-marker reference databases for eDNA metabarcoding analysis of South African marine species in the Southern Atlantic and Indian Ocean. It uses [CRABS](https://github.com/gjeunen/reference_database_creator) (Creating Reference databases for Amplicon-Based Sequencing) to integrate sequences from four databases:

| Database | Sequences | Strength |
|----------|-----------|----------|
| **BOLD** | COI-5P barcodes | Gold standard for COI; curated, standardised metadata |
| **NCBI** | 12S, 16S, COI | Broadest taxonomic coverage including regional species |
| **EMBL** | Vertebrate mitochondrial | European sequence data |
| **MitoFish** | Fish mitochondrial genomes | Complete fish mitogenomes |

### Target Taxa

- **Fishes:** Actinopterygii, Chondrichthyes, Sarcopterygii (inc. *Latimeria chalumnae*), Myxini, Petromyzontida
- **Marine Mammals:** Cetacea, Pinnipedia (inc. Cape fur seal *Arctocephalus pusillus*)

### Target Markers

| Marker | Target | Amplicon | Purpose |
|--------|--------|----------|---------|
| MiFish-U (12S rRNA) | Fish, Cetaceans | 160–180 bp | Universal fish detection |
| MarVer1 (12S rRNA) | Cetaceans | ~202 bp | Marine mammal specific |
| MarVer3 (16S rRNA) | Pinnipeds, Sirenians | ~245 bp | Marine mammal specific |
| STAT (16S rRNA) | Cetaceans, marine mammals | 70–80 bp | Marine mammal specific |
| Leray-XT (COI) | Fish, Cetaceans, Invertebrates | ~313 bp | Species-level ID |

### Primer Sequences

| Marker | Direction | Sequence |
|--------|-----------|----------|
| MiFish-U | Forward | `GTCGGTAAAACTCGTGCCAGC` |
| MiFish-U | Reverse | `CATAGTGGGGTATCTAATCCCAGTTTG` |
| MarVer1 | Forward | `CGTGCCAGCCACCGCG` |
| MarVer1 | Reverse | `GGGTATCTAATCCYAGTTTG` |
| MarVer3 | Forward | `AGACGAGAAGACCCTRTG` |
| MarVer3 | Reverse | `GGATTGCGCTGTTATCCC` |
| STAT | Forward | `TTAGACACTTTGGGAGGCTG` |
| STAT | Reverse | `CTGTGTAGCCGAAGGTAGAC` |
| Leray-XT | Forward | `GGWACWRGWTGRACWITITAYCCYCC` |
| Leray-XT | Reverse | `TAIACYTCIGGRTGICCRAARAAYCA` |

---

## Workflow Summary

```
Phase 1: Setup & Taxonomy Files
Phase 2: Download sequences (BOLD, NCBI, EMBL, MitoFish)
Phase 3: Import & standardise all sequences
    ↓
Workflow A: MiFish-U (12S)   → Merge → PCR → Filter → Derep → Subset → Export
Workflow B: MarVer (16S/12S) → Merge → PCR → Filter → Derep → Subset → Export
Workflow C: Leray-XT (COI)   → Merge → PCR → Filter → Derep → Subset → Export
    ↓
Phase 7: Create BLAST databases
Phase 8: Visualisation & QC
Phase 9: Downstream analysis (BLAST / DADA2 / QIIME2)
```

---

## Prerequisites

### Software

```bash
conda --version       # Conda must be installed
conda activate crabs
crabs --version       # Requires CRABS 1.12.0+
```

### Species List (`merged_sa_species.txt`)

The pipeline filters databases to South African marine species. This file must be prepared before starting:

```bash
# Convert FishBase download to pipeline format
bash prepare_species_lists.sh --convert-fish

# Merge fish and cetacean lists
bash prepare_species_lists.sh --merge

# Verify output (~2,087 species)
wc -l species_lists/merged_sa_species.txt
head -5 species_lists/merged_sa_species.txt
```

Species are downloaded from [FishBase](https://fishbase.de/) and [SeaLifeBase](https://www.sealifebase.org/). See the `prepare_species_lists.sh` script for full details.

---

## Phase 1: Environment Setup & Taxonomy Files

### 1.1 Activate CRABS Environment

```bash
conda activate crabs
crabs --version
```

### 1.2 Create Project Directory Structure

```bash
mkdir -p raw_data processed_data reference_dbs taxonomy results
cd /home/evilliers/work/edna_method/edna_sa_project
```

### 1.3 Download NCBI Taxonomy Files

!!! warning "One-time setup — ~13 GB, takes 30–60 minutes"
    These files are required for all import operations. Download them once and reuse.

```bash
cd taxonomy

# NCBI taxonomy database
wget ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz
tar -xzf taxdump.tar.gz

# Accession-to-taxonomy mapping
wget ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/accession2taxid/nucl_gb.accession2taxid.gz
gunzip nucl_gb.accession2taxid.gz

cd ..
```

**Expected files in `taxonomy/`:**

| File | Size |
|------|------|
| `names.dmp` | ~272 MB |
| `nodes.dmp` | ~205 MB |
| `nucl_gb.accession2taxid` | ~13 GB |

---

## Phase 2: Download Sequence Data

### 2.1 Download from BOLD (COI-5P only)

```bash
# Ray-finned fishes
crabs --download-bold --taxon 'Actinopterygii' --marker 'COI-5P' --version-v3 --output raw_data/bold_actinopterygii_coi.fasta

# Cartilaginous fishes
crabs --download-bold --taxon 'Chondrichthyes' --marker 'COI-5P' --version-v3 --output raw_data/bold_chondrichthyes_coi.fasta

# Coelacanths
crabs --download-bold --taxon 'Sarcopterygii' --marker 'COI-5P' --version-v3 --output raw_data/bold_sarcopterygii_coi.fasta

# Hagfishes
crabs --download-bold --taxon 'Myxini' --marker 'COI-5P' --version-v3 --output raw_data/bold_myxini_coi.fasta

# Lampreys
crabs --download-bold --taxon 'Petromyzonti' --marker 'COI-5P' --version-v3 --output raw_data/bold_lampreys_coi.fasta

# Cetaceans
crabs --download-bold --taxon 'Cetacea' --marker 'COI-5P' --version-v3 --output raw_data/bold_cetacea_coi.fasta

# All Chordata (includes Pinnipeds/seals)
crabs --download-bold --taxon 'Chordata' --marker 'COI-5P' --version-v3 --output raw_data/bold_chordata_coi.fasta
```

!!! tip "Manual download fallback"
    If the automated download fails, download manually from [boldsystems.org](https://www.boldsystems.org) → Public Data Portal → Taxonomy Browser. Filter by **COI-5P** and download as FASTA.

### 2.2 Download from NCBI

!!! note
    Replace `your.email@example.com` with your actual email address. NCBI requires an email for API access.

```bash
EMAIL="your.email@example.com"

# ===== 12S rRNA (MiFish-U marker) =====
crabs --download-ncbi --database nucleotide --query 'Actinopterygii[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_actinopterygii.fasta
crabs --download-ncbi --database nucleotide --query 'Chondrichthyes[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_chondrichthyes.fasta
crabs --download-ncbi --database nucleotide --query 'Sarcopterygii[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_sarcopterygii.fasta
crabs --download-ncbi --database nucleotide --query 'Myxini[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_myxini.fasta
crabs --download-ncbi --database nucleotide --query 'Petromyzontidae[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_lampreys.fasta
crabs --download-ncbi --database nucleotide --query 'Cetacea[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_cetacea.fasta
crabs --download-ncbi --database nucleotide --query 'Pinnipedia[Organism] AND ("12S ribosomal RNA"[Title] OR 12S[Gene])' --email $EMAIL --output raw_data/ncbi_12s_pinnipedia.fasta

# ===== 16S rRNA (STAT / MarVer markers) =====
crabs --download-ncbi --database nucleotide --query '(Cetacea[Organism] AND 16S[All Fields]) AND 1000:2000[Sequence Length]' --email $EMAIL --output raw_data/ncbi_16s_cetacea.fasta
crabs --download-ncbi --database nucleotide --query '(Sirenia[Organism] AND 16S[All Fields]) AND 1000:2000[Sequence Length]' --email $EMAIL --output raw_data/ncbi_16s_sirenia.fasta
crabs --download-ncbi --database nucleotide --query '("Pinnipedia"[Organism] AND 16S[All Fields]) AND 1000:2000[Sequence Length]' --email $EMAIL --output raw_data/ncbi_16s_pinnipedia.fasta

# ===== COI (Leray-XT marker) =====
crabs --download-ncbi --database nucleotide --query '("Actinopterygii"[Organism] AND ("COX1"[Gene] OR "cytochrome c oxidase subunit I"[Gene])' --email $EMAIL --output raw_data/ncbi_coi_actinopterygii.fasta
crabs --download-ncbi --database nucleotide --query '("Chondrichthyes"[Organism] AND ("COX1"[Gene] OR "cytochrome c oxidase subunit I"[Gene]))' --email $EMAIL --output raw_data/ncbi_coi_chondrichthyes.fasta
crabs --download-ncbi --database nucleotide --query '("Sarcopterygii"[Organism] AND ("COX1"[Gene] OR "cytochrome c oxidase subunit I"[Gene]))' --email $EMAIL --output raw_data/ncbi_coi_sarcopterygii.fasta
crabs --download-ncbi --database nucleotide --query '("Cetacea"[Organism] AND ("COX1"[Gene] OR "cytochrome c oxidase subunit I"[Gene]))' --email $EMAIL --output raw_data/ncbi_coi_cetacea.fasta
crabs --download-ncbi --database nucleotide --query '("Pinnipedia"[Organism] AND ("COX1"[Gene] OR "cytochrome c oxidase subunit I"[Gene]))' --email $EMAIL --output raw_data/ncbi_coi_pinnipedia.fasta
```

### 2.3 Download from EMBL

```bash
# Vertebrate mitochondrial sequences
crabs --download-embl --taxon 'VRT_\d+\' --output raw_data/embl_vertebrates.fasta

# Mammal mitochondrial sequences
crabs --download-embl --taxon 'MAM_\d+\' --output raw_data/embl_mammals.fasta
```

### 2.4 Download from MitoFish

**Recommended: manual download**

1. Navigate to [mitofish.aori.u-tokyo.ac.jp/download.html](http://mitofish.aori.u-tokyo.ac.jp/download.html)
2. Download **Complete + Partial Mitochondrial Genomes**
3. Save as `raw_data/mitofish_complete.fasta`

Or use automated download:
```bash
crabs --download-mitofish --output raw_data/mitofish_complete.fasta
```

### 2.5 Verify Downloads

```bash
ls -lh raw_data/
```

**Expected:** ~27–30 FASTA files (BOLD: ~7, NCBI: ~17, EMBL: ~2, MitoFish: ~1)

---

## Phase 3: Import & Merge Sequences

All sequences must be converted to CRABS format with standardised NCBI taxonomy before processing.

### 3.1 Import BOLD Sequences

```bash
for taxon in actinopterygii_coi chondrichthyes_coi sarcopterygii_coi myxini_coi lampreys_coi cetacea_coi; do
  crabs --import \
    --import-format boldv3 \
    --input raw_data/bold_${taxon}.fasta \
    --names taxonomy/names.dmp \
    --nodes taxonomy/nodes.dmp \
    --acc2tax taxonomy/nucl_gb.accession2taxid \
    --output processed_data/bold_${taxon}.txt \
    --ranks 'superkingdom;phylum;class;order;family;genus;species'
done
```

!!! note
    Skip import commands for any files that don't exist or contain no sequences (e.g., Myxini, Sarcopterygii may have limited BOLD data).

### 3.2 Import NCBI Sequences

```bash
for file in ncbi_12s_actinopterygii ncbi_12s_chondrichthyes ncbi_12s_sarcopterygii ncbi_12s_myxini ncbi_12s_lampreys ncbi_12s_cetacea ncbi_12s_pinnipedia ncbi_16s_cetacea ncbi_16s_pinnipedia ncbi_16s_sirenia ncbi_coi_actinopterygii ncbi_coi_chondrichthyes ncbi_coi_sarcopterygii ncbi_coi_cetacea ncbi_coi_pinnipedia; do
  crabs --import \
    --import-format ncbi \
    --input raw_data/${file}.fasta \
    --names taxonomy/names.dmp \
    --nodes taxonomy/nodes.dmp \
    --acc2tax taxonomy/nucl_gb.accession2taxid \
    --output processed_data/${file}.txt \
    --ranks 'superkingdom;phylum;class;order;family;genus;species'
done
```

### 3.3 Import EMBL Sequences

```bash
for source in vertebrates mammals; do
  crabs --import \
    --import-format embl \
    --input raw_data/embl_${source}.fasta \
    --names taxonomy/names.dmp \
    --nodes taxonomy/nodes.dmp \
    --acc2tax taxonomy/nucl_gb.accession2taxid \
    --output processed_data/embl_${source}.txt \
    --ranks 'superkingdom;phylum;class;order;family;genus;species'
done
```

### 3.4 Import MitoFish Sequences

```bash
crabs --import \
  --import-format mitofish \
  --input raw_data/mitofish_complete.fasta \
  --names taxonomy/names.dmp \
  --nodes taxonomy/nodes.dmp \
  --acc2tax taxonomy/nucl_gb.accession2taxid \
  --output processed_data/mitofish_complete.txt \
  --ranks 'superkingdom;phylum;class;order;family;genus;species'
```

**Checkpoint:** You should have 10+ `.txt` files in `processed_data/`.

---

## Phase 4–6: Marker-Specific Workflows

Each marker follows the same six steps: **Merge → In Silico PCR → Filter → Dereplicate → Subset → Export**

---

### Workflow A: MiFish-U (12S rRNA)

**Target:** Universal fish and cetacean detection | **Amplicon:** 160–180 bp

#### A1 — Merge 12S Sources

```bash
crabs --merge \
  --output processed_data/merged_12s.txt \
  --import-format boldv5 \
  --uniq \
  --input "processed_data/ncbi_12s_actinopterygii.txt;
           processed_data/ncbi_12s_chondrichthyes.txt;
           processed_data/ncbi_12s_sarcopterygii.txt;
           processed_data/ncbi_12s_myxini.txt;
           processed_data/ncbi_12s_lampreys.txt;
           processed_data/ncbi_12s_cetacea.txt;
           processed_data/ncbi_12s_pinnipedia.txt;
           processed_data/mitofish_complete.txt;
           processed_data/embl_vertebrates.txt;
           processed_data/embl_mammals.txt"
```

Expected: ~500,000–2,000,000 sequences (~2–4 GB)

#### A2 — In Silico PCR

```bash
crabs --in-silico-pcr \
  --input processed_data/merged_12s.txt \
  --output processed_data/mifish_amplicons.txt \
  --forward GTCGGTAAAACTCGTGCCAGC \
  --reverse CATAGTGGGGTATCTAATCCCAGTTTG \
  --mismatch 4 \
  --buffer-size 988978900
```

!!! tip "OverflowError fix"
    If you get a buffer overflow error, calculate the correct buffer size with:
    ```bash
    awk -F'\t' '{ print length($NF) }' processed_data/merged_12s.txt | sort -nr | head -n 1 | awk '{ print $1*2 }'
    ```

Expected: ~50,000–200,000 sequences

#### A3 — Quality Filter

```bash
crabs --filter \
  --input processed_data/mifish_amplicons.txt \
  --output processed_data/mifish_filtered.txt \
  --minimum-length 100 \
  --maximum-length 250 \
  --maximum-n 2 \
  --environmental
```

#### A4 — Dereplicate

```bash
crabs --dereplicate \
  --input processed_data/mifish_filtered.txt \
  --output processed_data/mifish_derep.txt \
  --dereplication-method unique_species
```

Expected: ~8,000–30,000 sequences (50–80% reduction)

#### A5 — Subset to South African Species

```bash
crabs --subset \
  --input processed_data/mifish_derep.txt \
  --output processed_data/mifish_sa_subset.txt \
  --include merged_sa_species.txt

# Check how many SA species have 12S sequences
cut -f7 processed_data/mifish_sa_subset.txt | sort -u | wc -l
```

Expected: ~1,000–5,000 sequences

#### A6 — Export

```bash
crabs --export --input processed_data/mifish_sa_subset.txt --output reference_dbs/mifish_qiime.fasta --export-format qiime-fasta
crabs --export --input processed_data/mifish_sa_subset.txt --output reference_dbs/mifish_blast.fasta --export-format blast
crabs --export --input processed_data/mifish_sa_subset.txt --output reference_dbs/mifish_sintax.fasta --export-format sintax
```

---

### Workflow B: MarVer (Marine Mammals — 12S & 16S)

**Target:** Cetaceans, Pinnipeds, Sirenians | **Markers:** MarVer1 (12S) + MarVer3 (16S)

#### B1 — Merge Marine Mammal Sources

```bash
crabs --merge \
  --output processed_data/marine_mammals_16s_merged.txt \
  --uniq \
  --input "processed_data/ncbi_16s_cetacea.txt;
           processed_data/ncbi_16s_pinnipedia.txt;
           processed_data/ncbi_16s_sirenia.txt"
```

!!! note "Why not include EMBL MAM?"
    EMBL MAM contains all mammals — 96% are terrestrial primates. STAT/MarVer primers are marine mammal-specific and won't amplify terrestrial species, so including them wastes processing time.

#### B2 — In Silico PCR (both primers)

```bash
crabs --in-silico-pcr \
  --input processed_data/marine_mammals_16s_merged.txt \
  --output processed_data/amplicons_MarVer1.txt \
  --forward CGTGCCAGCCACCGCG \
  --reverse GGGTATCTAATCCYAGTTTG \
  --mismatch 4.5

crabs --in-silico-pcr \
  --input processed_data/marine_mammals_16s_merged.txt \
  --output processed_data/amplicons_MarVer3.txt \
  --forward AGACGAGAAGACCCTRTG \
  --reverse GGATTGCGCTGTTATCCC \
  --mismatch 4.5
```

#### B3 — Tag and Merge Amplicons

```bash
# Add primer tags to accession IDs to distinguish sources after merging
awk -F'\t' 'BEGIN {OFS="\t"} {$1=$1"_MarVer1"; print}' processed_data/amplicons_MarVer1.txt > processed_data/amplicons_MarVer1_tagged.txt
awk -F'\t' 'BEGIN {OFS="\t"} {$1=$1"_MarVer3"; print}' processed_data/amplicons_MarVer3.txt > processed_data/amplicons_MarVer3_tagged.txt

cat processed_data/amplicons_MarVer1_tagged.txt processed_data/amplicons_MarVer3_tagged.txt > processed_data/amplicons_combined_tagged.txt
```

#### B4 — Quality Filter

```bash
crabs --filter \
  --input processed_data/amplicons_combined_tagged.txt \
  --output processed_data/amplicons_cleaned.txt \
  --minimum-length 100 \
  --maximum-length 500 \
  --maximum-n 0
```

#### B5 — Dereplicate

```bash
crabs --dereplicate \
  --input processed_data/amplicons_cleaned.txt \
  --output processed_data/amplicons_dereplicated.txt \
  --dereplication-method unique_species
```

#### B6 — Subset to SA Species

```bash
crabs --subset \
  --input processed_data/amplicons_dereplicated.txt \
  --output processed_data/amplicons_sa_subset.txt \
  --include merged_sa_species.txt
```

#### B7 — Export

```bash
crabs --export --input processed_data/amplicons_sa_subset.txt --output reference_dbs/marine_mammals_qiime.fasta --export-format qiime-fasta
crabs --export --input processed_data/amplicons_sa_subset.txt --output reference_dbs/marine_mammals_blast.fasta --export-format blast
crabs --export --input processed_data/amplicons_sa_subset.txt --output reference_dbs/marine_mammals_sintax.fasta --export-format sintax
```

---

### Workflow C: Leray-XT (COI)

**Target:** Fish, cetaceans, invertebrates | **Amplicon:** ~313 bp | **Largest dataset**

#### C1 — Merge COI Sources

```bash
crabs --merge \
  --output processed_data/merged_coi.txt \
  --uniq \
  --input "processed_data/bold_actinopterygii_coi.txt;
           processed_data/bold_chondrichthyes_coi.txt;
           processed_data/bold_sarcopterygii_coi.txt;
           processed_data/bold_myxini_coi.txt;
           processed_data/bold_lampreys_coi.txt;
           processed_data/bold_cetacea_coi.txt;
           processed_data/ncbi_coi_actinopterygii.txt;
           processed_data/ncbi_coi_chondrichthyes.txt;
           processed_data/ncbi_coi_sarcopterygii.txt;
           processed_data/ncbi_coi_cetacea.txt;
           processed_data/ncbi_coi_pinnipedia.txt;
           processed_data/mitofish_complete.txt;
           processed_data/embl_vertebrates.txt"
```

Expected: ~1,000,000–3,000,000 sequences (~5–10 GB)

#### C2 — In Silico PCR

```bash
# Calculate buffer size first
awk -F'\t' '{ print length($NF) }' processed_data/merged_coi.txt | sort -nr | head -n 1 | awk '{ print $1*2 }'

crabs --in-silico-pcr \
  --input processed_data/merged_coi.txt \
  --output processed_data/leray_amplicons.txt \
  --forward GGWACWRGWTGRACWITITAYCCYCC \
  --reverse TAIACYTCIGGRTGICCRAARAAYCA \
  --mismatch 4 \
  --buffer-size 452178200
```

#### C3 — Quality Filter

```bash
crabs --filter \
  --input processed_data/leray_amplicons.txt \
  --output processed_data/leray_filtered.txt \
  --minimum-length 250 \
  --maximum-length 400 \
  --maximum-n 2 \
  --environmental
```

#### C4 — Dereplicate

```bash
crabs --dereplicate \
  --input processed_data/leray_filtered.txt \
  --output processed_data/leray_derep.txt \
  --dereplication-method unique_species
```

#### C5 — Subset to SA Species

```bash
crabs --subset \
  --input processed_data/leray_derep.txt \
  --output processed_data/leray_sa_subset.txt \
  --include merged_sa_species.txt
```

Expected: ~3,000–15,000 sequences

#### C6 — Export

```bash
crabs --export --input processed_data/leray_sa_subset.txt --output reference_dbs/leray_qiime.fasta --export-format qiime-fasta
crabs --export --input processed_data/leray_sa_subset.txt --output reference_dbs/leray_blast.fasta --export-format blast
crabs --export --input processed_data/leray_sa_subset.txt --output reference_dbs/leray_sintax.fasta --export-format sintax
```

---

## Phase 7: Create BLAST Databases

```bash
makeblastdb -in reference_dbs/mifish_blast.fasta -dbtype nucl -parse_seqids -out reference_dbs/mifish_blastdb
makeblastdb -in reference_dbs/marine_mammals_blast.fasta -dbtype nucl -parse_seqids -out reference_dbs/marine_mammals_blastdb
makeblastdb -in reference_dbs/leray_blast.fasta -dbtype nucl -parse_seqids -out reference_dbs/leray_blastdb
```

Each marker should produce `.nhr`, `.nin`, and `.nsq` index files.

---

## Phase 8: Visualisation & Quality Control

### Database Diversity Figures

```bash
# --tax-level: 1=superkingdom, 2=phylum, 3=class, 4=order, 5=family, 6=genus, 7=species
crabs --diversity-figure --input processed_data/mifish_sa_subset.txt --output results/mifish_diversity.png --tax-level 4
crabs --diversity-figure --input processed_data/amplicons_sa_subset.txt --output results/marver_diversity.png --tax-level 4
crabs --diversity-figure --input processed_data/leray_sa_subset.txt --output results/leray_diversity.png --tax-level 4
```

### Amplicon Length Distributions

```bash
crabs --amplicon-length-figure --input processed_data/mifish_sa_subset.txt --tax-level 4 --output results/mifish_lengths.png
crabs --amplicon-length-figure --input processed_data/leray_sa_subset.txt --tax-level 4 --output results/leray_lengths.png
```

### Species Coverage Report

```bash
echo "MiFish-U (12S) species:"; cut -f7 processed_data/mifish_sa_subset.txt | sort -u | wc -l
echo "MarVer (16S/12S) species:"; cut -f7 processed_data/amplicons_sa_subset.txt | sort -u | wc -l
echo "Leray-XT (COI) species:"; cut -f7 processed_data/leray_sa_subset.txt | sort -u | wc -l
echo "Total SA species in list:"; wc -l merged_sa_species.txt

# Find species covered by both 12S and COI
cut -f7 processed_data/mifish_sa_subset.txt | sort -u > results/mifish_species_list.txt
cut -f7 processed_data/leray_sa_subset.txt | sort -u > results/leray_species_list.txt
comm -12 results/mifish_species_list.txt results/leray_species_list.txt > results/species_in_both_12S_COI.txt
wc -l results/species_in_both_12S_COI.txt
```

---

## Phase 9: Downstream Analysis

### BLAST Assignment

```bash
blastn \
  -query your_asvs.fasta \
  -db reference_dbs/mifish_blastdb \
  -out results/blast_results.txt \
  -outfmt '6 qseqid sseqid pident length qcovs evalue bitscore staxids sscinames scomnames' \
  -max_target_seqs 5 \
  -perc_identity 97 \
  -num_threads 4
```

### DADA2 Assignment (R)

```r
library(dada2)

taxa <- assignTaxonomy(
  seqs = seqtab.nochim,
  refFasta = "reference_dbs/mifish_qiime.fasta",
  minBoot = 80,
  multithread = TRUE,
  tryRC = TRUE
)
write.csv(taxa, "results/mifish_dada2_taxonomy.csv")
```

### QIIME2 Assignment

```bash
conda activate qiime2-amplicon-2024.10

# Import reference sequences
qiime tools import \
  --type 'FeatureData[Sequence]' \
  --input-path reference_dbs/mifish_qiime.fasta \
  --output-path reference_dbs/mifish_seqs.qza

# Train classifier
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads reference_dbs/mifish_seqs.qza \
  --i-reference-taxonomy reference_dbs/mifish_taxonomy.qza \
  --o-classifier reference_dbs/mifish_classifier.qza

# Classify
qiime feature-classifier classify-sklearn \
  --i-classifier reference_dbs/mifish_classifier.qza \
  --i-reads your_rep_seqs.qza \
  --o-classification results/mifish_taxonomy.qza
```

---

## Troubleshooting

| Problem | Solution |
|---------|---------|
| `FileNotFoundError: taxonomy/names.dmp` | Re-download taxonomy files (Phase 1.3) |
| In silico PCR returns < 1,000 sequences | Increase `--mismatch` to 5 or 6; verify primer sequences |
| `OverflowError: FASTA record does not fit into buffer` | Recalculate `--buffer-size` (see Phase A2 tip above) |
| Subset returns empty file | Check species name format matches between database and `merged_sa_species.txt` (e.g., `Genus species` not `Genus_species`) |
| `MemoryError` during merge | Process taxa groups separately, then merge the subsetted results |
| BLAST database creation fails | Run `makeblastdb -version`; install with `conda install -c bioconda blast` |

---

## Summary Checklist

- [ ] Phase 1: Conda environment active; taxonomy files downloaded
- [ ] Phase 2: All sequences downloaded (BOLD, NCBI, EMBL, MitoFish)
- [ ] Phase 3: All sequences imported to CRABS format
- [ ] **Workflow A (MiFish-U):** Merged → PCR → Filtered → Dereplicated → Subsetted → Exported → BLAST DB
- [ ] **Workflow B (MarVer):** Merged → PCR → Tagged → Filtered → Dereplicated → Subsetted → Exported → BLAST DB
- [ ] **Workflow C (Leray-XT):** Merged → PCR → Filtered → Dereplicated → Subsetted → Exported → BLAST DB
- [ ] Phase 8: Diversity figures and species coverage reports generated
- [ ] Phase 9: Ready for taxonomic assignment (BLAST / DADA2 / QIIME2)

---

## Final Directory Structure

```
edna_sa_project/
├── merged_sa_species.txt        (2,087 species)
├── taxonomy/
│   ├── names.dmp                (272 MB)
│   ├── nodes.dmp                (205 MB)
│   └── nucl_gb.accession2taxid  (13 GB)
├── raw_data/                    (~27–30 FASTA files)
├── processed_data/              (~20+ CRABS .txt files)
├── reference_dbs/
│   ├── mifish_qiime.fasta       mifish_blast.fasta   mifish_sintax.fasta
│   ├── mifish_blastdb.nhr/nin/nsq
│   ├── marine_mammals_*.fasta   + BLAST index files
│   └── leray_*.fasta            + BLAST index files
└── results/
    ├── *_diversity.png
    ├── *_lengths.png
    └── *_species_list.txt
```

---

## References

Jeunen G-J, Dowle E, Edgecombe J, von Ammon U, Gemmell NJ, Cross H (2022). crabs — A software program to generate curated reference databases for metabarcoding sequencing data. *Molecular Ecology Resources*. [doi:10.1111/1755-0998.13741](https://doi.org/10.1111/1755-0998.13741)

Valsecchi E et al. (2020). Novel universal primers for metabarcoding environmental DNA surveys of marine mammals and other marine vertebrates. *Environmental DNA*, 2(4), 602–617.

- [CRABS GitHub](https://github.com/gjeunen/reference_database_creator)
- [FishBase](https://fishbase.de/)
- [SeaLifeBase](https://www.sealifebase.org/)
