# Maistro Lessons Health Check — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Verify every Maistro lesson (`Les2`–`Les12`) still installs, runs (at a pragmatic bar), and references AI models that are still reachable — and document every break with the exact fix needed, in `docs/health-check-TR-4556.md`.

**Architecture:** Three phases. (1) Pull each lesson's *solution* code + model IDs from the studio slides (repo notebooks are empty scaffolds) via Playwright login + the studio GraphQL API. (2) For each lesson, build a venv from its pinned `requirements.txt`, reconstruct the slide solution, and verify it to a per-lesson bar (light = full run, heavy = load+one inference, HF-inference = reachability only). (3) Assemble the findings into one committed markdown report.

**Tech Stack:** Playwright (MCP browser), Node.js (studio pull + HTML→markdown extract, reusing the eRiders2 tooling), Python venvs + pip (per-lesson), `huggingface_hub`/HTTP for model reachability. Windows + PowerShell host.

## Global Constraints

- **Spec:** `docs/superpowers/specs/2026-09-09-maistro-lessons-health-check-design.md` (authoritative).
- **Lessons in scope:** `Les2, Les3, Les4, Les5, Les6, Les7, Les8, Les9, Les10, Les11, Les12`. (`Les1` was deleted.)
- **Deliverable:** report only. **Do NOT edit** the lesson `notebook.ipynb` / `requirements.txt` files — findings are documented, not applied.
- **Secrets hygiene:** the studio credentials (`.env`) and any captured idToken/Bearer token are secrets. Never commit them, never write them into the report, never paste them into a commit message. Ensure `.env` is git-ignored.
- **Out of git:** the raw studio dump and per-lesson slide extracts are third-party course content — they live only in the scratchpad, never committed.
- **Pragmatic bar (per lesson class):**
  - **Light** — run end-to-end: `Les2`, `Les5`, `Les6`, `Les7`, `Les8`, `Les11`.
  - **Heavy** — install + load/download model + one representative inference: `Les3`, `Les4`, `Les12`.
  - **HF-inference** — reachability only, no authenticated call: `Les9`, `Les10`.
- **Install failure is a finding, not a blocker** — record it and continue with the other lessons.
- **Env faithfulness:** one venv per *unique* `requirements.txt`. Identical files: `Les7`≡`Les8`, `Les9`≡`Les10` (share one venv each). Every other lesson gets its own.
- **Paths** (define once; used by every task):
  - `REPO` = `C:\Users\koen\Documents\Maistros1`
  - `SCRATCH` = `C:\Users\koen\AppData\Local\Temp\claude\C--Users-koen-Documents-Maistros1\28e9ab3d-1292-4fc6-a3b4-470ea24ccb9d\scratchpad`
  - `SCRATCH\studio-dump.json` — raw GraphQL dump
  - `SCRATCH\extracts\LesN.md` — per-lesson solution extract (code + model refs)
  - `SCRATCH\venvs\lesN\` — per-lesson venv
  - `SCRATCH\run\LesN\` — reconstructed runnable solution + any run outputs
  - `SCRATCH\findings\LesN.md` — per-lesson finding (intermediate; feeds the report)
  - `SCRATCH\tools\` — the two Node scripts below
- **Reachability helper** (used by every verification task) — write once at `SCRATCH\tools\check_model.py`:

```python
# check_model.py — usage: python check_model.py <hf_repo_id> [<hf_repo_id> ...]
import sys, requests
from huggingface_hub import HfApi
api = HfApi()
for repo in sys.argv[1:]:
    try:
        info = api.model_info(repo)  # unauthenticated; works for public repos
        gated = getattr(info, "gated", False)
        dls = getattr(info, "downloads", "?")
        print(f"OK        {repo}  (gated={gated}, downloads={dls})")
    except Exception as e:
        # Fall back to a raw HEAD so we can distinguish 404 (gone) from 401 (gated) from network
        try:
            r = requests.head(f"https://huggingface.co/{repo}", timeout=15, allow_redirects=True)
            print(f"HTTP {r.status_code}  {repo}  ({e.__class__.__name__})")
        except Exception as e2:
            print(f"UNREACHABLE {repo}  -> {e2}")
```

- **Finding file format** (every verification task writes this shape to `SCRATCH\findings\LesN.md`):

```markdown
# LesN — <one-line what the lesson does>
- **Class:** light | heavy | hf-inference
- **Models referenced:** <repo id(s) / endpoint / "local GloVe" / "none">
- **Installs?** yes | no (<pip error summary>)
- **Runs (to bar)?** yes | no (<error summary>)
- **Model reachable?** yes | no | n/a (<check_model.py output>)
- **Severity:** ok | warning | broken
- **Fix needed:** <exact, actionable fix, or "none">
- **Notes:** <download size/time, versions, anything relevant>
```

---

## Task 1: Workspace hygiene + toolchain probe

**Files:**
- Modify: `REPO\.gitignore` (ensure `.env` is ignored)
- Create: `SCRATCH\extracts\`, `SCRATCH\findings\`, `SCRATCH\venvs\`, `SCRATCH\run\`, `SCRATCH\tools\` (directories)
- Create: `SCRATCH\tools\check_model.py` (from Global Constraints)

**Interfaces:**
- Produces: a clean git working tree, the recorded Python/Node versions (for the report's environment note), and the scratchpad skeleton every later task uses.

- [ ] **Step 1: Surface the stray git index state and get the user's OK to unstage it**

The index has ~53 files staged as deletions (a `git rm -r --cached .` artifact); all files are present on disk and byte-identical to `HEAD`. Confirm with the user, then unstage (non-destructive — touches no files):

```powershell
git --no-pager status --short | Select-Object -First 5   # show the state
# After user confirms:
git reset            # unstage everything; working files untouched
git --no-pager status --short
```
Expected after reset: no staged changes; untracked shows only `.env` (and `docs/` already committed). If the user declines the reset, skip it and path-scope the final report commit instead (Task 12).

- [ ] **Step 2: Ensure `.env` is git-ignored**

Read `REPO\.gitignore` (present on disk). If `.env` is not already matched, append it under an appropriate section. Verify:

```powershell
git check-ignore .env    # expected output: .env
```
Expected: prints `.env`. If it prints nothing, add the line and re-check.

- [ ] **Step 3: Probe Python + Node toolchain and record versions**

```powershell
py -0p 2>$null; python --version; py --version; node --version; npm --version
```
Record the Python version chosen for venvs and the Node version — these go in the report's environment note. Prefer the newest Python that exists (venvs are created with `py -3.X -m venv`).

- [ ] **Step 4: Create the scratchpad skeleton + reachability helper**

```powershell
foreach ($d in 'extracts','findings','venvs','run','tools') { New-Item -ItemType Directory -Force "$env:SCRATCH_PATH\$d" | Out-Null }
```
(Use the literal `SCRATCH` path from Global Constraints.) Then write `SCRATCH\tools\check_model.py` with the exact content from Global Constraints.

- [ ] **Step 5: Sanity-check the helper runs**

```powershell
# quick smoke test against a definitely-existing public model, using any venv or base python with huggingface_hub+requests
python "SCRATCH\tools\check_model.py" sentence-transformers/all-MiniLM-L6-v2
```
Expected: a line starting `OK ` or `HTTP 200`. (If `huggingface_hub`/`requests` aren't in base Python, defer this smoke test to the first lesson venv that has them — note that in the finding.)

- [ ] **Step 6: No commit** — this task changes only `.gitignore` (staged, committed with the report in Task 12) and scratchpad. Leave `.gitignore` staged.

---

## Task 2: Extract lesson solutions from studio (Phase 1)

**Files:**
- Create: `SCRATCH\tools\pull-studio.mjs`
- Create: `SCRATCH\tools\extract.mjs`
- Create: `SCRATCH\studio-dump.json` (generated)
- Create: `SCRATCH\extracts\LesN.md` (generated, one per matched lesson)
- Create: `SCRATCH\extracts\_mapping.md` (studio lesson → repo `LesN` mapping table)

**Interfaces:**
- Consumes: `.env` credentials; the studio GraphQL API `https://studio-backend.ftrprf.be/graphql`.
- Produces: `SCRATCH\extracts\Les2.md` … `Les12.md` (each = the lesson's solution code + concrete model IDs, as markdown), plus `_mapping.md`. Every later verification task reads its `SCRATCH\extracts\LesN.md`.

> **Run this task in the main session (interactive-capable), not a detached background subagent** — the login may need a human for SSO/2FA.

- [ ] **Step 1: Log in to studio with Playwright**

Using the Playwright MCP: `browser_navigate` to `https://studio.ftrprf.be`, then fill the login form with the `.env` values (`login=DIGAIKoen`, the password) via `browser_fill_form`/`browser_type` + submit. Read `.env` for the exact values; never echo the password into logs.

**Decision point:** if the page redirects to a federated identity provider (Google/Microsoft/Cognito hosted UI) or asks for 2FA that the credentials alone can't satisfy → **fallback**: ask the user to complete the login live in the same browser, OR to run `eRiders2/tools/studio-dump.js` manually and hand over the JSON (then skip to Step 4 pointing `studio-dump.json` at their file). Do not spend more than a couple of attempts automating a federated wall.

- [ ] **Step 2: Capture the idToken**

After login, run `browser_evaluate` to locate the bearer/idToken the SPA uses. Try, in order: `localStorage`/`sessionStorage` keys (Cognito stores it under a key containing `idToken`), then a decoded copy of any `authorization` header. Return only the token string.

```js
// browser_evaluate body — returns the idToken or null
() => {
  const store = { ...localStorage, ...sessionStorage };
  for (const [k, v] of Object.entries(store)) {
    if (/idtoken/i.test(k) && typeof v === 'string' && v.split('.').length === 3) return v;
  }
  // Cognito sometimes nests it in a JSON blob:
  for (const v of Object.values(store)) {
    try { const o = JSON.parse(v); if (o && typeof o.idToken === 'string') return o.idToken; } catch {}
  }
  return null;
}
```
Treat the returned token as a secret: pass it to the next step via an environment variable, never write it to a committed file. If `null`, capture it instead from a live GraphQL request's `authorization` header (reload the app with the Network panel / use `browser_network_requests` to read the header).

- [ ] **Step 3: Pull all Maistro decks with the Node puller**

Write `SCRATCH\tools\pull-studio.mjs` (adapted from `eRiders2/tools/studio-dump.js`, but running under Node with the token from an env var and writing to disk instead of a browser download):

```js
// pull-studio.mjs — usage: STUDIO_TOKEN=<idToken> node pull-studio.mjs <out.json>
import { writeFileSync } from 'node:fs';
const token = (process.env.STUDIO_TOKEN || '').replace(/^Bearer\s+/i, '').trim();
const out = process.argv[2] || 'studio-dump.json';
if (!token) { console.error('Set STUDIO_TOKEN'); process.exit(1); }
const endpoint = 'https://studio-backend.ftrprf.be/graphql';
async function gql(query, variables) {
  const res = await fetch(endpoint, {
    method: 'POST',
    headers: { 'content-type': 'application/json', authorization: 'Bearer ' + token },
    body: JSON.stringify({ query, variables }),
  });
  if (res.status === 401) throw new Error('401 — token expired; grab a fresh one');
  const j = await res.json();
  if (j.errors) throw new Error('GraphQL: ' + JSON.stringify(j.errors).slice(0, 400));
  return j.data;
}
const LIST_Q = `query FindLessons($page:Int!){ findAllLessonContent(page:$page, size:200, filter:[]){ total pages currentPage content { id title level clan published version type } } }`;
const LESSON_Q = `query FindLessonContent($id:Int!){ findLessonContent(id:$id){ id title type language level version published summary coach description slides { id title content sequence viewModes motivation info questions { id type ... on QuestionOpen { value solution placeholder } ... on QuestionMultipleChoice { value questionAnswersMultipleChoice { id value correct explanation } } } hints { id title content } } } }`;

const first = await gql(LIST_Q, { page: 0 });
const meta = first.findAllLessonContent;
let all = meta.content.slice();
for (let p = 1; p < meta.pages; p++) all = all.concat((await gql(LIST_Q, { page: p })).findAllLessonContent.content);

// Print every title once so we can eyeball the Maistro filter, then keep the AI-course lessons.
console.error('--- ALL TITLES (' + all.length + ') ---');
for (const l of all) console.error(`${l.id}\t${l.clan || ''}\t${l.title}`);
const RX = /(maistro|dig[\s-]*ai|\bles\s*(?:0?[2-9]|1[0-2])\b)/i;   // widen/narrow after seeing titles
const picked = all.filter(l => RX.test(l.title || '')).sort((a,b) => (a.title||'').localeCompare(b.title||'', 'nl', { numeric: true }));
console.error(`Picked ${picked.length} lessons.`);

const lessons = [], failed = [];
for (const { id, title } of picked) {
  try { lessons.push((await gql(LESSON_Q, { id })).findLessonContent); console.error(`  ok  ${title} (${id})`); }
  catch (e) { failed.push({ id, title, error: String(e) }); console.error(`  FAIL ${title} (${id}) ${e}`); }
}
writeFileSync(out, JSON.stringify({ pickedIndex: picked, failed, lessons }, null, 2));
console.error(`Wrote ${out} (${lessons.length} lessons).`);
```

Run it (token via env var so it never lands on disk):

```powershell
$env:STUDIO_TOKEN = "<idToken captured in Step 2>"
node "SCRATCH\tools\pull-studio.mjs" "SCRATCH\studio-dump.json"
Remove-Item Env:STUDIO_TOKEN
```
Expected: stderr prints the full title list + `Wrote …studio-dump.json (N lessons)`. **Inspect the printed titles**; if the `RX` filter caught the wrong set, adjust `RX` and re-run. Confirm the picked set covers lessons 2–12.

- [ ] **Step 4: Convert decks to per-lesson markdown extracts**

Write `SCRATCH\tools\extract.mjs` — the `analyzeHtml` HTML→markdown logic from `eRiders2/tools/extract-dump.js`, but keyed to write `Les<N>.md` where `N` is the lesson number parsed from each title (`/les\s*(\d+)/i`), into `SCRATCH\extracts\`. (Copy the `decode`/`analyzeHtml` functions verbatim from `extract-dump.js`; change only the input path, the output dir, and the filename key.) Run:

```powershell
node "SCRATCH\tools\extract.mjs" "SCRATCH\studio-dump.json" "SCRATCH\extracts"
```
Expected: writes `Les2.md`…`Les12.md`. Each should contain fenced code blocks (the exercise/solution code) and the concrete model IDs.

- [ ] **Step 5: Write the mapping table and verify coverage**

Create `SCRATCH\extracts\_mapping.md`: a table of `repo folder LesN | studio lesson title | studio id | slides | code blocks | model refs spotted`. Cross-check every repo folder `Les2`–`Les12` has a matching extract. **Flag any repo lesson with no matching studio deck (or vice-versa)** — that is itself a finding for the report.

- [ ] **Step 6: No commit** — all outputs are scratchpad-only (course content + token-adjacent). Nothing enters git here.

---

## Verification tasks (Tasks 3–11)

Each verification task follows the **same protocol** (stated here once; each task below gives only its lesson-specific specifics — env, requirements path, class/bar, what to run, models to check):

1. **Build/reuse venv:** `py -3.X -m venv SCRATCH\venvs\lesN`, then `SCRATCH\venvs\lesN\Scripts\python -m pip install -r REPO\LesN\requirements.txt`. Capture the pip result (success, or the first real error). For shared envs (`Les7`/`Les8`, `Les9`/`Les10`) build once and reuse.
2. **Reconstruct the solution:** copy the fenced code from `SCRATCH\extracts\LesN.md` into `SCRATCH\run\LesN\solution.py` (or a runnable `.ipynb`), wiring in the lesson's local data files from `REPO\LesN\`.
3. **Run to the bar** for the lesson's class (light/heavy/hf-inference — see Global Constraints).
4. **Check models:** run `python SCRATCH\tools\check_model.py <repo ids from the extract>` for every model the slide code names.
5. **Write** `SCRATCH\findings\LesN.md` in the finding-file format (Global Constraints). Set severity: `ok` (installs+runs+reachable), `warning` (runs with deprecation/workaround, or non-fatal gap), `broken` (install fails, run errors, or model gone).
6. **No git commit** (findings are scratchpad; the report is committed once in Task 12).

Tasks 3–11 are mutually independent and may be dispatched in parallel (distinct venv/run/finding paths; the only shared read-only inputs are the extracts and `check_model.py`).

### Task 3: Verify Les2 — language detection (light)

**Files:** venv `SCRATCH\venvs\les2`; reqs `REPO\Les2\requirements.txt` (`langdetect==1.0.9`); extract `SCRATCH\extracts\Les2.md`; finding `SCRATCH\findings\Les2.md`.
**Interfaces:** Consumes `Les2.md`. Produces `findings\Les2.md`.

- [ ] **Step 1:** Build venv + `pip install -r Les2/requirements.txt`; record result.
- [ ] **Step 2:** Reconstruct the slide solution into `run\Les2\solution.py`.
- [ ] **Step 3:** Run it end-to-end (light bar). `langdetect` is fully local — expect a language label as output. Record any error.
- [ ] **Step 4:** Models referenced: none (local library). Record "Model reachable? n/a".
- [ ] **Step 5:** Write `findings\Les2.md`.

### Task 4: Verify Les3 — face detection/recognition (heavy)

**Files:** venv `les3`; reqs `Les3/requirements.txt` (mediapipe, opencv, face_recognition_models, torch, torchvision); data `Les3\gezichten.jpg`; extract `Les3.md`; finding `Les3.md`.
**Interfaces:** Consumes `Les3.md` + `gezichten.jpg`. Produces `findings\Les3.md`.

- [ ] **Step 1:** Build venv + install. **Watch for Windows/Python wheel availability** of `mediapipe==0.10.20`, `face_recognition_models`, `torch==2.5.1` — an install failure here is the finding.
- [ ] **Step 2:** Reconstruct the slide solution into `run\Les3\solution.py`, pointing at `REPO\Les3\gezichten.jpg`.
- [ ] **Step 3:** Heavy bar — load the model(s) and run one representative inference (detect faces in `gezichten.jpg`); confirm it returns detections. Note model download size/time.
- [ ] **Step 4:** Model reachability: whatever HF/model assets the slide code downloads (e.g. mediapipe face model, any HF repo) — run `check_model.py` for HF repos; for mediapipe's bundled assets note whether the download URL resolves.
- [ ] **Step 5:** Write `findings\Les3.md`.

### Task 5: Verify Les4 — text-to-speech / speech (heavy)

**Files:** venv `les4`; reqs `Les4/requirements.txt` (coqui-tts, gruut, transformers, torchaudio, librosa); data `Les4\*.wav`, `Les4\woorden-fr\*.wav`, `Les4\words.txt`; extract `Les4.md`; finding `Les4.md`.
**Interfaces:** Consumes `Les4.md` + the wav/words data. Produces `findings\Les4.md`.

- [ ] **Step 1:** Build venv + install. `coqui-tts==0.26.2` + its native deps are the risk — record install outcome.
- [ ] **Step 2:** Reconstruct the slide solution into `run\Les4\solution.py`. Identify the exact TTS model name the slides use (coqui model id like `tts_models/<lang>/…`, and any STT/Whisper HF repo).
- [ ] **Step 3:** Heavy bar — load the TTS model and synthesize one short phrase to a wav in `run\Les4\`; if the lesson also does STT, transcribe one provided wav. Note download size/time.
- [ ] **Step 4:** Model reachability: run `check_model.py` for any HF repo ids; for the coqui model id, confirm it still resolves in the coqui model registry (`python -c "from TTS.api import TTS; print([m for m in TTS().list_models() if '<lang>' in m])"` or equivalent).
- [ ] **Step 5:** Write `findings\Les4.md`.

### Task 6: Verify Les5 — word embeddings / GloVe (light)

**Files:** venv `les5`; reqs `Les5/requirements.txt` (numpy); data `Les5\subset_lower_glove.42B.300d.zip`; extract `Les5.md`; finding `Les5.md`.
**Interfaces:** Consumes `Les5.md` + the GloVe zip. Produces `findings\Les5.md`.

- [ ] **Step 1:** Build venv + install (numpy only — should be trivial).
- [ ] **Step 2:** **Unzip** `subset_lower_glove.42B.300d.zip` → `subset_lower_glove.42B.300d.txt` into `run\Les5\` (the slide code reads the `.txt`, but the repo ships only the `.zip` — record this as a known gap needing a fix, e.g. an unzip step or shipping the `.txt`). Reconstruct the solution into `run\Les5\solution.py`.
- [ ] **Step 3:** Light bar — run end-to-end: load embeddings into the dict, print word count + the embedding for `chicken`, and compute a `cosine_similarity` (the helper is already in the notebook). Confirm sensible output.
- [ ] **Step 4:** Models referenced: local GloVe file (not a hosted model) → "Model reachable? n/a", but note the `.zip`/`.txt` mismatch under Fix needed.
- [ ] **Step 5:** Write `findings\Les5.md`.

### Task 7: Verify Les6 — word-embedding game (light)

**Files:** venv `les6`; reqs `Les6/requirements.txt` (numpy, ipywidgets); GloVe from Les5; extract `Les6.md`; finding `Les6.md`.
**Interfaces:** Consumes `Les6.md` + GloVe txt (reuse the unzip from Task 6 if present). Produces `findings\Les6.md`.

- [ ] **Step 1:** Build venv + install.
- [ ] **Step 2:** Reconstruct the solution into `run\Les6\solution.py`; reuse the unzipped GloVe txt. Note: `ipywidgets` UI can't render headless — exercise the underlying logic (the embedding/similarity functions), not the widget display.
- [ ] **Step 3:** Light bar — run the non-widget core end-to-end; confirm output. Record if any part is inherently notebook-widget-only.
- [ ] **Step 4:** Models referenced: local GloVe → n/a.
- [ ] **Step 5:** Write `findings\Les6.md`.

### Task 8: Verify Les7 & Les8 — sentence embeddings (light, shared env)

**Files:** venv `les7-8` (shared; reqs identical); reqs `Les7/requirements.txt`; data `Les7\imdb_labelled.txt`, `Les8\imdb_labelled.txt`; extracts `Les7.md`, `Les8.md`; findings `Les7.md`, `Les8.md`.
**Interfaces:** Consumes `Les7.md`, `Les8.md` + imdb data. Produces `findings\Les7.md` and `findings\Les8.md`.

- [ ] **Step 1:** Build the shared venv `les7-8` once + install. **Expect trouble:** `sentence-transformers==2.2.2` + `InstructorEmbedding==1.0.1` is known-broken against modern `huggingface_hub` (removed `cached_download`). Record the exact import/runtime error if it occurs.
- [ ] **Step 2:** Reconstruct both solutions into `run\Les7\solution.py` and `run\Les8\solution.py`. Identify the embedding model id the slides use (e.g. `hkunlp/instructor-large` / a `sentence-transformers/*` model).
- [ ] **Step 3:** Light bar — run each end-to-end (embed the imdb sentences; whatever downstream task the slides do). Record errors per lesson.
- [ ] **Step 4:** `check_model.py <embedding repo id>` for the model(s) named in each extract.
- [ ] **Step 5:** Write `findings\Les7.md` and `findings\Les8.md` (may share the same root cause; state it in each).

### Task 9: Verify Les9 & Les10 — LLM via HuggingFace (hf-inference, shared env)

**Files:** venv `les9-10` (shared); reqs `Les9/requirements.txt` (langchain, langchain-community, huggingface-hub); extracts `Les9.md`, `Les10.md`; findings `Les9.md`, `Les10.md`.
**Interfaces:** Consumes `Les9.md`, `Les10.md`. Produces `findings\Les9.md`, `findings\Les10.md`.

- [ ] **Step 1:** Build shared venv `les9-10` + install. Record result.
- [ ] **Step 2:** From each extract, identify the exact model repo id(s) and the langchain class used (e.g. `HuggingFaceHub`, `HuggingFaceEndpoint`, `HuggingFaceEndpointEmbeddings`). Note: `HuggingFaceHub` LLM is deprecated in current `langchain-community`.
- [ ] **Step 3:** HF-inference bar — **do not run the authenticated call**. Instead: (a) `check_model.py <repo id>` to confirm the model repo still exists; (b) note whether the model is still served by the serverless HF Inference API and whether the langchain class the slides use is deprecated/removed (import it and record any `DeprecationWarning`/`ImportError`). No HF token is used.
- [ ] **Step 4:** Write `findings\Les9.md`, `findings\Les10.md`. Severity `warning` if the model exists but the code path is deprecated; `broken` if the model repo or the langchain class is gone.

### Task 10: Verify Les11 — RAG over PDFs with FAISS (light)

**Files:** venv `les11`; reqs `Les11/requirements.txt` (langchain, faiss-cpu, sentence-transformers, InstructorEmbedding, pdfplumber); data `Les11\*.pdf`, `Les11\sporten_faiss_index.bin`; extract `Les11.md`; finding `Les11.md`.
**Interfaces:** Consumes `Les11.md` + the PDFs + the prebuilt FAISS index. Produces `findings\Les11.md`.

- [ ] **Step 1:** Build venv `les11` + install. Same `sentence-transformers 2.2.2`+`InstructorEmbedding` risk as Task 8 — record.
- [ ] **Step 2:** Reconstruct the solution into `run\Les11\solution.py`; point PDF loading + FAISS load at `REPO\Les11\` files. Identify the embedding model id.
- [ ] **Step 3:** Light bar — run end-to-end: load/parse the PDFs, load `sporten_faiss_index.bin` (or rebuild it), embed a query, and do a similarity search returning a hit. Record errors. Note if the prebuilt `.bin` is incompatible with the installed faiss/embedding versions (a likely finding).
- [ ] **Step 4:** `check_model.py <embedding repo id>`.
- [ ] **Step 5:** Write `findings\Les11.md`.

### Task 11: Verify Les12 — transformers model (heavy)

**Files:** venv `les12`; reqs `Les12/requirements.txt` (transformers, torch, huggingface-hub, tokenizers); extract `Les12.md`; finding `Les12.md`.
**Interfaces:** Consumes `Les12.md`. Produces `findings\Les12.md`.

- [ ] **Step 1:** Build venv `les12` + install. Record result.
- [ ] **Step 2:** Reconstruct the solution into `run\Les12\solution.py`. Identify the exact HF model repo id the slides load.
- [ ] **Step 3:** Heavy bar — load the model via `transformers` and run one representative inference (the task the lesson demonstrates); confirm output. Note download size/time.
- [ ] **Step 4:** `check_model.py <repo id>`.
- [ ] **Step 5:** Write `findings\Les12.md`.

---

## Task 12: Assemble and commit the report (Phase 3)

**Files:**
- Create: `REPO\docs\health-check-TR-4556.md`
- (already staged from Task 1: `REPO\.gitignore`)

**Interfaces:**
- Consumes: `SCRATCH\findings\Les2.md` … `Les12.md`, `SCRATCH\extracts\_mapping.md`, and the toolchain versions from Task 1.
- Produces: the committed report — the ticket deliverable.

- [ ] **Step 1: Verify all findings exist**

```powershell
Get-ChildItem "SCRATCH\findings\Les*.md" | Select-Object Name
```
Expected: `Les2`…`Les12` (11 files). Any missing lesson must appear in the report as "could not verify — <reason>", not silently dropped.

- [ ] **Step 2: Write `docs/health-check-TR-4556.md`**

Structure:
- **Header:** ticket link, date, environment note (OS, Python version, Node version, date run), and which lessons were full-run vs reachability-only.
- **Summary table:** `Lesson | What it does | Model(s) | Installs? | Runs? | Model reachable? | Severity`.
- **Per-lesson detail:** one subsection per lesson, lifting its `findings\LesN.md` — including the **exact fix needed** for anything `warning`/`broken`.
- **Slide/lesson mapping note:** from `_mapping.md`, including any lesson with no matching studio deck.
- **No secrets:** no credentials, tokens, or raw slide text.

- [ ] **Step 3: Self-check the report against the acceptance criteria**

Confirm each ticket AC is answered: (1) code executes — table's Runs? column complete for all 11; (2) models reachable — Model reachable? column complete; (3) every break has a documented fix — scan for any `broken`/`warning` row lacking a "Fix needed". Fix gaps inline.

- [ ] **Step 4: Commit**

If the Task 1 `git reset` was done (clean tree), commit report + `.gitignore`:
```powershell
git add docs/health-check-TR-4556.md .gitignore
git commit -m "Add TR-4556 Maistro lessons health-check report`n`nCo-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```
If the reset was declined (stray deletions still staged), path-scope instead so they aren't swept in:
```powershell
git add -- docs/health-check-TR-4556.md .gitignore
git commit -m "<message>" -- docs/health-check-TR-4556.md .gitignore
```
Expected: one commit adding the report (+ `.gitignore`), nothing else.

- [ ] **Step 5:** Report the summary back to the user (counts of ok/warning/broken, headline breakages) and hand off to `superpowers:finishing-a-development-branch` for PR/merge choice.

---

## Self-Review (author's check against the spec)

- **Spec §4 (studio extraction):** Task 2 — login, GraphQL pull, extract, mapping, fallback. ✓
- **Spec §5 (per-lesson pragmatic bar, per-unique venv, install-failure-as-finding, reachability method, known traps):** Tasks 3–11 + the shared protocol + `check_model.py`; traps called out in Tasks 8/9/10/11. ✓
- **Spec §6 (report format):** Task 12. ✓
- **Spec §7 (housekeeping: git reset w/ confirmation, `.env` ignored, dumps out of git):** Task 1 + Global Constraints + Task 2 Step 6. ✓
- **Spec §3 (non-goal: don't edit notebooks/requirements):** Global Constraints states it; no task edits lesson files. ✓
- **Spec §8 risks (login type, heavy installs, mapping gaps):** Task 2 fallback, "install failure is a finding", Task 2 Step 5 mapping-gap flag. ✓
- **Placeholder scan:** model IDs are intentionally read from extracts at runtime (unknown until Phase 1) — not placeholders, but data dependencies stated explicitly. `py -3.X` is a real branch on the Task 1 probe. No TODO/TBD. ✓
- **Consistency:** finding-file format, `check_model.py` signature, and path variables are defined once and referenced identically everywhere. ✓
