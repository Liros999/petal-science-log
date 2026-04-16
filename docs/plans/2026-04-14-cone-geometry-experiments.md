# Cone Geometry Experiments Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Run five cross-domain geometry experiments on the Flower Cone structure in both SAM3 FPN (256-dim, 85 species) and BioCLIP visual (1024-dim, 1681 species) spaces, producing per-species risk scores, morphological clusters, sample complexity bounds, and a vMF detectability prior.

**Architecture:** Each experiment is a self-contained CPU Python script that reads from existing NPZ/JSON files, computes purely geometric quantities (no model inference), and saves results to JSON + NPZ. All scripts submitted via SLURM CPU jobs. No GPU required.

**Tech Stack:** Python 3, numpy, scipy, scikit-learn, matplotlib. All in `/groups/itay_mayrose_nosnap/leardistel/petal_env/bin/python`.

---

## Experiment Numbering

- **E286** — Information Theory: Channel Capacity of the Azimuth Space
- **E287** — Astronomy: vMF Stellar Population Fitting + Detectability Prior
- **E288** — Graph Theory: Spectral Clustering + Voronoi + MST on Azimuth Graph
- **E289** — Crystallography: Debye-Scherrer Sample Complexity per Species
- **E290** — Wild Card: Topological Data Analysis + Sphere Painting

Science Log entries: 283–290 (continuing from Entry 282).

---

## Shared Data Paths (read-only — NEVER write back to these)

```python
# SAM3 FPN space (85 species, 256-dim)
SAM3_NPZ   = "/scratch200/leardistel/petal_benchmark/results/exp33_sam3_fpn_geometry/f_SAM3.npz"
# keys: f_SAM3(85,256), D_flower_SAM3_n(256,), theta_sam3_deg(85,),
#       eps_hat_sam3(85,256), azim_mag_sam3(85,), elev_mag_sam3(85,), species(85,)

# BioCLIP visual space (1681 species, 1024-dim)
CONE_NPZ   = "/scratch200/leardistel/petal_benchmark/results/exp_E70_bioclip_text_2174/cone_geometry.npz"
# keys: f_vis_n(1681,1024), theta_deg(1681,), eps_hat(1681,1024),
#       D_flower_vis_n(1024,), n_masks(1681,), species(1681,)

# Per-mask CLS tokens (52049 masks, 1024-dim, 2174 species)
CLS_NPZ    = "/scratch200/leardistel/petal_benchmark/results/exp_E76c_cls_all_israel/cls_tokens.npz"
# keys: cls(52049,1024), combo_score(52049,), mask_idx(52049,),
#       photo_ids(52049,), species(52049,)

# K2P DNA distances (223 species)
K2P_NPZ    = "/scratch200/leardistel/petal_benchmark/results/exp_E06_k2p_distances/D_k2p.npz"
# keys: D(223,223), species(223,)

# E72 sphere geometry (already computed for BioCLIP — read for vMF params)
E72_SUMMARY = "/scratch200/leardistel/petal_benchmark/results/exp_E72_sphere_geometry/summary.json"
E72_VMF     = "/scratch200/leardistel/petal_benchmark/results/exp_E72_sphere_geometry/vMF.json"

# Output root
RESULTS = "/scratch200/leardistel/petal_benchmark/results/"
```

---

## Task 1: E286 — Information Theory: Channel Capacity

### What we're measuring

The azimuth hyperplane (⊥ D_flower) is a communication channel. Species ε̂[s] are codewords. The channel noise is intra-species per-mask CLS scatter. Channel capacity C bounds how many species can be reliably discriminated.

**Key formula:**
```
C = (1/2) log₂ det(I + SNR · Σ_signal / Σ_noise)
```

For the spherical case with ID=20.3, the effective capacity in the azimuth subspace is:
```
C_azim = (ID/2) · log₂(1 + σ²_between / σ²_within)
```

where σ²_between = variance of species centroids in azimuth, σ²_within = mean intra-species mask scatter.

We also compute the **Kent distribution ellipticity** for SAM3 space (already done for BioCLIP: ellipticity=0.091 → nearly circular). If SAM3 shows higher ellipticity → preferred azimuthal axes → potential structure to exploit.

**Files:**
- Create: `/scratch200/leardistel/petal_benchmark/experiments/exp_E286_channel_capacity.py`
- Output: `/scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity/`
  - `summary.json` — channel capacity numbers
  - `per_species_snr.json` — per-species SNR in FPN and BioCLIP
  - `kent_sam3.json` — SAM3 Kent ellipticity

**Step 1: Write the script**

```python
#!/usr/bin/env python3
"""
E286 — Information Theory: Channel Capacity of the Azimuth Space
Computes per-species SNR, channel capacity (bits), and Kent ellipticity
in both SAM3 FPN (256-dim, 85 species) and BioCLIP visual (1024-dim, 1681 species) spaces.

Reads from existing NPZ files. No model inference.
"""
import functools
print = functools.partial(print, flush=True)

import numpy as np
import json
import os
from pathlib import Path
from scipy.special import iv  # modified Bessel function

# ── paths ──────────────────────────────────────────────────────────────────
SAM3_NPZ = "/scratch200/leardistel/petal_benchmark/results/exp33_sam3_fpn_geometry/f_SAM3.npz"
CONE_NPZ = "/scratch200/leardistel/petal_benchmark/results/exp_E70_bioclip_text_2174/cone_geometry.npz"
CLS_NPZ  = "/scratch200/leardistel/petal_benchmark/results/exp_E76c_cls_all_israel/cls_tokens.npz"
OUT_DIR  = "/scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity"
Path(OUT_DIR).mkdir(parents=True, exist_ok=True)

SEED = 42
np.random.seed(SEED)
print(f"SEED={SEED}")


# ── helpers ────────────────────────────────────────────────────────────────
def normalize(x):
    n = np.linalg.norm(x, axis=-1, keepdims=True)
    assert np.all(n > 1e-8), "zero vector encountered in normalize"
    return x / n

def kent_ellipticity(eps_hat):
    """
    Estimate Kent distribution ellipticity in the top-5 principal subspace
    of the azimuth cloud. eps_hat: (N, D) unit vectors already ⊥ D_flower.
    Returns lambda1, lambda2, ellipticity = (lambda1 - lambda2) / lambda1.
    """
    # scatter matrix in azimuth space (use top-5 PCA directions)
    C = eps_hat.T @ eps_hat / len(eps_hat)
    eigvals = np.linalg.eigvalsh(C)[::-1]  # descending
    l1, l2 = eigvals[0], eigvals[1]
    ell = (l1 - l2) / l1 if l1 > 0 else 0.0
    return float(l1), float(l2), float(ell)

def compute_channel_capacity(eps_hat_centroids, cls_tokens, species_labels, intrinsic_dim=20.3):
    """
    eps_hat_centroids: (N_species, D) — per-species azimuth directions (unit vectors)
    cls_tokens: (N_masks, D) — per-mask CLS tokens (unit vectors)
    species_labels: (N_masks,) str — species name per mask
    intrinsic_dim: effective dimension of azimuth manifold

    Returns per-species SNR and overall channel capacity.
    """
    unique_sp = np.unique(species_labels)
    species_set = set(np.unique(eps_hat_centroids[:, 0].view()))  # placeholder — overridden below

    per_species = []
    sigma_within_list = []

    for sp in unique_sp:
        mask_idx = np.where(species_labels == sp)[0]
        if len(mask_idx) < 3:
            continue
        masks = cls_tokens[mask_idx]   # (n_masks, D)
        masks_n = normalize(masks)

        # project onto azimuth plane — subtract elevation component
        # we don't have D_flower here; use mean direction as proxy
        mean_dir = normalize(masks_n.mean(axis=0, keepdims=True))[0]
        elev = masks_n @ mean_dir  # (n_masks,)
        azim_residual = masks_n - elev[:, None] * mean_dir[None, :]  # (n_masks, D)
        norms = np.linalg.norm(azim_residual, axis=1)
        valid = norms > 1e-6
        if valid.sum() < 3:
            continue
        azim_n = azim_residual[valid] / norms[valid, None]

        # within-species azimuth scatter = mean pairwise angle
        # approximate: std of dot products with mean direction
        mean_azim = normalize(azim_n.mean(axis=0, keepdims=True))[0]
        dots = azim_n @ mean_azim
        sigma_within = float(np.arccos(np.clip(dots, -1, 1)).std())  # radians
        sigma_within_list.append(sigma_within)
        per_species.append({
            "species": sp,
            "n_masks": int(len(mask_idx)),
            "sigma_within_rad": sigma_within,
            "sigma_within_deg": float(np.degrees(sigma_within))
        })

    sigma_within_mean = float(np.mean(sigma_within_list))
    sigma_within_deg  = float(np.degrees(sigma_within_mean))

    # between-species variance in azimuth: mean pairwise angle between centroids
    pairwise = eps_hat_centroids @ eps_hat_centroids.T
    np.fill_diagonal(pairwise, np.nan)
    sigma_between_rad = float(np.nanmean(np.arccos(np.clip(pairwise, -1, 1))))

    # SNR: ratio of between-species to within-species angular variance
    snr = (sigma_between_rad / sigma_within_mean) ** 2

    # Channel capacity (bits): MIMO formula on ID-dimensional azimuth subspace
    # C = (ID/2) * log2(1 + SNR)
    capacity_bits = (intrinsic_dim / 2.0) * np.log2(1.0 + snr)

    # Max resolvable species: 2^C
    max_species = 2 ** capacity_bits

    return {
        "sigma_within_mean_rad": sigma_within_mean,
        "sigma_within_mean_deg": sigma_within_deg,
        "sigma_between_mean_rad": sigma_between_rad,
        "sigma_between_mean_deg": float(np.degrees(sigma_between_rad)),
        "snr": snr,
        "snr_db": float(10 * np.log10(snr)),
        "intrinsic_dim": intrinsic_dim,
        "capacity_bits": float(capacity_bits),
        "max_resolvable_species": float(max_species),
        "n_species_used": len(per_species)
    }, per_species


# ── load SAM3 data ─────────────────────────────────────────────────────────
print("Loading SAM3 FPN data...")
d = np.load(SAM3_NPZ, allow_pickle=True)
f_SAM3       = d["f_SAM3"]          # (85, 256)
D_flower_sam3 = d["D_flower_SAM3_n"] # (256,)
theta_sam3   = d["theta_sam3_deg"]   # (85,)
eps_sam3     = d["eps_hat_sam3"]     # (85, 256) — already unit vectors ⊥ D_flower
species_sam3 = d["species"]          # (85,)
print(f"  SAM3: {len(species_sam3)} species, {f_SAM3.shape[1]}-dim")

# Kent ellipticity for SAM3
l1_sam3, l2_sam3, ell_sam3 = kent_ellipticity(eps_sam3)
kent_sam3 = {
    "lambda1": l1_sam3, "lambda2": l2_sam3, "ellipticity": ell_sam3,
    "lambda1_lambda2_ratio": l1_sam3 / l2_sam3 if l2_sam3 > 0 else float("inf"),
    "interpretation": f"Ratio={l1_sam3/l2_sam3:.4f}. <1.1=circle, >1.5=ellipse, >3=strong axis"
}
print(f"  SAM3 Kent: ellipticity={ell_sam3:.4f}, λ1/λ2={l1_sam3/l2_sam3:.4f}")
with open(f"{OUT_DIR}/kent_sam3.json", "w") as f:
    json.dump(kent_sam3, f, indent=2)

# Per-species SNR in SAM3 (approximation: use pairwise eps angles as between-species,
# and theta std as proxy for within-species scatter since we lack per-mask SAM3 FPN vectors)
pairwise_sam3 = eps_sam3 @ eps_sam3.T
np.fill_diagonal(pairwise_sam3, np.nan)
sigma_between_sam3 = float(np.nanmean(np.arccos(np.clip(pairwise_sam3, -1, 1))))
# within-species proxy: use inter-mask variation approximated by theta std
# (true within-species SAM3 scatter requires re-running SAM3 per mask — not available here)
# Use conservative estimate: sigma_within ≈ theta_std converted to radians
sigma_within_sam3_proxy = float(np.radians(theta_sam3.std()))
snr_sam3 = (sigma_between_sam3 / sigma_within_sam3_proxy) ** 2
capacity_sam3 = (18.0 / 2.0) * np.log2(1.0 + snr_sam3)  # ID≈18 for SAM3 (scaled from BioCLIP)

sam3_channel = {
    "space": "SAM3 FPN 256-dim",
    "n_species": int(len(species_sam3)),
    "sigma_between_mean_rad": sigma_between_sam3,
    "sigma_between_mean_deg": float(np.degrees(sigma_between_sam3)),
    "sigma_within_proxy_rad": sigma_within_sam3_proxy,
    "sigma_within_proxy_deg": float(np.degrees(sigma_within_sam3_proxy)),
    "note": "within-species is proxy (theta_std) — no per-mask SAM3 FPN vectors cached",
    "snr": snr_sam3,
    "snr_db": float(10 * np.log10(snr_sam3)),
    "intrinsic_dim_assumed": 18.0,
    "capacity_bits": float(capacity_sam3),
    "max_resolvable_species": float(2 ** capacity_sam3)
}
print(f"  SAM3 channel capacity: {capacity_sam3:.1f} bits → {2**capacity_sam3:.0f} max species")
with open(f"{OUT_DIR}/channel_sam3.json", "w") as f:
    json.dump(sam3_channel, f, indent=2)


# ── load BioCLIP data ──────────────────────────────────────────────────────
print("Loading BioCLIP visual cone data...")
c = np.load(CONE_NPZ, allow_pickle=True)
f_vis_n      = c["f_vis_n"]        # (1681, 1024)
theta_bioclip = c["theta_deg"]     # (1681,)
eps_bioclip  = c["eps_hat"]        # (1681, 1024)
D_flower_vis  = c["D_flower_vis_n"] # (1024,)
n_masks_arr  = c["n_masks"]        # (1681,)
species_bioclip = c["species"]     # (1681,)
print(f"  BioCLIP: {len(species_bioclip)} species, {f_vis_n.shape[1]}-dim")

print("Loading per-mask CLS tokens...")
cls_data = np.load(CLS_NPZ, allow_pickle=True)
cls_tokens   = cls_data["cls"]          # (52049, 1024)
cls_species  = cls_data["species"]      # (52049,) str
print(f"  CLS tokens: {len(cls_tokens)} masks across {len(np.unique(cls_species))} species")

# Compute channel capacity for BioCLIP using per-mask CLS tokens
# Only use species that appear in both BioCLIP cone AND CLS token set
bioclip_sp_set = set(species_bioclip.tolist())
cls_sp_set     = set(cls_species.tolist())
overlap_sp     = sorted(bioclip_sp_set & cls_sp_set)
print(f"  Species overlap (BioCLIP cone ∩ CLS tokens): {len(overlap_sp)}")

eps_overlap  = np.array([eps_bioclip[list(species_bioclip).index(s)] for s in overlap_sp])
# filter CLS tokens to overlap species
overlap_mask = np.isin(cls_species, list(overlap_sp))
cls_overlap  = cls_tokens[overlap_mask]          # unit-normalize
cls_overlap_n = normalize(cls_overlap)
cls_sp_overlap = cls_species[overlap_mask]

bioclip_channel, per_sp_bioclip = compute_channel_capacity(
    eps_overlap, cls_overlap_n, cls_sp_overlap, intrinsic_dim=20.3
)
bioclip_channel["space"] = "BioCLIP visual 1024-dim"
bioclip_channel["n_species_in_overlap"] = len(overlap_sp)

print(f"  BioCLIP channel capacity: {bioclip_channel['capacity_bits']:.1f} bits "
      f"→ {bioclip_channel['max_resolvable_species']:.0f} max species")
print(f"  SNR={bioclip_channel['snr']:.2f} ({bioclip_channel['snr_db']:.1f} dB)")

with open(f"{OUT_DIR}/channel_bioclip.json", "w") as f:
    json.dump(bioclip_channel, f, indent=2)
with open(f"{OUT_DIR}/per_species_snr_bioclip.json", "w") as f:
    json.dump(per_sp_bioclip, f, indent=2)

# ── combined summary ───────────────────────────────────────────────────────
summary = {
    "exp": "E286_channel_capacity",
    "date": "2026-04-14",
    "seed": SEED,
    "SAM3_FPN": sam3_channel,
    "BioCLIP_visual": bioclip_channel,
    "kent_sam3": kent_sam3,
    "kent_bioclip_from_E72": {
        "ellipticity": 0.0906, "lambda1": 0.0661, "lambda2": 0.0551,
        "ratio": 0.0661/0.0551
    },
    "interpretation": (
        "Channel capacity gives the maximum number of species whose azimuth fingerprint "
        "is reliably discriminable given per-mask within-species noise. "
        "If max_resolvable_species < actual species count, some species are "
        "geometrically indistinguishable — not a detector failure but a capacity limit."
    )
}
with open(f"{OUT_DIR}/summary.json", "w") as f:
    json.dump(summary, f, indent=2)
print("E286 DONE. Results in", OUT_DIR)
```

**Step 2: Write SLURM submit script**

```bash
#!/bin/bash
#SBATCH --job-name=E286_channel
#SBATCH --partition=power-general-public-pool
#SBATCH --qos=public
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=00:30:00
#SBATCH --output=/scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity/slurm_%j.out
#SBATCH --error=/scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity/slurm_%j.err

mkdir -p /scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity
/groups/itay_mayrose_nosnap/leardistel/petal_env/bin/python \
    /scratch200/leardistel/petal_benchmark/experiments/exp_E286_channel_capacity.py
```

**Step 3: Submit**
```bash
mkdir -p /scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity
sbatch /scratch200/leardistel/petal_benchmark/experiments/submit_E286.sh
```

**Step 4: Validate output**
```
cat /scratch200/leardistel/petal_benchmark/results/exp_E286_channel_capacity/summary.json
```
Expected: capacity_bits > 0, max_resolvable_species computable, kent ellipticity ratio for SAM3.

---

## Task 2: E287 — Astronomy: vMF Stellar Population + Detectability Prior

### What we're measuring

Fit a mixture of vMF distributions — one per species — to the per-mask CLS token cloud. This gives:
1. Per-species κ_s (concentration = how "pure" a species' flower appearance is)
2. Per-species membership probability for any new mask observation
3. A detectability prior: P(θ_s > θ_boundary) from the vMF marginal on elevation, computed WITHOUT running the detector

We already know from E72: κ_full=1835 (population-level, BioCLIP). Now we want κ_s per species from CLS tokens, and κ_SAM3 from the 85-species distribution.

**Key formula — vMF concentration from mean resultant length R̄:**
```
R̄ = (1/n) ||Σ x_i||    (mean resultant length of unit vectors)
d = embedding dimension

κ̂ = R̄(d - R̄²) / (1 - R̄²)    (approx for large d)
```

**Detectability prior:**
```
P(θ_s > θ_max | κ_population) = 1 - I_{d/2}(κ) / I_{d/2-1}(κ) × [CDF term]
```
(computed numerically for each species using their predicted θ from W_0)

**Files:**
- Create: `/scratch200/leardistel/petal_benchmark/experiments/exp_E287_vmf_stellar.py`
- Output: `/scratch200/leardistel/petal_benchmark/results/exp_E287_vmf_stellar/`
  - `summary.json`
  - `per_species_kappa.json` — per-species κ_s and R̄_s from CLS tokens
  - `detectability_prior.json` — P(detectable) per species from vMF prior
  - `sam3_vmf.json` — vMF fit on the 85-species SAM3 distribution

**Step 1: Write the script**

```python
#!/usr/bin/env python3
"""
E287 — Astronomy: vMF Stellar Population Fitting + Detectability Prior
Fits per-species vMF concentration κ_s from per-mask CLS tokens.
Computes detectability prior P(θ_s > θ_max) from vMF marginal.
Runs in both SAM3 FPN (85 sp) and BioCLIP visual (1681 sp) spaces.
"""
import functools
print = functools.partial(print, flush=True)

import numpy as np
import json
from pathlib import Path
from scipy.special import iv as bessel_i

SAM3_NPZ = "/scratch200/leardistel/petal_benchmark/results/exp33_sam3_fpn_geometry/f_SAM3.npz"
CONE_NPZ = "/scratch200/leardistel/petal_benchmark/results/exp_E70_bioclip_text_2174/cone_geometry.npz"
CLS_NPZ  = "/scratch200/leardistel/petal_benchmark/results/exp_E76c_cls_all_israel/cls_tokens.npz"
E72_VMF  = "/scratch200/leardistel/petal_benchmark/results/exp_E72_sphere_geometry/vMF.json"
OUT_DIR  = "/scratch200/leardistel/petal_benchmark/results/exp_E287_vmf_stellar"
Path(OUT_DIR).mkdir(parents=True, exist_ok=True)

SEED = 42
np.random.seed(SEED)
print(f"SEED={SEED}")

def normalize(x):
    n = np.linalg.norm(x, axis=-1, keepdims=True)
    assert np.all(n > 1e-8), "zero vector in normalize"
    return x / n

def vmf_kappa_estimate(unit_vectors):
    """
    Estimate vMF concentration κ from unit vectors.
    Uses approximation: κ ≈ R̄(d - R̄²) / (1 - R̄²)
    Valid for d >> 1 (our d=1024 or d=256 — both qualify).
    unit_vectors: (N, d) already unit-normalized.
    Returns: kappa (float), R_bar (float)
    """
    N, d = unit_vectors.shape
    # mean resultant vector
    mean_vec = unit_vectors.mean(axis=0)
    R_bar = float(np.linalg.norm(mean_vec))
    # approximate MLE estimator (Banerjee et al. 2005)
    kappa = R_bar * (d - R_bar**2) / (1.0 - R_bar**2 + 1e-12)
    return float(kappa), R_bar

def vmf_detectability_prior(theta_deg_array, kappa_population, d, theta_boundary_deg):
    """
    For each species with predicted elevation θ_s, compute P(detectable).
    P(detectable | θ_s) = P(cos(θ_mask) > cos(θ_boundary)) under vMF(κ_s, f̂[s]).
    Approximation: treat as 1D Gaussian with σ = 1/sqrt(κ_s) (large κ, large d limit).
    Returns array of P(detectable) per species.
    """
    from scipy.stats import norm
    theta_rad = np.radians(theta_deg_array)
    theta_boundary_rad = np.radians(theta_boundary_deg)
    # angular std from kappa: σ_angle ≈ 1/sqrt(κ) in high-d vMF
    sigma_angle = 1.0 / np.sqrt(kappa_population + 1e-12)
    # margin: how many std devs above threshold
    margin = (theta_boundary_rad - theta_rad) / sigma_angle
    # P(detectable) = P(θ_mask < θ_boundary) = Φ(margin)
    p_detectable = norm.cdf(margin)
    return p_detectable

# ── SAM3 vMF fit ───────────────────────────────────────────────────────────
print("SAM3 FPN vMF fit...")
d_sam3 = np.load(SAM3_NPZ, allow_pickle=True)
f_SAM3      = d_sam3["f_SAM3"]          # (85, 256), NOT unit normalized
D_flower_n  = d_sam3["D_flower_SAM3_n"] # (256,)
theta_sam3  = d_sam3["theta_sam3_deg"]  # (85,)
eps_sam3    = d_sam3["eps_hat_sam3"]    # (85, 256) unit vectors ⊥ D_flower
species_sam3 = d_sam3["species"]        # (85,)

# Normalize f_SAM3 to unit sphere for vMF fit
f_sam3_n = normalize(f_SAM3)
kappa_sam3_pop, Rbar_sam3 = vmf_kappa_estimate(f_sam3_n)

# Detectability prior for 85 species — theta_boundary = 59.2° (E70 max from BioCLIP space)
# Use SAM3 empirical boundary: 42.5° is max observed; set threshold at 50° (conservative)
theta_boundary_sam3 = 50.0
p_det_sam3 = vmf_detectability_prior(theta_sam3, kappa_sam3_pop, 256, theta_boundary_sam3)

sam3_vmf = {
    "space": "SAM3 FPN 256-dim",
    "n_species": 85,
    "kappa_population": kappa_sam3_pop,
    "R_bar": Rbar_sam3,
    "mean_theta_deg": float(theta_sam3.mean()),
    "theta_boundary_deg": theta_boundary_sam3,
    "n_at_risk": int((p_det_sam3 < 0.95).sum()),
    "n_high_risk": int((p_det_sam3 < 0.80).sum()),
    "per_species": [
        {
            "species": str(species_sam3[i]),
            "theta_deg": float(theta_sam3[i]),
            "p_detectable": float(p_det_sam3[i])
        }
        for i in np.argsort(p_det_sam3)  # sorted low to high
    ]
}
print(f"  SAM3 κ_pop={kappa_sam3_pop:.1f}, R̄={Rbar_sam3:.4f}")
print(f"  At-risk species (P<0.95): {sam3_vmf['n_at_risk']}/85")
with open(f"{OUT_DIR}/sam3_vmf.json", "w") as f:
    json.dump(sam3_vmf, f, indent=2)

# ── BioCLIP vMF: population-level ─────────────────────────────────────────
print("BioCLIP visual vMF fit...")
c = np.load(CONE_NPZ, allow_pickle=True)
f_vis_n      = c["f_vis_n"]          # (1681, 1024) already unit normalized
theta_bioclip = c["theta_deg"]        # (1681,)
eps_bioclip  = c["eps_hat"]           # (1681, 1024)
D_flower_vis = c["D_flower_vis_n"]    # (1024,)
n_masks_arr  = c["n_masks"]           # (1681,)
species_bioclip = c["species"]        # (1681,)

kappa_bioclip_pop, Rbar_bioclip = vmf_kappa_estimate(f_vis_n)
print(f"  BioCLIP κ_pop={kappa_bioclip_pop:.1f}, R̄={Rbar_bioclip:.4f}")
# Cross-check against E72: should be ≈1835
with open(E72_VMF) as f:
    e72_vmf = json.load(f)
print(f"  E72 reference κ={e72_vmf['kappa_full']:.1f} — difference: "
      f"{abs(kappa_bioclip_pop - e72_vmf['kappa_full']):.1f}")

# Per-species vMF fit from CLS tokens
print("Loading CLS tokens for per-species vMF fit...")
cls_data = np.load(CLS_NPZ, allow_pickle=True)
cls_tokens  = cls_data["cls"]     # (52049, 1024)
cls_species = cls_data["species"] # (52049,) str

cls_tokens_n = normalize(cls_tokens)
species_bioclip_set = set(species_bioclip.tolist())

per_species_kappa = []
for sp in np.unique(cls_species):
    if sp not in species_bioclip_set:
        continue
    idx = np.where(cls_species == sp)[0]
    if len(idx) < 5:
        continue
    masks_n = cls_tokens_n[idx]
    kappa_s, Rbar_s = vmf_kappa_estimate(masks_n)
    # project masks onto azimuth plane (subtract elevation)
    elev = masks_n @ D_flower_vis  # (n_masks,)
    theta_masks_deg = float(np.degrees(np.arccos(np.clip(elev, -1, 1))).mean())
    per_species_kappa.append({
        "species": sp,
        "n_masks": int(len(idx)),
        "kappa_s": float(kappa_s),
        "R_bar_s": float(Rbar_s),
        "mean_theta_masks_deg": theta_masks_deg,
        "interpretation": "high kappa_s = tight cluster = reliable fingerprint"
    })

per_species_kappa.sort(key=lambda x: x["kappa_s"])
print(f"  Per-species vMF: {len(per_species_kappa)} species with ≥5 masks")
print(f"  Median κ_s: {np.median([x['kappa_s'] for x in per_species_kappa]):.1f}")
print(f"  Min κ_s: {per_species_kappa[0]['kappa_s']:.1f} ({per_species_kappa[0]['species']})")
print(f"  Max κ_s: {per_species_kappa[-1]['kappa_s']:.1f} ({per_species_kappa[-1]['species']})")
with open(f"{OUT_DIR}/per_species_kappa.json", "w") as f:
    json.dump(per_species_kappa, f, indent=2)

# Detectability prior for all 1681 species
theta_boundary_bioclip = 59.2  # empirical max from E70
p_det_bioclip = vmf_detectability_prior(theta_bioclip, kappa_bioclip_pop, 1024, theta_boundary_bioclip)
n_at_risk = int((p_det_bioclip < 0.95).sum())
n_structurally_undetectable = int((p_det_bioclip < 0.50).sum())

detectability = {
    "space": "BioCLIP visual 1024-dim",
    "n_species": 1681,
    "kappa_population": kappa_bioclip_pop,
    "theta_boundary_deg": theta_boundary_bioclip,
    "n_at_risk_p95": n_at_risk,
    "n_structurally_undetectable_p50": n_structurally_undetectable,
    "fraction_at_risk": n_at_risk / 1681,
    "top10_highest_risk": [
        {"species": str(species_bioclip[i]), "theta_deg": float(theta_bioclip[i]),
         "p_detectable": float(p_det_bioclip[i])}
        for i in np.argsort(p_det_bioclip)[:10]
    ]
}
with open(f"{OUT_DIR}/detectability_prior.json", "w") as f:
    json.dump(detectability, f, indent=2)
print(f"  At-risk (P<0.95): {n_at_risk}/1681 = {n_at_risk/1681*100:.1f}%")
print(f"  Structurally undetectable (P<0.50): {n_structurally_undetectable}/1681")

summary = {
    "exp": "E287_vmf_stellar",
    "date": "2026-04-14",
    "seed": SEED,
    "SAM3": sam3_vmf,
    "BioCLIP_population_kappa": kappa_bioclip_pop,
    "BioCLIP_R_bar": Rbar_bioclip,
    "BioCLIP_detectability": detectability,
    "n_per_species_kappa_fitted": len(per_species_kappa)
}
with open(f"{OUT_DIR}/summary.json", "w") as f:
    json.dump(summary, f, indent=2)
print("E287 DONE.")
```

**Step 2: SLURM script** (same structure as E286, substitute job name and paths)

**Step 3: Submit and validate**
Key output to check: `per_species_kappa.json` sorted by κ_s — lowest κ_s species are the "disordered grains" (bad morphological consistency). Compare with high-θ species list from E70 — do they correlate?

---

## Task 3: E288 — Graph Theory: Spectral Clustering + Voronoi + MST

### What we're measuring

Three graph algorithms on the azimuth sphere, in both SAM3 (85 sp) and BioCLIP (1681 sp):

**A. Spectral clustering:** Adjacency A[s,t] = max(0, ε̂_s·ε̂_t − threshold). Laplacian eigenvectors → morphological clusters independent of taxonomy. Test: do clusters cut across families? (family_cities_ratio already=0.96 suggests yes)

**B. Voronoi cell volumes:** For each species ε̂[s], the Voronoi cell volume ∝ probability a random mask is correctly attributed. Hard species = small Voronoi cells. Computable via nearest-neighbor assignment on random sphere samples.

**C. MST on azimuth graph:** Minimum spanning tree on geodesic distances between ε̂[s]. Cross-correlate MST edge lengths with K2P DNA distances (223 species overlap) to find convergent/divergent evolution events.

**Files:**
- Create: `/scratch200/leardistel/petal_benchmark/experiments/exp_E288_azimuth_graph.py`
- Output: `/scratch200/leardistel/petal_benchmark/results/exp_E288_azimuth_graph/`
  - `summary.json`
  - `spectral_clusters_sam3.json` — cluster assignments + family cross-cutting score
  - `spectral_clusters_bioclip.json`
  - `voronoi_sam3.json` — per-species Voronoi cell volume proxy
  - `voronoi_bioclip.json`
  - `mst_dna_correlation.json` — MST vs K2P Mantel test result

**Step 1: Write the script**

```python
#!/usr/bin/env python3
"""
E288 — Graph Theory: Spectral Clustering + Voronoi + MST on Azimuth Graph
Runs three graph algorithms on ε̂[s] azimuth unit vectors.
No model inference. CPU only. ~10-20 min on 85/1681 species.
"""
import functools
print = functools.partial(print, flush=True)

import numpy as np
import json
from pathlib import Path
from scipy.sparse.linalg import eigsh
from scipy.sparse import csr_matrix
from scipy.spatial.distance import squareform
from scipy.cluster.hierarchy import dendrogram

SAM3_NPZ = "/scratch200/leardistel/petal_benchmark/results/exp33_sam3_fpn_geometry/f_SAM3.npz"
CONE_NPZ = "/scratch200/leardistel/petal_benchmark/results/exp_E70_bioclip_text_2174/cone_geometry.npz"
K2P_NPZ  = "/scratch200/leardistel/petal_benchmark/results/exp_E06_k2p_distances/D_k2p.npz"
OUT_DIR  = "/scratch200/leardistel/petal_benchmark/results/exp_E288_azimuth_graph"
Path(OUT_DIR).mkdir(parents=True, exist_ok=True)

SEED = 42
np.random.seed(SEED)

def geodesic_distance_matrix(unit_vecs):
    """Pairwise geodesic angles (degrees) between unit vectors."""
    dots = unit_vecs @ unit_vecs.T
    np.clip(dots, -1, 1, out=dots)
    return np.degrees(np.arccos(dots))

def spectral_clusters(eps_hat, species, n_clusters_list=[5, 10, 20], threshold_deg=30.0):
    """
    Build adjacency from eps_hat dot products, run spectral clustering.
    Returns cluster labels and family cross-cutting score.
    """
    from sklearn.cluster import SpectralClustering
    N = len(species)
    # affinity: max(0, cos(eps_i, eps_j) - cos(threshold))
    dots = eps_hat @ eps_hat.T
    threshold_cos = np.cos(np.radians(threshold_deg))
    affinity = np.maximum(0, dots - threshold_cos)
    np.fill_diagonal(affinity, 1.0)

    results = {}
    for k in n_clusters_list:
        if k >= N:
            continue
        sc = SpectralClustering(n_clusters=k, affinity="precomputed",
                                random_state=SEED, n_init=10)
        labels = sc.fit_predict(affinity)
        results[f"k{k}"] = {
            "labels": labels.tolist(),
            "species": species.tolist()
        }
        print(f"  Spectral k={k}: {len(np.unique(labels))} clusters assigned")
    return results, affinity

def voronoi_cell_volumes(eps_hat, species, n_samples=50000):
    """
    Monte Carlo estimate of Voronoi cell volumes on the azimuth sphere.
    Sample random unit vectors on S^{d-1}, assign each to nearest eps_hat species.
    Cell volume ∝ fraction of samples assigned to each species.
    """
    N, d = eps_hat.shape
    # Sample random unit vectors in d-dim
    raw = np.random.randn(n_samples, d)
    raw_n = raw / np.linalg.norm(raw, axis=1, keepdims=True)
    # Project out D_flower component — we want azimuth-only voronoi
    # eps_hat is already ⊥ D_flower, so just assign by dot product
    dots = raw_n @ eps_hat.T  # (n_samples, N)
    assignments = np.argmax(dots, axis=1)
    counts = np.bincount(assignments, minlength=N)
    volumes = counts / n_samples  # fraction = proxy for cell volume
    return volumes

def mst_from_distance_matrix(D_deg, species):
    """Compute MST using Kruskal's algorithm on geodesic distance matrix."""
    from scipy.sparse.csgraph import minimum_spanning_tree
    D_rad = np.radians(D_deg)
    mst = minimum_spanning_tree(csr_matrix(D_rad))
    mst_arr = mst.toarray()
    # collect edges
    rows, cols = np.where(mst_arr > 0)
    edges = [
        {"s1": str(species[r]), "s2": str(species[c]),
         "dist_deg": float(np.degrees(mst_arr[r, c]))}
        for r, c in zip(rows, cols)
    ]
    return edges

def mantel_test(D1, D2, n_perm=999, seed=42):
    """
    Mantel test between two distance matrices (flattened upper triangles).
    Returns r, p_value.
    """
    rng = np.random.default_rng(seed)
    N = D1.shape[0]
    idx = np.triu_indices(N, k=1)
    v1 = D1[idx]
    v2 = D2[idx]
    r_obs = float(np.corrcoef(v1, v2)[0, 1])
    count = 0
    for _ in range(n_perm):
        perm = rng.permutation(N)
        D2_perm = D2[np.ix_(perm, perm)]
        v2_p = D2_perm[idx]
        r_p = np.corrcoef(v1, v2_p)[0, 1]
        if r_p >= r_obs:
            count += 1
    p_val = (count + 1) / (n_perm + 1)
    return r_obs, p_val

# ── SAM3 graphs ────────────────────────────────────────────────────────────
print("=== SAM3 FPN graphs ===")
d = np.load(SAM3_NPZ, allow_pickle=True)
eps_sam3    = d["eps_hat_sam3"]  # (85, 256) unit vectors
species_sam3 = d["species"]      # (85,)
theta_sam3  = d["theta_sam3_deg"]

D_sam3 = geodesic_distance_matrix(eps_sam3)
print(f"  SAM3 geodesic: mean={D_sam3[D_sam3>0].mean():.1f}°, "
      f"min={D_sam3[D_sam3>0].min():.1f}°")

# A: Spectral clustering
print("  Running spectral clustering (SAM3)...")
clusters_sam3, affinity_sam3 = spectral_clusters(
    eps_sam3, species_sam3, n_clusters_list=[5, 10, 20], threshold_deg=60.0
)
with open(f"{OUT_DIR}/spectral_clusters_sam3.json", "w") as f:
    json.dump(clusters_sam3, f, indent=2)

# B: Voronoi volumes
print("  Computing Voronoi volumes (SAM3, 50k samples)...")
vol_sam3 = voronoi_cell_volumes(eps_sam3, species_sam3, n_samples=50000)
voronoi_sam3 = [
    {"species": str(species_sam3[i]), "theta_deg": float(theta_sam3[i]),
     "voronoi_volume_frac": float(vol_sam3[i])}
    for i in np.argsort(vol_sam3)  # sorted smallest to largest
]
with open(f"{OUT_DIR}/voronoi_sam3.json", "w") as f:
    json.dump(voronoi_sam3, f, indent=2)
print(f"  Smallest Voronoi: {voronoi_sam3[0]['species']} ({voronoi_sam3[0]['voronoi_volume_frac']:.4f})")
print(f"  Largest Voronoi:  {voronoi_sam3[-1]['species']} ({voronoi_sam3[-1]['voronoi_volume_frac']:.4f})")

# C: MST (SAM3, no DNA overlap — save for inspection)
print("  Computing MST (SAM3)...")
mst_sam3 = mst_from_distance_matrix(D_sam3, species_sam3)
with open(f"{OUT_DIR}/mst_sam3.json", "w") as f:
    json.dump(mst_sam3, f, indent=2)

# ── BioCLIP graphs ─────────────────────────────────────────────────────────
print("=== BioCLIP visual graphs (1681 species) ===")
c = np.load(CONE_NPZ, allow_pickle=True)
eps_bioclip  = c["eps_hat"]        # (1681, 1024)
species_bioclip = c["species"]     # (1681,)
theta_bioclip = c["theta_deg"]     # (1681,)

print("  Computing geodesic distance matrix (1681×1681)... this takes ~2 min")
D_bioclip = geodesic_distance_matrix(eps_bioclip)
print(f"  BioCLIP geodesic: mean={D_bioclip[D_bioclip>0].mean():.1f}°, "
      f"min={D_bioclip[D_bioclip>0].min():.2f}°")

# A: Spectral clustering (BioCLIP)
print("  Running spectral clustering (BioCLIP)...")
clusters_bioclip, affinity_bioclip = spectral_clusters(
    eps_bioclip, species_bioclip, n_clusters_list=[10, 20, 50], threshold_deg=70.0
)
with open(f"{OUT_DIR}/spectral_clusters_bioclip.json", "w") as f:
    json.dump(clusters_bioclip, f, indent=2)

# B: Voronoi volumes (BioCLIP)
print("  Computing Voronoi volumes (BioCLIP, 100k samples)...")
vol_bioclip = voronoi_cell_volumes(eps_bioclip, species_bioclip, n_samples=100000)
voronoi_bioclip = [
    {"species": str(species_bioclip[i]), "theta_deg": float(theta_bioclip[i]),
     "voronoi_volume_frac": float(vol_bioclip[i])}
    for i in np.argsort(vol_bioclip)
]
with open(f"{OUT_DIR}/voronoi_bioclip.json", "w") as f:
    json.dump(voronoi_bioclip, f, indent=2)

# C: MST + Mantel test against K2P DNA distances
print("  Computing MST (BioCLIP, 1681 species)...")
mst_bioclip = mst_from_distance_matrix(D_bioclip, species_bioclip)
with open(f"{OUT_DIR}/mst_bioclip.json", "w") as f:
    json.dump(mst_bioclip, f, indent=2)

# Mantel test: MST azimuth distances vs K2P
print("  Loading K2P distances for Mantel test...")
k = np.load(K2P_NPZ, allow_pickle=True)
D_k2p      = k["D"]        # (223, 223)
species_k2p = k["species"] # (223,)

# find overlap between BioCLIP and K2P species
bioclip_list = list(species_bioclip)
k2p_list     = list(species_k2p)
overlap = [s for s in k2p_list if s in set(bioclip_list)]
print(f"  K2P ∩ BioCLIP overlap: {len(overlap)} species")

if len(overlap) >= 10:
    bioclip_idx = [bioclip_list.index(s) for s in overlap]
    k2p_idx     = [k2p_list.index(s) for s in overlap]
    D_vis_overlap = D_bioclip[np.ix_(bioclip_idx, bioclip_idx)]
    D_k2p_overlap = D_k2p[np.ix_(k2p_idx, k2p_idx)]
    r_mantel, p_mantel = mantel_test(D_vis_overlap, D_k2p_overlap, n_perm=999, seed=SEED)
    mantel_result = {
        "n_species_overlap": len(overlap),
        "overlap_species": overlap,
        "r_mantel": r_mantel,
        "p_value": p_mantel,
        "n_permutations": 999,
        "interpretation": (
            "r>0.3, p<0.05: visual azimuth distances co-evolve with DNA distances "
            "(morphology tracks genotype). Divergence from this = convergent evolution."
        )
    }
    print(f"  Mantel r={r_mantel:.4f}, p={p_mantel:.4f}")
else:
    mantel_result = {"error": f"Only {len(overlap)} overlapping species — insufficient for Mantel"}

with open(f"{OUT_DIR}/mst_dna_correlation.json", "w") as f:
    json.dump(mantel_result, f, indent=2)

summary = {
    "exp": "E288_azimuth_graph",
    "date": "2026-04-14",
    "seed": SEED,
    "sam3": {
        "n_species": 85,
        "mean_geodesic_deg": float(D_sam3[D_sam3>0].mean()),
        "min_geodesic_deg": float(D_sam3[D_sam3>0].min()),
        "spectral_k_values": [5, 10, 20],
        "voronoi_min_species": voronoi_sam3[0]["species"],
        "voronoi_max_species": voronoi_sam3[-1]["species"]
    },
    "bioclip": {
        "n_species": 1681,
        "mean_geodesic_deg": float(D_bioclip[D_bioclip>0].mean()),
        "min_geodesic_deg": float(D_bioclip[D_bioclip>0].min()),
        "spectral_k_values": [10, 20, 50],
        "mantel_r": mantel_result.get("r_mantel"),
        "mantel_p": mantel_result.get("p_value")
    }
}
with open(f"{OUT_DIR}/summary.json", "w") as f:
    json.dump(summary, f, indent=2)
print("E288 DONE.")
```

**SLURM:** 8 CPUs, 64G RAM (1681×1681 matrix ~11GB float32 — use float16 or process in chunks if OOM), 2h time limit.

---

## Task 4: E289 — Crystallography: Debye-Scherrer Sample Complexity

### What we're measuring

The Debye-Scherrer formula relates ring width to minimum resolvable grain size. Here:
- Ring width σ_θ = 5.65° (SAM3) or 5.27° (BioCLIP) → the "coherence length"
- Minimum pairwise separation = 8.634° (from E72, BioCLIP min pairwise)
- **Question:** How many masks n_s are needed per species to estimate ε̂[s] to within the inter-species separation (so retrieval is reliable)?

**Debye-Scherrer analog:**

In crystallography: `τ = Kλ / (β cos θ)` where β = peak width = our σ_θ.

Our analog: to resolve two species separated by Δθ_min = 8.634°, we need the estimated centroid to have angular uncertainty < Δθ_min/2. The uncertainty on the centroid estimate from n masks is:

```
σ_centroid = σ_within / sqrt(n)
```

Requiring σ_centroid < Δθ_min/2:
```
n_required[s] = (2 σ_within[s] / Δθ_min)²
```

This gives a **per-species sample complexity** — the minimum number of masks needed for reliable azimuth recovery. Compare against actual n_masks[s] from E76c to find which species are currently under-sampled.

**Files:**
- Create: `/scratch200/leardistel/petal_benchmark/experiments/exp_E289_debye_scherrer.py`
- Output: `/scratch200/leardistel/petal_benchmark/results/exp_E289_debye_scherrer/`
  - `summary.json`
  - `per_species_sample_complexity.json` — n_required vs n_actual per species
  - `undersampled_species.json` — species where n_actual < n_required

**Step 1: Write the script**

```python
#!/usr/bin/env python3
"""
E289 — Crystallography: Debye-Scherrer Sample Complexity
Computes per-species minimum mask count needed for reliable azimuth fingerprint recovery.
Formula: n_required = (2 * sigma_within / delta_min)^2
where sigma_within = intra-species CLS angular scatter, delta_min = min inter-species separation.
"""
import functools
print = functools.partial(print, flush=True)

import numpy as np
import json
from pathlib import Path

CONE_NPZ  = "/scratch200/leardistel/petal_benchmark/results/exp_E70_bioclip_text_2174/cone_geometry.npz"
CLS_NPZ   = "/scratch200/leardistel/petal_benchmark/results/exp_E76c_cls_all_israel/cls_tokens.npz"
SAM3_NPZ  = "/scratch200/leardistel/petal_benchmark/results/exp33_sam3_fpn_geometry/f_SAM3.npz"
OUT_DIR   = "/scratch200/leardistel/petal_benchmark/results/exp_E289_debye_scherrer"
Path(OUT_DIR).mkdir(parents=True, exist_ok=True)

SEED = 42
np.random.seed(SEED)

DELTA_MIN_BIOCLIP_DEG = 8.634  # E72 confirmed minimum pairwise angle between species centroids

def normalize(x):
    n = np.linalg.norm(x, axis=-1, keepdims=True)
    assert np.all(n > 1e-8), "zero vector"
    return x / n

def per_species_sigma_within(cls_tokens_n, cls_species, D_flower_vis, target_species):
    """
    For each species in target_species, compute sigma_within =
    std of angular deviation of per-mask CLS tokens from species centroid,
    AFTER projecting onto azimuth plane (subtracting elevation component).
    Returns dict: species -> sigma_within_deg
    """
    results = {}
    for sp in target_species:
        idx = np.where(cls_species == sp)[0]
        if len(idx) < 3:
            continue
        masks = cls_tokens_n[idx]  # (n, 1024)
        centroid = normalize(masks.mean(axis=0, keepdims=True))[0]
        # project out elevation (D_flower direction)
        elev = masks @ D_flower_vis
        azim_res = masks - elev[:, None] * D_flower_vis[None, :]
        norms = np.linalg.norm(azim_res, axis=1)
        valid = norms > 1e-6
        if valid.sum() < 3:
            continue
        azim_n = azim_res[valid] / norms[valid, None]
        centroid_azim = normalize(centroid - (centroid @ D_flower_vis) * D_flower_vis)[None]
        dots = azim_n @ centroid_azim.T
        angles = np.degrees(np.arccos(np.clip(dots.squeeze(), -1, 1)))
        results[sp] = {
            "sigma_within_deg": float(angles.std()),
            "mean_angle_deg": float(angles.mean()),
            "n_masks": int(valid.sum())
        }
    return results

print("Loading data...")
c = np.load(CONE_NPZ, allow_pickle=True)
f_vis_n       = c["f_vis_n"]         # (1681, 1024)
theta_bioclip = c["theta_deg"]        # (1681,)
D_flower_vis  = c["D_flower_vis_n"]   # (1024,)
n_masks_cone  = c["n_masks"]          # (1681,)
species_bioclip = c["species"]        # (1681,)

cls_data = np.load(CLS_NPZ, allow_pickle=True)
cls_tokens    = cls_data["cls"]       # (52049, 1024)
cls_species   = cls_data["species"]   # (52049,)
cls_tokens_n  = normalize(cls_tokens)

# find overlap
bioclip_set = set(species_bioclip.tolist())
overlap_sp = [s for s in np.unique(cls_species) if s in bioclip_set]
print(f"Computing sigma_within for {len(overlap_sp)} species with CLS tokens...")

sigma_within = per_species_sigma_within(
    cls_tokens_n, cls_species, D_flower_vis, overlap_sp
)

# Debye-Scherrer sample complexity
delta_min_rad = np.radians(DELTA_MIN_BIOCLIP_DEG)
results = []
for sp, sw in sigma_within.items():
    sigma_rad = np.radians(sw["sigma_within_deg"])
    n_required = max(1, int(np.ceil((2 * sigma_rad / delta_min_rad) ** 2)))
    n_actual   = sw["n_masks"]
    bioclip_idx = list(species_bioclip).index(sp) if sp in bioclip_set else -1
    theta_s = float(theta_bioclip[bioclip_idx]) if bioclip_idx >= 0 else None
    results.append({
        "species": sp,
        "theta_deg": theta_s,
        "sigma_within_deg": sw["sigma_within_deg"],
        "n_required": n_required,
        "n_actual": n_actual,
        "undersampled": n_actual < n_required,
        "deficit": max(0, n_required - n_actual),
        "ratio_actual_to_required": n_actual / n_required if n_required > 0 else float("inf")
    })

results.sort(key=lambda x: x["ratio_actual_to_required"])
undersampled = [r for r in results if r["undersampled"]]

print(f"Total species analyzed: {len(results)}")
print(f"Undersampled: {len(undersampled)} / {len(results)} = {len(undersampled)/len(results)*100:.1f}%")
if results:
    print(f"Median n_required: {np.median([r['n_required'] for r in results]):.0f}")
    print(f"Median n_actual: {np.median([r['n_actual'] for r in results]):.0f}")

with open(f"{OUT_DIR}/per_species_sample_complexity.json", "w") as f:
    json.dump(results, f, indent=2)
with open(f"{OUT_DIR}/undersampled_species.json", "w") as f:
    json.dump(undersampled, f, indent=2)

# SAM3 version (proxy sigma_within = theta std / sqrt(n_feats))
print("\nSAM3 Debye-Scherrer proxy...")
d = np.load(SAM3_NPZ, allow_pickle=True)
theta_sam3  = d["theta_sam3_deg"]
species_sam3 = d["species"]
# per_species.json has n_flower_feats
import os
per_sp_path = SAM3_NPZ.replace("f_SAM3.npz", "per_species.json")
with open(per_sp_path) as fp:
    per_sp = json.load(fp)

# min pairwise in SAM3 azimuth space
eps_sam3 = d["eps_hat_sam3"]
dots_sam3 = eps_sam3 @ eps_sam3.T
np.fill_diagonal(dots_sam3, 1.0)
off_diag = dots_sam3[np.triu_indices(85, k=1)]
delta_min_sam3_deg = float(np.degrees(np.arccos(np.clip(off_diag.min(), -1, 1))))
print(f"  SAM3 min pairwise azimuth separation: {delta_min_sam3_deg:.2f}°")
delta_min_sam3_rad = np.radians(delta_min_sam3_deg)

sam3_results = []
for entry in per_sp:
    sp = entry["species"]
    n_feats = entry.get("n_flower_feats", 5)
    theta_s = entry["theta_sam3_deg"]
    # proxy sigma_within: use theta_std / sqrt(N) as coherence estimate
    sigma_proxy_rad = np.radians(theta_sam3.std()) / np.sqrt(n_feats)
    n_required = max(1, int(np.ceil((2 * sigma_proxy_rad / delta_min_sam3_rad) ** 2)))
    sam3_results.append({
        "species": sp, "theta_deg": theta_s,
        "n_actual": n_feats, "n_required_proxy": n_required,
        "undersampled_proxy": n_feats < n_required
    })

sam3_undersampled = [r for r in sam3_results if r["undersampled_proxy"]]
print(f"  SAM3 undersampled (proxy): {len(sam3_undersampled)}/85")

summary = {
    "exp": "E289_debye_scherrer",
    "date": "2026-04-14",
    "seed": SEED,
    "BioCLIP": {
        "delta_min_deg": DELTA_MIN_BIOCLIP_DEG,
        "n_species_analyzed": len(results),
        "n_undersampled": len(undersampled),
        "fraction_undersampled": len(undersampled) / max(1, len(results)),
        "median_n_required": float(np.median([r["n_required"] for r in results])),
        "median_n_actual": float(np.median([r["n_actual"] for r in results])),
    },
    "SAM3": {
        "delta_min_deg": delta_min_sam3_deg,
        "n_species": 85,
        "n_undersampled_proxy": len(sam3_undersampled)
    },
    "formula": "n_required = ceil( (2 * sigma_within / delta_min)^2 )",
    "interpretation": (
        "Undersampled species cannot be reliably distinguished from azimuthal neighbors "
        "with current mask counts. These are candidates for targeted image collection."
    )
}
with open(f"{OUT_DIR}/summary.json", "w") as f:
    json.dump(summary, f, indent=2)
print("E289 DONE.")
```

---

## Task 5: E290 — Wild Card: Sphere Painting + Topological Depth

### What we're measuring

Two crazy ideas that use the geometry in a completely different way:

**A. Sphere Painting:** For each species, paint its azimuth direction onto a 3D projection (first 3 PCA dims of eps_hat) using its dominant color (from the color analysis experiments). Do species with similar flower colors cluster together in azimuth? This is a visual/quantitative test of whether BioCLIP's visual space organizes flowers by color.

**B. Topological depth:** The persistent homology result from E72 showed H0 max lifetime = 54.6°, H1 max lifetime = 5.1°. The H1 loops are topological "holes" in the species distribution — these are regions where no species exists, despite nearby species on both sides. Each H1 loop is a potential **speciation gap** — morphological space that evolution has not filled. The species on the boundary of the largest H1 loop are the most morphologically isolated — isolated not just from neighbors but from the entire connected topology of floral morphology.

**Files:**
- Create: `/scratch200/leardistel/petal_benchmark/experiments/exp_E290_sphere_painting.py`
- Output: `/scratch200/leardistel/petal_benchmark/results/exp_E290_sphere_painting/`
  - `summary.json`
  - `pca3_projection.json` — 3D PCA coords of all eps_hat vectors
  - `h1_loop_species.json` — species on boundaries of H1 topological loops

**Note on A (color):** Requires per-species dominant color data. Check if `exp_E23_mask_color_extraction` or `exp_E275_color` has per-species color vectors before implementing. If absent, skip color and do PCA projection only.

**Note on B (topology):** E72 already ran ripser. The H1 persistence diagram is in `persistent_homology.json`. This experiment recovers WHICH species are near the H1 holes — ripser gives death/birth radii, not species membership. Use birth radius to threshold the azimuth graph and identify connected components that merge at that scale.

---

## SLURM Submit Scripts

Create one submit script per experiment:

```bash
# submit_E286.sh through submit_E290.sh
# All use: --partition=power-general-public-pool --qos=public
# E286, E287, E289: --mem=32G --cpus-per-task=4 --time=00:30:00
# E288: --mem=64G --cpus-per-task=8 --time=02:00:00 (1681x1681 matrix)
# E290: --mem=16G --cpus-per-task=2 --time=00:20:00
```

---

## Validation Checklist

After all jobs complete:

| Experiment | Key output to check | Pass criterion |
|---|---|---|
| E286 | `channel_bioclip.json`.capacity_bits | > 0, < 1000 |
| E286 | `kent_sam3.json`.ratio | Report whether > or < BioCLIP's 1.20 |
| E287 | `per_species_kappa.json` | All κ_s > 0, sorted ascending |
| E287 | `detectability_prior.json`.n_at_risk | Report fraction |
| E288 | `mst_dna_correlation.json`.r_mantel | Report r and p — hypothesis test |
| E288 | `voronoi_bioclip.json`[0] | Smallest cell species name |
| E289 | `summary.json`.fraction_undersampled | Report — if >50% → targeted collection needed |
| E290 | `summary.json` exists | Any output |

---

## Science Log Entries to Write After Results

- **Entry 283** — E286: Channel Capacity (what is the azimuth bandwidth?)
- **Entry 284** — E287: vMF Stellar Population (per-species κ_s, detectability prior)
- **Entry 285** — E288: Spectral Clusters + Voronoi + MST Mantel (does azimuth track DNA?)
- **Entry 286** — E289: Debye-Scherrer (which species are under-sampled?)
- **Entry 287** — E290: Sphere Painting + Topological Holes
- **Entry 288** — Cross-experiment synthesis: what do all 5 tell us together?
