# HANDOFF — cordis-dmp-analysis

Repo: https://github.com/MaastrichtU-IDS/cordis-dmp-analysis (org MaastrichtU-IDS, branch `master`, public)
Local: `/Users/remzicelebi/workspace/claude-ws/cordis-dmp-analysis` (venv: `.venv`, Python 3.14, `pip install -e .`)

## Goal

Build the corpus and tooling for a study evaluating **Data Management Plans
(DMPs)** from EU CORDIS (Horizon 2020 + Horizon Europe) with the **TAPS-RM
responsibility matrix** (Transparency, Accountability, Privacy &
Confidentiality, Social Values × Actors, Objects, Processes, Impacts), in the
domains **health, agriculture, climate**. Research questions: RQ1 template
adherence + content similarity; RQ2 TAPS-RM coverage; RQ3 influence of project
factors; RQ4 TAPS quality vs. FAIR/open-science commitment. The README's
"Next steps" section captures the agreed analysis roadmap.

## Current Progress

Pipeline (all stages implemented, CLI `cordis-dmp`, see README):

1. `metadata` — CORDIS deliverables dumps. Done for both programmes
   (h2020: 194k deliverables, horizon: 49k).
2. `filter` — title-matched DMPs → `data/dmp_deliverables.csv`. Done:
   **12,351 DMP rows** (h2020 4,421 + horizon 7,930).
3. `enrich` — `data/corpus.csv` with domain labels (`domains_esv`,
   `domains_cluster`), covariates, `version_rank` + `is_latest`. Done for both
   programmes. Health, latest-only: 2,040 DMPs (horizon 1,411, h2020 629).
4. `download` — **Health corpus downloaded: 2,326/2,330 PDFs ok, 4 failures,
   1.6 GB in `data/pdfs/`** (790 h2020 + 1,542 horizon files on disk).
5. `extract` — **26 PDFs extracted** (`data/text/`): the 6 original samples
   (horizon: OPALS, GREENGAGE, IS2H4C, DefenCB1, iENERGY, RIXISAC) + 20 health
   DMPs, which turned out to be **all H2020** (see bug below). 0 OCR, 0 errors.
6. `template` — HE template (39 canonical questions) lexical matcher.
   `--all` was run over the 26 extracted docs →
   `data/template_{matches,adherence}.csv`.

`--programmes` now filters `download` and `extract` too (was previously
ignored by `load_corpus_selection` — that is why the "20 horizon health" run
picked 20 H2020 docs). `data/` is gitignored — corpus lives only on this machine.

## Findings from the 26-doc template run

- Adherence is bimodal: OPALS 35/39 (0.90), everything else ≤ 18/39, most 0–5.
- **Lexical matched ≠ answered.** In OPALS, R-1 and RES-1 "answers" are only
  leftover question text; DS-1, R-4, ETH-1 contain question fragments; R-5's
  question text ended up inside R-4's answer.
- **Threshold 0.55 is too low.** In DefenCB1 (101150763_1), 5 of 6 matches
  score 0.56–0.61 and are mostly wrong (A-12 → resources text, OUT-1 → project
  title; F-2/F-4/A-10 all point to the same "Metadata:" section).
- **DefenCB1 actually follows the template at part level** — every HE part is
  present as a heading, answered as prose. Question-level lexical matching
  scores it 0.15. → Need part-level adherence in addition to question-level.
- Long docs (LETHE 157 p, RAMCIP 72 p) score higher partly by chance (hundreds
  of sections = more chances to hit a heading).
- H2020 DMPs follow the **H2020 FAIR template**; scoring them against HE
  questions understates adherence. Need a separate H2020 question set.
- Extraction loses tables (DefenCB1 data-summary table → empty one-word
  "headings"). PyMuPDF `find_tables()` could recover them.
- **`contentUpdateDate` is not the DMP date.** DefenCB1: header says
  20.11.2024, PDF `creationDate` 2024-11-25, CORDIS `contentUpdateDate`
  2025-05-19. Several H2020 DMPs share 2026-04-20 (bulk CORDIS refresh).
- The 39-question list in `template.py` is a condensed paraphrase; not yet
  checked against the official `data-management-plan_he_en.docx`.

## What Worked

- **EC document download**: `downloadPublic?documentIds=...&appId=PPGMS`
  returns an interstitial HTML page, NOT the file. Parse
  `window.location='...'` from it and fetch that tokenised URL **with the same
  session cookies** (requests.Session). Implemented in
  `src/cordis_dmp/download.py:download_one`.
- **Domain labels**: HE clusters via topic prefix `HORIZON-HLTH` (health —
  NOT "CL1"), `HORIZON-CL5` (climate), `HORIZON-CL6` (agri) and legal basis
  `HORIZON.2.1/2.5/2.6`, `H2020-EU.3.1/3.5/3.2`; euroSciVoc paths as the
  independent second signal. `policyPriorities.csv` (horizon only) gives the
  EU climate marker.
- **Heading detection**: font-size/bold/numbering heuristic + folding
  text-less heading fragments into the next heading, filtering `....` TOC lines.
- **Template matching**: difflib on the same-length *prefix* of the question
  handles truncated headings.

## What Didn't Work

- Direct `cordis-h2020projectDeliverables.csv` → 404. Correct:
  `https://cordis.europa.eu/data/cordis-{h2020,HORIZON}projectDeliverables-csv.zip`.
- HEAD requests against the EC portal return no Content-Length — must GET.
- Fetching the tokenised URL without session cookies → HTML error page.
- `--programmes` as argparse `nargs="+"` swallowed the subcommand; now a
  comma-separated single value.
- Lexical matcher on freely-restructured DMPs scores ~0/39 — expected; that is
  the motivation for the embedding-based matcher, not a bug in difflib.

## Next Steps

1. Extract 20 **Horizon Europe** health DMPs for the intended comparison:
   `.venv/bin/cordis-dmp --programmes horizon extract --domains health --latest-only --limit 20`
   then `cordis-dmp template --all`.
2. Improve the template matcher: part-level adherence; "contains real answer"
   check (strip question words, require min chars); raise threshold (~0.65)
   and/or one section per question; score per part to avoid the 12-question
   Accessible part dominating.
3. Add H2020 FAIR DMP template question set; select template by programme.
4. Extraction: add `doc_date` (header "Date:/Version") and `pdf_date`
   (PDF `creationDate`); table extraction via `find_tables()`.
5. Verify `HE_TEMPLATE` against the official .docx.
6. Then run full health extraction (~2,326 PDFs), inspect the 4 download
   failures in `data/manifest.jsonl`, and proceed to the analysis stages
   (embedding matcher RQ1, TAPS-RM codebook RQ2, regressions RQ3, FAIR/OpenAIRE RQ4).
7. Later: agriculture + climate downloads, OCR for flagged scans, LICENSE file.

## Conventions

- Commit messages end with the `Co-Authored-By:` attribution line for the
  current model; push to `origin master`.
- Be polite to the EC portal: defaults 4 workers / 0.5 s delay.
- All long jobs are resumable; journals: `manifest.jsonl` (download),
  `extract_log.jsonl` (extraction).
