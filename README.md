# Group6: Gene Expression-Based Molecular Generation Using Deep Learning

This project implements a transcriptome-conditioned molecular generation framework that generates novel SMILES molecules from target gene expression profiles.

The core model is **GRRM-FiLM-LSTM**, with scripts for training, generation, Tanimoto-based evaluation, and interpretability analysis.

The thesis document is in the project root: `Group6_Project.docx`.

## 1. Project Objective

Most molecular generators are chemistry-centric. This project introduces transcriptomic conditioning so the model is not only chemically valid, but also better aligned with target biological response patterns.

## 2. Method Overview

Main model file: `GRRM-FiLM-LSTM/xin4_GxRNNstar.py`

The architecture has four key modules:

1. **GRRM-1 (ImprovedSTAR)**  
Learns global representations from 978-dimensional gene expression vectors using multi-head projections, residual connections, feed-forward blocks, and layer normalization.

2. **GRRM-2 (ProgressiveGeneEncoder)**  
Compresses high-dimensional gene information into a compact conditioning feature (default `gene_feature_dim=256`).

3. **Adaptive FiLM**  
Generates `alpha / beta / gate` from gene features to modulate each SMILES token embedding dynamically.

4. **LSTM Decoder**  
Autoregressively generates SMILES from FiLM-modulated token embeddings plus gene conditioning vectors.

Entry point: `GRRM-FiLM-LSTM/main.py`  
Training loop: `GRRM-FiLM-LSTM/train_gxrnn.py`

## 3. Repository Structure

```text
.
├── Group6_Project.docx                # Thesis
├── Group6_Project_docx.pdf            # Thesis PDF export
├── env/
│   ├── gxrnn.yml                      # Original conda environment
│   └── requirements_new.txt           # Updated dependencies (Python 3.9+)
├── GRRM-FiLM-LSTM/
│   ├── main.py                        # Train / generate / evaluate entry point
│   ├── train_gxrnn.py                 # Training loop
│   ├── xin4_GxRNNstar.py              # Main GRRM-FiLM-LSTM implementation
│   ├── GxRNN.py                       # Earlier baseline implementation
│   ├── utils.py
│   ├── datasets/
│   │   ├── LINCS/mcf7.zip
│   │   ├── test_protein/*.csv         # Target-conditioned expression profiles
│   │   └── ligands/source_*.csv       # Known ligands from ExCAPE
│   ├── results/                       # Training, generation, and analysis outputs
│   └── pbs/                           # Cluster job scripts
└── README.md
```

## 4. Environment Setup

### Option A: Original conda environment (closest to historical runs)

```bash
cd GRRM-FiLM-LSTM
conda env create -f ../env/gxrnn.yml
conda activate gxrnn
```

### Option B: Updated dependencies (Python 3.9+)

```bash
cd GRRM-FiLM-LSTM
python3 -m venv .venv
source .venv/bin/activate
pip install -r ../env/requirements_new.txt
```

If `rdkit` installation fails via pip, install it from conda-forge:

```bash
conda install -c conda-forge rdkit
```

## 5. Data Preparation

Training expects `datasets/LINCS/mcf7.csv`. The repository includes `mcf7.zip`, so unzip first:

```bash
cd GRRM-FiLM-LSTM
unzip -o datasets/LINCS/mcf7.zip -d datasets/LINCS/
```

By default, `main.py` uses:

- Training data: `datasets/LINCS/mcf7.csv`
- Test protein expression profiles: `datasets/test_protein/<protein>.csv`
- Known ligands: `datasets/ligands/source_<protein>.csv`

## 6. Training and Generation

Run all commands below inside `GRRM-FiLM-LSTM/`.

### 6.1 Training (thesis-style settings)

```bash
python main.py \
  --train_gxrnn \
  --gene_expression_file "datasets/LINCS/" \
  --cell_name mcf7 \
  --gene_num 978 \
  --hidden_size 1024 \
  --smiles_epochs 450 \
  --smiles_dropout 0.3 \
  --gene_batch_size 64 \
  --emb_size 128 \
  --train_rate 0.9
```

Model checkpoints are saved as `results/saved_gxrnn.pkl_<epoch>.pkl`, for example:

- `results/saved_gxrnn.pkl_450.pkl`

### 6.2 Single-target generation

```bash
python main.py \
  --generation \
  --hidden_size 1024 \
  --protein_name AKT1 \
  --saved_gxrnn results/saved_gxrnn.pkl_450.pkl
```

Output file:

- `results/generation/res-AKT1.csv`

### 6.3 Batch generation for all 10 targets

```bash
for p in AKT1 AKT2 AURKB CTSK EGFR HDAC1 MTOR PIK3CA SMAD3 TP53; do
  python main.py \
    --generation \
    --hidden_size 1024 \
    --protein_name "$p" \
    --saved_gxrnn results/saved_gxrnn.pkl_450.pkl
done
```

### 6.4 Tanimoto similarity evaluation

```bash
python main.py \
  --calculate_tanimoto \
  --hidden_size 1024 \
  --protein_name AKT1 \
  --candidate_num 1000
```

## 7. Key Output Files

- `results/gxrnn_train_results*.csv`: epoch-wise loss / validity / uniqueness / similarity
- `results/predicted_valid_smiles*.csv`: generated valid SMILES on validation set and paired labels
- `results/generation/res-*.csv`: generated molecules per target protein
- `results/train_violin_plot.png`, `results/valid_violin_plot.png`: property distribution comparisons
- `results/film_cluster_analysis/*`: FiLM parameter clustering and interpretability outputs

## 8. Thesis Result Snapshot (from `Group6_Project.docx`)

The thesis reports that GRRM-FiLM-LSTM achieves the best Tanimoto score on 6 out of 10 targets and remains competitive on the rest.

Reported scores for **GRRM-FiLM-LSTM (ours)**:

| Target | Tanimoto |
|---|---:|
| AKT1 | 0.654 |
| AKT2 | 0.547 |
| AURKB | 0.774 |
| CTSK | 0.492 |
| EGFR | 0.674 |
| HDAC1 | 0.400 |
| MTOR | 0.596 |
| PIK3CA | 0.740 |
| SMAD3 | 0.641 |
| TP53 | 0.756 |

## 9. Interpretability Scripts

- `jiyincujvlei.py`: FiLM parameter clustering, heatmaps, and PCA
- `makecusters.py`: regenerate `clusters.npy`
- `jiyincujvlei5.py`: extract generated molecules from cluster 5
- `jiyincujvlei5bijiao.py`: deeper analysis of cluster 5
- `results/xiaotiqin.py`: violin plots for generated vs. original molecular properties

## 10. Troubleshooting

1. **`FileNotFoundError: datasets/LINCS/mcf7.csv`**  
Unzip `mcf7.zip` into `datasets/LINCS/`.

2. **`ModuleNotFoundError: rdkit`**  
Install RDKit via conda-forge.

3. **Checkpoint file not found**  
Training uses the naming pattern `saved_gxrnn.pkl_<epoch>.pkl`; pass the full filename in `--saved_gxrnn`.

4. **No GPU available**  
The code automatically falls back to CPU (`utils.get_device()`).

## 11. Citation

Dong Yizhe, Liu Hongxi, Tian Yi, Wen Yuxin, Wang Yikai.  
*Gene Expression-Based Molecular Generation Using Deep Learning*.  
Master of Science in Biomedical Informatics, National University of Singapore, 2026.

