# Validation note: `cambrian_p_vsibench_failures.ipynb`

**Status: ready for its GPU pilot; not yet run on a GPU.**
- Everything except model inference was executed.
- The inference worker is the VSI-590K worker with two changes: it takes the official VSI-Bench prompt, and it records a SHA-1 for every input frame. It has not run here (no GPU).
- Run section 6 in `pilot` mode before relying on anything else.

The source notebook, `cambrian_p_vsi590k_failures.ipynb`, is unchanged. This is a separate notebook with its own output and dataset-cache folders.

**Drive output:** `MyDrive/Cambrian-P/vsibench_runs/vsibench-video10pct-seed42-cambrianp7b-v1/`
**Zip** (cell 9b): next to that folder, as `<RUN_ID>_verified_failures.zip`.

## Pinned identities (re-resolved and saved by cells 1c and 2b)

| | Value |
|---|---|
| Dataset | `nyu-visionx/VSI-Bench`, config `full`, split `test` @ `3a6a1cf9ac54c403509b15260897cc6ddc41b408` |
| Model | `nyu-visionx/Cambrian-P-7B` @ `d219bdf9c83ec63424f51f4b26be424c94211b3b` (128 frames) |
| Code, formatter and scorer | `cambrian-mllm/cambrian-p` @ `b3c15527a9f9c4b70020b874eab66f67b2b03901`. `lmms-eval/lmms_eval/tasks/vsibench/utils.py` has SHA-256 `1cdfd6a8…693bc97`. |
| Generation | From the pinned `vsibench.yaml`: `max_new_tokens 16`, `temperature 0`, `top_p 1.0`, `num_beams 1`, no sampling. Cell 2b checks this against the file. |

## The real data at that revision (inspected, not assumed)

- **The `full` config:** it is `test*.parquet`, which is `test_debiased.parquet` plus `test_pruned.parquet`, so **5,130 rows**, with a `pruned` column. Those rows are identical to `test.jsonl`; the notebook checks this. The `debiased` config would give only 2,362 rows.
- **Fields:** `id` (unique integers 0–5155, not contiguous), `dataset`, `scene_name`, `question_type`, `question`, `ground_truth`, `options`. All rows are valid.
- **Videos:** V = **288** unique `(dataset, scene_name)` pairs: ARKitScenes 150, ScanNet 88, ScanNet++ 50.
- **The sample:** ceil(10% × 288) = **29 videos**: 21 ARKitScenes, 5 ScanNet, 3 ScanNet++. They carry **453 questions**, 8.8% of all questions.
- **Media:**
  - `arkitscenes.zip`, `scannet.zip` and `scannetpp.zip` total 5.7 GB.
  - All 288 videos are at `<dataset>/<scene_name>.mp4`. `scannet.zip` also holds 224 videos the benchmark doesn't use.
  - Zip files allow random access, so only the selected videos are transferred, by HTTP range requests: **0.40 GB** in all.
  - Tested against the real archive: 3 selected ScanNet++ videos (64 MB) were read out of the 1.03 GB zip, with their CRCs checked.
  - If a server ignores range requests, the notebook downloads the whole archive instead, printing its size first.
- **Prompt:** the official `vsibench_doc_to_text`, with the YAML's default settings. Every one of the 5,130 prompts is unchanged when the reference is replaced by a sentinel value, so no prompt can contain the answer. Every reference scores 1.0 against itself under the official scorer.

## What changed from the VSI-590K notebook

| Area | Now |
|---|---|
| Dataset, settings, run ID, paths | VSI-Bench `full`/`test` with its own output and dataset-cache folders. The environment and model caches are keyed by code commit and model revision, so they are shared with the VSI-590K notebook and nothing is reinstalled. |
| Schema adapter | The real VSI-Bench fields. Native IDs are kept (`qa_id = vsibench-<id>`), and video identity is `(dataset, scene_name)`. |
| Sampling | Seeded uniform sampling of unique videos: sorted keys, a local RNG, then every question of each selected video. The selection hash is bound to the run in `run_config.json`. |
| Media | Zip member reads by range request, with size and CRC checks and safe paths. A missing or corrupt video stays in coverage and is never replaced. |
| Scoring | The pinned official `vsibench_process_results`, `fuzzy_matching`, MRA and aggregator, executed from the pinned file itself, unedited. Automatic status sits beside the official score and never changes it. |
| Run identity | Adds dataset config and split, sampling unit, scorer hash and generation settings. A changed identity is refused under an existing RUN_ID. |
| Frame provenance | The worker records indices, timestamps, reader and a SHA-1 for each input frame. The export re-decodes those indices and compares hashes, and only a verified match allows `sufficient_in_input`. |
| Review and verified count | One shared validator for UI, counts, CSV, report, manifest and completion gate. Reviews are tied to one question and one run, with a blind judgment first and append-only history. The 5 evidence statuses come with validated citations, plus reserve promotion. `len(CASES)` and default-to-wrong are gone. |
| Deliverable | `verified_failure_manifest.json` decides what goes into `failure_videos/`, `cases/` and the zip. Candidates and overridden cases stay in `candidate_cases/` as history. |

## Executed here

The clean-namespace run executed every code cell in order in one namespace.
- **Replaced:** only Colab/Drive, the Hugging Face client and the GPU worker.
- **Served as real data:** the actual pinned annotation files (README, both parquets, `test.jsonl`), from a local HTTP server with real range support.
- **Archives:** all 288 member paths, each holding a tiny real MP4. One selected video was deliberately corrupted.
- **Stand-in worker:** it ran the real worker's decode, provenance and per-frame hashing, and gave deterministic answers covering every automatic status.

| Check (brief §9) | How | Result |
|---|---|---|
| 1. 288 videos give 29; repeated rows never increase V; all questions of selected videos included | Unit tests (synthetic and real), notebook asserts | Pass. Real data: 29 videos, 453 questions. |
| 2. Determinism; changed identity can't reuse results | Unit tests: shuffled input; changed revision, config, seed, scorer, frames or generation; selection binding | Pass |
| 3. Real annotations map to native IDs, types, options and media paths | Selftest on real `test.jsonl` and parquets, and the real zip listings | Pass: 5,130 rows valid; 288/288 media paths exist |
| 4. Official-scorer parity | Selftest: correct and wrong MC, format failures, numeric perfect, partial and zero, invalid inputs. Each score equals the official function's output, and the scorer file is byte-identical to the pinned one. | Pass |
| 5. Interrupted inference resumes | Unit test (stop, torn journal line, unmarked shard) and the notebook's own 6c check | Pass: no duplicates, no lost answers |
| 6–13. Review and verification | `vsibench_review_selftest.py` (52 checks, real MP4s including a real 7 MB VSI-Bench video). Includes test 7: 8 wrong + 5 correct + 4 uncertain + 3 unreviewed gives **8**, minimum not met. | Pass |
| UI in the notebook | The harness drove cell 8c by setting widgets and clicking buttons, using the full-mode data | Pass, 20/20. Blank and default saves were rejected. Extra-frame contact sheets appeared as images. Ambiguous evidence labelled "visual reasoning" was rejected. An override decremented the count and promoted a reserve. History was kept. |
| Deliverable | Cells 9a and 9b, then the report opened in Chromium | Pass. Manifest, counts and `coverage.json` agree (2). Every relative link resolves. Both videos play and 10/10 images load. The zip holds only the verified cases. |
| 14. Clean-kernel run, failing tests stop | Whole notebook, pilot then full; pyflakes on all cells in order; V1 raises if any selftest exits nonzero | Pass |
| Drive cache | A second session with local disk wiped and Drive kept | Pass: environment restored, no model download, no Hugging Face listing or file transfer, selection verified rather than redrawn |
| Corrupt or missing media | One fixture video with a bad CRC | Counted as 1 failed video with 38 execution errors. Never replaced, never counted as wrong. |

### Bugs found by these tests and fixed
- **Torn journal line:** a write interrupted mid-line made the journal unreadable after the next append. Appends now start on a fresh line, and the reader skips torn lines. The VSI-590K notebook has the same latent bug.
- **Range-refusal fallback:** Python's `zipfile` hid the "range ignored" error, so the full-download fallback could never trigger. A dedicated probe now raises `RangeNotSupported`.
- **Pilot picker:** it crashed when a selected video is missing from its archive.
- **Completion gate:** it read the old `full_run_complete` field, so it always reported inference as incomplete.
- **Review self-test in Colab (found in the first full Colab run):** it ran in the model venv, which has no IPython, so V1 failed after every other check had passed. It now runs in the notebook kernel, and only video decoding goes to the venv (`--frames_python`). This was reproduced and fixed with two fresh environments that split the packages the same way.

## Not executed here
- **Model inference on a GPU** (`vsibench_worker.py`). The pilot (6b) is its first real test: model load, camera tokens (the worker refuses to run without them), non-empty short answers under `max_new_tokens = 16`, and time per question.
- **Colab and Drive themselves:** Drive mount, real Drive speed and free space, and the Hugging Face download client in Colab. Small files and zip ranges use plain HTTPS, which worked here against the real URLs.
- **The full 29-video run and real reviews.** No real failure cases exist yet. Cases from the stand-in worker are not results.

## Choices to know about
- **Run identity and the selection hash:** the selection hash isn't in the config hash, because the selection is computed after the identity is fixed. It is bound to the run in `run_config.json`, and the selection is fully determined by values that are in the identity: revision, config, split, fraction, seed and sampling unit.
- **Review queue:** only automatic failure candidates (`wrong_candidate`) and `format_error` cases enter it. Partial-credit (`imperfect`) answers are inspection candidates in `failure_candidates.csv` but are never verified failures by score alone. A confirmed format failure must be labelled `format_failure`.
- **Subset scores:** official subset scores in `coverage.json` are labelled as "selected subset only", using lmms-eval's macro average. They are not a benchmark score, and the exported cases say nothing about a failure rate.
- **Frame evidence is a manual stage.** Exported frames alone never verify anything.
