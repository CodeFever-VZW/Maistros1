# Maistro lessons health check — TR-4556

**Ticket:** [TR-4556](https://codefever.atlassian.net/browse/TR-4556) · **Date run:** 2026-09-10 · **Branch:** `✨/TR-4556-verify-lesson-code-and-models`

## Method

- **Source of truth = studio slides.** The repo `notebook.ipynb` files are student scaffolds (empty solution cells). The real solution code + model IDs were pulled from the studio decks of the **`2025_1 – mAIstros 01`** series (les02–les12 → repo Les2–Les12), via the studio GraphQL API.
- **Per-lesson environments** built with `uv` on **CPython 3.12.13**, one venv per unique `requirements.txt` (the machine only has Python 3.13/3.14; the lessons target ~3.11/3.12, so uv provisioned 3.12 for a fair install test).
- **Pragmatic bar:** light lessons run end-to-end; heavy lessons install + load model + one inference; the HF-inference lessons (Les9/10) are reachability-only (no HF token this pass).
- **Environment:** Windows 11 · uv 0.11.23 · Node 22 · git-lfs 3.7.1.

## Summary

**9 ok · 2 warning · 0 broken** (of 11 lessons). *Update 2026-09-17: all three previously-broken lessons resolved — Les5 & Les6 (GloVe data) and Les3 (SDXL → SD3-medium); see the Updates below.*

| Lesson | Topic | Model(s) | Installs | Runs | Model reachable | Severity |
|---|---|---|:--:|:--:|:--:|---|
| Les2 | GenAI? (langdetect) | none (local) | ✅ | ✅ | n/a | **ok** |
| Les3 | Beelden Genereren | PGAN celebAHQ-512; ~~SDXL~~ → SD3-medium | ✅ | code ✅ | ✅ (SD3-medium) | ~~broken~~ → **resolved** (2026-09-17) |
| Les4 | Text2Speech | coqui `tts_models/*` (×5) | ✅ | ✅ | ✅ | **ok** |
| Les5 | Word Embeddings | none (GloVe file) | ✅ | code ✅ / data ✅ | n/a | ~~broken (data)~~ → **resolved** (2026-09-17) |
| Les6 | Word Embedding Game | none (GloVe file) | ✅ | code ✅ / data ✅ | n/a | ~~broken (data)~~ → **resolved** (2026-09-17) |
| Les7 | Sentence Embeddings | `hkunlp/instructor-large` | ✅ | ✅ | ✅ | **ok** |
| Les8 | Sentiment/Clustering | `hkunlp/instructor-large` | ✅ | ✅ | ✅ | **ok** |
| Les9 | LLMs | zephyr-7b-beta, nllb-200, Falconsai/text_summarization | ✅ | import ✅ / runtime deprecated | ✅ (repos) | **warning** |
| Les10 | Prompt engineering | zephyr-7b-beta (+ Llama-3-8B, gated) | ✅ | import ✅ / runtime deprecated | ✅ (repos) | **warning** |
| Les11 | RAG | instructor-large + distilbert-squad + FAISS | ✅ | ✅ | ✅ | **ok** |
| Les12 | Chatbots | distilbert-squad, `google/flan-t5-large` | ✅ | ✅ | ✅ | **ok** |

## Update 2026-09-17 — GloVe data resolved

Cross-cutting issue #1 (Les5 & Les6) is fixed:

- The GloVe subset is **re-hosted as a plain `.txt`** (no git-LFS, no unzip step) at `https://codefeverpublic.blob.core.windows.net/public-content/maistros1/subset_lower_glove.42B.300d.txt` — 934,857,304 B, 300-dim, verified live (HTTP 200).
- **Studio slides updated** to point at the new URL: **les 05** slide 29 (id 2155026); **les 06** slides 16 & 17 (ids 2152357, 2152358).
- **Repo cleaned:** removed the broken LFS `.zip` pointers from `Les5/` and `Les6/`, dropped the now-dead `*.zip` LFS rule from `.gitattributes`, and added a "Data" markdown cell with the download link to both notebooks.

## Update 2026-09-17 — Les3 SDXL resolved (SD3-medium)

Re-investigation corrected two things about Les3:

- **No content mismatch.** The current studio deck *"les 03: Beelden Genereren"* (id 5111, 56 slides) is a single **three-part** lesson: PGAN face *generation* (`torch.hub` celebAHQ-512) → SDXL text→image *generation* → face *recognition* (`cv2`/`mediapipe` on `gezichten.jpg`). The repo's `mediapipe`/`opencv`/`face_recognition`/`gezichten.jpg` are **used by part 3**, not stale, and `requirements.txt` is byte-identical (mod line endings) to the file the slide serves for download. **No repo files needed changing.**
- **The only real breakage was the SDXL endpoint**, now fixed. The old `api-inference.huggingface.co` host is retired (DNS gone), and the naive replacement `router.huggingface.co/hf-inference/…/stable-diffusion-xl-base-1.0` now returns **HTTP 410** ("no longer supported by provider hf-inference") — so the earlier router-swap recommendation is void. Of the current text-to-image models, only **`stabilityai/stable-diffusion-3-medium-diffusers`** is still served by the free `hf-inference` provider and returns raw image bytes to the lesson's existing `requests.post({"inputs": …})` code (verified live: HTTP 200 → `image/jpeg`). **Fix applied on a draft copy of the deck (id 7389):** the two SDXL code blocks (slides seq 37 & 43) now point at `https://router.huggingface.co/hf-inference/models/stabilityai/stable-diffusion-3-medium-diffusers`, and the token slide (seq 36) got a note that the model is gated (`gated=auto` → one-time "Agree and access repository"). The content team reviewed the copy (SD3-medium approved) and **published it on 2026-09-18**; the original deck 5111 was left as-is.

## Not in scope: Les1 & les -1 (no runnable code)

The repo covers **Les2–Les12**. `Les1` was removed from the repo (commit `5e437be` "Delete first lesson files"), so there is no Les1 code on disk. Its studio deck (**"les 01: AI?"**, id 4916) is a **concept-only** lesson — 131 slides, **zero code cells** — so there is nothing to run or check. There is also a setup deck **"les -1: Huggingface"** (id 4946) — a screenshot walkthrough for creating a HuggingFace account/token, likewise **no runnable code**. Both are correctly excluded from the install/run/reachability checks; they were pulled from studio and inspected to confirm they contain no code.

## Cross-cutting issues (fix once, helps several lessons)

1. **Repo git-LFS is disabled** → `git lfs pull` returns *"Git LFS is disabled for this repository."* The two 353 MB GloVe files (Les5, Les6) are LFS-tracked and therefore **unobtainable** — blocks both word-embedding lessons. *Fix: re-enable LFS on the GitHub repo, or re-host the GloVe subset (release asset/direct link) and update the lessons.* **→ RESOLVED 2026-09-17 (re-hosted; see [Update](#update-2026-09-17--glove-data-resolved)).**
2. **Old HF serverless Inference API is retired.** `api-inference.huggingface.co` no longer serves (confirmed: host doesn't resolve); it was replaced by **Inference Providers** at `router.huggingface.co`. This broke the *runtime* of Les3 (SDXL), Les9 and Les10 (langchain `HuggingFaceHub`). Note the target moved again since: image models are now served only by (mostly paid) third-party providers, and even `hf-inference` has dropped SDXL (returns 410). **Les3 → RESOLVED 2026-09-17** by switching to `stabilityai/stable-diffusion-3-medium-diffusers`, still on the free `hf-inference` provider (see [Update](#update-2026-09-17--les3-sdxl-resolved-sd3-medium)). **Les9/Les10 still pending:** migrate to `langchain_huggingface.HuggingFaceEndpoint` / the router with an HF token, or run models locally.
3. **All referenced model repos still exist** — none were deleted. The failures are about *access method*, not missing models.

## Per-lesson detail

### Les2 — GenAI? — ok
`langdetect==1.0.9` installs and detects nl/en/fr correctly. No hosted model. **No fix needed.**

### Les3 — Beelden Genereren — ~~broken + content mismatch~~ → resolved
**Correction (2026-09-17):** the earlier "content mismatch" was wrong. The current deck (id 5111) is a single **three-part** lesson — PGAN face *generation* (`torch.hub` `celebAHQ-512`), SDXL text→image *generation*, and face *recognition* (`cv2`/`mediapipe` on `gezichten.jpg`) — so the repo's mediapipe/opencv/`gezichten.jpg` are part of the lesson, and `requirements.txt` matches (byte-identical to the slide's download). **No repo change was needed.**
- PGAN via torch.hub **works** (264 MB weights, generated a 512×512 face).
- SDXL text→image **was broken**: `api-inference.huggingface.co` is retired (DNS gone) and `hf-inference` no longer serves SDXL (`router.huggingface.co/hf-inference/…/stable-diffusion-xl-base-1.0` → **410**). **Fixed** by switching to `stabilityai/stable-diffusion-3-medium-diffusers` — the one current text-to-image model still on the free `hf-inference` provider, returning raw image bytes to the lesson's unchanged `requests.post` code (verified: HTTP 200 → `image/jpeg`).
- Face recognition **works** on the pre-shipped `gezichten.jpg` regardless of the generator.
**Applied on a draft copy of the deck (id 7389):** the two SDXL slides (seq 37 & 43) re-pointed to the SD3-medium router URL, plus a gate note on the token slide (seq 36). Reviewed and **published** by the content team (2026-09-18). See [Update](#update-2026-09-17--les3-sdxl-resolved-sd3-medium).

### Les4 — Text2Speech — ok
`coqui-tts==0.26.2` installs on 3.12; **all 5 `tts_models/*` named on the slides still resolve** in the registry; synthesized a wav with `tts_models/en/ljspeech/tacotron2-DDC`. (The original coqui.ai shut down, but the maintained `coqui-tts` fork + its rehosted registry work.) **No fix needed.**

### Les5 — Word Embeddings — ~~broken (data unavailable)~~ → resolved
Code logic verified against a stand-in (loader + `cosine_similarity`: falcon≈eagle 0.99, falcon≠armchair 0.06). But the lesson can't run as shipped: the GloVe file is git-LFS and **LFS is disabled** (cross-cutting #1), and the code reads `...txt` while the repo ships `...zip` (needs an unzip step). **Fix:** restore GloVe availability (re-enable LFS / re-host) and add an unzip step or ship the `.txt`. **Resolved 2026-09-17:** GloVe re-hosted as `.txt`, slide 29 updated, repo `.zip` removed, notebook links to the new URL (see [Update](#update-2026-09-17--glove-data-resolved)).

### Les6 — The Word Embedding Game — ~~broken (data unavailable)~~ → resolved
Same GloVe data problem as Les5. Core embedding/similarity logic is fine; the interactive part is `ipywidgets` (notebook-only, expected). **Fix:** same as Les5. **Resolved 2026-09-17:** same re-host; slides 16 & 17 updated, repo `.zip` removed, notebook links to the new URL (see [Update](#update-2026-09-17--glove-data-resolved)).

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

1. ~~**Re-enable git-LFS (or re-host GloVe)** — unblocks Les5 & Les6.~~ **✅ DONE 2026-09-17** (re-hosted as `.txt`; slides + repo updated). *(cross-cutting #1)*
2. **Migrate off the retired HF Inference API** — **Les3 ✅ DONE 2026-09-17** (SD3-medium on `hf-inference`); **Les9 & Les10 still pending** (→ `langchain_huggingface` / `router.huggingface.co` + HF token, or local models). *(cross-cutting #2)*
3. ~~**Reconcile the Les3 repo folder**~~ — **dropped:** re-investigation found no mismatch. The repo folder matches the current three-part lesson; `requirements.txt` and `gezichten.jpg` are correct as shipped.
4. *(Optional, non-blocking)* bump `pypdfium2` in Les11; migrate deprecated `langchain.embeddings` imports in Les7/8/11. *(The Les5 `.zip`→`.txt` step is now moot — the file is served as `.txt`.)*

## Notes & limitations

- Les9/10 were verified **reachability + import/deprecation only** (no HF token), per scope. Their model *repos* are confirmed present; a full live-inference test would need a token and the migrated endpoint.
- The verification env is Python 3.12 (via uv). The host default Python is 3.14, on which several of these pinned stacks would not install — worth noting for anyone reproducing without uv.
