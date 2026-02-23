# Assessment: mothur+BLAST Pipeline for Marine eDNA Analysis

**Date**: January 13, 2026
**Assessed SOP**: eDNA_mothur_and_BLAST_analysis_SOP_v3.md
**Context**: Marine environmental DNA metabarcoding

---

## Executive Summary

The mothur+BLAST pipeline documented in v3 of the SOP is a **functional but somewhat dated** approach for marine eDNA analysis. While it provides thorough quality control and works well with custom databases, it lacks several modern best practices that have become standard in the field as of 2026.

**Overall Assessment**: ⚠️ **Adequate but not state-of-the-art**

---

## Strengths ✓

### 1. Thorough Quality Control
- Excellent sequence quality filtering via mothur
- Comprehensive primer removal
- Robust chimera detection using VSEARCH
- Multiple quality checkpoints throughout workflow

### 2. Custom Database Support
- Works well with specialized/curated reference databases
- Good for taxonomic groups not well-represented in public databases
- Flexible BLAST-based approach

### 3. Clear Documentation
- Well-documented workflow with detailed explanations
- Reproducible methodology
- Good troubleshooting guidance

### 4. Conservative Approach
- Strict quality filtering ensures high-confidence results
- Reduces false positive detections
- Appropriate for presence/absence studies

---

## Concerns for Marine eDNA (2026 Context)

### 1. Tool Selection Issues

**mothur** was originally designed for bacterial 16S rRNA microbial community analysis, not marine metazoan eDNA.

**More appropriate alternatives for fish/invertebrate eDNA**:
- **OBITools**: Specifically designed for metabarcoding workflows
- **DADA2**: Generates Amplicon Sequence Variants (ASVs) with sophisticated error correction
- **QIIME2**: Comprehensive metabarcoding platform with extensive plugin ecosystem
- **Cutadapt + VSEARCH/UNOISE3**: Modern quality filtering and denoising

### 2. Taxonomic Assignment Limitations

The BLAST + hard threshold method is **overly simplistic**:

```
Current approach:
- ≥99% identity → Species level
- 95-98.9% identity → Family level
- <95% identity → NO MATCH
```

**Problems**:
- No consideration of multiple equally-good matches
- Ignores alignment coverage and quality
- Binary thresholds don't reflect biological reality
- No confidence scores or uncertainty quantification

**Better alternatives**:
- **LCA (Lowest Common Ancestor)** algorithms: Conservative consensus approach
- **SINTAX** or **RDP classifier**: Probabilistic assignment with confidence scores
- **BLAST + phylogenetic placement**: Tree-based taxonomic inference
- **Database-specific tools**: BOLD Identification Engine, MIDORI2 classifiers

### 3. Missing Modern Best Practices

The SOP lacks several components now considered essential for marine eDNA studies:

| Missing Element | Why It Matters | Impact |
|----------------|----------------|---------|
| **PCR technical replicates** | Detect stochastic PCR dropout of rare species | May miss rare/sporadic taxa |
| **Occupancy modeling** | Account for imperfect detection probability | Overestimate true absence |
| **Index-hopping correction** | Critical for Illumina NovaSeq/NextSeq chemistry | False positives in samples |
| **Rarefaction/accumulation curves** | Assess sampling completeness | Unknown if diversity is fully captured |
| **Tag-jump filtering** | Remove low-abundance cross-contamination | False positives from abundant samples |
| **Positive controls** | Validate taxonomic assignment accuracy | No verification of pipeline performance |
| **Read depth normalization** | Account for varying sequencing depth | Biased abundance comparisons |
| **Blank subtraction algorithms** | Statistically robust contamination filtering | May retain or over-remove contaminants |

### 4. Database Considerations

**No mention of standard marine databases**:
- **MIDORI2**: Most comprehensive marine eukaryote reference database (2+ million sequences)
- **BOLD** (Barcode of Life Data System): Gold standard for COI barcoding
- **GenBank**: Needs careful curation but valuable for validation

**Current approach**: Relies entirely on custom database without validation against public databases.

**Risk**: Misidentifications if custom database has errors or gaps.

---

## State-of-the-Art Marine eDNA Pipeline (2026)

### Recommended Modern Workflow

```
┌─────────────────────────────────────────────────────────────┐
│              MODERN MARINE eDNA PIPELINE                    │
└─────────────────────────────────────────────────────────────┘

1. EXPERIMENTAL DESIGN
   ├─ PCR technical replicates (3x per sample)
   ├─ Negative controls (extraction + PCR blanks)
   ├─ Positive controls (mock communities)
   └─ Adequate sequencing depth (>50,000 reads/sample)

2. QUALITY CONTROL & DENOISING
   ├─ Cutadapt or Trimmomatic: Remove primers/adapters
   ├─ DADA2 or UNOISE3: Error correction & ASV generation
   └─ No arbitrary length/quality cutoffs (algorithm-based)

3. CHIMERA REMOVAL
   └─ DADA2 (de novo + reference-based) or VSEARCH

4. TAXONOMIC ASSIGNMENT (Multi-tier approach)
   ├─ Primary: BLAST/VSEARCH against MIDORI2 or BOLD
   ├─ Algorithm: LCA or SINTAX with confidence scores
   ├─ Threshold: >97% identity, >90% coverage
   └─ Validation: Cross-check against GenBank

5. CONTAMINATION FILTERING
   ├─ Subtract negative control taxa (with statistical thresholds)
   ├─ Tag-jump correction (remove <0.03% reads per sample)
   ├─ Filter low-abundance ASVs (<0.1% per sample or <10 reads)
   └─ Remove singletons/doubletons globally

6. REPLICATION ANALYSIS
   ├─ Check concordance across PCR replicates
   ├─ Apply occupancy modeling (multi-scale if available)
   └─ Define detection thresholds (e.g., 2/3 replicates)

7. DIVERSITY & STATISTICAL ANALYSIS
   ├─ Rarefaction curves (assess sampling effort)
   ├─ Alpha diversity (richness, Shannon, etc.)
   ├─ Beta diversity (Bray-Curtis, Jaccard, UniFrac)
   ├─ Multivariate statistics (PERMANOVA, NMDS, etc.)
   └─ Indicator species analysis

8. REPORTING & VALIDATION
   ├─ Report ASV sequences and abundances
   ├─ Provide taxonomy tables with confidence scores
   ├─ Include negative/positive control results
   ├─ Document filtering thresholds and rationale
   └─ Deposit data in public repositories (NCBI SRA, DRYAD)
```

---

## When Current SOP Is Appropriate

### ✅ **Use this pipeline if**:

1. **Maintaining consistency** with previous studies using identical methods
2. **Custom database** is highly specialized, curated, and validated
3. **Target organisms** are well-known with excellent reference sequences
4. **Simple presence/absence** is the primary goal (not quantitative)
5. **Conservative results** preferred over maximum sensitivity
6. **Resources limited** and modern pipeline training not feasible
7. **Peer-reviewed precedent** exists for this approach in your system

### ⚠️ **Reconsider if**:

1. **Rare or cryptic species** detection is important
2. **High-impact publication** planned (reviewers expect modern methods)
3. **Poorly-characterized taxa** are targets
4. **Quantitative abundance** estimates needed
5. **Cross-study comparisons** with ASV-based studies required
6. **Contamination** is a major concern (urban/coastal sites)
7. **Grant/regulatory requirements** specify modern standards

---

## Specific Recommendations for Improvement

### Priority 1: High Impact, Moderate Effort

1. **Add ASV-based analysis**
   - Implement DADA2 pipeline in parallel
   - Compare results to validate consistency
   - Transition to ASVs for future work

2. **Improve taxonomic assignment**
   - Use LCA algorithm instead of hard thresholds
   - Add confidence scores to all assignments
   - Cross-validate against MIDORI2/BOLD

3. **Implement contamination filtering**
   - Develop blank subtraction algorithm
   - Filter low-abundance reads (<0.1% per sample)
   - Document all filtering decisions

### Priority 2: Medium Impact, Low Effort

4. **Include technical replicates**
   - Run 2-3 PCR replicates per sample
   - Report detection frequency
   - Apply simple occupancy thresholds (2/3 rule)

5. **Add positive controls**
   - Use mock communities or tissue samples
   - Validate taxonomic assignment accuracy
   - Check for primer bias

6. **Generate rarefaction curves**
   - Assess sampling completeness
   - Determine if additional sequencing needed
   - Report in supplementary materials

### Priority 3: Lower Priority, High Effort

7. **Implement occupancy modeling**
   - Use multi-scale occupancy models
   - Account for imperfect detection
   - Requires statistical expertise

8. **Develop integrated database**
   - Merge custom database with MIDORI2/BOLD
   - Version control and document provenance
   - Regularly update with new sequences

---

## Alternative Pipelines to Consider

### For General Marine eDNA (Fish, Invertebrates)

**DADA2 Pipeline** (Most popular in 2026)
- **Pros**: ASV-based, excellent error correction, well-documented, R-based integration
- **Cons**: Computationally intensive, requires R proficiency
- **Best for**: Diversity studies, cross-study comparisons, publication in top journals

**OBITools Pipeline**
- **Pros**: Designed specifically for metabarcoding, excellent for length-variable markers
- **Cons**: Python-based (different ecosystem), less active development
- **Best for**: COI, 18S, other variable-length markers

**QIIME2 Pipeline**
- **Pros**: Comprehensive ecosystem, many plugins, great visualization
- **Cons**: Steep learning curve, overkill for simple projects
- **Best for**: Complex studies, multiple marker genes, integration with microbiome data

### For Specific Applications

**Fish eDNA**: DADA2 + MiFish/12S database + FishTaxa classifier
**Marine Mammals**: DADA2 + custom cetacean/pinniped database
**Zooplankton**: OBITools + BOLD COI database
**Harmful Algae**: DADA2 + PR2 database (18S)

---

## Modern Marine eDNA Resources (2026)

### Key Publications

1. **Gold et al. (2023)** - "Synthesizing eDNA metabarcoding standards for aquatic biodiversity"
2. **Stoeckle et al. (2024)** - "Best practices for marine fish eDNA metabarcoding"
3. **Marques et al. (2023)** - "Marine metabarcoding: A practical guide from sampling to data analysis"

### Databases

- **MIDORI2**: http://reference-midori.info (Marine eukaryote reference)
- **BOLD**: http://www.boldsystems.org (COI barcode database)
- **FishBase**: Integration with genetic data
- **PR2**: Protist 18S rRNA database

### Software Tools

- **DADA2**: https://benjjneb.github.io/dada2/
- **QIIME2**: https://qiime2.org/
- **OBITools**: https://pythonhosted.org/OBITools/
- **ANACAPA**: Toolkit specifically for eDNA metabarcoding
- **CRABS**: Creating Reference databases for Amplicon-Based Sequencing

### Standardization Efforts

- **eDNA Collaborative**: https://www.ednacollab.org/
- **eDNA Society**: Standards and best practices working group
- **EMBRC**: European Marine Biological Resource Centre protocols

---

## Validation Recommendations

If you choose to continue with the current mothur+BLAST pipeline, **validate it** by:

1. **Run mock communities** with known composition
   - Compare detected vs. expected species
   - Quantify false positives/negatives
   - Assess taxonomic resolution accuracy

2. **Parallel analysis with modern pipeline**
   - Process subset of samples with DADA2
   - Compare species lists and relative abundances
   - Document concordance and discrepancies

3. **Cross-validate taxonomic assignments**
   - BLAST top hits against NCBI GenBank
   - Check for conflicting identifications
   - Review low-confidence assignments manually

4. **Test contamination filtering**
   - Examine negative control composition
   - Develop sample-specific blank subtraction thresholds
   - Document contamination rates

5. **Assess reproducibility**
   - Re-run subset of samples from raw data
   - Check consistency across runs
   - Document which steps introduce variability

---

## Conclusion

The mothur+BLAST pipeline documented in your SOP v3 is a **solid, conservative approach** that will produce reliable presence/absence data for marine eDNA samples. However, it represents **2015-2020 era methodology** rather than current best practices.

### Key Decision Points:

**Stick with current pipeline if**:
- Maintaining consistency with existing dataset
- Resources/time limited
- Simple species detection is sufficient
- Custom database is critical

**Upgrade to modern pipeline if**:
- Starting new projects
- Publishing in high-impact journals
- Quantitative data needed
- Comparing with other recent studies
- Grant/regulatory requirements

### Recommended Path Forward:

1. **Short term**: Use current pipeline with enhanced contamination filtering and positive controls
2. **Medium term**: Implement DADA2 pipeline in parallel and compare results
3. **Long term**: Transition to ASV-based workflow with multi-database taxonomic assignment

The good news: Your detailed SOP provides an excellent foundation. Many of the same principles (quality control, chimera removal, careful data interpretation) apply regardless of the specific tools used.

---

**Document prepared**: January 13, 2026
**Next review recommended**: When starting new project or before manuscript submission
**Contact**: Refer to lab bioinformatician or eDNA working group for pipeline updates
