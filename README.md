# EcoMap — Modular Unified Teacher-Student Pipeline

> **Branch:** `student` | **Purpose:** End-to-end knowledge distillation, student model trained from teacher knowledge.

---

## Table of Contents

1. [Pipeline Overview](#1-pipeline-overview)
2. [Quick Start](#2-quick-start)
3. [Dataset Requirements](#3-dataset-requirements)
4. [Configuration](#4-configuration)
5. [Model Architecture](#5-model-architecture)
6. [Key Hyperparameters](#6-key-hyperparameters)
7. [Output Structure](#7-output-structure)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Pipeline Overview

A single command runs all stages end-to-end:

| Stage | Description |
|-------|-------------|
| **Stage 0** | Load input embeddings (UNI, scVI, cell composition) |
| **Stage 0.5** | PCA preprocessing and feature reduction |
| **Phase 1** | Train teacher model (multimodal, 5-fold cross-validation) |
| **Phase 2** | Build ensemble teacher from all folds |
| **Phase 3** | Train student model with knowledge distillation |
| **Phase 4** | Generate visualizations (spatial, confidence, neighborhood) |

---

## 2. Quick Start

### Step 1 — Download the datasets

The three reference datasets are available here:

> [Google Drive — Dataset Folder](https://drive.google.com/drive/folders/1h2mY0to3B52E_IKbG4DPKqhckzK-08o-?usp=sharing)

Unzip the folders into your **project root**.

### Step 2 — Choose or edit a config file

| Config File | Use When |
|-------------|----------|
| `config/modular_unified_teacher_student_flexible.yaml` | Custom dataset (edit paths here) |
| `config/modular_unified_teacher_student_GEO.yaml` | Pre-configured for GEO dataset |
| `config/modular_unified_teacher_student_Zenodo.yaml` | Pre-configured for ZENODO dataset |

For a custom dataset, open `modular_unified_teacher_student_flexible.yaml` and update the data directory paths to match your file locations.

### Step 3 — Run the pipeline

```bash
./run_unified_pipeline.sh config/modular_unified_teacher_student_flexible.yaml
```

Replace the config path with `_GEO.yaml` or `_Zenodo.yaml` if using a reference dataset.

---

## 3. Dataset Requirements

Each dataset folder must contain the following files:

| File | Description |
|------|-------------|
| `barcode_labels.csv` | Sample labels (one per row) |
| `barcode_metadata.csv` | Metadata — must include `x_coord`, `y_coord`, `patient_id` |
| `label_mapping.json` | Class name → index mapping |
| `UNI_EMBEDDINGS_COMBINED.CSV` | 1024D morphology embeddings **(student + teacher input)** |
| `SCVI_EMBEDDINGS_COMBINED.CSV` | 128D gene expression embeddings **(teacher only)** |
| `CELL_COMPOSITION_EMBEDDINGS*.CSV` | Cell type composition **(teacher only)** |

> `x_coord` and `y_coord` in `barcode_metadata.csv` are required for spatial visualizations. Other visualizations will still generate without them.

---

## 4. Configuration

All tunable settings live in the YAML config files. The three key sections to be aware of:

**Selecting the right config:**
- Start with `_flexible.yaml` for any new or custom dataset.
- The `_GEO.yaml` and `_Zenodo.yaml` files are ready to run as-is for those datasets, no edits needed beyond verifying your data paths are correct after unzipping.

---

## 5. Model Architecture

### Teacher Model (Multimodal)

| Property | Detail |
|----------|--------|
| Input dimensionality | 1177D — UNI (1024D) + scVI (128D) + Cell Composition (25D for GEO, 15D for ZENODO) |
| Hidden layers | [256, 128, 64] |
| Output | 5-class ecotype predictions |

### Student Model (Morphology-Only)

| Property | Detail |
|----------|--------|
| Input dimensionality | 1024D — UNI embeddings only, PCA-reduced relative to teacher |
| Hidden layers | [128, 64, 32] |
| Output | 5-class ecotype predictions |

> The student intentionally uses fewer inputs. Its lower accuracy (~79% vs teacher ~88%) reflects the information cost of dropping gene expression and cell composition data — this is expected, not a bug.

---

## 6. Key Hyperparameters

Adjust these in your chosen YAML config:

```yaml
teacher:
  n_epochs: 200
  batch_size: 32
  learning_rate: 0.001
  early_stopping_patience: 20

student:
  n_epochs: 150
  batch_size: 32
  learning_rate: 0.001
  early_stopping_patience: 15

distillation:
  alpha: 0.7        # Weight for hard targets (ground truth labels)
  beta: 0.2         # Weight for soft targets (teacher probability outputs)
  gamma: 0.1        # Weight for feature matching loss
  temperature: 4.0  # Controls softness of teacher logits

embeddings:
  image_encoder:
    pca_variance: 0.6   # Reduces 1024D → ~150D
```

---

## 7. Output Structure

Results are organized under the dataset name:

```
[GEO/ZENODO] Ablation Study/
│
├── TEACHER_UNIFIED_[DATASET]_0.6_PCA_ver5/
│   ├── training/metrics/       # Cross-validation metrics, per-fold results
│   ├── training/models/        # Fold-specific teacher model weights
│   └── .working/               # Preprocessed embeddings, PCA models
│
└── STUDENT_UNIFIED_[DATASET]_0.6_PCA_ver5/
    ├── training/metrics/       # Student accuracy, F1, predictions CSV
    ├── training/models/        # Final student model weights
    ├── post-training/visualizations/
    │   ├── spatial/            # Ecotype maps with tissue coordinates
    │   ├── spatial_confidence/ # Prediction confidence heatmaps
    │   ├── spatial_comparison/ # Teacher vs student side-by-side
    │   ├── neighborhood/       # Neighbor composition analysis
    │   └── preprocessing/      # Embedding quality metrics
    └── pipeline_execution.log  # Full execution log
```

---

## 8. Troubleshooting

**Pipeline fails during teacher training**
Check that all input embedding CSVs exist and that column names match what the config expects.

**Student model accuracy is low (~79%)**
This is expected. The student only sees morphology (UNI embeddings); the teacher uses three modalities. The accuracy gap directly reflects the missing information, it is not a misconfiguration.

**Visualizations not generated**
Spatial visualizations require `x_coord` and `y_coord` in `barcode_metadata.csv`. If those columns are missing, only the spatial outputs are skipped — all other visualizations will still generate.
