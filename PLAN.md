# Assignment 3 plan

Working notes, not part of the submission (delete before the PR, or keep; it's harmless).

## Where we are

The repo is the empty starter: `index.py`, `server.py`, `agent.py` and `evaluate.py` all raise `NotImplementedError`; `AGENTS.md`, `eval/questions.csv`, `eval/human_labels.csv` and `.claude/agents/` are templates; there is no `.env` yet.

**Hard requirement:** `main.py index`, `main.py ask`, `main.py evaluate` run end to end and `check.py` passes.

## The constraint that shapes everything: ~1,000 embeddings a day

- A 50k-word corpus in ~150-word chunks ≈ 420 embeddings. Each extra chunking = another full index.
- Every query embedding counts too (30+ questions × each eval run).
- So: **cache every vector to disk as soon as it arrives** (chunk vectors in `index/`, query vectors in a cache file), and pick the two retrieval designs so that only **one** of them needs a second full embedding pass.
  - Design A: fixed ~150-word chunks with overlap, dense search.
  - Design B: structure-based chunks (by article/section), dense search. Costs one more index.
  - Design C (free): B + hybrid (BM25 + dense) or a metadata filter. No new embeddings.

## Test set format (eval/questions.csv)

`expected_location` = `<doc_part> <section number>`, e.g. `leaflet 3` (package leaflet section 3) or `smpc 4.4`. It matches a chunk when the chunk's `source` and `doc_part` match and its `section` starts with that number. Several expected passages (multi questions) are `;`-separated, aligned with `expected_source`; acceptable alternatives for one passage are `|`-separated. 39 questions: 25 single, 7 multi, 7 unanswerable (q36 = Lixiana, answered after the add-document demo).

## Steps

| # | Step | Output | Embeddings |
|---|---|---|---|
| 0 | Setup: `uv sync`, Gemini key in `.env`, `uv run python api_key.py` | working key | 0 |
| 1 | **Use case + corpus** (decision below). Keep one extra document *out* of the corpus for the "add a document" demo | `corpus/` (git-ignored), corpus table in REPORT | 0 |
| 2 | **Test set first**, before tuning: 30+ Qs, ≥5 `multi`, ≥5 `unanswerable` (one answered by the held-out doc) | `eval/questions.csv` | 0 |
| 3 | `index.py`: download → extract → clean → chunk with `source` + `location` → embed with saved vectors → ChromaDB. Functions `chunk_file(path)` and `embed_and_store(chunks)` | `index/` committed (<90 MB) | ~420/design |
| 4 | Retrieval eval: hit rate @1/3/5 for each design, words added to prompt; pick the winner | table in REPORT | ~35 (queries, cached) |
| 5 | `server.py`: `search(query)` with citations, `add_document(path)` importing index.py; skip-or-replace rule for duplicates (hash of file) | MCP tools | 1/query |
| 6 | Subagent `.claude/agents/corpus-researcher.md` (search tool only); run it once in Claude Code, paste transcript | transcript in REPORT | few |
| 7 | `AGENTS.md` rules + `agent.py` (LangChain `create_agent`, Gemini, MCP tools, approval before `add_document`, call limit, extra guardrail, e.g. reject answers whose citations aren't in the retrieved chunks) | `main.py ask` works | few |
| 8 | `evaluate.py`: agent on every Q; code checks (citations valid, refusals); LLM judge with rubric; you grade 20+ by hand **before** seeing the judge; agreement %; before/after adding the held-out doc | `eval/` results | ~35 |
| 9 | Failure classification, decision, cost (tokens + seconds per question) | REPORT | 0 |
| 10 | `check.py` + `check.py --evaluate`, enable GitHub Action, push, open PR, Blackboard link + zip | submission | 0 |

Suggested pacing: day 1 = steps 0–3 (design A), day 2 = design B + steps 4–7, day 3 = steps 8–10.

## Use case (chosen): blood thinners for home caregivers

**Who asks:** a family caregiver looking after an older relative who takes a blood thinner (a DOAC: apixaban, rivaroxaban or dabigatran).
**What they ask:** what the official product information says, e.g. "a dose was missed, what now?", "which side effects mean calling a doctor?", "can she take ibuprofen with it?", "can the tablet be crushed?".
**What they do with it:** decide whether to wait, give the dose, or call the pharmacist/doctor.
**Cost of a wrong answer vs "I don't know":** a wrong answer can cause a serious bleed or a stroke (double dose, missed dose, risky combination). "I don't know, ask your pharmacist" costs a phone call. So the assistant must cite every claim, refuse when the corpus doesn't cover it, and never give personal dosing advice beyond what the leaflet says.

## Corpus

EMA product information (EPAR), one PDF per medicine: Annex I (SmPC, for professionals) + Annex III (package leaflet, for patients). Licence: EMA allows reproduction with the source acknowledged. No personal data.

| Medicine | Brand | URL | Raw size |
|---|---|---|---|
| apixaban | Eliquis | https://www.ema.europa.eu/en/documents/product-information/eliquis-epar-product-information_en.pdf | 185 pages, ~68,500 words |
| rivaroxaban | Xarelto | https://www.ema.europa.eu/en/documents/product-information/xarelto-epar-product-information_en.pdf | 312 pages, ~105,000 words |
| dabigatran | Pradaxa | https://www.ema.europa.eu/en/documents/product-information/pradaxa-epar-product-information_en.pdf | 190 pages, ~63,500 words |
| edoxaban | Lixiana | https://www.ema.europa.eu/en/documents/product-information/lixiana-epar-product-information_en.pdf | **held out**: added live for the "add a document" demo |

**Problem: ~237k raw words ≈ 1,600 chunks**, beyond the ~1,000 embeddings/day limit. Most of it is near-duplicate text: each PDF repeats the SmPC and the leaflet for every strength (2.5 mg, 5 mg, 15 mg, 20 mg, oral suspension…) plus labelling (Annex IIIA, box text). Cleaning plan:
- drop Annex II and the labelling (Annex IIIA): boxes and blister text, no answers there;
- de-duplicate SmPCs/leaflets that are the same apart from the strength: keep the main tablet strength's text, record the others in metadata;
- target **~60–80k words → ~450 chunks**, which fits in one day per design and still clears 50k.

This is the "extraction problem I fixed" for the Ingestion section of REPORT.md.

**Done in index.py (measured):** 236,312 raw words → 72,308 kept, by (a) skipping Annex II, the box labelling and the leaflet's country contact list, (b) keeping only caregiver-relevant SmPC sections (1–3, 4.x, 6.3–6.6; dropping 5.x clinical-trial/pharmacology data, ~44k words), (c) dropping paragraphs repeated word for word across strengths (~50% of the rest). Result: **461 chunks (section design), 604 (fixed design)**. Day 1: `uv run python index.py --design section`; day 2: `uv run python index.py` (adds fixed).

Metadata per chunk: `source` (e.g. `eliquis`), `doc_part` (`smpc`/`leaflet`), `section` (e.g. `4.5 Interaction with other medicinal products`, or leaflet section `3. How to take Eliquis`), `page`.

Good unanswerable questions (close to the topic): anything about edoxaban/Lixiana (until it is added), warfarin, specific food brands, cost or reimbursement in Spain, personal dose changes.

Note: EMA rate-limits downloads (HTTP 429). `index.py` should download with a pause between files and skip files already in `corpus/`.
