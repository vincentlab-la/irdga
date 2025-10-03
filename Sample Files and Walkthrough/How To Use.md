# How to Use the SNV and SV Pipelines

This repository provides pipelines for analyzing **single nucleotide variants (SNVs)**, **structural variants (SVs)**, and applying **inheritance filtering**. These tools are designed to facilitate variant discovery in inherited retinal dystrophies (IRDs) and other rare diseases.  

---

## 1. SNV Pipeline  

The SNV pipeline filters, scores, and ranks small variants (SNVs and indels).  

### Input Requirements  
Your input file must be **tab-delimited** and contain all required SNV columns outlined in the README.  

**Example SNV Input (excerpt):**  

```
gene_symbol    FILTER    AD_REF    AD_ALT    1000g_all    typeseq_priority    spliceAI_DS_AG    spliceAI_DS_AL    spliceAI_DS_DG    spliceAI_DS_DL    Clinvar_SIG    Clinvar_ReviewStatus    effect_priority    sift_score    polyphen_score    ma_score    spx_dpsi    phylopMam_avg    phylopVert100_avg    PROVEAN_score    CADD_phred    REVEL_score    MPO    HPO
ABCA4          PASS      35        12        0.0001       missense_variant    0.02              0.00              0.00              0.01              Pathogenic     reviewed_by_expert      missense           0.01          0.95             1.2        -0.05        1.75            2.41                -2.3             25.6          0.86           MP:0005390    HP:0000548
```

### Output  
In addition to the above columns, the SNV pipeline adds three new fields:  

```
gene_symbol    FILTER    AD_REF    AD_ALT    ...    sAI_delta    newDP    VariantScore
ABCA4          PASS      35        12        ...    0.42         47       6.5
```

- `sAI_delta` — combined SpliceAI prediction score  
- `newDP` — normalized read depth across reference and alternate alleles  
- `VariantScore` — cumulative variant score used for ranking  

---

## 2. SV Pipeline  

The SV pipeline processes structural variants and prepares them for integration with SNVs.  

### Input Requirements  
The pipeline accepts SV calls from:  
- **MANTA** (example input file provided in this repository)  
- **CNVnator**  
- Other SV callers (e.g., DELLY), provided the required information is present.  

**Required SV Columns:**  
- `CHROM`  
- `Start` (or `START`)  
- `Stop` (or `END`)  
- `gene_symbol`  

⚠ **Note:**  
- Column names do not have to match exactly — the pipeline will prompt you to map them.  
- If population frequency columns are missing, filtering may not run. You can annotate with gnomAD-SV, 1000 Genomes, or other SV frequency data if needed.  

**Example SV Input (MANTA excerpt):**  

```
Sample    CHROM    START    END      SVTYPE    REF    ALT     SVLEN    QUAL    FILTER    gene_symbol
Ex1       chr1     789481   789481   INS       G      <INS>   266      999     PASS      LINC01409
Ex2       chr1     934098   934898   DEL       G      <DEL>   800      339     PASS      SAMD11
```

### Output  
- Filtered SV file (same structure as input).  
- No new scoring columns are added.  


---

## 3. Inheritance Filtering  

The inheritance filtering utility is an **interactive R workflow** that integrates SNV and SV outputs and applies disease‑model rules. It operates **by gene name**, so your files must contain a `gene_symbol` column.

### What the script does
- Loads **CSV/TSV/XLS/XLSX** inputs; lets you pick **multiple files and (for Excel) specific sheets**.
- If any row has multiple genes, it **splits** them into separate rows via `separate_rows(gene_symbol, sep = ",")`.
- Supports **any column layouts** and safely binds heterogeneous tables.
- Writes all results to a **single Excel workbook** with multiple sheets and summaries.
- Prompts you to choose **inheritance mode**: Recessive or Dominant.

### Required / Helpful columns
- **Required:** `gene_symbol` (for matching).
- **Helpful (if present):**  
  - `Zygosity` (SNV): `hom-alt` flags homozygous variants.  
  - `GT_PreNorm` (SV): `1/1` or `1|1` flags homozygous SVs when `Zygosity` is absent.  
  - `VariantScore` (SNV): enables per‑gene score summaries.  
  - `CHROM`, `SV_Start`, `SV_Stop` (SV): when present, SV rows are grouped by unique `(CHROM, SV_Start, SV_Stop)` intervals during comparisons.

### Recessive mode (what is retained)
Within each loaded sheet, the script keeps genes that satisfy **any** of:
- **Duplicate gene hits** (≥2 variants in the same `gene_symbol`).  
- **Homozygous variants** in SNVs where `Zygosity == "hom-alt"`.  
- **Homozygous SVs** where `GT_PreNorm` is `1/1` or `1|1` (used when `Zygosity` is not available).  

If `VariantScore` exists, two **gene‑level aggregates** are appended beside each variant row:
- `GeneScore_AvgAll` — average of all `VariantScore` values per gene.
- `GeneScore_AvgTop2` — average of the top two `VariantScore` values per gene.

The script also computes **gene overlaps between inputs** (e.g., SNV vs SV): it creates per‑comparison sheets listing shared `gene_symbol` values and the matched rows from each source.

**Recessive output file name:** `{SampleNumber}_Inheritance_Filter_Recessive.xlsx` (auto‑increments to avoid overwriting).

### Dominant mode (what is retained)
- Applies a **population frequency filter** to the selected SNV sheet (if available):  
  keeps variants where both `A1000g_freq_max ≤ 0.0003` **and** `freq_max ≤ 0.0003` (i.e., **≤ 0.03%**).  
- Writes all resulting sheets into a single workbook plus a summary.

**Dominant output file name:** `{SampleNumber}_Inheritance_Filter_Dominant.xlsx` (auto‑increments to avoid overwriting).

### What the Excel output contains
- **Individual recessive candidate sheets** per input (suffix `_recessive`).  
- **Gene overlap comparison sheets** for every pairwise input combination.  
- **Analysis_Statistics**: per‑sheet metrics (variant counts, genes involved, retained counts).  
- **Summary_Statistics**: overall counts (sheets processed, total variants, unique genes, biallelic totals).  
- **All_Unique_Genes**: a de‑duplicated list of all retained `gene_symbol` entries (when any exist).

### Minimal input for this step
- The only mandatory column is **`gene_symbol`**. Additional columns enable richer filtering and summaries as described above.


## Workflow Summary  

1. Prepare SNV and/or SV input files with the required columns.  
2. Run the **SNV pipeline** → produces scored SNV file.  
3. Run the **SV pipeline** → produces filtered SV file.  
4. Merge SNV + SV outputs.  
5. Apply **inheritance filtering** → produces AR, AD, and merged output files.  
6. Review the resulting variant and gene candidates.  
