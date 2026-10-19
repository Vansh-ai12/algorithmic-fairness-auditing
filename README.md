# CSET485 – AI and Society: Assignment #1
**Audit, Stress-Testing, and Fairness Trade-Offs in Criminal Risk Scoring**

- **Assigned Dataset:** COMPAS Recidivism Risk Score Dataset
- **Random Seed:** `161` (70% Train, 30% Test split)
- **Primary Deliverables:** `report.pdf`, `report.docx`, `notebook.ipynb`

---

## 1. Project Directory Structure

```
c:/Projects/AI_Society_Proj/
│
├── data/
│   └── compas-scores-two-years.csv     # Filtered ProPublica COMPAS dataset (N=6,172)
│
├── figures/
│   ├── fairness_tradeoff.png          # DP vs EO Pareto trade-off curve across thresholds
│   ├── model_comparison_cm.png        # Confusion Matrix between Model A and Model B
│   ├── ovb_shift.png                  # Omitted-Variable Bias bar chart across predictors
│   ├── adversarial_priors_dist.png    # Prior conviction distribution across adversarial subsets
│   ├── fairness_metrics.csv           # Sweep metrics (DP, EO, PP, Acc, TPR, FPR)
│   ├── ovb_table.csv                  # Full vs Reduced model coefficients & % change
│   ├── adversarial_50.csv             # Sampled 50-case adversarial cohort
│   └── summary_stats.json             # Key aggregate statistics
│
├── notebook.ipynb                     # Fully reproducible Jupyter Notebook (no code comments)
├── report.docx                        # Complete academic & executive report with tables and figures
├── report.pdf                         # Rendered submission-ready PDF document
├── requirements.txt                   # Dependency manifest for pip
├── run_analysis.py                    # Standalone end-to-end Python script
└── README.md                          # Documentation and defense preparation guide
```

---

## 2. Environment Setup & Execution

### Activate Virtual Environment
```powershell
.\venv\Scripts\Activate.ps1
```

### Install Dependencies
```powershell
pip install -r requirements.txt
```

### Run Notebook or Standalone Script
```powershell
# To run Jupyter Lab or Notebook:
jupyter notebook notebook.ipynb

# Or to re-run the complete pipeline and regenerate all figures:
python run_analysis.py
python generate_plots.py
```

---

## 3. Key Computed Empirical Results (Ground Truth for Submission)

### Part 1: Model Comparison
- **Test Sample Size:** 1,852 individuals (30% holdout with seed 161)
- **Model A (OLS on decile_score):** Empirical median threshold = **4.4197**
- **Model B (Logistic Regression on recidivism):** Cutoff = **0.50**
- **Confusion Matrix:**
  - Low / Low: **906**
  - Model A Low / Model B High: **17**
  - Model A High / Model B Low: **282**
  - High / High: **647**
- **Total Mismatches:** **299** out of 1,852 (**16.14%**)

### Part 2.1: Simpson's Paradox / Subgroup Heterogeneity
- **Youth cohort (< 25):** `priors_count` coefficient = **+0.2893** ($p = 2.55 \times 10^{-11}$, 95% CI: [0.2043, 0.3743])
- **Mature cohort (>= 25):** `priors_count` coefficient = **+0.1528** ($p = 3.45 \times 10^{-70}$, 95% CI: [0.1359, 0.1697])
- **Divergence:** Non-overlapping confidence intervals indicate a single prior doubles recidivism log-odds for youth compared to adults.
- **Sign Reversal:** `juv_misd_count` is **+0.1072** for youth but flips to **-0.0575** for mature defendants.

### Part 2.2: Omitted-Variable Bias (Dropping priors_count)
| Covariate | Full Model Coeff | Reduced Model Coeff | Absolute Shift | % Change |
|---|---|---|---|---|
| `age` | -0.0465 | -0.0308 | +0.0157 | +33.88% |
| `sex_male` | +0.2766 | +0.4373 | +0.1607 | +58.08% |
| `charge_felony` | +0.2192 | +0.3820 | +0.1628 | +74.28% |
| `race_black` | +0.0799 | +0.4126 | +0.3327 | **+416.30%** |
- **Insight:** Omitting `priors_count` causes `race_black` to act as an unshielded proxy for criminal history, artificially inflating its predictive weight by over 5x.

### Part 2.3: Adversarial Subset Construction
- **High-Confidence Failures:** **96** cases (**5.18%** of test set)
  - **False Positives ($P \ge 0.75, Y=0$):** 34 cases (Mean priors = **14.68**)
  - **False Negatives ($P \le 0.25, Y=1$):** 62 cases (Mean priors = **0.77**)
- **Adversarial Subset Mean Age:** **48.10 years** (vs. 34.49 years full test set)
- **Failure Mechanism:** The monotonic linear logit specification fails to account for age-related criminal desistance in older individuals with high past convictions, while ignoring unobserved triggers in individuals with zero priors.

### Part 2.4: Fairness Trade-Offs (Black vs. White Defendants)
- **t = 0.15:** DP = 1.059, EO = 1.027, Accuracy = 47.79% (Black FPR = 98.66%, White FPR = 91.41%)
- **t = 0.35:** DP = 1.538, EO = 1.270, PP = 1.113, Accuracy = 61.05% (Black FPR = 73.05%, White FPR = 42.24%)
- **t = 0.45:** DP = 1.894, EO = 1.518, PP = 1.081, Accuracy = 66.27% (Black FPR = 44.77%, White FPR = 21.00%)
- **Trade-off Cost:** Moving from t=0.45 to t=0.35 gains 18.80% in DP at the expense of a 16.34% loss in EO (Trade-off ratio = 0.869).
