# UNSW-NB15 Intrusion Detection: Reproduction and Extension

This repository reproduces the binary-classification comparison in Omarov et al. (2024) and evaluates two extensions on the official UNSW-NB15 split:

1. **Binary intrusion detection:** class-balanced LightGBM with a decision threshold calibrated using Youden's J on duplicate-aware out-of-fold predictions.
2. **Ten-class attack categorization:** LightGBM with training-only SMOTENC, evaluated separately from the binary task.

The project also includes data-quality and distribution-shift audits, per-category diagnostics, statistical comparisons, feature reduction, runtime measurements, plots, and a Google Colab notebook.

## Paper

**Improving UNSW-NB15 Network Intrusion Detection Through Cost-Sensitive Gradient Boosting and Threshold Calibration**

**Authors:** Muhammad Shariq, Saud Nasir, and Rohma Shaikh

- Paper source: [`paper/paper.tex`](paper/paper.tex)
- Compiled paper: [`Improving_UNSW_NB15_Network_Intrusion_Detection_Through_Cost_Sensitive_Gradient_Boosting_and_Threshold_Calibration.pdf`](Improving_UNSW_NB15_Network_Intrusion_Detection_Through_Cost_Sensitive_Gradient_Boosting_and_Threshold_Calibration.pdf)
- Figures: [`paper/roc.pdf`](paper/roc.pdf), [`paper/percat.pdf`](paper/percat.pdf)

The binary and ten-class experiments use different targets and must be reported as separate tasks. The binary target is `label` (normal vs. attack); the multiclass target is `attack_cat` (ten categories).

## Main results in the paper

All values below are from the five-seed runs on the official test partition. The cited paper's reported accuracy and this project's reproduced scores are different references.

| Task and model | Accuracy | Macro-F1 | False-alarm rate | AUC |
|---|---:|---:|---:|---:|
| Binary RF reproduction, default threshold | 87.06% ± 0.06 | 86.44% | 26.97% | 97.85% |
| Binary proposed class-balanced GBDT + grouped-CV Youden-J threshold | **90.97% ± 0.10** | **90.74%** | **16.28%** | **98.58%** |
| Ten-class RF reproduction | 75.43% ± 0.08 | 46.51% ± 0.21 | — | — |
| Ten-class LightGBM + SMOTENC | **76.59% ± 0.12** | **51.35% ± 0.49** | — | — |

The binary improvement trades a lower false-alarm rate for a higher miss rate (3.12% for the proposed detector versus 1.49% for RF). In the ten-class task, performance does not improve for every class; see the per-class results, especially the Shellcode F1 change, before making class-specific claims. UNSW-NB15 is general network traffic and should not be described as an IoT-only dataset.

## Dataset

Download the official partition files from the [UNSW-NB15 dataset page](https://research.unsw.edu.au/projects/unsw-nb15-dataset) and place them here:

```text
Dataset/UNSW_NB15_training-set.csv
Dataset/UNSW_NB15_testing-set.csv
```

The expected files contain 175,341 training rows and 82,332 test rows. The scripts check these sizes. Do not replace the official split with a random re-split if you want to reproduce the reported experiment.

UNSW grants free use of the dataset for academic research and requires citation of the dataset papers; commercial use requires permission from its authors. Review the [dataset provider's current terms and citation requirements](https://research.unsw.edu.au/projects/unsw-nb15-dataset) before using or redistributing the data. The dataset is not covered by this repository's MIT software license.

## Environment setup

The Colab notebook records the following package versions for its reproduction environment: scikit-learn 1.9.1, LightGBM 4.7.0, and imbalanced-learn 0.14.2. The result artifacts also record NumPy, pandas, SciPy, and Python versions. Runtime and small numeric differences may vary by hardware and package versions.

Example setup from the repository root (PowerShell):

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install numpy pandas scipy matplotlib scikit-learn==1.9.1 lightgbm==4.7.0 imbalanced-learn==0.14.2
```

On macOS/Linux, activate with `source .venv/bin/activate` after creating the virtual environment. The repository also provides [`requirements-multiclass.txt`](requirements-multiclass.txt) for the multiclass dependencies, but the command above installs the full set used by the scripts and plots.

## Reproduce the paper results

Run the scripts from the `src/` directory. This is important because a few analysis scripts read or write files using paths relative to that working directory.

From the repository root in PowerShell:

```powershell
Set-Location src

# Reproduce six non-SVM binary baselines.
python baseline.py KNN "Naive Bayes" DT "Logistic Regression" AdaBoost "Random Forest"

# Run the stratified 40,000-row RBF-SVM and Naive Bayes encoding sensitivity.
python svm_sub.py

# Run grouped-CV binary extension, ablations, and the five-seed multiclass RF reference.
python extended.py

# Align the saved RF default-threshold results with predict() tie handling.
python patch_rf.py

# Run ten-class LightGBM + SMOTENC. This uses the RF reference from extended.json.
python multiclass_smote.py

# Supplementary results and paper figures.
python timing.py
python fp_analysis.py
python figs.py
python summarize.py
```

The baseline script supports selecting individual models by name. Running it without arguments runs every model, including the full-data RBF-SVM, which can take substantially longer than the stratified 40,000-row SVM used for the paper comparison. The `extended.py` and SMOTENC runs are compute-intensive; allow time for grouped cross-validation and repeated fits. The test partition is for final reporting, not model or threshold selection. The multiclass sampler is fitted only inside training folds and on the full training partition for final fits; it never resamples the test rows.

The scripts write JSON/CSV/text outputs under `results/` and figures under `figs/`. To rerun the full workflow cleanly, archive any existing outputs first; scripts may overwrite files with the same names.

## Google Colab notebooks

The repository contains two notebooks for different stages of the work. They are alternatives; choose the one matching the experiment you want to run.

### Original paper reproduction

Open [`reproduce_unsw_nb15_code.ipynb`](reproduce_unsw_nb15_code.ipynb) in Colab and run the cells in order. This notebook follows the original paper's binary-classification workflow: data cleaning and preprocessing, seven classifiers, comparison with the paper's Table 1, metric-consistency checks, plots, and execution-time measurements.

By default, `DATA_SOURCE = "upload"` and `USE_DRIVE = False`. Upload the official training and testing CSVs when prompted. Results are stored in the notebook's local `results/` folder, then packaged as `paper_method_results.zip` and downloaded to your computer. The default settings do not save results to Drive.

### Final enhanced reproduction and extension

Open [`UNSW_NB15_Reproduction_and_Extension_Colab.ipynb`](UNSW_NB15_Reproduction_and_Extension_Colab.ipynb) in Colab to run the final enhanced workflow. It covers the binary baselines and cost-sensitive GBDT extension, grouped-CV threshold calibration, statistical/per-category diagnostics, compact-feature and runtime analysis, ten-class SMOTENC, and optional data-shift and false-alarm scenarios.

This notebook reads the official CSVs from `MyDrive/UNSW_NB15/Dataset/`; Drive is mounted for input access only. Generated results remain in the temporary runtime folder `/content/UNSW_NB15_colab_results/`, not on Drive. The final notebook section packages those outputs into a ZIP download. Download it before the runtime ends because Colab's local files are temporary.

The enhanced notebook includes optional switches for long sections, including binary baselines, the compact-feature/runtime experiment, and shift/scenario diagnostics. Read the section notes before running all cells; the main grouped-CV and SMOTENC workflows can take a long time.

## Results and artifacts

- `results/baseline_KNN_NaiveBayes_DT_LogisticRegression_AdaBoost_RandomForest.json`, `results/baseline_svm.json`: binary baseline metrics.
- `results/extended.json`: binary extension, grouped-CV hyperparameter/threshold results, significance, per-category metrics, compact variants, and binary RF reference runs.
- `results/multiclass_smote.json`: ten-class RF and SMOTENC run details.
- `results/multiclass_smote_test_per_class.csv`: per-class scores by model and seed.
- `results/multiclass_smote_summary.txt`: human-readable multiclass summary.
- `results/fp_analysis.json`: false-alarm group diagnostics.
- `results/timing.json`: fitting time, inference time per flow, and serialized model size.
- `results/summary.txt`: consolidated baseline and extension tables.
- `figs/roc.pdf` / `figs/roc.png`, `figs/percat.pdf` / `figs/percat.png`: paper figures.

Numbers in the paper are rounded summaries of saved results. A fresh run may differ slightly due to software and hardware. Report the actual seeds, package versions, and completed runs when publishing a rerun.

## Citation

If you use this project, cite the paper and the original UNSW-NB15 dataset publications. The original study reproduced here is:

> B. S. Omarov, O. A. Auelbekov, B. O. Kulambayev, and B. S. Omarov, “IoT Network Intrusion Detection Using Machine Learning on UNSW-NB15 Dataset,” *Herald of the Kazakh-British Technical University*, vol. 21, no. 3, pp. 48–57, 2024. doi: [10.55452/1998-6688-2024-21-3-48-57](https://doi.org/10.55452/1998-6688-2024-21-3-48-57).

The UNSW dataset page lists the required dataset publications and citation details: [UNSW-NB15 dataset](https://research.unsw.edu.au/projects/unsw-nb15-dataset).

## License

The repository's software and documentation are provided under the [MIT License](LICENSE). The license file currently names `MuhammadShariq12` as the copyright holder. The paper, third-party dataset, and third-party papers or assets may have separate rights and terms; the repository MIT license does not grant rights to those materials. Follow the dataset provider's terms when downloading, using, citing, or redistributing UNSW-NB15.

## Authors

- Muhammad Shariq
- Saud Nasir
- Rohma Shaikh
