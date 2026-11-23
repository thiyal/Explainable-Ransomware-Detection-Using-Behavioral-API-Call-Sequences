# Explainable-Ransomware-Detection-Using-Behavioral-API-Call-Sequences
An explainable ransomware-detection project that analyzes sequences of monitored Windows API calls. The implementation combines API-trace preprocessing, TF-IDF n-gram features, classical machine-learning classifiers, an Attention-BiLSTM model, and explanation methods intended to help an analyst understand a prediction.
Project status: This repository contains the Python implementation extracted from the project report. It does not include the original dataset, trained model artifacts, or a complete model-training/export workflow for every model. Follow the setup and requirements below before attempting to run the dashboard.

1. Project objectives
- Validate and normalize behavioral API-call traces.
- Filter calls to a predefined set of monitored Windows APIs.
- Reduce repeated consecutive API calls (API-hammering suppression) and truncate sequences to configured windows.
- Represent traces using TF-IDF API n-grams or integer token sequences.
- Evaluate classical classifiers with family-grouped stratified cross-validation.
- Provide an Attention-BiLSTM architecture for sequential classification.
- Generate model explanations using TreeSHAP feature attributions or temporal attention peaks.
- Measure explanation faithfulness by masking highly attributed features or tokens and observing prediction-confidence changes.
- Expose an analyst-facing Flask endpoint for trace analysis.
2. Repository structure
.
├── app/
│   └── server.py              # Flask REST endpoint and simple web interface
├── scripts/
│   └── run_experiments.py     # Grouped cross-validation experiments
├── src/
│   ├── __init__.py            # Add this empty file if your checkout does not contain it
│   ├── config.py              # Central experiment configuration
│   ├── taxonomy.py            # Monitored Windows API vocabulary and subsystems
│   ├── preprocess.py          # Trace validation and API-sequence normalization
│   ├── features.py            # TF-IDF features and integer API vocabulary
│   ├── models_tree.py         # Logistic regression, Linear SVM, RF and XGBoost
│   ├── model_bilstm.py        # Attention-BiLSTM model and training utilities
│   ├── metrics.py             # Classification metrics
│   ├── explainability.py      # TreeSHAP, attention peaks and deletion tests
│   └── pipeline.py            # End-to-end inference pipeline
└── README.md
Note: The Python files use package-relative imports such as from .config import CONFIG. Add an empty src/__init__.py file if it is absent in your GitHub repository so Python treats src as a package.
3. Implementation overview
Trace preprocessing
src/preprocess.py validates JSON telemetry documents, filters calls against the monitored API vocabulary, suppresses consecutive repetitions beyond the configured run-length threshold, and truncates the resulting sequence to the selected window. It also retains source-event indices so explanations can be mapped back to the original trace events.
Monitored API groups
src/taxonomy.py defines five API subsystems:
- Filesystem
- Cryptography
- Process
- Registry
- Network
The monitored API names are defined directly in this module.
Feature representations and models
src/features.py builds case-sensitive TF-IDF features from API-call n-grams and provides an integer vocabulary for sequential models.

src/models_tree.py implements the following tabular classifiers:
- Logistic Regression (logreg)
- Linear Support Vector Machine (svm)
- Random Forest (rf)
- XGBoost (xgb)
The experiment runner uses StratifiedGroupKFold to keep related ransomware families (and benign samples grouped by sample ID) separated between training and test folds.

src/model_bilstm.py defines a bidirectional LSTM with masked additive attention, along with fold-training and prediction utilities.
Explainability
src/explainability.py includes:
- TreeSHAP attribution for TF-IDF API n-gram features.
- Ranking of high-attention API events for the sequential model.
- Deletion-faithfulness tests that mask important features or tokens and measure the change in predicted ransomware probability.
- A random-masking control for comparison.
Inference dashboard
app/server.py provides a Flask interface and the POST /api/analyze endpoint. It accepts a JSON trace upload and selects either the XGBoost + TreeSHAP model or the Attention-BiLSTM model. The dashboard loads model files from the configured artifacts directory.
4. Requirements
Use Python 3.10 or later. Install the packages used by the implementation:
pip install numpy pandas scikit-learn flask joblib shap xgboost torch
The XGBoost package is needed for XGBoost experiments and the XGBoost dashboard option. PyTorch is needed for the Attention-BiLSTM implementation. SHAP is needed for TreeSHAP explanations.

For reproducible work, create and activate a virtual environment before installing dependencies.
5. Input telemetry format
The preprocessing code expects one JSON document per trace, with the following required structure:
{
  "sample_id": "64-character-hexadecimal-SHA256-value",
  "process_name": "example.exe",
  "api_trace": [
    {
      "api_name": "CreateFileW",
      "subsystem": "filesystem",
      "timestamp_ms": 12.5,
      "arguments": {}
    },
    {
      "api_name": "WriteFile",
      "subsystem": "filesystem",
      "timestamp_ms": 18.2,
      "arguments": {}
    }
  ]
}
The sample_id must be a 64-character hexadecimal string. api_trace must be a non-empty array. Each event must contain api_name, subsystem, and timestamp_ms; timestamps must be in non-decreasing order. The arguments field is used by the explanation mapping when available, but is not one of the required event fields checked by the validator.

The example above is a format illustration, not a real dataset sample.
6. Running the experiment script
Prepare a CSV manifest with these columns:
Column	Description
sample_id	Sample identifier; benign samples are grouped by this value
label	Binary class label: 0 for benign and 1 for ransomware
family	Ransomware family name; used for grouping positive samples
trace_path	Path to the corresponding JSON trace file


Example manifest layout:
sample_id,label,family,trace_path
<64-character-hash>,0,benign,data/traces/benign_001.json
<64-character-hash>,1,family_a,data/traces/malware_001.json
Run the experiment script from the repository root:
python -m scripts.run_experiments --manifest data/manifest.csv --out results.json
The script evaluates Logistic Regression, Linear SVM, Random Forest, and XGBoost over the configured sequence windows (200, 500, and 1000 by default), then writes summarized cross-validation metrics to results.json.

The script expects the manifest and trace files to exist. No dataset is included in this repository. The current experiment script evaluates the four tabular classifiers; it does not train or export the Attention-BiLSTM model.
7. Running the Flask dashboard
The dashboard expects pre-trained artifacts in an artifacts/ directory by default:
artifacts/
├── xgb.joblib
├── tfidf.joblib
└── bilstm.pt
These files are not generated by scripts/run_experiments.py and are not included in this repository. They must be produced using a compatible training and model-export workflow before the dashboard can load the corresponding model.

To use a different artifact directory, set ARTIFACT_DIR before starting the server.
# Linux / macOS
export ARTIFACT_DIR="artifacts"
python -m app.server
# Windows PowerShell
$env:ARTIFACT_DIR = "artifacts"
python -m app.server
Open http://127.0.0.1:5000 in your browser. Upload a trace JSON file and select a model. The server exposes POST /api/analyze for programmatic use; the uploaded file field is named trace, and the optional form field model accepts xgb or bilstm.

The server binds to 127.0.0.1 by default, so it is intended for local use. Do not expose an unreviewed development dashboard to a public network.
8. Configuration
Most experiment settings are centralized in src/config.py, including:
- Random seed: 42
- API repetition limit: 2
- Sequence windows: 200, 500, 1000
- TF-IDF n-gram range: (1, 3)
- Maximum TF-IDF features: 5000
- Cross-validation folds: 5
- Attention-BiLSTM embedding and hidden dimensions, dropout, learning rate, batch size, epochs, and early-stopping patience
- Decision threshold: 0.5
- Explanation top-k values and deletion steps
Change these values in the configuration module when conducting a new experiment, and document any changes when reporting results.
9. Evaluation metrics
src/metrics.py calculates accuracy, precision, recall, F1-score, false-positive rate (FPR), ROC-AUC, and confusion-matrix counts (TP, FP, TN, FN). The experiment runner reports the mean and standard deviation of selected metrics across folds.
10. Limitations and reproducibility notes
- The repository does not contain the trace dataset or a data-acquisition/collection tool.
- The repository does not contain trained model weights or serialized vectorizers.
- scripts/run_experiments.py runs grouped cross-validation for the four TF-IDF-based classifiers; it does not save a fitted model/vectorizer pair for dashboard inference.
- The Attention-BiLSTM module contains model and fold-training utilities, but the repository does not include a complete end-to-end BiLSTM experiment script or checkpoint-export command.
- Dashboard inference therefore requires compatible artifacts to be trained and saved separately before use.
- The code has been checked for Python syntax, but a full end-to-end execution with a real dataset and trained artifacts has not been established by this repository extraction.
- Results depend on dataset quality, trace collection methodology, class balance, and the training/evaluation protocol. Do not interpret example JSON or configuration defaults as experimental results.
11. Responsible use
This project is intended for academic research, defensive security analysis, and malware-behavior understanding. Use only telemetry and datasets that you are authorized to access. The classifier is a decision-support component, not a guarantee that a file or process is safe or malicious; predictions should be reviewed alongside other security evidence.
12. Author
M. THIYAL
B.Tech – Computer Science and Engineering (Cloud Technology and Mobile Applications)
JAIN (Deemed-to-be University)
