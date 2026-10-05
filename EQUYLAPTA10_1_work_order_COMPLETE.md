# EQUYLAPTA 10.1 — Work Order

> **Status:** Appendix A, Appendix B and the Drive file list (§11.1) are filled in; there are no remaining placeholders.
>
> **Human, before pasting:** review the lines marked `◀ CONFIRM` in §3. The important one is what *Phenotype D* means: the v10 files never defined it, so §5 supplies a default definition. Everything else can stay as written. Then paste this whole file into the Colab agent.

**Environment:** Google Colab, T4 GPU, Google Drive mountable at `/content/drive`.  **Run ID:** `equylapta10_1`

---

## 1. Mission

v10 reported "Phenotype D = 92.4% (95% CI 90.1–94.7%)". Independent audits found nothing in the v10 package that supports that number (§4 lists the defects). Re-run Locate-and-Edit Model Surgery so that the answer to *"does the located edit reach Phenotype D ≥ 90%?"* rests on raw per-example data and executable code, and is whatever the data say.

A failing result reported honestly is a **successful** run. A passing result that cannot be recomputed from the files you ship is a **failed** run.

**Method constraint.** The recipient model is never trained, distilled, merged, or LoRA-adapted. The only change to it is the located edit, and a parameter diff must prove that (gate G8).

---

## 2. Hard rules

| # | Rule |
|---|---|
| R1 | **No result literals.** Every result (rates, counts, CIs, p-values, losses, hashes, file counts) is computed by code in this session from raw logs and written to files by code. Never type, estimate, "infer", or back-solve a result or a sample size. Thresholds and config values may be literals; results may not. |
| R2 | **Real code only.** Every script in `code/` is executed in this session and its stdout/stderr saved under `logs/`. Comment-only files, stubs, and scripts that were never run are prohibited. |
| R3 | **One source of truth.** `final_results.json` holds every gate status and headline number. README, verdict and reports are rendered from it by code. P13 verifies that no file disagrees with it. |
| R4 | **Status vocabulary:** `PASS`, `FAIL`, `NOT_RUN`, `NOT_APPLICABLE` (with reason). `NOT_RUN` counts as not passed. The word VERIFIED appears only as the computed verdict. |
| R5 | **Freeze before you look.** The analysis plan (`spec/PHENOTYPE_D_SPEC.json`) is hashed at the end of P0; the selections (`spec/SELECTIONS_FROZEN.json`) are hashed before the test run. The test set is evaluated once (P9). If a bug forces a re-run, keep both results, report both, and state that the test set was accessed twice. |
| R6 | **Test data never influences a choice.** Layer, latents, k, α, detector and thresholds are chosen on train/val only. |
| R7 | **After seeing any result:** never lower a threshold, loosen the agreement definition, drop prompts or conditions, relabel a condition, or swap the metric. |
| R8 | Donor and recipient must be different weight sets. Identical → `RUN_INVALID` (G2). |
| R9 | Every file named in any message or report exists in the zip. File lists are generated from the zip itself, never typed. |
| R10 | Describe results neutrally ("not supported by the data"); never characterize intent ("simulated", "fake"). |
| R11 | One final message (§12). No repeated "complete" banners, no emojis. |
| R12 | Aim for ≤ 4 h on the T4. If time is short, cut crosscoder steps or val iterations, never `N_TEST_PROMPTS` below 500. Checkpoint to `DRIVE_ROOT/_checkpoints/<phase>/` (with `.sha256`) after P1, P3, P5, P7, P9 so a disconnect can resume. |
| R13 | **Everything lands on Google Drive.** After every phase (P0–P14) mirror the whole `WORK_DIR/equylapta10_1/` tree to `DRIVE_ROOT/_live_mirror/` with `shutil.copytree(..., dirs_exist_ok=True)` and log file count and bytes. Local Colab disk is scratch only; if the session dies, Drive must already hold every file produced so far. Section 11.1 lists the file names that must exist on Drive at the end. |

---

## 3. Configuration

```
RUN_ID                 = "equylapta10_1"
WORK_DIR               = "/content/equylapta10_1_work"   # local disk (fast); Drive holds checkpoints + final zip only
DRIVE_ROOT             = "/content/drive/MyDrive/EQUYLAPTA10_1"
INPUT_ZIPS (read-only, keep this exact spelling) in /content/drive/MyDrive/:
    equylapta10potato.zip
    equylapta10potatoverification.zip
    equylapta10reverificaion.zip

DESIGN                 = "planted_donor"            # ◀ CONFIRM   or "user_pair" (then fill DONOR_MODEL)
RECIPIENT_MODEL        = "EleutherAI/pythia-70m"    # ◀ CONFIRM
DONOR_MODEL            = ""                         # user_pair only
TARGET_BEHAVIOR        = "math_style_continuation"  # ◀ CONFIRM — this is what Phenotype D measures (§5.2)
PHENOTYPE_D_THRESHOLD  = 0.90
N_TEST_PROMPTS         = 1000                       # hard minimum 500
N_VAL_PROMPTS          = 500
CONFIDENCE             = 0.95                       # Wilson interval
MAX_VAL_ITERATIONS     = 5
SEED                   = 1010
INCLUDE_HEAVY_IN_ZIP   = False                      # donor weights / activation shards stay on Drive, SHA-256 recorded in the zip
```

If a field is blank, use the default shown and record `"source": "default"` in `config_used.json` (user-set fields get `"source": "user"`). Do not stop to ask questions mid-run unless a required input has no default. If an input zip is missing, record `NOT_FOUND` and continue.

---

## 4. What went wrong in v10, and what 10.1 must do

| v10 finding | 10.1 requirement |
|---|---|
| 92.4% and its CI were static values in `metric_sanity_check.json`; no code computed them | R1; `10_statistics_and_verdict.py` computes every statistic from `per_example_results.csv` |
| The agreement metric was never defined | §5.2 definition frozen in the spec; `metric.py` with unit tests |
| Only 20 generations from 2 prompts. 20 trials cannot give 92.4% (nearest values are 90% and 95%); an exact 92.4% needs a multiple of 250 trials | ≥ 500 distinct held-out prompts (default 1,000); the prompt is the unit of independence |
| Dataset of 5 items, a test split of 1, activation statistics from 4 vectors | Real corpora, document-level splits, leakage check (P1) |
| The SAE was an untouched random init (all weights within ±1/√fan_in, excess kurtosis −1.200); the crosscoder had barely moved and latent 608 was unremarkable | Real training with health checks (P5, G9); latents chosen on train, k and α on val (P6–P7) |
| Wrong-donor and wrong-layer controls produced text identical to the primary vector; "low" and "high" α were both 1.2; a sweep over 0.5–2.0 was claimed but only 0.0 and 1.2 were run | Controls with logged vector hashes (P6, §7); the α grid is actually run (P7); G6 requires negative controls to fail |
| Steering was ≈ 1% of the activation norm; on one prompt every condition matched baseline (trivial agreement) | α in units of the median residual norm; headroom gate G3; no-op rate reported |
| Donor and recipient were the same model and layer | R8, G2 |
| First verification copied the claim as "independent", "inferred" n = 100, labelled a normal-approximation interval as Wilson, and passed a CI whose lower bound was 87.2% | A second implementation recomputes every statistic; no inferred n; CI methods named correctly (§8) |
| Every script in both verification packages was one comment line; the claimed T4 re-run does not appear in the logs | R2; executed code with saved stdout; G10 |
| README, JSON and chat gave contradictory statuses (e.g. "controls valid: PASS" vs `controls_verified: false`) | R3; consistency check (P13, G11) |
| The chat reported two reports (`hardcoded_claim_detection.txt`, `final_zip_self_verification.txt`) that are not in the delivered zip | R9 |
| Step reports 1–7 were identical boilerplate with no metrics | Every phase writes a log with counts, durations and hashes |

---

## 5. Phenotype D: definition, gates, verdicts

### 5.1 Design

**`planted_donor` (default).** To have a ground-truth donor, copy the recipient and fine-tune the *copy* briefly on `TARGET_BEHAVIOR` text. The donor is a test fixture only. The method under test is locating the behaviour in activation space and editing the **untouched** recipient.

**`user_pair`.** Use `DONOR_MODEL` and `RECIPIENT_MODEL` as given. If hidden sizes differ or the models do not share a lineage, learn a linear map donor→recipient on paired train activations and require R² ≥ 0.9 on val.

### 5.2 Test unit and agreement (default definition — replace only if yours differs)

- **Test unit:** one neutral prefix — a 32–64-token document-initial span from a held-out general-text document, cut at a sentence boundary, whose own detector label is 0 (the prefix is not mathematical). Each prefix comes from a different document.
- For every condition *c*: greedy decoding, 48 new tokens, fp32 → continuation `y_c(x)`. The donor's continuation is `y_D(x)`.
- `label(y)` is the frozen deterministic detector (P2). `distinct2(y)` = distinct bigrams ÷ total bigrams of the continuation.
- Prefixes where `label(y_D) = 0` are discarded for every condition. Over-sample candidates (2 × N) so that at least N remain.
- **`agree_c(x) = 1` iff `label(y_c(x)) = 1` and `distinct2(y_c(x)) ≥ 0.5`; otherwise 0.**
- **Phenotype D agreement of condition *c* = k_c / n**, with k_c = Σ agree_c(x) and n = number of test prefixes.
- Reported, not gating: fraction of continuation tokens equal to the donor's; next-token KL recovery = 1 − KL(donor‖edited) / KL(donor‖recipient), with bootstrap CI.

If your Phenotype D means something else (e.g. agreement with a specific larger donor), replace this subsection and the detector in P2 only; every other rule stays. The behaviour must be checkable by a deterministic detector validated against source labels. LLM-as-judge may not gate.

### 5.3 Gates (all computed by code; thresholds frozen in the spec)

| ID | Gate | Pass condition |
|---|---|---|
| G1 | Sample size | n_test ≥ 500 distinct documents (plan: `N_TEST_PROMPTS`) |
| G2 | Donor validity | ≥ 1 parameter tensor differs between donor and recipient; donor exhibit rate ≥ 0.95 on candidate test prefixes |
| G3 | Headroom | agreement(A) ≤ 0.50 (the unedited recipient must not already "agree") |
| G4 | Primary agreement | Wilson 95% **lower bound** of F ≥ `PHENOTYPE_D_THRESHOLD` |
| G5 | Causal gain | exact McNemar F vs A: p < 0.001, F better |
| G6 | Negative controls fail | for each of D, G, H, I, J: agreement(F) − agreement(control) ≥ 0.30 and McNemar p < 0.001 (for replicated controls G, I, J use the best-scoring replica); H must be identical to A on 100% of prompts. D may be `NOT_APPLICABLE` (< 4 recipient-only latents, count recorded); J may be `NOT_APPLICABLE_LAYER_AGNOSTIC` (§7) |
| G7 | Collateral damage | general-text perplexity ratio F/A ≤ max(1.25, donor/A ratio); median donor-NLL of F continuations ≤ that of A continuations; corpus distinct-2 of F outputs ≥ 0.8 × that of donor outputs |
| G8 | Surgery integrity | parameter diff shows only the declared edit tensor(s) differ from the recipient (all others bitwise equal); hook vs folded-bias logits max abs diff ≤ 1e-3 |
| G9 | Locate health | `init_signature_test` is false for every crosscoder weight tensor; final train loss ≤ 0.5 × initial; val FVU ≤ 0.35 per model; selected latents beat the best of 100 random-latent sets on val by ≥ 0.30 |
| G10 | Reproducibility | re-run of A and F on 200 fixed test prompts in a fresh process: ≥ 99% identical outputs |
| G11 | Integrity | P13 finds 0 violations; every number in every report traces to a results file |

### 5.4 Verdict (computed by code, never typed)

| Verdict | Condition |
|---|---|
| `RUN_INVALID` | any of G1, G2, G8, G9, G10, G11 fails |
| `STOPPED_BEFORE_TEST` | the P7 readiness gate failed; the test set is untouched |
| `INCOMPLETE` | the session ended before P10 finished |
| `PHENOTYPE_D_NOT_REACHED` | run valid; G4 fails |
| `PHENOTYPE_D_NOT_ESTABLISHED` | run valid; G4 passes but at least one of G3, G5, G6, G7 fails (high agreement that cannot be attributed to a specific, clean edit) |
| `PHENOTYPE_D_VERIFIED` | every gate passes (`NOT_APPLICABLE` allowed only where §5.3 permits, with the caveat written in the verdict file) |

---

## 6. Pipeline

Write one script per phase (names in §9). Each phase logs counts, durations, seeds and file hashes to `logs/`.

**P0 Setup and plan freeze.** Mount Drive; create `WORK_DIR` and `DRIVE_ROOT`; save `nvidia-smi` and `pip freeze` to `logs/`; set seeds, `torch.use_deterministic_algorithms(True, warn_only=True)`, `CUBLAS_WORKSPACE_CONFIG=:4096:8`, batch size 32 with left padding. Record SHA-256 of the three input zips (read-only; never modify them; do not use their contents as evidence for 10.1). Write `config_used.json`. Write and hash `spec/PHENOTYPE_D_SPEC.json` (§5 definitions, gate thresholds, §7 conditions, §8 statistics, seeds, sample sizes). Write `common.py`, `metric.py`, `hooks.py` and `reference_utils.py` (Appendix A) with unit tests (known Wilson values; exact McNemar; detector on synthetic strings; `agree` arithmetic; degenerate-repetition case; hook vs fold equivalence; init-signature test). Run them; save output to `logs/unit_tests.txt`. All must pass before P1.

**P1 Data and splits.** Verify each source loads; record any substitution in the spec. General text: `NeelNanda/pile-10k`. Math-style text: `openai/gsm8k` and `EleutherAI/hendrycks_math` (train splits). Split at **document** level into train/val/test pools (fixed seed); no document in two pools; exact-hash and 13-gram near-duplicate check across pools → `data/leakage_check.json` (overlaps must be 0, otherwise remove and log). Reserve 500 extra test-pool documents (128 tokens each) for perplexity. Build neutral prefixes (§5.2); save jsonl files and SHA-256 of each in `split_manifest.json`.

**P2 Detector.** A frozen deterministic classifier for `TARGET_BEHAVIOR` (rules, or sklearn trained on the train pool only). Validate on 48-token segments cut from val documents using their *source* labels (math-style vs general): balanced accuracy ≥ 0.95, else fix on train/val. Save definition, parameters, SHA-256, confusion matrix.

**P3 Donor.** `planted_donor`: fine-tune a **copy** of the recipient on train-pool math-style text (small LR, a few hundred steps) until the val exhibit rate is ≥ 0.95. The recipient's weights are never touched. Save donor weights to `DRIVE_ROOT/heavy_artifacts/`; record SHA-256 and the recipe in `artifacts/donor_build/`. `user_pair`: load the models and learn the map if needed (§5.1).

**P4 Layer sweep (cheap).** For each layer ℓ = 0..5: `v_ℓ` = unit(mean(donor − recipient activations at block ℓ output)) over ≤ 200k train tokens; steer with `v_ℓ` at each α in the grid on 200 val prefixes. Record in `results/val_alpha_layer_sweep.csv`. `ℓ*` = layer with the highest best-α val agreement. (This table also decides whether control J is applicable; §7.)

**P5 Locate: crosscoder at ℓ\*.** Shared latent space over [donor, recipient] activations at ℓ\* (per-model encoder/decoder weights); `d_latent ≥ 4,096` (default 8,192); ≥ 5M training tokens; ≥ 3,000 optimizer steps; sparsity penalty weighted by decoder norms. Exclude position 0 (massive activations). Store activations as fp16 memmap shards on local disk. Log loss, L0, FVU per model and dead-latent fraction every 100 steps to `artifacts/training_curves.csv`. At the end run `init_signature_test` on every weight tensor and save the results. Failing G9 → `RUN_INVALID`.

**P6 Select latents and build vectors.** Score latents on **train prefixes only**: relative decoder norm ρ_j = ‖d_j^donor‖ / (‖d_j^donor‖ + ‖d_j^recipient‖) and the mean activation difference between donor-behaviour positions and neutral positions (scoring rule declared in the spec). Take the top-k (k ∈ {1, 4, 16}; final k on val). `v_F` = unit-norm sum of the selected latents' recipient-space decoder directions. Build every other vector per §7. Write `artifacts/vectors/vectors_index.json` with SHA-256, L2 norm and cosine to `v_F` for each vector. Assert |cos| < 0.5 against `v_F` for every replica of G and I (a control that equals the treatment is a bug).

**P7 Val calibration and readiness gate.** α is in units of the median per-token residual L2 norm at ℓ\* (train tokens, position 0 excluded); `alpha_abs = alpha_rel × median_norm`. Grid `alpha_rel ∈ {0, 0.1, 0.25, 0.5, 0.75, 1, 1.5, 2, 3}` × k ∈ {1, 4, 16} on `N_VAL_PROMPTS` val prefixes; record agreement and collateral metrics. `(k*, α*)` = highest val agreement subject to the G7 thresholds. **Readiness gate:** val agreement(F) ≥ 0.92, val agreement(A) ≤ 0.50, and F beats the best of 100 random-latent vectors by ≥ 0.30. If it fails, run up to `MAX_VAL_ITERATIONS` bounded improvement rounds (more crosscoder steps, different k or ℓ, larger `d_latent`), each logged to `results/val_iteration_log.csv`. Still failing → verdict `STOPPED_BEFORE_TEST`: leave the test set untouched, skip to P12 with test sections `NOT_RUN`, and put the diagnosis in `reports/13_limitations_next_steps.txt`.

**P8 Freeze selections and perform the surgery.** Write and hash `spec/SELECTIONS_FROZEN.json` (ℓ\*, k\*, latent ids, α\*, detector SHA-256, split hashes, vector hashes, seeds, `test_access_count: 0`). Fold the edit into a copy of the recipient: `model.gpt_neox.layers[ℓ*].mlp.dense_4h_to_h.bias += alpha_abs · v_F` (adds the vector to every position after block ℓ\*). Write `weight_diff_report.json` (per tensor: changed?, max abs diff, SHA-256 before/after; only that bias may change). Check hook-vs-fold logits on 64 val prompts (max abs diff ≤ 1e-3). Save `surgery_diff.safetensors` (changed tensor only) and `apply_surgery.py`, then prove it reconstructs the edited model from the public base weights plus the diff. Hooks (all other conditions) go on the block output (a tuple: modify element 0) at every position, including generated tokens.

**P9 Test evaluation (once).** First write `logs/test_set_access_log.json` (timestamp, spec and selections hashes, access #1); refuse to run if it already exists. Evaluate every condition in §7 on every test prefix with identical decoding. Condition F uses the folded checkpoint (no hooks); the others use hooks. Append one row per (prompt, condition, replica) to `results/per_example_results.csv` (columns in §9). Compute collateral metrics on the reserved perplexity set.

**P10 Statistics and verdict.** `10_statistics_and_verdict.py` reads only the CSVs. It computes everything in §8, recomputes with the second implementation, evaluates G1–G11 (G11 pending until P13), and writes `final_results.json` with `{status, value, threshold, evidence_file}` per gate plus the verdict.

**P11 Reproducibility re-run.** In a fresh process, re-run A and F on a fixed seeded subset of 200 test prompts; compare outputs row by row → `results/reproducibility_rerun.csv`. This is a re-execution of the frozen pipeline, not a second look at the test set; log it as such.

**P12 Render reports and spot-check sheet.** Render `README_FIRST.txt`, `FINAL_PHENOTYPE_D_VERDICT.txt` and all reports (§10) from `final_results.json` and the results files with templates. Write `results/spot_check_sample.csv` (30 seeded random test prompts: prompt, donor output, A output, F output, best-G output, labels, `agree`).

**P13 Consistency and integrity check.** Scan every `.txt`, `.json` and `.csv` in the package: (a) every PASS/FAIL/VERIFIED token agrees with `final_results.json`; (b) every file named anywhere exists; (c) banned patterns: code files with < 5 non-comment lines, `inferred_sample_size`, `TODO`, `placeholder`, `simulated`, and headline numbers appearing as literals in `code/`; (d) counts (prompts, conditions, files) agree across files. Write the result to `reports/12_integrity_consistency_report.txt`, write G11 into `final_results.json`, recompute the verdict, re-render the three top-level files, and re-check until stable (max 3 passes). Do not package while violations remain.

**P14 Package and copy to Drive.** §11.

---

## 7. Conditions

All steering conditions add `alpha_abs · unit_vector` to the block output at every position of block ℓ\* unless stated. Every row of `per_example_results.csv` records condition, replica, layer, `alpha_rel`, `alpha_abs` and the vector SHA-256.

| Code | Name | Definition | Expected |
|---|---|---|---|
| A | Baseline | unedited recipient, no hook | reference |
| B | Low-α uniform | `v_F` at 0.5·α\* | ≤ F |
| C | High-α uniform | `v_F` at 2·α\* | report; may degrade fluency |
| D | Recipient-only subtraction | subtract the unit sum of the top-k recipient-only latents' directions (ρ_j < 0.1) at α\* | ≈ A (negative control) |
| E | Donor-only projection | projection of mean(donor − recipient) onto the span of the selected latents' directions, at α\* | ablation, not gated |
| F | **Primary donor vector** | `v_F` at layer ℓ\*, α\* (folded into the bias) | the test of Phenotype D |
| G | Matched random vectors | 20 random unit vectors, same `alpha_abs`, layer ℓ\* | ≈ A (negative control) |
| H | Sham | hook installed, zero vector | **identical to A** |
| I | Wrong-latent (v10's "wrong-donor") | 20 vectors from random non-selected latents, norm-matched | ≈ A (negative control). If time allows, also `I2`: a vector located by the same pipeline from a second planted donor with a different behaviour (e.g. code-style) |
| J | Wrong-layer | `v_F` injected at each layer ≠ ℓ\* (one replica per layer), same `alpha_abs` | below F. If the P4 sweep shows max − min val agreement across layers < 0.30, the direction is layer-agnostic: mark J `NOT_APPLICABLE_LAYER_AGNOSTIC` and write "layer specificity not demonstrated" in the verdict file |
| K | *(optional)* v10 legacy vector | `direction_vector_layer2.npy` at layer 2, α\* — only if the recipient is `pythia-70m` | report only |
| L | *(optional)* diff-of-means reference | unit(mean(donor − recipient)) at ℓ\*, α\* | report only; F should match or beat it, otherwise say so |

---

## 8. Statistics

- Unit of analysis = prompt. Report every rate with **k and n**.
- Rates: Wilson 95% interval (gating) and Clopper–Pearson (reported). Recompute with a second implementation; the two must agree to 1e-9 (`wilson_ci` vs `wilson_ci_crosscheck`, Appendix A).
- Paired comparisons: exact McNemar on discordant pairs (report b and c). Paired differences: bootstrap over prompts, 10,000 resamples, fixed seed.
- Never infer a sample size from a percentage. Never label a normal-approximation interval "Wilson".
- Only G1–G11 gate. Anything else is exploratory and labelled as such.

Required successes for the Wilson lower bound to reach 90% (computed with `min_successes_for_lower_bound`):

| n | min k | observed rate needed |
|---|---|---|
| 500 | 464 | 92.8% |
| 1,000 | 919 | 91.9% |
| 2,000 | 1,827 | 91.3% |

For scale: 231/250 = 92.4% has a Wilson lower bound of only 88.4%, so it would **fail** the gate.

---

## 9. Package contents

Top-level folder inside the zip is `equylapta10_1/`.

```
equylapta10_1/
├── README_FIRST.txt                      rendered; ≤ 25 lines; verdict, headline numbers, file map, how to recompute
├── FINAL_PHENOTYPE_D_VERDICT.txt         rendered: verdict, gate table, one-paragraph justification, caveats
├── final_results.json                    single source of truth
├── manifest.json                         per-file bytes + SHA-256 (generated by code at the end)
├── SHA256SUMS.txt
├── spec/
│   ├── PHENOTYPE_D_SPEC.json   + PHENOTYPE_D_SPEC.sha256       (frozen end of P0)
│   ├── SELECTIONS_FROZEN.json  + SELECTIONS_FROZEN.sha256      (frozen before P9)
│   └── config_used.json                  every field marked "user" or "default"
├── data/
│   ├── train_prompts.jsonl  val_prompts.jsonl  test_prompts.jsonl
│   ├── split_manifest.json               doc ids, sources, counts, SHA-256 per file
│   └── leakage_check.json
├── artifacts/
│   ├── donor_build/                      recipe, val exhibit rate, donor weight SHA-256
│   ├── detector/                         definition, parameters, validation metrics
│   ├── crosscoder_weights.safetensors    crosscoder_config.json    training_curves.csv
│   ├── selected_latents.json
│   ├── vectors/                          *.npy + vectors_index.json (SHA-256, norm, cosine to v_F)
│   ├── surgery_diff.safetensors          ONLY the changed tensor(s)
│   ├── weight_diff_report.json           apply_surgery.py
│   └── external_artifacts.json           heavy files kept on Drive: path, bytes, SHA-256
├── results/
│   ├── per_example_results.csv           test: one row per (prompt, condition, replica)
│   ├── val_alpha_layer_sweep.csv         val_iteration_log.csv
│   ├── condition_summary.csv             n, k, rate, Wilson and CP CIs, McNemar p, b, c, gap to F
│   ├── paired_tests.json  collateral_metrics.json  secondary_metrics.json
│   ├── reproducibility_rerun.csv
│   └── spot_check_sample.csv
├── reports/                              (§10)
├── code/
│   ├── common.py  metric.py  hooks.py  reference_utils.py  tests/
│   ├── 00_setup_and_plan_freeze.py            08_freeze_selections_and_surgery.py
│   ├── 01_build_splits.py                     09_test_evaluation_ONCE.py
│   ├── 02_build_detector.py                   10_statistics_and_verdict.py
│   ├── 03_build_donor.py                      11_reproducibility_rerun.py
│   ├── 04_layer_sweep.py                      12_render_reports.py
│   ├── 05_train_crosscoder.py                 13_consistency_and_integrity_check.py
│   ├── 06_select_latents_build_vectors.py     14_package_and_drive_copy.py
│   └── 07_val_calibration_and_readiness.py
├── logs/                                 execution_log.txt, nvidia_smi.txt, pip_freeze.txt, unit_tests.txt,
│                                         one stdout file per phase, test_set_access_log.json
└── inputs_reference/                     v10_input_hashes.json, v10_defect_register.json (§4 as JSON)
```

`per_example_results.csv` columns: `prompt_id, doc_id, condition, replica, layer, alpha_rel, alpha_abs, vector_sha256, prompt_text, donor_output, output, donor_label, label, distinct2, agree, token_match_frac, identical_to_A, seed, batch_size`.

Target zip size ≤ 300 MB. Heavy artifacts (donor weights, activation shards, any full edited checkpoint) live in `DRIVE_ROOT/heavy_artifacts/` unless `INCLUDE_HEAVY_IN_ZIP = True`.

---

## 10. Reports (all in `reports/`, plain `.txt`, every number rendered by code)

| File | Must contain |
|---|---|
| `00_EXECUTIVE_SUMMARY.txt` | verdict; F, A and best negative control with k/n and Wilson CIs; gate table G1–G11 (status, value, threshold); three plain sentences on what the result does and does not show; test-set access count |
| `01_spec_and_config_report.txt` | resolved config with each field's source; Phenotype D as implemented; SHA-256 of spec and selections; timestamps proving the spec was frozen before the test run |
| `02_data_splits_leakage_report.txt` | sources and substitutions; counts per pool; prefix construction; donor exhibit rate on candidates; overlap results; split-file hashes |
| `03_donor_detector_report.txt` | donor construction (or user pair and mapping R²); donor ≠ recipient evidence; detector definition, validation balanced accuracy, confusion matrix; 5 detector hits and 5 misses |
| `04_locate_training_report.txt` | crosscoder config; first/last loss, L0, FVU per model, dead fraction; init-signature result per tensor; latent scores; selected-vs-random gap on val |
| `05_calibration_val_report.txt` | layer sweep; α × k grid; collateral per α; chosen (ℓ\*, k\*, α\*); readiness outcome; every val iteration |
| `06_surgery_integrity_report.txt` | edit definition (tensor name, `alpha_abs`, ‖v‖); parameter-diff summary; hook-vs-fold difference; reconstruct-from-diff result |
| `07_test_results_report.txt` | table of A–L: n, k, rate, Wilson and CP CIs, no-op rate (outputs identical to A), paired tests vs A; headroom; secondary metrics |
| `08_controls_report.txt` | each control's definition, replicas, vector hashes and cosines, rates, gap to F, McNemar p; J applicability decision |
| `09_collateral_damage_report.txt` | perplexity ratios, donor-NLL medians, distinct-2; 5 seeded example outputs each for A, F and the donor |
| `10_statistics_audit_report.txt` | formulas; own-vs-scipy max abs difference; n/k table; confirmation that every headline number was recomputed from `per_example_results.csv` |
| `11_reproducibility_report.txt` | re-run design; seeds; versions; identical-output fraction; any differing rows |
| `12_integrity_consistency_report.txt` | each P13 check, items scanned, violations (must be 0), banned-pattern findings |
| `13_limitations_next_steps.txt` | limitations (one model pair, planted donor, one behaviour, detector validity). If the verdict is not VERIFIED: a ranked diagnosis from the data and the specific experiments for 10.2 (not run) |
| `14_human_spot_check_guide.txt` | how to read `spot_check_sample.csv`; what to count by eye; how to compare with `agree` (human and detector labels should agree on ≥ 90% of rows if the detector is valid) |

---

## 11. Packaging and Google Drive procedure (P14)

1. After P13 shows zero violations, write `artifacts/external_artifacts.json` and copy heavy artifacts to `DRIVE_ROOT/heavy_artifacts/` (unless `INCLUDE_HEAVY_IN_ZIP`).
2. Run `package_and_copy(f"{WORK_DIR}/{RUN_ID}", RUN_ID, DRIVE_ROOT)` from Appendix A. It builds `manifest.json` and `SHA256SUMS.txt`, creates `equylapta10_1.zip` (top-level folder `equylapta10_1/`, deterministic file order), tests it, copies it to Drive, reads the Drive copy back, and compares SHA-256, `testzip` and the file list against the manifest. An existing file of the same name is renamed `*_prev_<timestamp>`, never overwritten.
3. Call `write_drive_verification(...)`. The zip cannot contain its own hash or the result of verifying itself, so this record sits **beside** the zip on Drive.
4. If SHA-256 or `testzip` fails: delete the Drive copy, copy once more; if it still fails, report `FAILED_PACKAGING` and stop.

Result on Drive:

```
MyDrive/EQUYLAPTA10_1/
├── equylapta10_1.zip
├── equylapta10_1_SHA256.txt                 sidecar: "<sha256>  equylapta10_1.zip"
├── equylapta10_1_DRIVE_VERIFICATION.txt     local/Drive SHA-256, match, testzip, manifest match, file listing of the zip
├── heavy_artifacts/
└── _checkpoints/
```

### 11.1 Files that must exist on Google Drive at the end (names are exact)

All paths are under `/content/drive/MyDrive/EQUYLAPTA10_1/`. Generate this list by code from the Drive folder itself and compare it with the table below (R9); report any missing name as a violation.

| Location on Drive | File / folder name | Written by |
|---|---|---|
| `EQUYLAPTA10_1/` | `equylapta10_1.zip` | P14 (`package_and_copy`) |
| `EQUYLAPTA10_1/` | `equylapta10_1_SHA256.txt` | P14 (`write_drive_verification`) |
| `EQUYLAPTA10_1/` | `equylapta10_1_DRIVE_VERIFICATION.txt` | P14 (`write_drive_verification`) |
| `EQUYLAPTA10_1/` | `equylapta10_1_prev_<timestamp>.zip` (only if an earlier zip existed) | P14 |
| `EQUYLAPTA10_1/heavy_artifacts/` | `donor_model/` (planted-donor weights, HF format), `donor_model.sha256`, `activations_<layer>_shard_*.f16.npy` (memmap shards), `external_artifacts_copy.json` | P3, P5, P14 |
| `EQUYLAPTA10_1/_checkpoints/<phase>/` | one folder each for `P1`, `P3`, `P5`, `P7`, `P9`, each holding its files plus a `.sha256` | R12 |
| `EQUYLAPTA10_1/_live_mirror/equylapta10_1/` | full mirror of the work tree (R13), including every file below | after every phase |

Files inside `equylapta10_1.zip` (and therefore also in `_live_mirror/`), exactly as in §9:

- top level: `README_FIRST.txt`, `FINAL_PHENOTYPE_D_VERDICT.txt`, `final_results.json`, `manifest.json`, `SHA256SUMS.txt`
- `spec/`: `PHENOTYPE_D_SPEC.json`, `PHENOTYPE_D_SPEC.sha256`, `SELECTIONS_FROZEN.json`, `SELECTIONS_FROZEN.sha256`, `config_used.json`
- `data/`: `train_prompts.jsonl`, `val_prompts.jsonl`, `test_prompts.jsonl`, `split_manifest.json`, `leakage_check.json`
- `artifacts/`: `crosscoder_weights.safetensors`, `crosscoder_config.json`, `training_curves.csv`, `selected_latents.json`, `surgery_diff.safetensors`, `weight_diff_report.json`, `apply_surgery.py`, `external_artifacts.json`, and the folders `donor_build/`, `detector/`, `vectors/` (with `vectors_index.json`)
- `results/`: `per_example_results.csv`, `val_alpha_layer_sweep.csv`, `val_iteration_log.csv`, `condition_summary.csv`, `paired_tests.json`, `collateral_metrics.json`, `secondary_metrics.json`, `reproducibility_rerun.csv`, `spot_check_sample.csv`
- `reports/`: `00_EXECUTIVE_SUMMARY.txt`, `01_spec_and_config_report.txt`, `02_data_splits_leakage_report.txt`, `03_donor_detector_report.txt`, `04_locate_training_report.txt`, `05_calibration_val_report.txt`, `06_surgery_integrity_report.txt`, `07_test_results_report.txt`, `08_controls_report.txt`, `09_collateral_damage_report.txt`, `10_statistics_audit_report.txt`, `11_reproducibility_report.txt`, `12_integrity_consistency_report.txt`, `13_limitations_next_steps.txt`, `14_human_spot_check_guide.txt`
- `code/`: `common.py`, `metric.py`, `hooks.py`, `reference_utils.py`, `tests/test_reference_utils.py`, `00_setup_and_plan_freeze.py`, `01_build_splits.py`, `02_build_detector.py`, `03_build_donor.py`, `04_layer_sweep.py`, `05_train_crosscoder.py`, `06_select_latents_build_vectors.py`, `07_val_calibration_and_readiness.py`, `08_freeze_selections_and_surgery.py`, `09_test_evaluation_ONCE.py`, `10_statistics_and_verdict.py`, `11_reproducibility_rerun.py`, `12_render_reports.py`, `13_consistency_and_integrity_check.py`, `14_package_and_drive_copy.py`
- `logs/`: `execution_log.txt`, `nvidia_smi.txt`, `pip_freeze.txt`, `unit_tests.txt`, `test_set_access_log.json`, and one stdout file per phase named `phase_00_stdout.txt` … `phase_14_stdout.txt`
- `inputs_reference/`: `v10_input_hashes.json`, `v10_defect_register.json`

If `STOPPED_BEFORE_TEST`, `per_example_results.csv`, `reproducibility_rerun.csv`, `spot_check_sample.csv` and `test_set_access_log.json` are replaced by `NOT_RUN` stubs that state the reason, and the file list in the final message says so.

---

## 12. Final message (send once)

Plain text, rendered from `final_results.json` and the zip listing:

```
EQUYLAPTA 10.1 — <verdict>
Test prompts n=…   A: k/n (…%)   F: k/n (…%, Wilson 95% [lo, hi])   best negative control: k/n (…%)
Gates G1–G11: one line each (status, value, threshold)
Drive: <path> | <bytes> | SHA-256 <hash> | local = Drive: yes/no | testzip: ok/failed
Files in zip: <count per folder, generated from the zip's own listing>
NOT_RUN / NOT_APPLICABLE: <list with reasons>
Test-set accesses: <n>
Next step: <one line from reports/13>
```

---

## 13. Failure handling

- Val readiness fails → `STOPPED_BEFORE_TEST`; package the val evidence and diagnosis; do not touch the test set.
- Crosscoder unhealthy (G9) → `RUN_INVALID`; report which check failed; do not select latents from it.
- Valid run, gate not met → report the number as it is (`NOT_REACHED` / `NOT_ESTABLISHED`). Do not retune on test. A new attempt is a new run (`equylapta10_1b`) with a fresh, never-used test pool.
- Bug found after P9 → fix, re-run from the affected phase, keep both outputs, report test access = 2.
- Session lost → resume from `_checkpoints/`; log the resume.
- Anything you cannot compute → `NOT_RUN` with the reason. Never fill a gap with a plausible number.

---

## Appendix A — Reference code (tested; adapt freely, keep the behaviour)

Save as `code/reference_utils.py` and import it from your scripts.

```python
"""EQUYLAPTA 10.1 reference utilities (Appendix A).

Statistics, metric helpers, init-signature test, parameter diff, hook/fold helpers,
and packaging/Drive verification. Every function here is pure and deterministic.
"""
import hashlib, json, math, os, shutil, time, zipfile
from statistics import NormalDist
import numpy as np
from scipy import stats

# ----------------------------------------------------------------- hashing
def sha256_bytes(b: bytes) -> str:
    return hashlib.sha256(b).hexdigest()

def sha256_file(path, chunk=1 << 20) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for blk in iter(lambda: f.read(chunk), b""):
            h.update(blk)
    return h.hexdigest()

def sha256_json(obj) -> str:
    return sha256_bytes(json.dumps(obj, sort_keys=True, separators=(",", ":")).encode())

def sha256_array(a) -> str:
    a = np.ascontiguousarray(np.asarray(a))
    return sha256_bytes(a.tobytes())

# -------------------------------------------------------------- statistics
def _z(conf):
    return NormalDist().inv_cdf(0.5 + conf / 2.0)

def wilson_ci(k, n, conf=0.95):
    """Wilson score interval, closed form. Returns (lo, hi)."""
    if n <= 0 or not (0 <= k <= n):
        raise ValueError("need 0 <= k <= n, n > 0")
    z = _z(conf); p = k / n; z2 = z * z
    denom = 1 + z2 / n
    centre = (p + z2 / (2 * n)) / denom
    half = z * math.sqrt(p * (1 - p) / n + z2 / (4 * n * n)) / denom
    return max(0.0, centre - half), min(1.0, centre + half)

def wilson_ci_crosscheck(k, n, conf=0.95):
    """Independent implementation: solve (p_hat - p)^2 = z^2 p(1-p)/n by bisection."""
    if n <= 0 or not (0 <= k <= n):
        raise ValueError("need 0 <= k <= n, n > 0")
    z2 = _z(conf) ** 2; ph = k / n
    f = lambda p: (ph - p) ** 2 - z2 * p * (1 - p) / n   # <= 0 inside the interval
    def bisect(a, b, sign_a):
        for _ in range(200):
            m = (a + b) / 2
            if (f(m) <= 0) == sign_a:
                a = m
            else:
                b = m
        return (a + b) / 2
    lo = 0.0 if k == 0 else bisect(0.0, ph, False)   # f>0 at 0, f<=0 at ph
    hi = 1.0 if k == n else bisect(ph, 1.0, True)    # f<=0 at ph, f>0 at 1
    return lo, hi

def clopper_pearson_ci(k, n, conf=0.95):
    a = 1 - conf
    lo = 0.0 if k == 0 else stats.beta.ppf(a / 2, k, n - k + 1)
    hi = 1.0 if k == n else stats.beta.ppf(1 - a / 2, k + 1, n - k)
    return float(lo), float(hi)

def mcnemar_exact(b, c):
    """Exact two-sided McNemar on discordant counts b (x only) and c (y only)."""
    m = b + c
    if m == 0:
        return 1.0
    return float(min(1.0, 2.0 * stats.binom.cdf(min(b, c), m, 0.5)))

def discordant_counts(x, y):
    """x, y: 0/1 arrays over the same prompts. b = x=1,y=0 ; c = x=0,y=1."""
    x = np.asarray(x, int); y = np.asarray(y, int)
    return int(((x == 1) & (y == 0)).sum()), int(((x == 0) & (y == 1)).sum())

def min_successes_for_lower_bound(n, threshold=0.90, conf=0.95):
    """Smallest k whose Wilson lower bound >= threshold (None if impossible)."""
    for k in range(n + 1):
        if wilson_ci(k, n, conf)[0] >= threshold:
            return k
    return None

def bootstrap_diff_ci(x, y, n_boot=10000, conf=0.95, seed=1010):
    """Paired bootstrap over prompts for mean(x) - mean(y). Returns (diff, lo, hi)."""
    x = np.asarray(x, float); y = np.asarray(y, float)
    d = x - y; n = len(d)
    rng = np.random.default_rng(seed)
    means = np.empty(n_boot)
    for i in range(n_boot):
        means[i] = d[rng.integers(0, n, n)].mean()
    a = (1 - conf) / 2
    return float(d.mean()), float(np.quantile(means, a)), float(np.quantile(means, 1 - a))

# ----------------------------------------------------------- metric helpers
def distinct2(tokens):
    """distinct bigrams / total bigrams. `tokens` is a list (token ids or words) or a str (split on whitespace)."""
    if isinstance(tokens, str):
        tokens = tokens.split()
    if len(tokens) < 2:
        return 0.0
    bg = list(zip(tokens[:-1], tokens[1:]))
    return len(set(bg)) / len(bg)

def agree(label, d2, min_distinct2=0.5):
    """agree = 1 iff the detector fires AND the continuation is not degenerate."""
    return int(int(label) == 1 and d2 >= min_distinct2)

def token_match_frac(a_ids, b_ids):
    n = min(len(a_ids), len(b_ids))
    if max(len(a_ids), len(b_ids)) == 0:
        return 1.0
    return sum(int(a_ids[i] == b_ids[i]) for i in range(n)) / max(len(a_ids), len(b_ids))

def ngrams13(text):
    w = text.split()
    return {" ".join(w[i:i + 13]) for i in range(max(0, len(w) - 12))}

# -------------------------------------------------------- init-signature test
def excess_kurtosis(a):
    a = np.asarray(a, np.float64).ravel()
    m = a.mean(); s2 = ((a - m) ** 2).mean()
    return float(((a - m) ** 4).mean() / (s2 ** 2) - 3.0) if s2 > 0 else float("nan")

def init_signature_test(w, init_w=None, kurt_tol=0.15):
    """True if tensor `w` looks like an untouched default init (a problem).

    Signals: (a) if init_w given, relative L2 distance < 1e-3; (b) all-zero tensor;
    (c) every |w| <= 1/sqrt(fan) for some fan in the shape AND excess kurtosis within
    kurt_tol of -1.2 (uniform(-b, b)). Returns a dict with the evidence.
    """
    a = np.asarray(w, np.float64)
    out = {"shape": list(a.shape), "max_abs": float(np.abs(a).max()),
           "excess_kurtosis": excess_kurtosis(a) if a.size > 3 else float("nan")}
    rel = None
    if init_w is not None:
        i = np.asarray(init_w, np.float64)
        rel = float(np.linalg.norm(a - i) / (np.linalg.norm(i) + 1e-12))
    out["rel_dist_from_init"] = rel
    zero = bool(np.abs(a).max() == 0.0)
    bounds_ok = False
    if a.ndim >= 2:
        bounds_ok = any(out["max_abs"] <= 1.0 / math.sqrt(f) + 1e-9 for f in a.shape if f > 0)
    kurt_ok = abs(out["excess_kurtosis"] + 1.2) <= kurt_tol if a.size > 3 else False
    out["looks_like_init"] = bool((rel is not None and rel < 1e-3) or zero or (bounds_ok and kurt_ok))
    return out

# ------------------------------------------------------------ parameter diff
def weight_diff_report(state_a, state_b):
    """state_*: dict name -> numpy array (or torch tensor). Returns per-tensor diff + changed list."""
    rep, changed = {}, []
    names = sorted(set(state_a) | set(state_b))
    for n in names:
        if n not in state_a or n not in state_b:
            rep[n] = {"changed": True, "reason": "missing in one side"}; changed.append(n); continue
        a = np.asarray(state_a[n].cpu().numpy() if hasattr(state_a[n], "cpu") else state_a[n])
        b = np.asarray(state_b[n].cpu().numpy() if hasattr(state_b[n], "cpu") else state_b[n])
        same = a.shape == b.shape and np.array_equal(a, b)
        rep[n] = {"changed": not same,
                  "max_abs_diff": 0.0 if same else (float(np.abs(a.astype(np.float64) - b).max()) if a.shape == b.shape else None),
                  "sha256_before": sha256_array(a), "sha256_after": sha256_array(b)}
        if not same:
            changed.append(n)
    return {"tensors": rep, "changed": changed, "n_tensors": len(names)}

# --------------------------------------------------------- hook / fold helpers
def make_block_output_hook(vector, alpha_abs):
    """forward hook for a GPT-NeoX block: adds alpha_abs*vector to output[0] at every position."""
    import torch
    def hook(module, inputs, output):
        v = (alpha_abs * torch.as_tensor(vector)).to(device=output[0].device, dtype=output[0].dtype)
        return (output[0] + v,) + tuple(output[1:])
    return hook

def fold_bias(model, layer, vector, alpha_abs):
    """Fold the edit into gpt_neox.layers[layer].mlp.dense_4h_to_h.bias (in place). Returns the tensor name."""
    import torch
    b = model.gpt_neox.layers[layer].mlp.dense_4h_to_h.bias
    with torch.no_grad():
        b += (alpha_abs * torch.as_tensor(vector)).to(device=b.device, dtype=b.dtype)
    return f"gpt_neox.layers.{layer}.mlp.dense_4h_to_h.bias"

# --------------------------------------------------------------- packaging
def _walk(root):
    out = []
    for dp, dn, fn in os.walk(root):
        dn.sort()
        for f in sorted(fn):
            out.append(os.path.relpath(os.path.join(dp, f), root).replace(os.sep, "/"))
    return sorted(out)

def build_manifest(run_dir, run_id):
    """Writes manifest.json (all files except itself and SHA256SUMS.txt) then SHA256SUMS.txt (all files except itself)."""
    skip = {"manifest.json", "SHA256SUMS.txt"}
    files = [f for f in _walk(run_dir) if f not in skip]
    man = {"run_id": run_id, "n_files": len(files),
           "files": [{"path": f, "bytes": os.path.getsize(os.path.join(run_dir, f)),
                      "sha256": sha256_file(os.path.join(run_dir, f))} for f in files]}
    with open(os.path.join(run_dir, "manifest.json"), "w") as fh:
        json.dump(man, fh, indent=1, sort_keys=True)
    sums = [f for f in _walk(run_dir) if f != "SHA256SUMS.txt"]
    with open(os.path.join(run_dir, "SHA256SUMS.txt"), "w") as fh:
        for f in sums:
            fh.write(f"{sha256_file(os.path.join(run_dir, f))}  {run_id}/{f}\n")
    return man

def make_zip(run_dir, run_id, zip_path):
    """Deterministic zip: sorted order, fixed timestamps, top-level folder run_id/."""
    with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
        for f in _walk(run_dir):
            zi = zipfile.ZipInfo(f"{run_id}/{f}", date_time=(2026, 1, 1, 0, 0, 0))
            zi.compress_type = zipfile.ZIP_DEFLATED; zi.external_attr = 0o644 << 16
            with open(os.path.join(run_dir, f), "rb") as fh:
                z.writestr(zi, fh.read())
    return zip_path

def package_and_copy(run_dir, run_id, drive_root, tmp_dir="/content"):
    """Build manifest + zip, test it, copy to Drive, read the Drive copy back and compare."""
    os.makedirs(drive_root, exist_ok=True)
    man = build_manifest(run_dir, run_id)
    local_zip = os.path.join(tmp_dir, f"{run_id}.zip")
    make_zip(run_dir, run_id, local_zip)
    with zipfile.ZipFile(local_zip) as z:
        local_testzip = z.testzip()
        listing = sorted(i.filename for i in z.infolist() if not i.is_dir())
    expected = sorted([f"{run_id}/{f['path']}" for f in man["files"]] +
                      [f"{run_id}/manifest.json", f"{run_id}/SHA256SUMS.txt"])
    local_sha = sha256_file(local_zip)
    dest = os.path.join(drive_root, f"{run_id}.zip")
    if os.path.exists(dest):
        os.rename(dest, os.path.join(drive_root, f"{run_id}_prev_{time.strftime('%Y%m%d_%H%M%S')}.zip"))
    for attempt in (1, 2):
        shutil.copy2(local_zip, dest)
        with open(dest, "rb") as fh: os.fsync(fh.fileno()) if hasattr(os, "fsync") else None
        drive_sha = sha256_file(dest)
        with zipfile.ZipFile(dest) as z:
            drive_testzip = z.testzip()
            drive_listing = sorted(i.filename for i in z.infolist() if not i.is_dir())
        ok = drive_sha == local_sha and drive_testzip is None
        if ok:
            break
        os.remove(dest)
    res = {"run_id": run_id, "local_zip": local_zip, "drive_zip": dest,
           "bytes": os.path.getsize(dest), "local_sha256": local_sha, "drive_sha256": drive_sha,
           "sha_match": drive_sha == local_sha, "testzip_ok": drive_testzip is None and local_testzip is None,
           "zip_matches_manifest": drive_listing == expected, "file_listing": drive_listing,
           "status": "OK" if (ok and drive_listing == expected) else "FAILED_PACKAGING"}
    return res

def write_drive_verification(res, drive_root, run_id):
    """Sidecar files beside the zip (the zip cannot contain its own hash)."""
    with open(os.path.join(drive_root, f"{run_id}_SHA256.txt"), "w") as fh:
        fh.write(f"{res['drive_sha256']}  {run_id}.zip\n")
    lines = [f"run_id: {run_id}", f"drive_path: {res['drive_zip']}", f"bytes: {res['bytes']}",
             f"local_sha256: {res['local_sha256']}", f"drive_sha256: {res['drive_sha256']}",
             f"sha_match: {res['sha_match']}", f"testzip_ok: {res['testzip_ok']}",
             f"zip_matches_manifest: {res['zip_matches_manifest']}", f"status: {res['status']}",
             f"files_in_zip: {len(res['file_listing'])}", "", "file listing:"] + res["file_listing"]
    with open(os.path.join(drive_root, f"{run_id}_DRIVE_VERIFICATION.txt"), "w") as fh:
        fh.write("\n".join(lines) + "\n")
```

Usage at P14 (Colab):

```python
from google.colab import drive; drive.mount('/content/drive')
from reference_utils import package_and_copy, write_drive_verification
DRIVE_ROOT = "/content/drive/MyDrive/EQUYLAPTA10_1"; RUN_ID = "equylapta10_1"
res = package_and_copy(f"/content/equylapta10_1_work/{RUN_ID}", RUN_ID, DRIVE_ROOT)
write_drive_verification(res, DRIVE_ROOT, RUN_ID)
assert res["sha_match"] and res["testzip_ok"] and res["zip_matches_manifest"]
```

Notes on Appendix A: `make_block_output_hook` and `fold_bias` need `torch` and were not exercised outside Colab, so P0 must add a hook-vs-fold equivalence test on a tiny random `GPTNeoXForCausalLM` (G8) and show it passing in `logs/unit_tests.txt`. `package_and_copy` uses a fixed zip timestamp so identical content gives an identical zip hash.

## Appendix B — Unit tests for `reference_utils.py`

Save as `code/tests/test_reference_utils.py`. These tests run without a GPU; extend them with the detector, `agree` on real strings and the hook-vs-fold test. Run with a plain loop over `test_*` functions if `pytest` is unavailable. Expected values already checked: Wilson lower bound for 231/250 is 0.884; `min_successes_for_lower_bound` gives 464 / 919 / 1827 for n = 500 / 1,000 / 2,000.

```python
import json, os, tempfile, zipfile, numpy as np
from scipy import stats
import reference_utils as R

def test_wilson_known():
    lo, hi = R.wilson_ci(231, 250); assert abs(lo - 0.884) < 0.001, lo
    assert abs(R.wilson_ci(0, 10)[0]) < 1e-12 and abs(R.wilson_ci(10, 10)[1] - 1.0) < 1e-12
def test_wilson_crosscheck():
    for n in (1, 7, 50, 250, 500, 1000, 2000):
        for k in sorted({0, 1, n // 3, n // 2, int(.92 * n), n - 1, n}):
            a, b = R.wilson_ci(k, n), R.wilson_ci_crosscheck(k, n)
            assert max(abs(a[0] - b[0]), abs(a[1] - b[1])) < 1e-9, (k, n, a, b)
def test_wilson_vs_scipy():
    r = stats.binomtest(231, 250).proportion_ci(0.95, method="wilson")
    a = R.wilson_ci(231, 250); assert abs(a[0] - r.low) < 1e-9 and abs(a[1] - r.high) < 1e-9
def test_clopper():
    r = stats.binomtest(231, 250).proportion_ci(0.95, method="exact")
    a = R.clopper_pearson_ci(231, 250); assert abs(a[0] - r.low) < 1e-9 and abs(a[1] - r.high) < 1e-9
def test_mcnemar():
    assert R.mcnemar_exact(0, 0) == 1.0
    assert abs(R.mcnemar_exact(10, 0) - 2 * 0.5 ** 10) < 1e-12
    assert abs(R.mcnemar_exact(5, 5) - 1.0) < 1e-12
    assert abs(R.mcnemar_exact(3, 9) - stats.binomtest(3, 12, 0.5).pvalue) < 1e-12
def test_min_successes_table():
    assert R.min_successes_for_lower_bound(500) == 464
    assert R.min_successes_for_lower_bound(1000) == 919
    assert R.min_successes_for_lower_bound(2000) == 1827
def test_bootstrap_deterministic():
    x = np.array([1] * 90 + [0] * 10); y = np.array([0] * 60 + [1] * 40)
    a = R.bootstrap_diff_ci(x, y, 500, seed=3); b = R.bootstrap_diff_ci(x, y, 500, seed=3)
    assert a == b and a[1] <= a[0] <= a[2]
def test_distinct2_and_agree():
    assert R.distinct2("a b c d") == 1.0
    assert R.distinct2("a a a a a a") < 0.5          # degenerate repetition
    assert R.distinct2("a") == 0.0 and R.distinct2([]) == 0.0
    assert R.agree(1, 0.9) == 1 and R.agree(1, 0.49) == 0 and R.agree(0, 1.0) == 0
    assert R.agree(1, R.distinct2("x x x x x x x x")) == 0   # label fires but repetition
def test_init_signature():
    rng = np.random.default_rng(0)
    fan = 512; b = 1 / np.sqrt(fan)
    w0 = rng.uniform(-b, b, (8192, fan))
    assert R.init_signature_test(w0)["looks_like_init"] is True
    assert abs(R.init_signature_test(w0)["excess_kurtosis"] + 1.2) < 0.05
    w1 = w0 + rng.normal(0, 0.05, w0.shape)           # trained-like: moved well beyond the bound
    assert R.init_signature_test(w1)["looks_like_init"] is False
    assert R.init_signature_test(w1, init_w=w0)["looks_like_init"] is False
    assert R.init_signature_test(w0 * 1.0, init_w=w0)["looks_like_init"] is True
    assert R.init_signature_test(np.zeros(64))["looks_like_init"] is True
def test_weight_diff():
    a = {"x": np.ones(3), "y": np.zeros(2)}; b = {"x": np.ones(3), "y": np.array([0., 1e-3])}
    r = R.weight_diff_report(a, b); assert r["changed"] == ["y"] and r["tensors"]["y"]["max_abs_diff"] == 1e-3
def test_hashing():
    assert R.sha256_bytes(b"abc") == "ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad"
def test_package_roundtrip():
    with tempfile.TemporaryDirectory() as t:
        run = os.path.join(t, "work", "rid"); os.makedirs(run + "/sub")
        open(run + "/a.txt", "w").write("hello"); open(run + "/sub/b.txt", "w").write("world")
        drive = os.path.join(t, "drive"); os.makedirs(t + "/tmp")
        res = R.package_and_copy(run, "rid", drive, tmp_dir=t + "/tmp")
        assert res["sha_match"] and res["testzip_ok"] and res["zip_matches_manifest"] and res["status"] == "OK"
        R.write_drive_verification(res, drive, "rid")
        assert open(drive + "/rid_SHA256.txt").read().split()[1] == "rid.zip"
        res2 = R.package_and_copy(run, "rid", drive, tmp_dir=t + "/tmp")   # existing zip is renamed, not overwritten
        assert any("_prev_" in f for f in os.listdir(drive))
        assert res2["local_sha256"] == res["local_sha256"]                  # deterministic zip
```

## Appendix C — Colab cell 1 (bootstrap)

Run this first, then paste the rest of the work order to the agent.

```python
from google.colab import drive
drive.mount('/content/drive')
import os, subprocess, sys
os.makedirs('/content/drive/MyDrive/EQUYLAPTA10_1/heavy_artifacts', exist_ok=True)
os.makedirs('/content/drive/MyDrive/EQUYLAPTA10_1/_checkpoints', exist_ok=True)
os.makedirs('/content/drive/MyDrive/EQUYLAPTA10_1/_live_mirror', exist_ok=True)
os.makedirs('/content/equylapta10_1_work/equylapta10_1/code/tests', exist_ok=True)
subprocess.run([sys.executable, '-m', 'pip', 'install', '-q', 'transformers', 'datasets', 'safetensors', 'scipy', 'scikit-learn', 'accelerate'], check=True)
for z in ['equylapta10potato.zip', 'equylapta10potatoverification.zip', 'equylapta10reverificaion.zip']:
    p = '/content/drive/MyDrive/' + z
    print(z, 'FOUND' if os.path.exists(p) else 'NOT_FOUND')
```
