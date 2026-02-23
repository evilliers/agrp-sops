# QIIME2 eDNA Metabarcoding Workflow

**SOP ID:** EDNA-002
**Version:** 2.0
**Date:** 2026-02-23
**Author:** AGRP

---

## Overview

This SOP guides you through a complete QIIME2 amplicon metabarcoding analysis — from raw paired-end FASTQ files to diversity statistics and R-ready export files. The workflow uses 16S rRNA data from *Exaiptasia diaphana* (sea anemone) bacterial communities as an example dataset.

!!! tip "Full Course Available"
    This SOP is based on the AGRP QIIME2 training course. For step-by-step exercises and detailed explanations, see the full tutorial:

    **[QIIME2 eDNA Metabarcoding Tutorial](https://lab417.saiab.ac.za/qiime2_course/qiime2_tutorial.html)**

**QIIME2 version:** `qiime2-amplicon-2024.10`

---

## Prerequisites

- Access to `lab417.saiab.ac.za` (see [Getting Started](../hpc/getting-started.md))
- Demultiplexed paired-end FASTQ files (Illumina MiSeq)
- A metadata file in TSV format (validated with Keemei)
- A pre-trained classifier matching your primer region (see [Section 6](#6-training-your-own-classifier))

---

## Environment Setup

Activate the shared QIIME2 environment:

```bash
source /usr/local/bin/qiime2-activate
qiime info
```

Use a `screen` session to protect long-running jobs from disconnection:

```bash
screen -S qiime_analysis
```

| Screen command | Action |
|---------------|--------|
| `ctrl-a ctrl-d` | Detach from session |
| `screen -r qiime_analysis` | Reattach session |

### Directory Setup

```bash
mkdir -p analysis/seqs
mkdir analysis/visualisations
mkdir analysis/tree
mkdir analysis/taxonomy
mkdir analysis/export
```

---

## Workflow Summary

```
Raw FASTQs → Import → Trim Primers → Quality Check → DADA2 Denoise
    → Taxonomic Classification → Filter Contaminants
    → Phylogenetic Tree → Diversity Metrics → Export for R
```

---

## 1. Data Import, Cleaning & Quality Control

### 1.1 Import Raw Data

Import demultiplexed paired-end FASTQ files (Casava format):

```bash
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path raw_data \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path analysis/seqs/combined.qza
```

**Output:** `analysis/seqs/combined.qza`

### 1.2 Remove Primers with Cutadapt

!!! note
    Confirm with your sequencing facility whether primers were already removed.

Replace the primer sequences below with your own if using a different region:

```bash
qiime cutadapt trim-paired \
  --i-demultiplexed-sequences analysis/seqs/combined.qza \
  --p-front-f AGGATTAGATACCCTGGTA \
  --p-front-r CRRCACGAGCTGACGAC \
  --p-error-rate 0.20 \
  --output-dir analysis/seqs_trimmed_data \
  --verbose
```

**Primers (16S V5V6):** Forward 784f / Reverse 1492r
**Output:** `analysis/seqs_trimmed_data/trimmed_sequences.qza`

### 1.3 Assess Sequence Quality

Generate a quality visualisation:

```bash
qiime demux summarize \
  --i-data analysis/seqs_trimmed_data/trimmed_sequences.qza \
  --o-visualization analysis/visualisations/trimmed_sequences.qzv
```

Download the `.qzv` file to your computer with `scp`, then open it at **[view.qiime2.org](https://view.qiime2.org)**.

**What to look for:**

- Sequence retention: 70–90% is optimal
- Forward and reverse quality plots — note where median quality drops below **Q30**
- Record truncation lengths for the DADA2 step (separate values for forward and reverse reads)

### 1.4 Denoise with DADA2

DADA2 corrects sequencing errors, merges paired reads, removes chimeras, and produces Amplicon Sequence Variants (ASVs).

!!! warning
    This step can take hours on large datasets. Adjust `--p-trunc-len-f` and `--p-trunc-len-r` based on **your own** quality plots — do not use these values blindly.

```bash
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs analysis/seqs_trimmed_data/trimmed_sequences.qza \
  --p-trunc-len-f 211 \
  --p-trunc-len-r 167 \
  --p-n-threads 0 \
  --output-dir analysis/dada2out \
  --verbose
```

**Outputs:**

| File | Contents |
|------|---------|
| `analysis/dada2out/table.qza` | ASV feature table |
| `analysis/dada2out/representative_sequences.qza` | ASV sequences |
| `analysis/dada2out/denoising_stats.qza` | Per-sample retention stats |

### 1.5 Check Denoising Statistics

```bash
qiime metadata tabulate \
  --m-input-file analysis/dada2out/denoising_stats.qza \
  --o-visualization analysis/visualisations/16s_denoising_stats.qzv
```

**Quality targets:**

| Step | Target retention |
|------|-----------------|
| Passed quality filter | > 70–80% |
| Merged | > 85–95% |
| Non-chimeric | > 85–95% |

### 1.6 Summarise Feature Table

```bash
qiime feature-table summarize \
  --i-table analysis/dada2out/table.qza \
  --m-sample-metadata-file metadata.tsv \
  --o-visualization analysis/visualisations/16s_table.qzv
```

In the **Interactive Sample Detail** tab, note the median per-sample frequency — you will use this as your rarefaction depth later.

---

## 2. Taxonomic Classification

### 2.1 Classify ASVs

Run the Naive Bayes classifier against the pre-trained SILVA reference:

```bash
qiime feature-classifier classify-sklearn \
  --i-classifier silva_138_16s_v5v6_classifier_2021-4.qza \
  --i-reads analysis/dada2out/representative_sequences.qza \
  --p-n-jobs 1 \
  --output-dir analysis/taxonomy \
  --verbose
```

**Output:** `analysis/taxonomy/classification.qza`

!!! note
    The classifier provided is specific to the **16S V5V6 region**. For different primers, you must train your own (see [Section 6](#6-training-your-own-classifier)).

### 2.2 Visualise Taxonomy

```bash
qiime metadata tabulate \
  --m-input-file analysis/taxonomy/classification.qza \
  --o-visualization analysis/visualisations/taxonomy.qzv
```

Taxonomy strings follow the format:
```
k__Bacteria;p__Firmicutes;c__Bacilli;o__Lactobacillales;f__Streptococcaceae
```

Confidence scores range from 0–1. Classifications with low confidence (< 0.7) should be treated cautiously.

### 2.3 Remove Mitochondria and Chloroplasts

```bash
qiime taxa filter-table \
  --i-table analysis/dada2out/table.qza \
  --i-taxonomy analysis/taxonomy/classification.qza \
  --p-exclude Mitochondria,Chloroplast \
  --o-filtered-table analysis/taxonomy/16s_table_filtered.qza
```

**Output:** `analysis/taxonomy/16s_table_filtered.qza`

### 2.4 Summarise Filtered Table

```bash
qiime feature-table summarize \
  --i-table analysis/taxonomy/16s_table_filtered.qza \
  --m-sample-metadata-file metadata.tsv \
  --o-visualization analysis/visualisations/16s_table_filtered.qzv
```

Use the median frequency from the **Interactive Sample Detail** tab to set your rarefaction depth in Section 4.

---

## 3. Phylogenetic Tree Construction

Build a rooted phylogenetic tree from ASV sequences using MAFFT + FastTree:

```bash
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences analysis/dada2out/representative_sequences.qza \
  --o-alignment analysis/tree/aligned_16s_representative_seqs.qza \
  --o-masked-alignment analysis/tree/masked_aligned_16s_representative_seqs.qza \
  --o-tree analysis/tree/16s_unrooted_tree.qza \
  --o-rooted-tree analysis/tree/16s_rooted_tree.qza \
  --p-n-threads 1 \
  --verbose
```

The **rooted tree** (`16s_rooted_tree.qza`) is required for phylogenetic diversity metrics in the next section.

---

## 4. Diversity Analysis

### 4.1 Taxonomic Bar Charts

```bash
qiime taxa barplot \
  --i-table analysis/taxonomy/16s_table_filtered.qza \
  --i-taxonomy analysis/taxonomy/classification.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization analysis/visualisations/barchart.qzv
```

In QIIME2 View, use the taxonomic level dropdown (Level 1–7) to explore community composition across samples.

### 4.2 Rarefaction Curves

Set `--p-max-depth` to the **median sample frequency** from `16s_table_filtered.qzv`:

```bash
qiime diversity alpha-rarefaction \
  --i-table analysis/taxonomy/16s_table_filtered.qza \
  --i-phylogeny analysis/tree/16s_rooted_tree.qza \
  --p-max-depth 9062 \
  --m-metadata-file metadata.tsv \
  --o-visualization analysis/visualisations/16s_alpha_rarefaction.qzv
```

**Plateauing curves** indicate sufficient sequencing depth. Non-plateauing curves mean diversity is undersampled.

### 4.3 Core Diversity Metrics

Set `--p-sampling-depth` to the **median frequency** from your filtered table (samples below this depth are excluded):

```bash
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny analysis/tree/16s_rooted_tree.qza \
  --i-table analysis/taxonomy/16s_table_filtered.qza \
  --p-sampling-depth 5583 \
  --m-metadata-file metadata.tsv \
  --output-dir analysis/diversity_metrics
```

**Alpha diversity metrics generated (within-sample):**

| Metric | What it measures |
|--------|----------------|
| Shannon index | Richness weighted by abundance |
| Observed features | Count of unique ASVs |
| Faith's PD | Phylogenetic diversity |
| Evenness (Pielou's) | How evenly distributed species are |

**Beta diversity metrics generated (between-sample):**

| Metric | Type |
|--------|------|
| Jaccard | Qualitative (presence/absence) |
| Bray-Curtis | Quantitative (abundance-weighted) |
| Unweighted UniFrac | Qualitative + phylogenetic |
| Weighted UniFrac | Quantitative + phylogenetic |

Copy visualisations to your visualisations folder:

```bash
cp analysis/diversity_metrics/*.qzv analysis/visualisations/
```

### 4.4 Alpha Diversity Group Significance

Test whether groups differ in phylogenetic diversity and evenness:

```bash
qiime diversity alpha-group-significance \
  --i-alpha-diversity analysis/diversity_metrics/faith_pd_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization analysis/visualisations/faith-pd-group-significance.qzv

qiime diversity alpha-group-significance \
  --i-alpha-diversity analysis/diversity_metrics/evenness_vector.qza \
  --m-metadata-file metadata.tsv \
  --o-visualization analysis/visualisations/evenness-group-significance.qzv
```

A p-value < 0.05 (Kruskal-Wallis test) indicates a significant difference between groups.

### 4.5 Beta Diversity Group Significance (PERMANOVA)

Test whether microbial composition differs between metadata groups:

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix analysis/diversity_metrics/unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column Genotype \
  --o-visualization analysis/visualisations/unweighted-unifrac-genotype-significance.qzv \
  --p-pairwise

qiime diversity beta-group-significance \
  --i-distance-matrix analysis/diversity_metrics/unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file metadata.tsv \
  --m-metadata-column Environment \
  --o-visualization analysis/visualisations/unweighted-unifrac-environment-significance.qzv \
  --p-pairwise
```

Replace `Genotype` and `Environment` with the relevant column names from your own metadata file.

---

## 5. Exporting Data for R

Export files in formats compatible with phyloseq and other R packages:

```bash
# Phylogenetic tree (Newick format)
qiime tools export \
  --input-path analysis/tree/16s_unrooted_tree.qza \
  --output-path analysis/export

# ASV feature table (BIOM format)
qiime tools export \
  --input-path analysis/taxonomy/16s_table_filtered.qza \
  --output-path analysis/export

# Convert BIOM to TSV
biom convert \
  -i analysis/export/feature-table.biom \
  -o analysis/export/feature-table.tsv \
  --to-tsv

# Taxonomy assignments
qiime tools export \
  --input-path analysis/taxonomy/classification.qza \
  --output-path analysis/export

# Remove extra header lines
sed '1d' analysis/export/taxonomy.tsv > analysis/export/taxonomy_noHeader.tsv
sed '1d' analysis/export/feature-table.tsv > analysis/export/feature-table_noHeader.tsv
```

**Files exported to `analysis/export/`:**

| File | Use |
|------|-----|
| `tree.nwk` | Phylogenetic tree for phyloseq |
| `feature-table.biom` | ASV table (BIOM format) |
| `feature-table_noHeader.tsv` | ASV table (TSV format) |
| `taxonomy_noHeader.tsv` | Taxonomy assignments |

---

## 6. Training Your Own Classifier

Required when using **different primers or a different gene region** to the V5V6 classifier above.

```bash
# Step 1 — Extract region-specific reads from SILVA reference
qiime feature-classifier extract-reads \
  --i-sequences silva-138-99-seqs.qza \
  --p-f-primer FORWARD_PRIMER_SEQUENCE \
  --p-r-primer REVERSE_PRIMER_SEQUENCE \
  --o-reads silva_138_marker_gene.qza

# Step 2 — Train the Naive Bayes classifier
qiime feature-classifier fit-classifier-naive-bayes \
  --i-reference-reads silva_138_marker_gene.qza \
  --i-reference-taxonomy silva-138-99-tax.qza \
  --o-classifier silva_138_marker_gene_classifier.qza \
  --verbose
```

Replace `FORWARD_PRIMER_SEQUENCE` and `REVERSE_PRIMER_SEQUENCE` with your actual primers.

---

## Troubleshooting

| Problem | Solution |
|---------|---------|
| Memory error during classification | Add `--p-reads-per-batch 10000` or reduce `--p-n-jobs` |
| Low read retention after DADA2 | Re-examine quality plots and adjust truncation lengths |
| Rarefaction curves don't plateau | Increase `--p-max-depth` or consider deeper sequencing |
| Sample dropped from diversity analysis | Its frequency was below `--p-sampling-depth`; lower the depth or check that sample |
| QIIME2 View won't load | Use Chrome or Firefox (not private/incognito mode) |
| Wrong classifier results | Ensure classifier matches your primer region |

!!! tip
    Use `qiime <plugin> <command> --help` at any time to see full parameter descriptions.

---

## Reference

Dungan AM, van Oppen MJH, and Blackall LL (2021) Short-Term Exposure to Sterile Seawater Reduces Bacterial Community Diversity in the Sea Anemone, *Exaiptasia diaphana*. *Front. Mar. Sci.* 7:599314. doi:10.3389/fmars.2020.599314

## Further Reading

- **[Full QIIME2 Tutorial — AGRP Course](https://lab417.saiab.ac.za/qiime2_course/qiime2_tutorial.html)**
- [Official QIIME2 Documentation](https://docs.qiime2.org/)
- [QIIME2 Forum](https://forum.qiime2.org/)
- [SILVA Reference Database](https://www.arb-silva.de/)
