# Validation note: `cambrian_p_vsibench_failures_fixed.ipynb`

This is a focused fix of `cambrian_p_vsibench_failures.ipynb`. The original is unchanged; a check confirms this against git.

Unchanged:
- dataset, sample, scoring rules and inference settings;
- the verified-failure validator, review records and candidate eligibility;
- `RUN_ID`, output and cache folders.

**Existing Drive results resume as they are. Nothing is deleted or reset.**

## What changed

- **Full is the default.**
  - 1a now reads `RUN_MODE = "full"`. Full means every question of the selected ~10% of videos (29 videos, 453 questions), not the whole benchmark.
  - `pilot` is still available when set explicitly, and the resume check (6c) is still pilot-only.
  - The run mode is not part of the run identity or config hash, so pilot answers are reused by the full run and existing prediction shards still match.
  - 6b now prints the runtime mode and the selected-subset size before inference starts. It also prints the last recorded inference mode separately: the setting is an intention, while the counts are the evidence.
- **One read-only diagnostics helper** (`vsibench_diagnostics.py`, embedded like the other helpers), used by 8a, 8b and 8C.
  - **Shown:** run mode (or "unknown"); last recorded mode; run folder and config hash; selected videos and questions.
  - **Prediction coverage:** from a fresh merge of the saved shards, counting only the frozen selected question IDs: answered, pending, execution errors, unavailable media. Ignored shards are reported too.
  - **Scoring and queue freshness:** each is current, stale, missing or unknown.
  - **Counts:** every `automatic_status`; inspection candidates and queue-eligible questions, kept distinct; eligible videos; queue entries, active and reserve; exports ready.
  - **Freshness uses content fingerprints.**
    - 7a writes `scoring_state.json`, fingerprinting each selected question's answer text or error plus the config and scorer.
    - 8a writes `queue_state.json`, fingerprinting the scores it queued.
    - So a changed answer is detected even when the counts don't change.
    - Runs scored before this fix have no fingerprints. Their freshness shows as "unknown" with "rerun 7a, then 8a"; nothing is discarded.
  - **Read-only:** it never starts inference, re-scores, edits the queue or deletes anything. A test checks this by hashing every file before and after.
- **8C no longer prints the unconditional "run sections 6-7 in full mode" message**, which appeared even in full mode.
  - When the queue is empty or a prerequisite is missing, 8C shows the diagnostic panel instead. It lists every supported finding with its next step: pilot scope, incomplete full inference, errors and media problems, missing or stale scores, missing or stale queue, all correct (with the denominator and any pending questions), partial credit only (pointing to `failure_candidates.csv`), reserve entries but no active ones, or exports missing (run 8b).
  - Missing globals are named, with the cells that define them, instead of raising `NameError`.
  - The review UI works as before when there are candidates; the panel shows above it only when something needs attention.

The old message was removed from the executable cells and the embedded `review_cells.py` template, so rerunning setup does not restore it.

- **Background runs (Colab Pro+).**
  - Everything the cells print is also appended to `logs/notebook-<time>.log` on Drive, with a start, finish or error line for each cell. This was tested in a real IPython shell: rerunning 1a does not log twice.
  - The last cell no longer unmounts Drive during Run all. Before, unmounting there would have sent later 8c review saves to a local folder that is lost when the runtime ends.

- **Worker bug (found in your full Colab run, where all 453 questions ended in `KeyError`).**
  - The inference worker copied a `record_id` field that only VSI-590K questions have. The model answered each question, then saving the answer failed, so all 453 were recorded as execution errors. The worker now saves the benchmark's `native_id`.
  - Its error handling now catches and retries only the model call. A bug in saving stops the worker immediately with its traceback, instead of being recorded as model failures and retried three times.
  - `vsibench_worker_selftest.py` runs the **real worker** on a real selected video, with only the model replaced by a stand-in: 11 checks covering decoding and frame hashes, the exact prompt and generation settings, saved rows, retry, out-of-memory, deliberate stop, and a planted bookkeeping bug. On the previously shipped worker it reproduces `KeyError: 'record_id'`; the fixed worker passes 11/11.
  - 6a runs it as a pre-flight before any GPU work, and V1 runs it again.
  - Your existing error rows need no cleanup: 6b re-answers every question without a saved answer.

## Tests executed (no GPU, no dataset download)

| Brief case | Where | Result |
|---|---|---|
| 1. Fresh settings give `RUN_MODE == "full"` | Notebook check (cell 1a as shipped) and the harness, which ran 1a unedited | Pass |
| 2–13 | `vsibench_diagnostics_selftest.py`: fixtures built with the real shard writer, merge, scoring state and queue | 23/23 pass |
| 14. Embedded copies agree | Notebook check: all 10 `%%writefile` copies equal their sources; executable 8a–9b cells equal the embedded template; the old message is gone everywhere | Pass |
| Syntax | All 28 code cells and every embedded script parse; pyflakes is clean on all cells in order and on every helper | Pass |
| Existing tests | `vsibench_selftest.py` 41/41, `vsibench_review_selftest.py` 52/52, review UI driven by the harness 20/20 | Pass |

The end-to-end harness also ran the whole notebook, with Colab, Hugging Face and the GPU worker stubbed:
- **Pilot:** set explicitly, top to bottom.
- **Full default:** 6b announced "Runtime mode: full -> 453 question(s) on 29 video(s)", with the last recorded mode shown separately.
- **8a and 8C right after a full 6b without 7a:** the panel says "Scoring: stale … Run 7a, then 8a -> 8b -> 8c. Full mode is already selected", and never "switch to full".
- **8C in a fresh runtime with only settings loaded:** it named the missing cells, with no traceback.
- **Second session:** nothing reinstalled or re-transferred.

## Not performed

- GPU inference and live Colab or Drive execution.
- I have not diagnosed your earlier Colab run. The confirmed bug is the unconditional 8C message; running the fixed notebook's 8a or 8C on your existing run folder will show its actual state.
