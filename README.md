# Evaluating LLMs for Ontology-Aware Information Extraction in Low-Resource Medieval Documents

Code, data partitions, prompts, model predictions, and evaluation results accompanying the paper
*Evaluating Large Language Models for Ontology-Aware Information Extraction in Low-Resource Medieval Documents*.

The study evaluates four LLM–API configurations on three independently evaluated extraction tasks
(document-level entity inventories, structured facts, and events) over 128 medieval notarial charters
from Asturias (Spain), using reference annotations derived from the Medieval Charters Knowledge Graph (MCKG).

All tables and figures reported in the paper can be regenerated from the recorded predictions in this
repository **without running any model inference**.

---

## Repository structure

```
.
├── README.md
├── requirements.txt
├── .env.example                 # template for API keys
├── notebooks/
│   ├── 01_benchmark_construction.ipynb   # RDF/Turtle (Attested Dataset) -> task-specific JSON references
│   ├── 02_splitter.ipynb         # deterministic demo / dev / test partition
│   └── 03_llm_evaluation.ipynb           # prompting, inference, parsing, matching, scoring, tables
├── data/
│   ├── corpus/                  # D1.txt … D128.txt — charter transcriptions (model input)
│   ├── ttl_attested/            # D1.ttl … D128.ttl — MCKG Attested Dataset annotations (reference source)
│   ├── ner/
│   │   └── ner.json             # NER entity spans (PERS/LOC) used for inventory completion
│   └── references/
│       ├── D1.json … D128.json  # task references derived from ttl_attested/ (+ ner.json)
│       └── all_documents_ie_eval.jsonl   # the same 128 references in a single file
├── splits/
│   ├── split.json               # test (109), dev_tune (10), dev_pool (9)
│   ├── test.txt, dev_tune.txt, dev_pool.txt
│   ├── split_summary.csv
│   └── documents_with_event_types.csv
├── results/
│   ├── paper_tables/            # aggregated metrics per task / model / prompting / run (.csv)
│   └── paper_calculations.xlsx  # run-level means and standard deviations, composition summaries
└── outputs/
    └── llm_eval_outputs.zip     # raw prompts, responses, parsed predictions, per-document metrics
```
---

## Data

- **Source.** The 128 charters and their annotations come from the Medieval Charters Knowledge Graph (MCKG).
  We use the **Attested Dataset**, which preserves documentary (non-normalized) forms.
- **References.** `notebooks/01_benchmark_construction.ipynb` derives one reference file per document
  (`data/references/D*.json`) deterministically from the Turtle files. Each file contains:
  - `entities`: persons and places with their attested names (`canonical_name`, `aliases`);
  - `facts`: subject–predicate–object assertions (`fact_type` = `attribute` or `relation`);
  - `events`: event `type`, optional `subtype`, `attributes` (e.g. `time`, `price`), and participant `arguments` (role–entity pairs);
  - `warnings`: notes produced during conversion.
- **NER-based completion.** Person and place entities found by NER (`data/ner/ner.json`) whose attested
  names are not already among the RDF-derived entities are added to `entities` with `"provenance": "ner"`
  (853 entities across the corpus). They are not normalized or reconciled across documents.
- **Entity inventories.** Two reference inventories are evaluated against the same predictions:
  - *restricted* (`annotated` in file names): excludes entities with `"provenance": "ner"`;
  - *unrestricted* (`all` in file names): includes them.
- **Output schemas.** The JSON schemas sent to the models are defined in
  `notebooks/03_llm_evaluation.ipynb` (section *JSON schemas per task*).
- **License.** Code is released under the MIT License (see LICENSE). Corpus and annotations are derived from MCKG and remain subject to its license.

## Data partition

| Partition | Documents | Use |
|---|---|---|
| `dev_pool` | 9 | Pool for in-context demonstrations |
| `dev_tune` | 10 | Prompt and output-convention development |
| `test` | 109 | Final evaluation |

The partition is deterministic and non-random (`notebooks/02_splitter.ipynb`). Documents with
rare event structures are assigned to the test set; development and demonstration documents are drawn
from Purchase/Sale documents stratified by a simple complexity score.

Fixed demonstrations: **one-shot** uses `D30`; **few-shot** uses `D30` and `D44`.
Both are Purchase/Sale documents from `dev_pool`.

## Models and inference settings

| Paper name | Folder name | Model identifier | Interface | Structured output |
|---|---|---|---|---|
| GPT-5.4 Nano | `gpt54nano` | `gpt-5.4-nano` | OpenAI Batch API | JSON-object response mode |
| LLaMA-4 Scout 17B | `llama17` | `meta-llama/llama-4-scout-17b-16e-instruct` | Groq API | Prompt-based JSON instructions |
| Gemini 3.1 Flash Lite | `gemini_native` | `gemini-3.1-flash-lite-preview` | Google GenAI API | JSON response mode + JSON Schema |
| Mistral Large 2411 | `mistrallarge` | `mistral-large-2411` | Mistral API | JSON-object response mode |

Common settings: temperature `0.0`; maximum output tokens `4096`; Gemini `thinking_level = "minimal"`.
Transient API errors (rate limits, 5xx, timeouts) are retried up to 6 times with exponential backoff.

## Experimental runs

| Prompting (paper) | Folder name | Runs |
|---|---|---|
| Zero-shot | `baseline` | 1 (`test`) |
| One-shot | `oneshot` | 1 (`test`); 3 for GPT-5.4 Nano events (`test`, `test_2`, `test_3`) |
| Few-shot | `fewshot` | 3 (`test`, `test_2`, `test_3`) |

The paper reports the **first run** (`test`) unless stated otherwise; `test_2` and `test_3` are used for
the run-to-run stability analysis (Section 5.4).

## Evaluation

- **Parsing.** Responses that cannot be parsed as JSON count as failed extractions (all reference items
  for that document become false negatives). Parseable but schema-non-conforming responses are still scored.
- **Normalization** (comparison only): case, whitespace, punctuation variants, and diacritics.
- **Entities:** document-level inventory matching on entity type and attested name.
- **Facts:** exact match on (subject type, subject, predicate, fact type, object type, object),
  with multiplicity handled by counts.
- **Events:**
  - `events_header` — Header-F1: exact type and subtype match, one-to-one greedy pairing;
  - `event_attrs` — Attribute-F1, `event_rels` — Role-F1, `event_all` — Frame-F1:
    computed only over header-matched event pairs (conditional measures);
  - `events_exact` — complete-event exact match (end-to-end).
- Micro-F1 is the primary measure; macro-F1 is reported alongside it.

## Where each paper result comes from

All paths are relative to `results/paper_tables/`. File names follow
`<task>__<model>__<prompting>__[<inventory>__]<run>__<table>.csv`.

| Paper result | File(s) | Column / row |
|---|---|---|
| Table 4 — Entities (restricted / unrestricted) | `entityonly/<model>/fewshot/*__annotated__test__main.csv`, `*__all__test__main.csv` | `entities`, `f1` |
| Table 4 — Facts | `factsonly/<model>/fewshot/*__test__main.csv` | `facts`, `f1` |
| Table 4 — Events (header / frame) | `eventsonly/<model>/fewshot/*__test__main.csv` | `events_header` / `event_all`, `f1` |
| Table 5 — Fact attributes / relations | `factsonly/<model>/fewshot/*__test__main.csv` | `fact_attrs` / `fact_rels`, `f1` |
| Table 5 — Event attributes / roles | `eventsonly/<model>/<prompting>/*__test__main.csv` | `event_attrs` / `event_rels`, `f1` |
| Table 6 — all prompting conditions | same `main.csv` files under `baseline/`, `oneshot/`, `fewshot/` | as above |
| Table 7 — predicate-level F1 and support | `factsonly/<model>/fewshot/*__test__all_facts.csv` | `f1`; N = `tp + fn` |
| Person / Place F1 (Sec. 5.1) | `entityonly/<model>/fewshot/*__test__breakdown.csv` | `Person`, `Place` |
| Macro-F1 (Sec. 5.2–5.3) | `*__main.csv` | `macro_f1` |
| Documentary compositions (Sec. 5.3) | `*/<model>/fewshot/*__test__by_type.csv` | one row per composition |
| Run-to-run stability (Sec. 5.4) | `*__test__`, `*__test_2__`, `*__test_3__` files | as above |

`results/paper_calculations.xlsx` collects the run-level values, means, and sample standard deviations.

## Raw model outputs (`llm_eval_outputs.zip`)

```
llm_eval_outputs/<task>/<model>/<prompting>/<task>__<model>__<prompting>__[<inventory>__]<run>/
    D*_prompt.txt          # exact prompt sent to the model
    D*.raw.json            # raw response, parse status, schema validation, provider metadata
    D*.parsed.json         # parsed prediction used for scoring
    D*.metrics.json        # per-document TP/FP/FN and scores
    D*.fact_errors.json    # (facts only) error analysis
    results_*.csv          # per-batch summaries
```

### Failed requests

A small number of documents have no usable prediction. They are scored as failed extractions, as
described above. `D*.raw.json` records the cause in `parse_error`: either an API error (`backend_error: …`)
or a response that could not be parsed, including truncation at the output-token limit
(`finish_reason = "length"`).

## Reproducing the results

All notebooks use `BASE_DIR = Path("..")`, so they must be run **from the `notebooks/` directory**
(the default when a notebook is opened from that folder in Jupyter or VS Code). All paths are
resolved relative to the repository root.

### Recompute metrics and tables (no API keys needed)

1. Install dependencies: `pip install -r requirements.txt`.
2. Unzip `outputs/llm_eval_outputs.zip` in the repository root. This creates `llm_eval_outputs/`,
   which is the output directory expected by the notebook (`OUT_DIR`).
3. Open `notebooks/03_llm_evaluation.ipynb` and run all cells from the beginning up to and including
   the *Summary* section. These cells only define configuration, helpers, and evaluation functions;
   they make no API calls.
4. Skip *Dev_tune execution* and *Test execution* (these call the model APIs) and go to
   *Summary of test execution*. Set:
   - `task_name`: `entityonly`, `factsonly`, or `eventsonly`;
   - `backend_name`: `gpt54nano`, `llama17`, `gemini_native`, or `mistrallarge`;
   - `prompt_variant`: `baseline`, `oneshot`, or `fewshot`;
   - `entity_extraction_mode` (entities only): `annotated` (restricted) or `all` (unrestricted);
   - `rerun = False` for the first run, or `rerun = True` with `rerun_number = 2` or `3`
     for the repeated runs.
5. Run the next cell. It recomputes the metrics from the saved per-document outputs and writes the
   tables to `llm_eval_outputs/paper_tables/`, where they can be compared with `results/paper_tables/`.

### Re-run inference

1. Copy `.env.example` to `.env` and add your own API keys.
2. In `notebooks/03_llm_evaluation.ipynb`, set `BACKEND_NAME`, `TASK_NAME`, and `PROMPT_VARIANT`,
   then run the *Test execution* section for the desired model.

Hosted model versions and provider behavior change over time, so re-running inference may not
reproduce the recorded predictions exactly.
