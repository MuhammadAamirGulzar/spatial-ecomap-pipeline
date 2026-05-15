# Modular Unified Teacher-Student Pipeline EcoMap (Student Model Training Branch)

**Branch**: `student`  
**Purpose**: End-to-end knowledge distillation pipeline for training compact student models using teacher knowledge  

---

One command handles all below mentioned stages:
1. **STAGE 0**: Load input embeddings (UNI, scVI, cell composition)
2. **STAGE 0.5**: Apply PCA preprocessing and feature reduction
3. **PHASE 1**: Train teacher model (multimodal, 5-fold cross-validation)
4. **PHASE 2**: Build ensemble teacher from all folds
5. **PHASE 3**: Train student model with knowledge distillation
6. **PHASE 4**: Generate comprehensive visualizations (spatial, confidence, neighborhood analysis)

---

## Configuration

Three config files are provided:

- **`config/modular_unified_teacher_student_flexible.yaml`** - Template for custom datasets
- **`config/modular_unified_teacher_student_GEO.yaml`** - Pre-configured for GEO dataset
- **`config/modular_unified_teacher_student_Zenodo.yaml`** - Pre-configured for ZENODO dataset

To use a custom dataset, modify `modular_unified_teacher_student_flexible.yaml` with your dataset paths and run:
```bash
./run_unified_pipeline.sh config/modular_unified_teacher_student_flexible.yaml
```

---

## Model Architecture

### Teacher Model (Multimodal)
- **Input**: 1177D (UNI:1024D + scVI:128D + Cell Composition:25D for GEO, 15D for ZENODO)
- **Hidden layers**: [256, 128, 64]
- **Output**: 5-class ecotype predictions


### Student Model (Morphology-Only)
- **Input**: 1024D (UNI embeddings, PCA reduced from teacher relatively)
- **Hidden layers**: [128, 64, 32]
- **Output**: 5-class ecotype predictions

---

## Output Structure

Results are organized by dataset:

```
[GEO/ZENODO] Ablation Study/
├── TEACHER_UNIFIED_[DATASET]_0.6_PCA_ver5/
│   ├── training/metrics/       # Cross-validation metrics, fold results
│   ├── training/models/        # Fold-specific teacher models
│   └── .working/               # Preprocessed embeddings, PCA models
│
└── STUDENT_UNIFIED_[DATASET]_0.6_PCA_ver5/
    ├── training/metrics/       # Student accuracy, F1, predictions CSV
    ├── training/models/        # Final student model weights
    ├── post-training/visualizations/
    │   ├── spatial/           # Ecotype maps with tissue coordinates
    │   ├── spatial_confidence/ # Prediction confidence heatmaps
    │   ├── spatial_comparison/ # Teacher vs student predictions
    │   ├── neighborhood/      # Neighbor composition analysis
    │   └── preprocessing/     # Embedding quality metrics
    └── pipeline_execution.log  # Complete execution log
```
---

## Dataset Requirements

Each dataset needs:
- `barcode_labels.csv` - Sample labels (one per row)
- `barcode_metadata.csv` - Metadata including x_coord, y_coord, patient_id for spatial visualization
- `label_mapping.json` - Class name to index mapping
- `UNI_EMBEDDINGS_COMBINED.CSV` - 1024D morphology embeddings (student input)
- `SCVI_EMBEDDINGS_COMBINED.CSV` - 128D gene expression embeddings (teacher only)
- `CELL_COMPOSITION_EMBEDDINGS*.CSV` - Cell type composition (teacher only)

---
---

## Troubleshooting

**Q: Pipeline fails during teacher training?**  
A: Check that all input embedding CSV files exist and have the correct column names.

**Q: Student model accuracy is low?**  
A: This is expected (~79% vs teacher ~88%). Student uses only morphology while teacher uses multimodal data. The gap reflects information lost.

**Q: Visualizations not generated?**  
A: Ensure `barcode_metadata.csv` contains `x_coord` and `y_coord` columns. Other visualizations generate regardless.

---

## Key Hyperparameters

Adjust in YAML configs:

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
  alpha: 0.7        # Hard target weight
  beta: 0.2         # Soft target weight
  gamma: 0.1        # Feature matching weight
  temperature: 4.0  # Teacher logit softness

embeddings:
  image_encoder:
    pca_variance: 0.6  # Reduce 1024D to ~150D
```

---
