# Maistro lessons health check — design (TR-4556)

- **Ticket:** [TR-4556](https://codefever.atlassian.net/browse/TR-4556) — "Check Maistro lessons: verify code still runs and models still available" (Task)
- **Repo:** https://github.com/CodeFever-VZW/Maistros1
- **Branch:** `✨/TR-4556-verify-lesson-code-and-models`
- **Date:** 2026-09-09

## 1. Problem

The Maistro lessons are an AI/NLP course (`Les2`–`Les12`; `Les1` was deleted). This is a **periodic health check**: confirm the lesson code still runs against its pinned environment and that every AI model/endpoint the lessons reference still exists and is reachable.

Key constraint discovered during exploration: **the repo notebooks are student scaffolds, not solutions.** Each `notebook.ipynb` contains only a title, comment-instructions, a few helper functions, and empty/placeholder cells (e.g. `Les9` sets `huggingfacehub_api_token = "..."` and stops). The actual working code — and the concrete model IDs — live on the **studio slides**, so the health check must read the slides to know what to run and which models to check.

## 2. Goals & success criteria

Maps directly to the ticket's acceptance criteria:

1. **Lesson code executes without errors against the current environment** — verified at the "pragmatic" bar defined in §5.
2. **All AI models referenced in the lessons still exist / are reachable** — checked per lesson.
3. **Any broken code or unavailable model is documented with the fix needed** — captured in the report.

**Definition of done:** `docs/health-check-TR-4556.md` committed, containing a per-lesson status table + the exact fix needed for every break, with every lesson `Les2`–`Les12` covered (or explicitly marked "could not verify" with the reason).

## 3. Scope / non-goals

**In scope:** the 11 lessons `Les2`–`Les12`; extracting their solution code + model references from studio; verifying install + run + model reachability; writing the report.

**Non-goals (this pass):**
- Editing the lesson notebooks or `requirements.txt` files. Findings are documented as "fix needed", not applied. (A follow-up ticket can apply fixes.)
- Fixing the unrelated stray git index state beyond a non-destructive unstage needed to make clean commits (see §7).
- Auditing the eRiders2 repo (used only as a reference for the studio mechanism).

## 4. Phase 1 — Extract solutions from studio

**Mechanism** (adapted from `eRiders2/tools/studio-dump.js` + `extract-dump.js`):

1. Playwright (MCP) opens `https://studio.ftrprf.be` and logs in with the `.env` credentials (`login=DIGAIKoen`).
2. Once authenticated, run the GraphQL enumerate-and-fetch **inside the page context** (`page.evaluate` → `fetch`), so the browser's own `Authorization: Bearer <idToken>` session is used and no token has to be extracted by hand:
   - `findAllLessonContent(page, size:200, filter:[])` — paginate to enumerate every lesson the account can see.
   - `findLessonContent(id){ … slides{ id title content sequence … questions hints } }` — full deck per lesson.
3. **Discover the Maistro lesson filter at runtime.** The eRiders tool filters titles by `/e\s*[123]\s*-\s*les/i`; the Maistro/DIG-AI titles are unknown until enumerated. List titles, identify the lesson set, map each studio lesson → repo folder `LesN` by lesson number.
4. Convert each slide's HTML `content` → readable markdown with fenced code blocks (reuse the `analyzeHtml` logic from `extract-dump.js`).

**Output location:** raw dump JSON + per-lesson markdown extracts go to the **scratchpad**, not git — they are third-party course content and the flow touches credentials.

**Fallback (main risk):** if the studio login is federated / has 2FA and Playwright can't complete it, fall back to (a) interactive login where the user completes the SSO/2FA step, or (b) the user runs the `studio-dump.js` snippet manually and hands over the JSON. Confirm the login type early; do not burn time fighting an SSO wall.

## 5. Phase 2 — Per-lesson verification (pragmatic bar)

**Environments:** one venv per **unique** `requirements.txt`. `Les7`≡`Les8` and `Les9`≡`Les10` are byte-identical, so ~9 venvs. Creating the venv **is** part of the test: **a pinned dependency that no longer installs on the current Python/OS is a finding**, recorded (not a blocker) so verification continues for the other lessons. Record the Python version used.

**Per-lesson bar** — reconstruct the slide solution and run it to the level below, logging the outcome either way:

| Class | Lessons | Bar |
|---|---|---|
| Light | `Les2` (langdetect), `Les5`/`Les6` (local GloVe), `Les7`/`Les8` (local sentence embeddings), `Les11` (embeddings + FAISS + PDF) | Run end-to-end. Includes unzipping the GloVe `.zip`→`.txt` (a known gap: `Les5` references `subset_lower_glove.42B.300d.txt` but the repo ships the `.zip`). |
| Heavy | `Les3` (mediapipe / face_recognition / opencv / torch), `Les4` (coqui-TTS + gruut + torchaudio), `Les12` (transformers + torch) | Install, load/download the model, run one representative inference. Note download size/time. |
| HF-inference | `Les9`/`Les10` (langchain + huggingface-hub, uses `HUGGINGFACEHUB_API_TOKEN`) | **Reachability only** — confirm the referenced model repo/endpoint still exists and is reachable; do **not** execute the authenticated inference call (no HF token this pass). |

**Model reachability method:** for HuggingFace repos, check existence/metadata via `huggingface_hub` (`model_info`) or an unauthenticated HTTP request to `huggingface.co/<repo>`. Flag known deprecation traps and confirm each against the actual slide code:
- `sentence-transformers==2.2.2` + `InstructorEmbedding` is a known-broken combo with modern `huggingface_hub` (`cached_download` removed). Affects `Les7`/`Les8`/`Les11`.
- langchain's `HuggingFaceHub` LLM class is deprecated / the serverless HF Inference API surface changed. Affects `Les9`/`Les10`.
- coqui-TTS (`Les4`) model names are resolved from its model registry / HF — verify the specific model named on the slide still resolves.

**Independence:** each lesson is a self-contained unit (own env, own data, own model) → the plan can fan these out to parallel subagents, one per lesson (or per env group).

## 6. Phase 3 — Report

Write `docs/health-check-TR-4556.md`:

- **Summary table:** lesson · installs? · runs (to bar)? · model reachable? · severity (ok / warning / broken).
- **Per-lesson detail:** what the lesson does, which model(s) it references, what happened, and — for anything broken — the **exact fix needed** (e.g. "bump `sentence-transformers` to ≥2.3 / pin `huggingface_hub<0.26`", "model `X` moved to `Y`", "unzip GloVe in a setup cell").
- **Environment note:** Python version, OS, date, and which lessons were verified full-run vs reachability-only.
- **No secrets:** no credentials, tokens, or raw slide dumps in the report.

## 7. Housekeeping & hygiene

- **Stray git index state:** the working tree currently has every tracked file staged as a deletion (a `git rm -r --cached .` artifact) while the files are all present on disk and byte-identical to `HEAD`. It is unrelated to this ticket. To keep commits clean, either path-scope each commit to only the intended files, or (preferred, non-destructive) `git reset` to unstage — this restores a pristine tree without touching any file. Confirm with the user before the reset since the state predates this work.
- **`.env`** stays untracked/ignored; never commit or echo the credentials.
- Studio dump + extracts stay out of git (scratchpad or gitignore).

## 8. Risks & open questions

1. **Studio login automation** — is it a plain username/password form, or federated (Cognito hosted UI / Google / MS) with 2FA? Determines whether Phase 1 is fully automated. Fallback ready (§4).
2. **Heavy installs on Windows** — some pinned versions (coqui-TTS, mediapipe, torch builds) may not install on the current Python/OS; those become documented findings rather than blockers.
3. **Slide ↔ lesson mapping** — confirmed only after enumeration; if a repo lesson has no matching slide deck (or vice versa) it is flagged in the report.
