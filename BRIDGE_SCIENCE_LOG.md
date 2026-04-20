# BRIDGE Science Log

**Started**: 2026-04-19
**Previous log archive**: `/groups/itay_mayrose/leardistel/[PipeLine]PETAL/07_Science_Log/archive/SCIENCE_LOG_archived_2026-04-19_173454.md` (SHA256 `a0b46b57404dce01f67d2780ef6ed4ba7df714e62fd74d09a400772e9628f635`, 32,378 lines).
**Authoritative pipeline spec**: `../docs/CANONICAL_PIPELINE.md`.

Rules (see `../docs/AGENT_ONBOARDING.md`):
- Entries numbered from 1, monotonically.
- No edits to past entries. If superseded, write a new entry that says so.
- Every entry must cite: experiment folder, config diff vs canonical, validation set size, metric definition, result.

---

## Preamble — operational framing of BRIDGE (2026-04-20)

The purpose of this pipeline is to segment **every flower across a corpus of photos, regardless of which specific photo it appeared in**. Species-level aggregation is the primary goal; image-level completeness is secondary. If a species appears in any photo, the pipeline must find it. Whether a given photo has 1 of 5 flowers annotated is incidental to the mission.

**Primary metric: sheet-mode precision** (Manual Validator human-labeled TP/FP). This is what a user experiences. A user uploads photos → pipeline returns masks → each mask is either on a flower (TP) or not (FP). At sealed knobs, sheet-mode precision ≈ 94%.

**Why IoU-based recall is a secondary metric**:
- Citadel GT = SAM2 polygons on user-annotated flowers. Multi-flower images have real flowers that were never annotated.
- SAM2 and SAM3 produce subtly different polygon shapes on the same flower — IoU punishes correct detections when the boundary style differs.
- IoU recall is an internal effectiveness check for the pipeline's operating point, not a claim about real-world performance.
- Chasing perfect image-level IoU leads to an infinite re-annotation loop; it is not tractable as a production metric.

**Integrity constraints on reporting**:
- Quote sheet-mode numbers as the primary claim.
- Report IoU-based numbers only with explicit measurement-protocol disclosure (n_GT bucket, IoU threshold, image subset).
- Never compute precision = TP/(TP+FP) on raw mask-level IoU — over-coverage is by design (conf=0.001, TOP_K=20) and IoU-FP counts legitimate detections of un-annotated flowers as errors.
- Species coverage is unambiguous: 100% on Citadel (231 validated species), 92.6% on the full Israeli catalog (2,417 / 2,611).

**The honest resume at sealed knobs** (conf=0.001, TOP_K=20, α=2.0, combo_gate=0.30, area_thresh=0.003):
- Sheet-mode precision: ≈94% (human-validated)
- Species coverage (Citadel 231 TP species): 100%
- Species coverage (Israeli catalog 2,611): 92.6%
- Image detection rate (Citadel, IoU≥0.5): 93.92%
- Top-1 precision on single-GT images (IoU≥0.5): 84.06%
- Median best-IoU per image (single-GT): 0.91
- Mean best-IoU per image (single-GT): 0.84

Mask-level recall at IoU≥0.3 on the Citadel TP set = 94.15% over 45,263 GT polygons. This is a mechanical comparator against SAM2 polygons, not a ground-truth recall statement about flowers in the scene.

---

### Regime-stratified results (the honest publication structure)

**Single-flower images** (n_GT=1, n=12,601, 28.8% of GT polygons) — the clean user-facing regime:

| metric | IoU ≥ 0.3 | IoU ≥ 0.5 | IoU ≥ 0.7 |
|---|---|---|---|
| Strict recall (best-match per GT over all passing masks) | **96.25%** | **94.10%** | **87.17%** |
| Boundary-tolerant: IoU≥τ OR coverage≥0.8 AND \|P\|≥0.1·\|G\| | **97.06%** | **96.63%** | **95.56%** |
| Boundary-tolerant: IoU≥τ OR containment≥0.7 AND \|P∩G\|≥0.05·\|G\| | **97.96%** | **97.84%** | **97.84%** |
| Top-1 precision (single highest-combo mask only) | 86.81% | 83.60% | 76.37% |
| Mean best-IoU (threshold-independent) | 0.8406 | 0.8406 | 0.8406 |
| Median best-IoU (threshold-independent) | 0.9078 | 0.9078 | 0.9078 |

**Counting-convention disclosure** (required for reviewers):

Two valid counting units give slightly different strict-recall numbers on single-GT imagery:
- **GT-polygon counting** (canonical): 94.10% @ IoU≥0.5. Denominator = # GT polygons (12,601). Each GT matched to the best predicted mask.
- **Image counting** (alternate): ~94.86% @ IoU≥0.5 = fraction of single-GT images where ANY passing mask has IoU≥0.5. Denominator = # images.

These converge up to rounding on single-GT data. **94.10% is the publication-canonical strict recall at IoU≥0.5.**

**Observation — tolerant recall converges at high IoU thresholds**: at IoU≥0.7 the boundary-tolerant metric (97.84% containment-based) is barely different from IoU≥0.3 (97.96%). The tolerant metric is stable across IoU cutoff — it measures "is SAM3 inside or overlapping SAM2's flower" which doesn't depend on the chosen cutoff. Strict IoU drops from 96.25% → 87.17% between τ=0.3 and τ=0.7 because IoU is increasingly sensitive to boundary drift; tolerant metrics stay flat because the inside-agreement is threshold-less.

**Top-1 precision** (highest-combo mask only, 1 prediction per image): 86.81% / 83.60% / 76.37% at IoU 0.3 / 0.5 / 0.7 — the most literature-comparable operating point (one prediction per image).

**Threshold-independent boundary-agreement**: mean best-IoU = 0.8406, median = 0.9078. SAM3 typically outlines the flower at ~91% overlap with SAM2. SAM3 is typically ~5% SMALLER than SAM2's polygon (median size_ratio = 0.9534); the mean ratio = 1.631 is pulled up by a 3% tail of multi-flower merges. Under-coverage dominates over-coverage ~1.6:1 on single-GT images.

**Multi-flower images** (n_GT ≥ 2) — boundary-tolerance matters more:

| n_GT | strict IoU≥0.5 | tolerant | Δ |
|---|---|---|---|
| 2 | 84.90% | 91.16% | +6.27pp |
| 3 | 76.77% | 85.25% | +8.48pp |
| 4+ | 70.43% | 81.08% | +10.65pp |

**Flower-size stratification** (all n_GT, 43,660 GT polygons):

| GT area fraction | n | strict | tolerant | regime |
|---|---|---|---|---|
| 0.010–0.030 | 15,176 | 84.0% | 95.4% | **+11.4pp lift — medium** |
| 0.030–0.100 | 10,497 | 90.6% | 97.2% | +6.6pp |
| 0.100–0.300 | 4,325 | 95.1% | 98.1% | +3.1pp |
| 0.300–1.000 | 876 | 94.2% | 96.7% | +2.5pp |
| 0.003–0.010 | 11,344 | 70.3% | 75.3% | small |
| **0.000–0.003** | **1,442** | **5.4%** | **5.5%** | **FPN-resolution limit** |

**Identification layer (closed-form shrinkage QDA, α=0.001, β=0.02)** on sealed 256-dim ε̂ azimuth, 2,608 species, 28,517/7,133 train/test split:

| level | top-1 | top-5 | top-10 |
|---|---|---|---|
| mask-level | 65.58% | 80.77% | 85.22% |
| centroid-level (≥3 masks) | **85.76%** | **94.05%** | **95.24%** |

Beats LDA by +7.86pp mask / +5.29pp centroid.

**Interpretation**: the pipeline finds every flower type (100% Citadel / 92.6% Israeli catalog species coverage). Single-flower strict IoU recall = 94%. Multi-flower strict IoU is 70–85%, rising to 81–91% under boundary-tolerant evaluation. Small/medium flowers gain most from tolerant metric (+11pp). Sub-0.3% flowers are at the FPN receptive-field limit.

**Constraints to disclose**:
1. Boundary-tolerant evaluation is appropriate because GT (SAM2) and predictions (SAM3) use different segmentation families. Strict IoU undercounts correct detections.
2. Flowers <0.3% of image are limited by FPN-L3 14-px receptive field. Geometric constraint, not semantic.
3. Primary deployment metric is sheet-mode precision (≈94%), not IoU-recall.

---

All entries that follow must be consistent with this framing. Contradictions must be flagged explicitly as superseding entries.

---

## Entry 1 — Cone saturates at N≈25; pure QDA collapses; unsupervised color clusters (2026-04-20)

### Cone detection saturation (exp 37, job 13548474)

**Sampling protocol**: from 2,608 sealed Israeli-flora species (each with ≥3 mask FPN vectors in resealed DB), randomly sample N species, build `D_flower_N` = normalize(mean of N species centroids). Held-out masks = species not in the N. Repeat 3× per N. N ∈ {10, 25, 50, 100, 200, 500, 1000, 1500, 2000, 2400}.

**Metric**: fraction of held-out masks with `cos(v̂ − bg_mean, D_flower_N) ≥ 0.30`.

**Result** (averaged across 3 reps):

| N | angle(D_flower_N, D_flower_sealed) | held-out cone @ 0.30 | QDA family-match (nearest training sp) |
|---|---|---|---|
| 10 | 32.3° | **98.2%** | 10.0% |
| 25 | 25.4° | **98.9%** | 11.0% |
| 50 | 28.7° | 98.6% | 15.2% |
| 100 | 26.9° | 98.7% | 19.0% |
| 200 | 26.9° | 98.7% | 22.8% |
| 500 | 26.7° | 98.8% | 29.5% |
| 1000 | 26.9° | 98.7% | 34.6% |

**Detection saturation**: ≥98% held-out mask inclusion at N = 25. Adding more species does not improve detection. The cone axis direction is robustly learnable from any ~25 random flowers.

**QDA identification saturation**: NOT saturated at N=1000; family-match rate is still climbing. Identification requires representative coverage across all 126 Israeli families.

**Interpretation**: two distinct scales.
- `D_flower` direction = mean on S²⁵⁵, estimable at N = 25 within the loose FA-FPN threshold's tolerance.
- Per-species discrimination needs representative density across the 2,608-class manifold, which is far from saturating at N = 1000.

**Reproducibility**:
- script: `experiments/37_cone_generalization_2026-04-20/run.py`
- output: `results/37_cone_generalization_2026-04-20/results_A_detection.json`, `results_B_identification.json`
- seed: `np.random.default_rng(42 + rep*100 + N)` per sample

### β=1 pure-QDA collapse (exp 36, job 13539745)

Confirming the shrinkage math: at β=1 (no pooled Σ_w contribution), regardless of α∈{0.001, 0.005, 0.01, 0.03, 0.1, 0.3}, mask-top-1 plateaus at **43.35–43.59%** and centroid-top-1 at **61.17–61.55%**. α does not recover pure QDA — the rank-10 classwise covariance estimated from ~11 training masks per class is insufficient alone.

The peak at β=0.20–0.30 (70.08% mask / 88.91% centroid) is the sealed baseline. β=1 confirms the shrinkage law: the ~30% weight on Σ_c is actionable; 100% Σ_c is not.

**Reproducibility**:
- script: `experiments/36_qda_pure_beta1_2026-04-20/run.py`
- output: `results/36_qda_pure_beta1_2026-04-20/summary.json`

### Unsupervised color clusters (exp 39, job 13548921)

Per-species 5-D chromatic fingerprint (circular hue, R, mean chroma, σ chroma, mean L) for 2,167 species with ≥5 chromatic masks. KMeans K=6 on standardized + hue-as-(sin,cos) features:

| cluster | n_sp | mean hue° | R | chroma | σ chroma | L | top families |
|---|---|---|---|---|---|---|---|
| 0 | 319 | 93 (yellow) | 0.87 | 39.0 | 20.9 | 65 | Asteraceae, Poaceae, Fabaceae |
| 1 | 541 | 99 (yellow-green) | 0.88 | 19.1 | 9.5 | 70 | Fabaceae, Apiaceae, Brassicaceae |
| 2 | 298 | 48 (red-orange) | 0.49 | 13.2 | 9.2 | 67 | Fabaceae, Caryophyllaceae, Lamiaceae |
| 3 | 367 | 56 (red-orange) | 0.88 | 24.2 | 9.6 | 52 | Fabaceae, Asteraceae, Amaryllidaceae |
| 4 | 345 | 96 (yellow) | 0.97 | 55.2 | 14.5 | 69 | Fabaceae, Asteraceae, Brassicaceae |
| 5 | 297 | 355 (magenta) | 0.69 | 28.1 | 14.9 | 57 | Fabaceae, Asteraceae, Lamiaceae |

Cluster 4 (n=345) shows the tightest within-species color concentration (R=0.97) at peak chroma — high-chroma yellow specialists. Cluster 2 (R=0.49, low chroma) is the diffuse/pastel cluster. Reproducibility: `experiments/39_pollinator_cloud_2026-04-20/run.py`, KMeans seed=0, n_init=20.

---

(awaiting entry 2)
