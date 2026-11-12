# Validation note: `cambrian_p_revsi_audit_colab.ipynb`

**Status: not yet run on a GPU or in Colab.**
- Everything except Cambrian-P inference was executed here: on the real ReVSI release, on the pinned official scorer, and end to end with a stand-in model.
- The first real test of the model is cell D2's probe, which runs one job before the subset and records time and peak VRAM.

This is a new, self-contained notebook: all helpers are embedded, and nothing else is needed at runtime. The VSI-Bench notebooks and their Drive results are untouched.

## Pinned sources (re-resolved once and saved in `pins.json`)

| | Value |
|---|---|
| ReVSI dataset | `3dlg-hcvc/ReVSI` @ `345a811321e2c9ec2b91a2e8df13e3a124584fa8`, config `32_frame`, split `test` |
| Official scorer | `EvolvingLMMs-Lab/lmms-eval` @ `c26b6b2713c3ec155962ff990f956bf78e0efe45`. `lmms_eval/tasks/revsi/utils.py` SHA-256 `fcb0c066…5cad`; `_default_template_yaml` SHA-256 `a081add9…19ff`. Executed from the file, unedited. |
| Model / code | `nyu-visionx/Cambrian-P-7B` @ `d219bdf9…`; `cambrian-mllm/cambrian-p` @ `b3c15527…` |
| VSI-Bench (overlap only) | `nyu-visionx/VSI-Bench` @ `3a6a1cf9…` |

## Facts checked on the real release

- **Questions:** `32_frame` has 6,158 questions on 380 unique (dataset, scene) videos from 5 sources and 13 subtypes. Scene ids are unique across sources.
- **Reference answers:** every one scores 1.0 against itself under the official scorer.
- **Cross-config IDs:** the same native id has the same question text across configs, but **different reference answers in some cases**: 29 of 500 checked between `32_frame` and `all_frame`. So question ids are config-qualified (`revsi:32_frame:<id>`) and are never joined across configs.
- **Official frames:** `sampled_video_frame_idx.json` gives 32 increasing indices per scene, nested 16 ⊂ 32 ⊂ 64 for all 381 scenes.
- **Released 32-frame videos:** each holds exactly 32 frames, so Cambrian-P at `NUM_FRAMES=32` takes every frame once. The worker refuses any other count (protocol violation, no resampling).
- **Match against `all_frame`:** close at the official indices. Mean pixel difference was 0.8–2.0 for 3RScan and ScanNet++, and about 10 (max 30) for ARKitScenes, which is consistent with re-encoding. Wrong indices give about 70. Every sampled scene is checked in C1, and a mismatch blocks that video rather than being accepted.
- **Media:** `video.zip` (4.87 GB) is range-readable: its 1,524 members were listed in 4 requests.
- **Sample:** 38 videos (`ceil(0.10 × 380)`) carrying 724 questions. Their inputs are 46.5 MB, plus 365 MB of `all_frame` video for verification.
- **Overlap:** 222 ReVSI scenes are also in VSI-Bench; 24 of the 38 sampled scenes are; 5 sampled scenes are in the historical 29-video VSI cohort.
- **QA matching:** 19 conservative template matches on the sample (object identity unverified). In some of them ReVSI and VSI-Bench give different reference answers, recorded as disagreement.
- **Frequency baseline:** the scene-disjoint baseline scores an official macro of 28.6 on the sample. It is a local diagnostic.

## Acceptance checks actually run (spec §5)

| # | Check | Result |
|---|---|---|
| 1 | nbformat validation; helpers compile; pyflakes on all cells and helpers; no laptop-only paths | Pass |
| 2 | Top-to-bottom run | **Not run in Colab, no GPU.** Run end to end with stubs for Colab, Hugging Face (a local server with real ReVSI files and a synthetic `video.zip`), installs and the model |
| 3 | Deterministic ~10% of unique videos, chosen without answers, reused unchanged | Pass (`revsi_selftest`, harness) |
| 4 | Frame fixtures: wrong index map, independent re-sampling, wrong frame count, config mismatch, budget-specific visibility | Pass |
| 5 | IDs across configs cannot collide or attach reviews to the wrong question | Pass |
| 6 | Scorer parity: MC, numeric exact/partial/zero, malformed, missing; subset-only aggregation; composite hierarchy; zero-target diagnostic | Pass |
| 7 | Resume skips finished jobs; changed frame, model, prompt or question hashes invalidate cached rows | Pass |
| 8 | Annotation errors, partial evidence, drafts, stale reviews and missing citations never count; two failures on one video count once | Pass |
| 9 | 18 candidates with 0 reviews gives the exact required message and no rerun request; empty queues explain their cause | Pass (exact string; harness shows "20 candidate videos exported; 0 reviewed; …") |
| 10 | Review-only run rebuilds every report from disk with no install and no model; CSV import goes through the same validator | Pass (fresh process and fresh-runtime harness pass) |
| 11 | No leakage: the worker never receives references; baseline folds are scene-disjoint and never fit on held-out answers; the control set is predeclared | Pass |
| 12 | Every exported case has valid paths; zero or too few cases are reported honestly | Pass |

Test counts:
- **Acceptance self-test:** 43/43.
- **Worker pre-flight:** 14/14, on a real ReVSI 32-frame video plus a wrong-count video.
- **End-to-end harness:** 23/23, across full, review form plus CSV import, fresh review-only, and smoke.

## Expected hardware

- **Inference (D2):** a single large GPU (A100-class), batch size one. **A T4 is not expected to fit** Cambrian-P-7B; the probe reports measured VRAM before the subset runs.
- **Everything else:** setup in review-only mode, scoring, controls analysis, review, exports and reports all run on CPU or T4 with no model.

## Output location

`/content/drive/MyDrive/Cambrian-P/revsi_runs/revsi32-video10pct-seed42-cambrianp7b-v1/`. Smoke runs use the same path with `-smoke` appended. Every file listed in spec §4 is written there, and printed output goes to `logs/notebook-*.log`.

## Remaining blockers, limits and untested steps

- **Not tested:** GPU inference with the real model (camera-token loading at 32 frames, answers, VRAM, time); the PyAV install in the Colab kernel (A2); Colab/Drive behaviour.
- **Network:** range reads of the real `video.zip` and the scorer download from raw.githubusercontent.com were exercised from this session, not from Colab. After the first run, the scorer is cached on Drive per commit.
- **Not implemented:** `RUN_ALL_FRAME_SECONDARY` stays off; no secondary all-frame protocol is implemented.
- **Skipped with a reason:** object-removal controls, because the release has per-budget, not per-frame, target visibility; text-only inference, because the adapter path is not validated.
- **Black-frame control:** the model computes camera tokens from the black images, and no real geometry is supplied. The control has only run with the stand-in model.
- **Historical comparison:** VSI-Bench at 128 frames versus ReVSI at 32 frames is labelled descriptive and confounded. The historical candidates are failure-selected.
- **Not available here:** the reference notebook `cambrian_p_colab (3).ipynb`. Its environment, install and worker steps reach this notebook through the validated VSI-Bench notebook, which was derived from it.
