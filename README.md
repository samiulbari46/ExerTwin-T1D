# PHYT1D — Physiological Twin for Type 1 Diabetes

**An exercise-aware digital twin framework for personalised glucose dynamics identification and simulation.**

PHYT1D extends the Dalla Man (2014) glucose–insulin ODE skeleton with an analytical exercise state function **Φ(u, d, t)** that modulates insulin sensitivity *SI*, endogenous glucose production *EGP*, non-insulin-mediated muscle uptake *Fc01*, and gastric emptying *kempt* as explicit functions of declared exercise intensity (%VO₂max), duration, and elapsed time — **no wearables required**. Eight core glucose–insulin parameters are identified per patient from CGM, insulin, and meal data via adaptive MCMC; six additional exercise-specific parameters — including the novel post-exercise SI decay constant *τ_post* — are identified from exercise-window CGM. Validated against the UVa/Padova T1D Simulator (T1DS). Pass threshold: **MARD < 10%** (Cappon et al. 2023).

> ⚠️ **Research use only.** Not a medical device. Not intended for clinical decision-making.

---

## Table of Contents

- [Highlights](#highlights)
- [Repository Structure](#repository-structure)
- [Model Architecture](#model-architecture)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Module Guide](#module-guide)
- [Parameter Groups](#parameter-groups)
- [Validation Scenarios](#validation-scenarios)
- [Evaluation Metrics](#evaluation-metrics)
- [Citation](#citation)
- [References](#references)
- [License](#license)

---

## Highlights

- **Analytical exercise model** — no neural network, no heart-rate sensor; intensity and duration are declared by the user.
- **Two-phase MCMC identification** — Window A (core glucose–insulin parameters, meal-only CGM) and Window B (exercise-specific parameters, exercise-window CGM).
- **Post-exercise SI tail** — patient-specific *τ_post* captures the 4–12 h late-onset hypoglycaemia mechanism, unique among existing T1D digital twins.
- **Differential aerobic / resistance physiology** at the ODE level.
- **Uncertainty propagation** — 1000 posterior realisations yield median + 25–75th percentile bands.
- **Validation harness** — four scenario families against UVa/Padova T1DS.

---

## Repository Structure

```
PHYT1D/
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
├── notebooks/
│   ├── 01_ode_base.ipynb              # Dalla Man ODE skeleton (10 states)
│   ├── 02_exercise_phi.ipynb          # Exercise state function Φ(u, d, t)
│   ├── 03_vo2max_estimation.ipynb     # VO2max/VO2rest priors (LowLands)
│   ├── 04_parameters.ipynb            # Full parameter catalogue (groups A–D)
│   ├── 05_mcmc_window_a.ipynb         # MCMC core parameter identification
│   ├── 06_simulator.ipynb             # Euler integrator + uncertainty propagation
│   ├── 07_mcmc_window_b.ipynb         # MCMC exercise parameter identification
│   ├── 08_evaluation_metrics.ipynb    # MARD, RMSE, TIR/TBR/TAR, Clarke EGA
│   ├── 09_scenarios.ipynb             # S1–S4 validation scenarios
│   └── 10_main.ipynb                  # End-to-end pipeline (single patient)
├── scenarios/
│   └── Scenario.scn           # UVa/Padova T1DS scenario file
├── data/
│   └── Example_data.csv               # (gitignored) patient CSVs, T1DS exports
```

---

## Model Architecture

PHYT1D operates in three phases:

1. **Phase 1 — Core identification (Window A).** Eight glucose–insulin parameters identified from meal-only CGM segments (≥ 6 h, ≥ 4 h post-bolus) via adaptive single-component Metropolis–Hastings, posterior fitted with a t-copula.
2. **Phase 2 — Exercise identification (Window B).** Six exercise parameters identified from the exercise bout + 6 h post-exercise CGM, conditioned on Phase 1 posteriors.
3. **Simulation.** Augmented ODE integrated (Euler, Δt = 5 min) with Φ(u, d, t) parameter increments. 1000 posterior realisations → median + uncertainty bands.

| Component | Description | Source |
|---|---|---|
| ODE skeleton | Dalla Man 2014 glucose–insulin (10 states) | Dalla Man 2014 |
| SC insulin PK | 3-compartment Schiavon; *k_d, k_a2* identified | Schiavon 2018  |
| CHO absorption | Nonlinear gastric emptying; *k_empt, k_abs* identified | Dalla Man 2006  |
| Exercise Φ(u,d,t) | Analytical; %VO₂max + duration → ODE increments | Riddell 2017 ; Yardley 2013  |
| Post-exercise tail | Exponential SI decay; *τ_post* identified | Hinshaw 2013  |
| MCMC | Adaptive SCMH; t-copula posterior; 2000 samples | Cappon 2023 ; Gilks 1996  |

---

## Installation

**Requirements:** Python 3.11+ and JupyterLab.

```bash
git clone https://github.com/<your-username>/PHYT1D.git
cd PHYT1D
python -m venv .venv
source .venv/bin/activate          # on Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter lab
```

### `requirements.txt`

```
numpy>=1.26
scipy>=1.11
pandas>=2.1
matplotlib>=3.8
seaborn>=0.13
jupyterlab>=4.0
nbformat>=5.9
tqdm>=4.66
```

---

## Quick Start

Run the notebooks in order, or jump straight to the end-to-end pipeline:

```bash
jupyter lab notebooks/10_main.ipynb
```

`10_main.ipynb` performs the full pipeline for a single patient:

1. Import patient CSV (CGM, insulin, meals).
2. Set user parameters (Group C — meal times, CHO amounts, CR, sex, age group).
3. Compute VO₂max prior from age/sex.
4. Build ground-truth signal and `USER_PARAMS` dictionary.
5. **MCMC Window A** — identify *θ_A* (8 core parameters).
6. **MCMC Window B** — identify *θ_B* (6 exercise parameters).
7. Run 1000-realisation ensemble simulation.
8. Compute evaluation metrics (MARD, TIR, Clarke EGA, …).
9. Plot 24 h glucose trajectory with 25–75th uncertainty bands.

---

## Module Guide

| # | Notebook | Purpose |
|---|---|---|
| 01 | `01_ode_base.ipynb` | Base ODE structure: subcutaneous insulin PK (Schiavon 2018), oral glucose absorption (Dalla Man 2006), glucose–insulin kinetics (Bergman 1979), hypoglycaemia risk function ρ(G). |
| 02 | `02_exercise_phi.ipynb` | Exercise state function Φ(u, d, t): ΔSI_ex (aerobic + resistance), ΔSI_tail (post-exercise), ΔEGP_res, ΔFc01_eff, kempt_eff, SI_eff composite. |
| 03 | `03_vo2max_estimation.ipynb` | VO₂max prior from age/sex (LowLands Fitness Registry); age-specific VO₂rest. |
| 04 | `04_parameters.ipynb` | Complete parameter catalogue: priors, bounds, biological meanings (Groups A–D). |
| 05 | `05_mcmc_window_a.ipynb` | MCMC Window A: log-prior, Gaussian log-likelihood, adaptive SCMH sampler, t-copula posterior fit, identifiability check, MARD evaluation. |
| 06 | `06_simulator.ipynb` | Euler integrator (Δt = 5 min), exercise activation logic, bolus computation, 1000-realisation ensemble. |
| 07 | `07_mcmc_window_b.ipynb` | MCMC Window B: exercise priors, log-likelihood conditioned on *θ_A* medians, *τ_post* and VO₂max recovery checks. |
| 08 | `08_evaluation_metrics.ipynb` | MARD, MAPE, gRMSE, TIR/TBR/TAR, Clarke Error Grid, parameter recovery, Wilcoxon signed-rank test. |
| 09 | `09_scenarios.ipynb` | S1–S4 validation scenarios against UVa/Padova T1DS; MARD profile and 24 h glucose plots. |
| 10 | `10_main.ipynb` | End-to-end pipeline (single patient): import → MCMC → simulate → evaluate → plot. |

---

## Parameter Groups

Parameters are organised into four groups (see Section 8 of the technical report for full priors and biological meanings).

### Group A — MCMC Window A (8 params, identified from CGM)
`SI, SG, Gb, p2, kd, ka2, kempt, kabs`

### Group B — MCMC Window B (6 params, identified from exercise CGM)
`beta_aer, beta_res, tau_post, tau_on, VO2max, phi`

### Group C — User-supplied at simulation time
`u, d, t_start, exercise_type, CR, CHO_BF/LU/DN/SN, t_BF/LU/DN/SN, bolus_factor, sex, age_group`

### Group D — Fixed population values
`VI, ke, beta, f, VG, alpha, r1, r2, Gth, Fc01, gamma_aer, gamma_res, tau_ramp, delta_res`

---

## Validation Scenarios

All scenarios: open loop, 1440-min simulation, 100 virtual adults, CR = 15, Q_basal = quest, Q_meals = Total.

| Scenario | Variable | Levels | Exercise state | Primary parameters tested |
|---|---|---|---|---|
| S1 | Intensity | rest, 10%, 25%, 50%, 60%, 75%, 90% VO₂max | Non-Aerobic/Aerobic, 60 min, 17:00 | beta_aer, tau_on, tau_post, VO2max |
| S2 | Duration | 15, 30, 45, 60, 75, 90, 105, 120 min | Aerobic, 50%, 16:00 - 18:00 | tau_post, Fc01, tail–meal interaction |
| S3 | Bolus modulation | ×0.4 – ×1.6 (6 levels) | Rest | SI, kd, ka2 |
| S4 | CHO modulation | ×0.7 – ×1.3 (6 levels) | Rest | kempt, kabs, Ra(t) |

**Total runs:** 27 × 50 = 1350 T1DS simulations.

---

## Evaluation Metrics

- **MARD** (Cappon 2023 eq. 9) — pass threshold < 10%
- **RMSE** / **gRMSE** [mg/dL]
- **Time-in-Range** — TIR (70–180), TBR (<70), TAR (>180) (Battelino 2019)
- **Clarke Error Grid Analysis** (Clarke 1987)
- **VO₂max recovery** — |VO2max_hat − VO2max_true| / VO2max_true < 15%
- **τ_post recovery** — posterior median vs T1DS IS controller implied decay
- **Parameter RE** — < 20% per Group A parameter
- **Posterior coverage** — ≥ 50% of subjects with *θ\** in 25–75th CI
- **Wilcoxon signed-rank test**, Bonferroni corrected, α = 0.05

---

## Citation

If you use PHYT1D in your research, please cite:

```bibtex
@techreport{phyt1d2025,
  title  = {PHYT1D: Physiological Twin for Type 1 Diabetes —
            An Exercise-Aware Digital Twin Framework},
  author = {<Samiul Bari>},
  year   = {2025},
  type   = {Technical Design Document},
  note   = {Version 2}
}
```

---

## References

Key references:

-  Riddell MC et al. Exercise management in type 1 diabetes: a consensus statement. *Lancet Diabetes Endocrinol* 2017;5(5):377–390.
-  Cappon G et al. ReplayBG. *IEEE Trans Biomed Eng* 2023;70(11):3227–3238.
-  Dalla Man C et al. The UVA/Padova T1D Simulator: new features. *J Diabetes Sci Technol* 2014;8(1):26–34.
-  Schiavon M et al. Modeling SC absorption of fast-acting insulin. *IEEE Trans Biomed Eng* 2018;65(9):2079–2086.
-  Dalla Man C et al. System model of oral glucose absorption. *IEEE Trans Biomed Eng* 2006;53(12):2472–2478.
-  Yardley JE et al. Resistance vs aerobic exercise: acute effects on glycemia in T1D. *Diabetes Care* 2013;36(3):537–542.
-  Hinshaw L et al. Diurnal pattern of insulin action in T1D. *Diabetes* 2013;62(7):2223–2229.
-  Van de Poppe T et al. Reference values for cardiorespiratory fitness — LowLands Fitness Registry. 2021.
-  Battelino T et al. Clinical targets for CGM data interpretation. *Diabetes Care* 2019;42(8):1593–1603.

---

## License

This project is released under the MIT License — see [`LICENSE`](LICENSE) for details.

**Disclaimer.** PHYT1D is research software. It is not a medical device, has not been evaluated by any regulatory body, and must not be used to make clinical or therapeutic decisions.
