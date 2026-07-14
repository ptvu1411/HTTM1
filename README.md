# CCTV Violence Detection & Automated Alerting

Final-year project: detect violent activity in CCTV-style video, classify violent vs
non-violent, and on a *confirmed* event send a proof-of-concept alert (email / WhatsApp).
Primary model: R(2+1)D-18 (Kinetics-pretrained). See the project plan for the full design.

## Phase 0 — setup (do this first)

### 1. Create the environment
```bash
python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
source .venv/bin/activate
```

### 2. Install PyTorch with CUDA (IMPORTANT — do this before requirements.txt)
Pick the command matching your CUDA version from https://pytorch.org/get-started/locally/.
For a typical RTX 3050 setup this is usually something like:
```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
```
(Use the selector on the PyTorch site to get the exact line for your machine.)

### 3. Install the rest
```bash
pip install -r requirements.txt
```

### 4. Verify the GPU is visible  ← Phase 0 "done when"
```bash
python -m src.check_env
```
You want to see `CUDA available: True` and your RTX 3050 listed. If it says False,
fix the PyTorch install before going further.

### 5. Set up secrets
```bash
cp .env.example .env      # Windows: copy .env.example .env
```
Then edit `.env` with your email app-password etc. `.env` is gitignored — never commit it.

### 6. Download the dataset
Download RWF-2000 and place it under `data/RWF-2000/` keeping its **official**
`train/` and `val/` folders (we use the official split, not a random one).
Optionally add RLVS under `data/RLVS/` for the cross-dataset test.

## Project layout
```
configs/config.yaml   all tunable settings
src/config.py         loads config + secrets
src/check_env.py      Phase 0 GPU/environment check
data/                 datasets (gitignored)
outputs/              checkpoints, logs, metrics (gitignored)
notebooks/            exploration
```

## Next phases
1. Data pipeline (official split manifest, clip validator, DataLoader)
2. Preprocessing & augmentation
3. Model & training (R(2+1)D-18)
4. Evaluation (metrics, multi-clip averaging, cross-dataset, VideoMAE comparison on Kaggle)
5. Temporal confidence gate
6. Inference pipeline (clips + webcam)
7. Alerting (email / WhatsApp)
8. Demo (Streamlit)
9. Report
