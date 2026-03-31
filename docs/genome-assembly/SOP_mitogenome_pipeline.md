# SOP: Fish Mitogenome Assembly and Annotation from ONT Data

**Version:** 1.1
**Date:** 2026-03-30
**Pipeline scripts:** `run_pipeline.sh`, `annotate_mitos2.sh`, `generate_report.sh`

---

## 1. Overview

This SOP describes how to assemble and annotate a fish mitochondrial genome (mitogenome) from Oxford Nanopore Technology (ONT) long reads. The pipeline takes raw reads, enriches mitochondrial reads by mapping to a reference, assembles and polishes the mitogenome, annotates it using two complementary tools (MITOS2 for gene annotation and MitoZ for circos visualisation), performs BLAST-based species identification against NCBI nt, and generates a final markdown report.

The pipeline is configured per species using a plain-text config file, allowing it to be reused across any fish species without modifying the main scripts.

**Expected outputs for a typical vertebrate fish mitogenome:**

- 1 circular contig, ~16–18 kb
- 13 protein-coding genes
- 2 ribosomal RNA genes (12S, 16S)
- 22 transfer RNA genes

---

## 2. Pipeline Code Repository

The pipeline scripts are version-controlled in a dedicated Gitea repository:

**[172.20.142.126:3000/evilliers/mitogenome-pipeline](http://172.20.142.126:3000/evilliers/mitogenome-pipeline)**

This repository contains:

| File / Folder | Description |
|---------------|-------------|
| `run_pipeline.sh` | Main assembly pipeline (Steps 0–8) |
| `annotate_mitos2.sh` | MITOS2 annotation (run separately) |
| `generate_report.sh` | Final report generator |
| `configs/` | Species config files (one per species) |
| `SOP_mitogenome_pipeline.md` | This SOP (kept in sync with the website) |

!!! note "Large files excluded from git"
    The following are **not** tracked in git due to size:
    `singularity/mitoz_3.6.sif` (~2 GB), `results/`, `reference/`, `logs/`, and raw read files.
    These are generated or downloaded automatically on first run.

### Contributing changes via pull request

If you update any pipeline script or config file, please submit a pull request rather than pushing directly to `main`:

```bash
# 1. Clone the repo (first time only)
git clone http://172.20.142.126:3000/evilliers/mitogenome-pipeline.git

# 2. Create a feature branch
git checkout -b fix/describe-your-change

# 3. Make your changes, then commit
git add run_pipeline.sh          # stage specific files, not git add -A
git commit -m "Fix: brief description of what changed"

# 4. Push the branch and open a PR on Gitea
git push origin fix/describe-your-change
```

Then open a pull request on Gitea at `172.20.142.126:3000/evilliers/mitogenome-pipeline/pulls`. In the PR description, note:

- What script(s) were changed and why
- Which species / config was used to test
- Any new or changed output files

!!! tip "Updating the SOP"
    If your change affects how the pipeline is run, update `SOP_mitogenome_pipeline.md` in the same PR. After merging, copy the updated SOP to the `agrp-sops` repository and push there too so the website stays in sync.

---

## 3. Prerequisites

### 3.1 Software and environments

The following tools must be installed before running the pipeline.

| Tool | Environment / method | Used in step |
|------|----------------------|--------------|
| NanoPlot | `ont` conda env | Step 1 (QC) |
| minimap2 | `ont` conda env | Step 3 (read enrichment) |
| samtools | `ont` conda env | Step 3 (read enrichment) |
| seqkit | `ont` conda env | Step 3, Step 6 |
| Flye | `ont` conda env | Step 4 (assembly) |
| QUAST | `ont` conda env | Step 6 (stats) |
| medaka | `medaka` conda env | Step 5 (polishing) |
| MITOS2 (runmitos) | `mitos2` conda env | Annotation script |
| BioPython | `mitos2` conda env | Step 8 (BLAST species ID) |
| MitoZ | Singularity image | Step 7 (circos / gene order) |
| efetch (NCBI E-utilities) | system PATH | Step 0 (reference download) |
| Singularity / Apptainer | system binary | Step 7 |

The `mitos2` conda environment and the MitoZ Singularity image are created automatically on first run if they do not exist. BioPython is already included in the `mitos2` environment.

### 3.2 Input data requirements

- ONT reads in `.fastq.gz` format (a single concatenated file per species)
- A known NCBI RefSeq accession for a closely related mitogenome to use as a mapping reference

---

## 4. Repository layout

```
mitogenome-pipeline/
├── run_pipeline.sh          # Main assembly pipeline (Steps 0–8)
├── annotate_mitos2.sh       # MITOS2 annotation (run separately)
├── generate_report.sh       # Final report generator
├── configs/
│   └── anguilla_marmorata.conf   # Example species config
├── reference/               # Reference FASTAs (auto-downloaded, not in git)
│   └── refseq89m/           # MITOS2 reference database (auto-downloaded)
├── singularity/
│   └── mitoz_3.6.sif        # MitoZ Singularity image (auto-downloaded, not in git)
└── results/
    └── <species_prefix>/    # All outputs for a species (not in git)
        ├── 01_qc/
        ├── 03_mito_reads/
        ├── 04_assembly/
        ├── 05_polished/
        ├── 06_stats/
        ├── 07_mitoz/
        ├── 07_annotation/
        ├── 08_blast/
        └── report.md
```

> **Note:** `07_annotation/` only appears after running `annotate_mitos2.sh`. `08_blast/` and `report.md` are created by `run_pipeline.sh` and `generate_report.sh` respectively.

---

## 5. Step 1 — Create a species config file

Each species requires a config file in the `configs/` directory. Copy the example and edit it for the new species.

```bash
cp configs/anguilla_marmorata.conf configs/<new_species>.conf
```

Open the new file and set the following variables:

```bash
# =============================================================================
# Species config: <Common name or full name>
# =============================================================================

SPECIES_NAME="Genus species"         # Full scientific name (for display only)
SPECIES_PREFIX="genus_species"       # Lowercase, underscore-separated (used for file names)

# Input reads (fastq or fastq.gz) — path relative to project root
READS="path/to/reads.fastq.gz"

# NCBI RefSeq accession for a closely related mitogenome
REFERENCE_ACCESSION="NC_XXXXXX.1"

# Expected mitogenome size — used by Flye (e.g., 17k, 16.5k)
GENOME_SIZE="17k"

# Flye read type:
#   --nano-hq   for R10+ flowcells with high-accuracy basecalling (recommended)
#   --nano-raw  for older R9.4.1 flowcells or standard basecalling
FLYE_MODE="--nano-hq"

# Medaka polishing model — must match the flowcell and basecaller version used
# See: https://github.com/nanoporetech/medaka#models
MEDAKA_MODEL="r1041_e82_400bps_sup_v5.0.0"
```

**Choosing `REFERENCE_ACCESSION`:** Search NCBI for a complete mitogenome from the same genus or a closely related genus. The reference is used only for read enrichment (mapping) and QUAST comparison; it does not need to be from the same species.

**Choosing `FLYE_MODE`:**

| Flowcell | Basecalling | Use |
|----------|-------------|-----|
| R10.4.1 + SUP/HAC | `--nano-hq` |
| R9.4.1 | `--nano-raw` |

**Choosing `MEDAKA_MODEL`:** The model name encodes the flowcell chemistry, pore version, basecaller speed, and model version. Check `medaka tools list_models` for all available models.

---

## 6. Step 2 — Run the assembly and MitoZ pipeline

Activate the `ont` conda environment, then run the pipeline with the species config:

```bash
conda activate ont
bash run_pipeline.sh configs/<species>.conf
```

**Example:**
```bash
conda activate ont
bash run_pipeline.sh configs/anguilla_marmorata.conf
```

The script will run non-interactively through all steps. The first run for a new species will also:

- Download the reference mitogenome from NCBI (Step 0, requires internet)
- Pull the MitoZ Singularity image (~1 GB, Step 7, requires internet)

Both are skipped on subsequent runs if the files already exist.

### Pipeline steps

| Step | Tool | Description | Output directory |
|------|------|-------------|-----------------|
| 0 | efetch | Download reference mitogenome from NCBI | `reference/` |
| 1 | NanoPlot | Read quality report | `results/<prefix>/01_qc/` |
| 3 | minimap2 + samtools | Map all reads to reference, keep mapped reads | `results/<prefix>/03_mito_reads/` |
| 4 | Flye | Assemble enriched mito reads | `results/<prefix>/04_assembly/` |
| 5 | Medaka | Polish assembly with raw reads | `results/<prefix>/05_polished/` |
| 6 | seqkit + QUAST | Assembly statistics and comparison to reference | `results/<prefix>/06_stats/` |
| 7 | MitoZ (Singularity) | Circos plot and gene order | `results/<prefix>/07_mitoz/` |
| 8 | BioPython + NCBI BLAST | Species identification — top 10 nt hits | `results/<prefix>/08_blast/` |

!!! note
    Step 2 (Filtlong pre-filtering) is intentionally skipped. Pre-filtering to a size limit before mapping discards most mitochondrial reads. All raw reads are mapped directly to the reference instead.

!!! note
    Step 8 submits `consensus.fasta` to NCBI BLAST over HTTP and may take 5–15 minutes depending on NCBI load. An internet connection is required.

### Checking assembly quality after Step 6

Before proceeding to annotation, verify the assembly looks correct:

1. **Check contig count and circularity** — `results/<prefix>/04_assembly/assembly_info.txt` should show 1 contig, `circ. = Y`, length ~16–18 kb.
2. **Check QUAST genome fraction** — `results/<prefix>/06_stats/report.txt` should show `Genome fraction (%) = ~100` and `# misassemblies = 0`.
3. **Check MitoZ gene order** — `results/<prefix>/07_mitoz/<prefix>.consensus.fasta.result/` should contain the circos plot and GenBank file.

---

## 7. Step 3 — Run MITOS2 annotation

MITOS2 is run as a separate script. Pass the same config file used for assembly — the script derives all paths from `SPECIES_PREFIX` and writes output to `results/<prefix>/07_annotation/`.

```bash
bash annotate_mitos2.sh configs/<species>.conf
```

**Example:**
```bash
bash annotate_mitos2.sh configs/anguilla_marmorata.conf
```

On first run, the script will automatically:

1. Create the `mitos2` conda environment (if not present)
2. Download the MITOS2 reference database `refseq89m` from Zenodo (~20 MB)

### MITOS2 output files

| File | Contents |
|------|----------|
| `result.bed` | Gene coordinates (BED format) |
| `result.gff` | Gene annotation (GFF3 format) |
| `result.fas` | Gene nucleotide sequences |
| `result.faa` | Protein sequences |
| `result.mito` | Summary of all annotated features |
| `result.geneorder` | Gene order string |
| `result.png` | Linear genome map |

**Expected result for a vertebrate fish:** 13 protein-coding genes, 2 rRNAs (12S, 16S), 22 tRNAs. Missing or fragmented genes may indicate a misassembly or a genuinely unusual mitogenome and should be investigated.

---

## 8. Step 4 — Generate the final report

Once the pipeline and MITOS2 annotation are complete, generate a markdown report that summarises all results in one place:

```bash
bash generate_report.sh configs/<species>.conf
```

**Example:**
```bash
bash generate_report.sh configs/anguilla_marmorata.conf
```

The report is written to `results/<prefix>/report.md` and contains:

- Assembly statistics (contig length, coverage, circularity, QUAST metrics)
- BLAST species identification table (top 10 NCBI nt hits)
- MitoZ annotation summary and full gene table
- MITOS2 annotation summary and gene order (if `annotate_mitos2.sh` has been run)
- Index of all output files

The report can be opened in any markdown viewer, text editor, or converted to PDF/HTML. If MITOS2 has not yet been run, section 4 will display a prompt with the command to run it — the report can be regenerated at any time.

---

## 9. Interpreting outputs

### Assembly quality indicators

| Metric | Acceptable range | File |
|--------|-----------------|------|
| Contig length | 15,500–18,000 bp | `results/<prefix>/04_assembly/assembly_info.txt` |
| Circular | Y | `results/<prefix>/04_assembly/assembly_info.txt` |
| Coverage | > 50× (ideally 100–200×) | `results/<prefix>/04_assembly/assembly_info.txt` |
| Genome fraction | > 95% | `results/<prefix>/06_stats/report.txt` |
| Misassemblies | 0 | `results/<prefix>/06_stats/report.txt` |
| Mismatches per 100 kbp | < 1,000 | `results/<prefix>/06_stats/report.txt` |

### MitoZ circos plot

MitoZ names its result directory after the input filename, so the outputs are at:

```
results/<prefix>/07_mitoz/<prefix>.consensus.fasta.result/circos.png
results/<prefix>/07_mitoz/<prefix>.consensus.fasta.result/circos.svg
```

The GenBank annotation file is at:

```
results/<prefix>/07_mitoz/tmp_<prefix>_consensus.fasta_mitoscaf.fa/<prefix>_consensus.fasta_mitoscaf.fa.gbf
```

The circos plot shows the circular mitogenome with gene positions, strand orientation, and a GC content track. Both PNG (289 kb) and SVG (44 kb) are produced — open the SVG in any web browser or vector graphics editor for a scalable version.

### BLAST species identification

Results are in `results/<prefix>/08_blast/blast_results.tsv` with columns: rank, accession, identity %, alignment length, e-value, bitscore, description.

The top hit identity percentage is the key value to interpret:

| Identity % | Interpretation |
|-----------|----------------|
| ≥ 99% | Strong species-level match |
| 97–99% | Likely same species, minor sequence divergence |
| 95–97% | Probable congeneric species |
| < 95% | Genus-level or higher match only |

If the top BLAST hits disagree with the sample label, treat the BLAST result as the authoritative identification. A mislabelled sample will show all top hits belonging to a different genus or family than expected.

!!! warning
    MitoZ's "closely related species" in `summary.txt` is determined from its own limited internal database and is used to guide annotation — it is **not** a species identification. Always use the BLAST result for species ID.

### Gene order

The gene order string from MITOS2 is in `results/<prefix>/07_annotation/result.geneorder`. A `-` prefix indicates the gene is on the minus strand. Compare against published gene orders for the taxon to confirm the assembly is correctly oriented.

---

## 10. Adding a new species — quick checklist

- [ ] Obtain concatenated ONT reads (`.fastq.gz`)
- [ ] Find a RefSeq accession for a related mitogenome on NCBI
- [ ] Clone (or pull) the pipeline repo: `git clone http://172.20.142.126:3000/evilliers/mitogenome-pipeline.git`
- [ ] Copy and edit a config: `cp configs/anguilla_marmorata.conf configs/<new_species>.conf`
- [ ] Set all 7 config variables: `SPECIES_NAME`, `SPECIES_PREFIX`, `READS`, `REFERENCE_ACCESSION`, `GENOME_SIZE`, `FLYE_MODE`, `MEDAKA_MODEL`
- [ ] Run: `conda activate ont && bash run_pipeline.sh configs/<new_species>.conf`
- [ ] Check assembly quality (contig count, circularity, QUAST)
- [ ] Verify BLAST top hit matches expected species/genus (`results/<prefix>/08_blast/blast_results.tsv`)
- [ ] Run: `bash annotate_mitos2.sh configs/<new_species>.conf`
- [ ] Verify gene count (expect 13 CDS + 2 rRNA + 22 tRNA)
- [ ] Generate final report: `bash generate_report.sh configs/<new_species>.conf`
- [ ] Commit the new config file and push (or open a PR): `git add configs/<new_species>.conf && git commit -m "Add config: <species name>"`

---

## 11. Troubleshooting

**Very few reads after Step 3 (mito read enrichment)**
The reference may be too divergent from your species. Try a closer relative as the mapping reference. Check `seqkit stats` output after Step 3 — at minimum a few thousand reads are needed for assembly.

**Flye produces 0 contigs or multiple short contigs**
Coverage is likely too low. Check the read count from Step 3. Consider using `--nano-raw` if reads were basecalled without super-accuracy mode.

**QUAST shows low genome fraction or many misassemblies**
The reference may be from a distantly related species. This does not necessarily indicate a bad assembly — compare GC content and length against published mitogenomes for the taxon instead.

**MitoZ Singularity pull fails**
Check internet connectivity and available disk space (`df -h`). The `.sif` file is ~1 GB. Re-run the pipeline once the image has been downloaded manually:
```bash
singularity pull singularity/mitoz_3.6.sif docker://guanliangmeng/mitoz:3.6
```

**BLAST step times out or returns no results**
NCBI BLAST can be slow under heavy load. The step may take up to 15 minutes. If it fails, the pipeline will exit — re-run from Step 8 by calling the BioPython command directly, or submit `results/<prefix>/05_polished/consensus.fasta` manually to [NCBI BLAST](https://blast.ncbi.nlm.nih.gov) and save the top 10 hits as `results/<prefix>/08_blast/blast_results.tsv`.

**BLAST top hits do not match the sample label**
This is a strong indicator of sample mislabelling. Trust the BLAST result over the label. See section 9 (Interpreting BLAST results) for identity % thresholds. Note that MitoZ's "closely related species" is not a species identification — always use BLAST for this purpose.

**MITOS2 misses genes**
Try the [MITOS2 web server](http://mitos.bioinf.uni-leipzig.de) with the polished consensus as input (Reference: RefSeq 89 Metazoa, Genetic code: 2 — vertebrate mt). The web server uses the same algorithm but may give slightly different results with default settings.
