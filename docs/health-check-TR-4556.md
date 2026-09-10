# Maistro lessons health check — TR-4556

**Ticket:** [TR-4556](https://codefever.atlassian.net/browse/TR-4556) · **Date run:** 2026-09-10 · **Branch:** `✨/TR-4556-verify-lesson-code-and-models`

## Method

- **Source of truth = studio slides.** The repo `notebook.ipynb` files are student scaffolds (empty solution cells). The real solution code + model IDs were pulled from the studio decks of the **`2025_1 – mAIstros 01`** series (les02–les12 → repo Les2–Les12), via the studio GraphQL API.
- **Per-lesson environments** built with `uv` on **CPython 3.12.13**, one venv per unique `requirements.txt` (the machine only has Python 3.13/3.14; the lessons target ~3.11/3.12, so uv provisioned 3.12 for a fair install test).
- **Pragmatic bar:** light lessons run end-to-end; heavy lessons install + load model + one inference; the HF-inference lessons (Les9/10) are reachability-only (no HF token this pass).
- **Environment:** Windows 11 · uv 0.11.23 · Node 22 · git-lfs 3.7.1.

## Summary

**6 ok · 2 warning · 3 broken** (of 11 lessons).

| Lesson | Topic | Model(s) | Installs | Runs | Model reachable | Severity |
|---|---|---|:--:|:--:|:--:|---|
| Les2 | GenAI? (langdetect) | none (local) | ✅ | ✅ | n/a | **ok** |
| Les3 | Beelden Genereren | PGAN celebAHQ-512; SDXL | ✅¹ | ⚠️ PGAN ok / SDXL ✗ | endpoint dead | **broken** + mismatch |
| Les4 | Text2Speech | coqui `tts_models/*` (×5) | ✅ | ✅ | ✅ | **ok** |
| Les5 | Word Embeddings | none (GloVe file) | ✅ | code ✅ / data ✗ | n/a | **broken** (data) |
| Les6 | Word Embedding Game | none (GloVe file) | ✅ | code ✅ / data ✗ | n/a | **broken** (data) |
| Les7 | Sentence Embeddings | `hkunlp/instructor-large` | ✅ | ✅ | ✅ | **ok** |
| Les8 | Sentiment/Clustering | `hkunlp/instructor-large` | ✅ | ✅ | ✅ | **ok** |
| Les9 | LLMs | zephyr-7b-beta, nllb-200, Falconsai/text_summarization | ✅ | import ✅ / runtime deprecated | ✅ (repos) | **warning** |
| Les10 | Prompt engineering | zephyr-7b-beta (+ Llama-3-8B, gated) | ✅ | import ✅ / runtime deprecated | ✅ (repos) | **warning** |
| Les11 | RAG | instructor-large + distilbert-squad + FAISS | ✅ | ✅ | ✅ | **ok** |
| Les12 | Chatbots | distilbert-squad, `google/flan-t5-large` | ✅ | ✅ | ✅ | **ok** |

¹ Les3's *slide* code installs (torch); the repo's shipped `requirements.txt` is for a different (stale) lesson — see below.

## Not in scope: Les1 & les -1 (no runnable code)

The repo covers **Les2–Les12**. `Les1` was removed from the repo (commit `5e437be` "Delete first lesson files"), so there is no Les1 code on disk. Its studio deck (**"les 01: AI?"**, id 4916) is a **concept-only** lesson — 131 slides, **zero code cells** — so there is nothing to run or check. There is also a setup deck **"les -1: Huggingface"** (id 4946) — a screenshot walkthrough for creating a HuggingFace account/token, likewise **no runnable code**. Both are correctly excluded from the install/run/reachability checks; they were pulled from studio and inspected to confirm they contain no code.

## Cross-cutting issues (fix once, helps several lessons)

1. **Repo git-LFS is disabled** → `git lfs pull` returns *"Git LFS is disabled for this repository."* The two 353 MB GloVe files (Les5, Les6) are LFS-tracked and therefore **unobtainable** — blocks both word-embedding lessons. *Fix: re-enable LFS on the GitHub repo, or re-host the GloVe subset (release asset/direct link) and update the lessons.*
2. **Old HF serverless Inference API is retired.** `api-inference.huggingface.co` no longer serves (confirmed: host doesn't resolve); it was replaced by **Inference Providers** at `router.huggingface.co` (alive, returns 401 → needs a token). This breaks the *runtime* of Les3 (SDXL), Les9 and Les10 (langchain `HuggingFaceHub`). *Fix: migrate to `langchain_huggingface.HuggingFaceEndpoint` / the router endpoint with an HF token, or run models locally.*
3. **All referenced model repos still exist** — none were deleted. The failures are about *access method*, not missing models.

## Per-lesson detail

### Les2 — GenAI? — ok
`langdetect==1.0.9` installs and detects nl/en/fr correctly. No hosted model. **No fix needed.**

### Les3 — Beelden Genereren — broken + content mismatch
The current slide teaches *image generation* (PGAN `celebAHQ-512` via `torch.hub` + Stable Diffusion XL via HF Inference API), but the **repo `Les3/` ships face-*detection* material** (mediapipe, face_recognition, `gezichten.jpg`) — a stale/older version of the lesson.
- PGAN via torch.hub **works** (downloaded 264 MB weights, generated a 512×512 face).
- SDXL **fails**: the slide's `api-inference.huggingface.co/...` endpoint is dead; the live replacement is `router.huggingface.co/hf-inference/models/stabilityai/stable-diffusion-xl-base-1.0` (needs a token).
**Fix:** (1) point the SDXL call at the Inference Providers router with a token (or use `diffusers` locally); (2) reconcile the repo folder with the current curriculum (remove the stale face-detection files / restore the image-gen materials).

### Les4 — Text2Speech — ok
`coqui-tts==0.26.2` installs on 3.12; **all 5 `tts_models/*` named on the slides still resolve** in the registry; synthesized a wav with `tts_models/en/ljspeech/tacotron2-DDC`. (The original coqui.ai shut down, but the maintained `coqui-tts` fork + its rehosted registry work.) **No fix needed.**

### Les5 — Word Embeddings — broken (data unavailable)
Code logic verified against a stand-in (loader + `cosine_similarity`: falcon≈eagle 0.99, falcon≠armchair 0.06). But the lesson can't run as shipped: the GloVe file is git-LFS and **LFS is disabled** (cross-cutting #1), and the code reads `...txt` while the repo ships `...zip` (needs an unzip step). **Fix:** restore GloVe availability (re-enable LFS / re-host) and add an unzip step or ship the `.txt`.

### Les6 — The Word Embedding Game — broken (data unavailable)
Same GloVe data problem as Les5. Core embedding/similarity logic is fine; the interactive part is `ipywidgets` (notebook-only, expected). **Fix:** same as Les5.

### Les7 — Sentence Embeddings — ok
Full pinned stack installs; `hkunlp/instructor-large` loads and embeds (768-dim). The feared `sentence-transformers 2.2.2` breakage does **not** occur because `huggingface-hub` is pinned to 0.24.0 (still ships `cached_download`). Only deprecation warnings. **No fix needed** (optional: migrate the deprecated `langchain.embeddings` import to `langchain_community`).

### Les8 — Sentiment Analysis en Clustering — ok
Identical env to Les7 (shared); Instructor embedding path verified; downstream clustering is plain scikit-learn. **No fix needed.**

### Les9 — LLMs — warning
The pinned `langchain==0.3.14` stack installs and `HuggingFaceHub` imports/constructs, but it's **deprecated** (removed in langchain 1.0) and calls the **retired** old Inference API at runtime. Model repos (zephyr-7b-beta, nllb-200-distilled-1.3B, Falconsai/text_summarization) all still exist. **Fix:** migrate to `langchain_huggingface.HuggingFaceEndpoint` + Inference Providers (or run locally via `transformers`); note `nllb-200` is a translation seq2seq model, better run via a local `pipeline("translation", ...)`.

### Les10 — Prompt engineering — warning
Same shared env / same `HuggingFaceHub` deprecation as Les9. `zephyr-7b-beta` exists; `meta-llama/Meta-Llama-3-8B-Instruct` exists but is **gated** (needs an approved access request per student). **Fix:** same migration as Les9; confirm the Llama-3 access request is still grantable.

### Les11 — RAG — ok
End-to-end verified: pdfplumber (240 chunks from `Badminton.pdf`) → Instructor embeddings → FAISS search → distilbert QA. The prebuilt `sporten_faiss_index.bin` also loads (316 × 768, compatible). **No fix needed** (minor: install warns `pypdfium2==4.30.1` is yanked; pdfplumber uses pdfminer.six for text so extraction still works).

### Les12 — Chatbots — ok
`distilbert-base-uncased-distilled-squad` QA → "Paris" (0.96) and `google/flan-t5-large` (3 GB, downloaded) → "paris". Both transformers pipelines run locally on CPU. **No fix needed.**

## Recommended actions (priority order)

1. **Re-enable git-LFS (or re-host GloVe)** — unblocks Les5 & Les6. *(cross-cutting #1)*
2. **Migrate off the retired HF Inference API** — fixes Les9, Les10, and the SDXL half of Les3 (→ `langchain_huggingface` / `router.huggingface.co` + HF token, or local models). *(cross-cutting #2)*
3. **Reconcile the Les3 repo folder** with the current "Beelden Genereren" curriculum (remove stale face-detection files or restore image-gen materials).
4. *(Optional, non-blocking)* fix the Les5 `.zip`→`.txt` unzip step; bump `pypdfium2` in Les11; migrate deprecated `langchain.embeddings` imports in Les7/8/11.

## Notes & limitations

- Les9/10 were verified **reachability + import/deprecation only** (no HF token), per scope. Their model *repos* are confirmed present; a full live-inference test would need a token and the migrated endpoint.
- The verification env is Python 3.12 (via uv). The host default Python is 3.14, on which several of these pinned stacks would not install — worth noting for anyone reproducing without uv.
