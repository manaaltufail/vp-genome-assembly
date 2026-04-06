# VGP Genome Assembly Pipeline
## De novo Assembly of Saccharomyces cerevisiae S288C

---

 Table of Contents
1. [Project Overview](#overview)
2. [Data Used](#data)
3. [Pipeline Steps](#pipeline)
4. [Results](#results)
5. [Tools & Versions](#tools)
6. [References](#references)



 Project Overview
This project performs a high-quality de novo 
genome assembly of the yeast Saccharomyces 
cerevisiae S288C using the Vertebrate Genome 
Project (VGP) pipeline on the Galaxy platform.

The goal is to generate a chromosome-level, 
haplotype-phased genome assembly using:
- PacBio HiFi long reads
- Illumina Hi-C chromatin conformation data
- Bionano optical maps



 Data Used
| Data Type | Source | Details |
|-----------|--------|---------|
| PacBio HiFi reads | Zenodo 6098306 | 3 files, ~50x coverage, FASTA format |
| Illumina Hi-C reads | Zenodo 5550653 | Forward + Reverse, fastqsanger.gz |
| Bionano optical maps | Zenodo 5887339 | CMAP format |



 Pipeline Steps

Step 1: Quality Control — Cutadapt
Tool: Cutadapt v4.4
Purpose: Remove PacBio adapter sequences 
from HiFi reads to improve assembly quality.

Adapters removed:
- First: `ATCTCTCTCAACAACAACAACGGAGGAGGAGGAAAAGAGAGAGAT`
- Second: `ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT`

Settings:
- Maximum error rate: 0.1
- Minimum overlap: 35bp
- Reads with adapters: DISCARDED

Output: 3 cleaned FASTA files (HiFi collection trimmed)

---

 Step 2: K-mer Analysis — Meryl
Tool: Meryl v1.3
Purpose: Count k-mer frequencies in reads 
to build a k-mer database for genome profiling.

Three runs:
1. Count k-mers (k=31) in each HiFi file
2. Merge all databases (Union-sum)
3. Generate frequency histogram

Why k=31?
Long enough to avoid repetitive k-mers,
short enough to be robust to sequencing errors.

Output: K-mer frequency histogram



 Step 3: Genome Profiling — GenomeScope2
Tool: GenomeScope2 v2.0
Purpose: Estimate genome properties from 
k-mer histogram using statistical modeling.

Settings:
- Ploidy: 2 (diploid)
- K-mer size: 31

Results:
![GenomeScope Linear Plot](images/01_genomescope_linear.png)
![GenomeScope Log Plot](images/02_genomescope_log.png)

| Property | Value |
|----------|-------|
| Genome haploid length | ~11.7 Mb |
| Heterozygosity | 0.576% |
| Homozygosity | 99.4% |
| Model fit | 92-96% |
| Read error rate | 0.00009% |

---

 Step 4: Genome Assembly — Hifiasm
Tool: Hifiasm v0.19.8
Mode: Hi-C phased assembly
**Purpose:** Assemble HiFi reads into contigs 
using Hi-C data for haplotype phasing.

Why Hi-C phased mode?
Hi-C data from the same individual allows 
hifiasm to separate the two haplotypes 
(hap1 and hap2) more accurately than 
solo mode.

Output:
- Hap1 contigs graph (GFA format)
- Hap2 contigs graph (GFA format)

---

Step 5: Assembly Evaluation — gfastats
Tool: gfastats v1.3.6
Purpose:Convert GFA to FASTA and generate 
assembly statistics.

Results:
| Statistic | Hap1 | Hap2 |
|-----------|------|------|
| # Contigs | 16 | 17 |
| Average contig length | 706,536 bp | 715,352 bp |
| Expected genome size | 11,747,160 bp | 11,747,160 bp |

Hap1 has exactly 16 contigs matching the 
16 chromosomes of S. cerevisiae! 

---

 Step 6: Quality Assessment — Merqury
Tool: Merqury v1.3
Purpose: Reference-free quality assessment 
using k-mer analysis.

Results:
![Merqury CN Plot](images/04_merqury_cn_plot.png)
![Merqury ASM Plot](images/05_merqury_asm_plot.png)

| Assembly | QV Score | Meaning |
|----------|----------|---------|
| Hap1 | 79.74 | 1 error per 100M bases! |
| Hap2 | 79.74 | 1 error per 100M bases! |
| Both | 79.74 | Exceptional quality! |

Note: Expected QV was ~40. 
Achieved QV of 79.74 far exceeds expectations! 

---

 Step 7: Bionano Hybrid Scaffolding
Tool: Bionano Hybrid Scaffold v3.7.0
Purpose:Use optical maps to order and 
orient contigs into larger scaffolds.

Settings:
- Configuration: VGP mode
- Conflict filter: Cut contig at conflict

Output: Hap1 assembly bionano (16 sequences, 11MB)

---

Step 8: Hi-C Scaffolding
Tools: BWA-MEM2 v2.2.1 + YaHS v1.2a.2
Purpose: Use Hi-C chromatin data to 
scaffold contigs to chromosome level.

Process:
1. Map Hi-C forward reads → BAM forward
2. Map Hi-C reverse reads → BAM reverse
3. Filter and merge BAM files
4. YaHS scaffolding with restriction enzyme CTTAAG

Why map forward and reverse separately?
Hi-C read pairs can come from different 
chromosomes, so standard paired mapping 
assumptions don't apply!

---

Step 9: Final Evaluation — PretextMap
Tools: PretextMap v0.1.9 + Pretext Snapshot v0.0.3
Purpose:Generate Hi-C contact maps to 
visually evaluate chromosome-level assembly.

Initial contact map (before YaHS):
![Initial Contact Map](images/06_initial_contact_map.png)

Final contact map (after YaHS):
![Final Contact Map](images/07_final_contact_map.png)

Strong diagonal signals confirm correct 
chromosome-level scaffolding! 

---

Results Summary

| Metric | Value | Status |
|--------|-------|--------|
| Genome size | ~11.7 Mb |  Matches reference |
| Hap1 contigs | 16 |  Matches chromosomes |
| Hap2 contigs | 17 |  Expected |
| QV Score | 79.74 |  Exceptional |
| Heterozygosity | 0.576% | Normal for yeast |
| Model fit | 92-96% |  Good fit |
| Assembly level | Chromosome |  Goal achieved |

---
Tools & Versions

| Tool | Version | Purpose |
|------|---------|---------|
| Cutadapt | 4.4+galaxy0 | Adapter trimming |
| Meryl | 1.3+galaxy6 | K-mer counting |
| GenomeScope2 | 2.0+galaxy2 | Genome profiling |
| Hifiasm | 0.19.8+galaxy0 | Genome assembly |
| gfastats | 1.3.6+galaxy0 | Assembly stats |
| Merqury | 1.3+galaxy3 | Quality assessment |
| Bionano Hybrid Scaffold | 3.7.0+galaxy3 | Optical map scaffolding |
| BWA-MEM2 | 2.2.1+galaxy1 | Hi-C read mapping |
| Filter and merge | 1.0+galaxy1 | BAM processing |
| YaHS | 1.2a.2+galaxy1 | Hi-C scaffolding |
| PretextMap | 0.1.9+galaxy0 | Contact map generation |
| Pretext Snapshot | 0.0.3+galaxy1 | Contact map visualization |

Platform: Galaxy (usegalaxy.org)



References
1. VGP Tutorial: https://training.galaxyproject.org
2. Rhie et al. 2020 - VGP pipeline
3. Cheng et al. 2021 - Hifiasm
4. Ranallo-Benavidez et al. 2020 - GenomeScope2
5. Zhou et al. 2022 - YaHS
6. Formenti et al. 2022 - gfastats
7. Rhie et al. 2020 - Merqury

---

 Platform Note
This pipeline was run on Galaxy public server 
(usegalaxy.org). Some steps experienced 
queue delays due to high server load on 
shared public computing resources.

---
Assembly completed: April 2026
Platform: Galaxy (usegalaxy.org)
