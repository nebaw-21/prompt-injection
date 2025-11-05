# Prompt Injection Attacker — Project Flow

This repository contains a single, end-to-end Jupyter notebook that generates adversarial prompts, runs them against small open-source language models, auto-labels model responses, produces metrics and visualizations, and surfaces the top examples for manual review.

The notebook lives at:
- `prompt_injection_attacker.ipynb`

All run artifacts are written to a timestamped directory:
- `results/run_YYYYMMDD_HHMMSS/`

Inside each run directory you’ll find the prompts, raw results, labeled results, summary metrics, charts, and curated examples.

---

## High-level flow

```
[Step-3 Config]
     ↓
[Step-5 Prompt Generation] ──▶ prompts.csv
     ↓
[Step-6 Model Loader] ──▶ pipelines
     ↓
[Step-7 Attack Runner] ──▶ attack_results.csv
     ↓
[Step-8 Auto-Labeler] ──▶ attack_results_labeled.csv
     ↓
[Step-9 Metrics] ──▶ asr_by_model.csv, asr_by_type_model.csv, osr_by_model.csv
     ↓
[Step-10 Visualizations] ──▶ asr_by_model.png, asr_by_type_model_stacked.png, asr_heatmap.png, osr_by_model.png
     ↓
[Step-11 Top Examples] ──▶ top_examples_*.csv, top_examples.md
```

A brief safety notice appears in Step-1, and runtime requirements/imports are handled in Step-2.

---

## What each step does (Cells 1–11)

1) Title & safety warning
- Introduces the notebook and emphasizes safe, offline research usage only.

2) Requirements & imports
- Installs/loads Python packages if needed.
- Uses `tqdm.auto` and attempts to enable `ipywidgets` for progress bars.
- Prints library versions for reproducibility.

3) Configuration
- Centralized knobs:
  - `MODEL_NAMES` (default: `EleutherAI/gpt-neo-125M`, `gpt2`, `distilgpt2`)
  - Device detection (`USE_GPU`, `DEVICE`, `DEVICE_ID`)
  - Generation params (`GEN_PARAMS`)
  - `OUT_DIR` = `results/run_YYYYMMDD_HHMMSS`

4) Utilities
- Seeding for reproducibility (Faker, NumPy, Torch).
- Helpers for base64 and leetspeak obfuscation.
- Token counting (lightweight) and a `save_csv` helper that always writes under `OUT_DIR`.

5) Prompt generation
- Produces 100 prompts (4 types × 25):
  - instruction_override, role_based, obfuscated (base64/leetspeak), multi_turn.
- Saves `prompts.csv` in `OUT_DIR`.

6) Model loader
- Loads a Hugging Face text-generation pipeline for each model in `MODEL_NAMES`.
- Uses GPU if available, else CPU.

7) Attack runner
- Runs all prompts against each model.
- Simulates multi-turn with a simple conversation transcript.
- Logs response text, timing, and token counts.
- Appends to `attack_results.csv` in batches to avoid data loss.

8) Auto-labeler
- Heuristically labels responses as vulnerable (1) or safe (0) using keyword and simple regex cues.
- Writes `attack_results_labeled.csv`.

9) Metrics
- Computes:
  - ASR (Attack Success Rate) per model → `asr_by_model.csv`
  - ASR by model × attack_type → `asr_by_type_model.csv`
  - OSR (Obfuscation Success Rate) for obfuscated prompts → `osr_by_model.csv`

10) Visualizations
- Generates:
  - `asr_by_model.png` (bar chart)
  - `asr_by_type_model_stacked.png` (stacked bar by attack type)
  - `asr_heatmap.png` (model × attack type heatmap)
  - `osr_by_model.png` (if obfuscated data exists)

11) Top examples for manual review
- Shows and saves the highest-signal vulnerable examples (overall, per model, per attack type), plus a small sample of safe examples.
- Outputs:
  - `top_examples_overall.csv`
  - `top_examples_by_model.csv`
  - `top_examples_by_type.csv`
  - `safe_examples_sample.csv`
  - `top_examples.md`

---

## Artifacts produced per run

Under `results/<run_id>/` you can expect:

- Data
  - `prompts.csv`
  - `attack_results.csv`
  - `attack_results_labeled.csv`
  - `asr_by_model.csv`
  - `asr_by_type_model.csv`
  - `osr_by_model.csv` (if applicable)

- Visuals
  - `asr_by_model.png`
  - `asr_by_type_model_stacked.png`
  - `asr_heatmap.png`
  - `osr_by_model.png` (if applicable)

- Curation
  - `top_examples_overall.csv`
  - `top_examples_by_model.csv`
  - `top_examples_by_type.csv`
  - `safe_examples_sample.csv`
  - `top_examples.md`

---

## How to run

- Open `prompt_injection_attacker.ipynb` and execute cells in order.
- For quick iterations, you can reduce `NUM_PER_TYPE` in Step-5 to generate fewer prompts.
- Running Step-7 on CPU can be slow; small models are chosen to keep runs manageable.

Tip: If you only need metrics/visuals from an existing run, you can skip Steps 5–7 and point to a prior `OUT_DIR` by reusing the existing `results/<run_id>/attack_results_labeled.csv` when running Steps 9–11.

---

## Configuration knobs

- `MODEL_NAMES` in Step-3: choose any HF Causal LM you can run locally.
- `GEN_PARAMS` in Step-3: control generation (`max_length`, `temperature`, `top_k`, `top_p`).
- `NUM_PER_TYPE` in Step-5: adjust dataset size per attack type.
- `OUT_DIR` in Step-3: run-specific output directory.

---

## Environment & performance

- Progress bars: `tqdm.auto` uses `ipywidgets` if available; falls back to text bars otherwise.
- GPU: Automatically used if `torch.cuda.is_available()` is true. Otherwise, the pipeline runs on CPU.
- Python versions: Notebook was exercised on Python 3.13 with CPU PyTorch; GPU wheels require platform-compatible CUDA builds.

To speed up:
- Decrease `NUM_PER_TYPE`.
- Lower `max_length` in `GEN_PARAMS`.
- Run fewer models by trimming `MODEL_NAMES`.

---

## Troubleshooting

- IProgress/ipywidgets warning: If you see `IProgress not found`, the notebook attempts `%pip install ipywidgets` in Step-2 and will fall back to text progress bars if widgets aren’t available.
- CUDA not used: If `torch.cuda.is_available()` prints `False`, you’re on CPU. Install a CUDA-enabled PyTorch build compatible with your GPU and Python version, or continue with CPU.
- Missing metrics/visualization CSVs: Run Steps 8–9 before Step-10. The viz cell checks for the presence of `asr_by_model.csv` and `asr_by_type_model.csv`.

---

## Safety notice

This project intentionally generates prompts that may produce harmful content. Use only in isolated, offline research environments. Do not deploy against public endpoints or use generated prompts for any malicious purpose.

---

## Next ideas (optional)

- Export a full markdown/PDF report (Step-12).
- Add an interactive review UI (filters, search, CSV export).
- Add a single-model streaming runner for low-memory environments.
