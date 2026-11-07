# Validation note: `cambrian_p_vsi590k_failures.ipynb`

**Status: ready for its GPU pilot, not yet run on a GPU.** Everything except model inference was executed. The Cambrian-P inference worker was checked statically against the pinned code, but no GPU was available here. Run section 6 in `pilot` mode before relying on the full run.

## Pinned identities (resolved during implementation, re-resolved and persisted by cell 1c)

| | Value |
|---|---|
| Dataset | `nyu-visionx/VSI-590K` (the brief's `vsi-590k` redirects here) @ `346fbd4e41dec974bf24894d0541a49327ee6669` |
| Annotation file | `vsi_590k.jsonl`, SHA-256 `025355ea…b319d231` (matches the repository's LFS hash) |
| Model | `nyu-visionx/Cambrian-P-7B` @ `d219bdf9c83ec63424f51f4b26be424c94211b3b` |
| Code | `cambrian-mllm/cambrian-p` @ `b3c15527a9f9c4b70020b874eab66f67b2b03901` |

## What the real data turned out to be (inspected, not assumed)

- One annotation file with 590,667 records. Its fields are `conversations` (`human`/`gpt` turns), `question_type`, and `video` or `image`; only ProcTHOR records also have an `id`.
- 374,148 video records from six sources, **all single-turn**. All 31,127 multi-turn records are image records.
- The review's population is the video records (the deliverable is videos), so image records fall outside it and are counted separately.
- A 10% sample with seed 42 is 37,415 questions on 4,475 of the 5,963 videos. It needs all six video archives (172 GB of the 236 GB).
- Each video is stored at its annotation path inside `<source>.tar.gz`; this was checked for all six archives. These are gzip archives, so they can't be read from the middle and must be streamed or downloaded.
- Answer formats are:
  - letters, sometimes with the option text (`C. 1`), for multiple choice;
  - plain numbers for measurements;
  - closed-set words for 5,683 ProcTHOR questions.

  Every reference answer scores as correct against itself under the scoring rules.
- The source notebook's camera-head workaround is still needed at the pinned commit:
  - `builder.py:230` still re-converts the config of any model folder whose name contains "cambrian";
  - the checkpoint's `rec_model_path` is JSON null, and `vggt_reconstructor.py:98` only skips loading starting weights for the string `'None'`.

  The workaround is kept, and the worker refuses to run unless the camera tokens are loaded.

## Executed here

| Check (brief's list) | How | Result |
|---|---|---|
| 1. Deterministic 10% selection, full-population denominator | Unit tests; real file | Pass: 37,415 = ceil(10% × 374,148) |
| 2. Stable IDs, video grouping, duplicate file names across folders | Unit tests; all 374,148 real rows | Pass: IDs unique |
| 3. Reference excluded from prompts, multi-turn handling | Unit tests; every real row | Pass: prompts come from the human turn only; multi-turn records are excluded with a reason |
| 4. Media lookup and safe extraction | Unit tests; real archive listings; streaming from real `.tar.gz` files | Pass: `..` and absolute paths refused; byte budget respected |
| 5. Resume | Unit tests; full-notebook run with a deliberate mid-run stop | Pass: kept answers unchanged, nothing answered twice, truncated or corrupt shards ignored, changed settings rejected |
| 6. Scoring | Unit tests (multiple choice, number-tolerance boundaries, angles, formatting errors, free text) | Pass |
| 7. Timestamps vs indices, image paths, exported videos | Real MP4 decoded with decord; report opened in Chromium | Pass: 20/20 videos play, 80/80 images load, links relative |
| 8. Distinct-video counting, honest shortfall | Unit tests; pilot pass (0 cases, "Minimum NOT met") | Pass |
| 9. Clean-namespace execution | Whole notebook run top to bottom in one fresh namespace, then switched to full mode and rerun | Pass: no undefined names (pyflakes on all cells in order) |
| Review interface | Driven with real ipywidgets | Pass: answers hidden until the blind judgment is saved; saves, reload and progress counter work |

The clean-namespace run replaced only Colab/Drive, the Hugging Face client and the GPU worker:
- **Hugging Face client:** it served the real annotation slice (1,477 records) and archives of real tiny MP4s, with 3 videos deliberately missing.
- **GPU worker:** the stand-in decoded the real video and gave deterministic answers.

Everything else ran as written: discovery, schema checks, freezing the selection, the media plan, streaming extraction, the batch loop, merging, scoring, case export with real frame extraction, the review interface and the report.

## Not executed here

- **Model inference (`vsi590k_worker.py`).** It reuses the source notebook's loader, patch, preprocessing and `ask()`. Every model-code name it uses was checked against the pinned commit, and it compiles and lints clean, but it has not run. The pilot (cell 6b) is the first real test.
  - The pilot will show whether the model loads, whether the camera tokens load (the worker checks this and stops if they don't), whether answers are non-empty, and how long a question takes.
- **Hugging Face download and streaming in Colab.** The repository listing, card, annotation download and archive streaming were run from this session against the real Hugging Face URLs. The archive downloads inside Colab were not.
- **The full 10% run.** It's estimated at 20–40 GPU-hours, from typical 7B prefill speeds. The pilot measures the real figure.
- **Real failure cases.** None exist until the model has run. Cases in the test run came from the stand-in worker and are not results.

## Choices to know about

- **Number scoring:** VSI-Bench's mean relative accuracy is used as a *diagnostic*, labelled as such. It hasn't been confirmed as VSI-590K's own metric.
  - Full credit means within 5%, which counts as `correct`; more than 50% off counts as `wrong`; anything between is `imperfect` and doesn't count as a failure.
  - Angles use circular error: up to 15° is correct, more than 45° is wrong.
- **What counts as a failure:** only `wrong` verdicts count toward the 10–20 videos. Format errors, unclear answers (`needs_review`), unscorable answers and execution errors never count.
- **The pilot is a setup check, not a representative sample.** It uses 2 videos from the smallest needed archive, and its answers count toward the full run because the settings match.
