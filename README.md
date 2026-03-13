# PV & BESS Hosting Capacity Sizing — IEEE 33-Bus Distribution System

[![Python](https://img.shields.io/badge/Python-3.8%2B-blue?logo=python)](https://www.python.org/)
[![Pyomo](https://img.shields.io/badge/Pyomo-6.x-orange)](http://www.pyomo.org/)
[![Solver](https://img.shields.io/badge/Solver-HiGHS%20%7C%20CBC%20%7C%20Gurobi-green)](https://highs.dev/)
[![License](https://img.shields.io/badge/License-MIT-lightgrey)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Active%20Research-brightgreen)]()

> **Thesis project** — Optimal siting and sizing of rooftop PV and battery energy storage (BESS) on the IEEE 33-bus radial distribution feeder using Mixed-Integer Linear Programming (MILP) with LinDistFlow power flow approximation.

---

## Table of Contents

- [Overview](#overview)
- [System Description](#system-description)
- [Mathematical Formulation](#mathematical-formulation)
- [Repository Structure](#repository-structure)
- [Requirements](#requirements)
- [How to Run](#how-to-run)
- [Key Results](#key-results)
- [Debugging Journey](#debugging-journey)
- [Parameters Reference](#parameters-reference)
- [Citation](#citation)

---

## Overview

This repository contains a Jupyter Notebook that solves the **PV and BESS hosting capacity problem** on the IEEE 33-bus benchmark distribution network. The optimizer decides:

- **Where** to place PV and BESS (bus selection via binary variables)
- **How large** each unit should be (continuous sizing variables)
- **How to dispatch** BESS charge/discharge across 24 hours (operational variables)

The objective is to **minimize Energy Not Served (ENS)** — a proxy for maximizing the utilization of available DER while keeping all bus voltages within limits.

---

## System Description

| Parameter | Value |
|-----------|-------|
| Network | IEEE 33-bus radial feeder (Baran & Wu, 1989) |
| Buses | 33 |
| Branches | 32 |
| Nominal voltage | 12.66 kV |
| System base | 10 MVA |
| Impedance base | 16.034 Ω |
| Peak active load | 3,715 kW |
| Peak reactive load | 2,300 kVAr |
| Time horizon | 24 hours (hourly dispatch) |
| Solver | HiGHS 1.x (auto-detected: CBC, Gurobi, GLPK fallback) |

The feeder has three laterals branching from the main feeder:

```
Bus 1 ──── 2 ──── 3 ──── 4 ──── 5 ──── 6 ──── 7 ── ... ── 18   (main feeder)
            │      │             │
           19     23            26
           20     24            27
           21     25            28
           22                   29 ── 30 ── 31 ── 32 ── 33
```

> **Weakest bus:** Bus 18 (V = 0.9115 pu at peak load, no DER)  
> **Worst power factor:** Bus 30 (Q = 600 kVAr, P = 200 kW → PF = 0.316)

---

## Mathematical Formulation

### Power Flow — LinDistFlow (Baran & Wu 1989)

$$V_{j,t} = V_{i,t} - R_\ell \cdot P_{\ell,t} - X_\ell \cdot Q_{\ell,t}$$

Active and reactive power balance at each load bus $j$ (buses 2–33):

$$P_{in} - P_{out} = -\left(P_d \cdot \lambda_t - P_{PV} - P_{dis} + P_{ch} - P_{LS}\right)$$

$$Q_{in} - Q_{out} = -Q_d \cdot \lambda_t$$

> The slack bus (bus 1) is **excluded** from power balance — it represents the infinite grid and supplies whatever P and Q is needed.

### BESS State of Charge

$$E_{n,t} = E_{n,t-1} + \left(\eta_{ch} \cdot P_{ch,n,t} - \frac{1}{\eta_{dis}} \cdot P_{dis,n,t}\right) \cdot \Delta t$$

$$0 \leq E_{n,t} \leq E^{max}_n \qquad \text{(usable energy window)}$$

$$E_{n,23} = E_{n,init} \qquad \text{(daily cycle)}$$

**Important:** $E^{max}_n$ is the **usable** energy (20%–90% SOC window).  
Physical rated capacity $= E^{max}_n / 0.70$.  
Physical SOC for plotting $= 0.20 + (E_{n,t}/E^{max}_n) \times 0.70$

### Objective

$$\min \sum_{n \in \mathcal{N}} \sum_{t \in \mathcal{T}} P_{LS,n,t} \cdot \Delta t$$

---

## Repository Structure

```
.
├── Distribution-PVBESS.ipynb    # Main MILP notebook (Pyomo, HiGHS)
├── ieee33_visualization.html    # Interactive 33-bus feeder explorer (open in browser)
├── project_summary.docx         # Full debug & methodology report (Word)
├── README.md                    # This file
└── results/
    └── pv_bess_hosting_capacity_ieee33.csv   # Output after solving
```

### Notebook structure

| Cell | Content |
|------|---------|
| 0 | Title & fix log |
| 1–2 | IEEE 33-bus data: loads, branches, PV/load profiles, base-case voltage check |
| 3–4 | MILP formulation: variables, planning limits |
| 5 | Constraints (all linear, MILP-safe) |
| 6–7 | Solver call with robust termination check |
| 8 | Optional: base-case LP feasibility scanner |
| 9–10 | Results: siting, sizing, hosting capacity |
| 11–12 | Plots: voltage bar chart, SOC daily cycle |
| 13–14 | CSV export |
| 15–18 | Standalone diagnostics (voltage check, SOC check, SOC simulation) |

---

## Requirements

```bash
pip install pyomo numpy pandas matplotlib
```

**Solver (choose one):**

```bash
# Option 1 — HiGHS (recommended, free)
pip install highspy

# Option 2 — CBC (free)
conda install -c conda-forge coincbc

# Option 3 — Gurobi (commercial, free academic licence)
pip install gurobipy
```

**Tested environment:**
- Python 3.9+
- Pyomo 6.7
- HiGHS 1.11.0 (via `highspy`)
- Anaconda on Windows 11

---

## How to Run

1. **Clone the repository**

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
```

2. **Install dependencies**

```bash
pip install pyomo numpy pandas matplotlib highspy
```

3. **Open the notebook**

```bash
jupyter notebook thesis-Rai-improved.ipynb
```

4. **Run all cells in order** — do not skip cells; each cell depends on variables defined in the previous one.

   > **Important:** Always use **Kernel → Restart & Run All** after downloading a new version of the notebook. Jupyter caches old variable values in memory — running only the solver cell will use the broken old model, not the fixed one.

5. **View the interactive feeder map**  
   Open `ieee33_visualization.html` in any browser (Chrome, Edge, Firefox). No server needed — it runs entirely in the browser. Drag the hour slider to see how voltages change across the day.

---

## Key Results

After solving, the notebook reports:

- **HC_PV** — total PV hosting capacity (kW) across all selected buses
- **HC_BESS_P / HC_BESS_E** — BESS power (kW) and usable energy (kWh)
- **Rated BESS energy** = HC_BESS_E / 0.70 (physical capacity including buffer zones)
- **Voltage profile** at peak load hour (t = 19h)
- **SOC daily cycle** for each BESS site

Results are exported to `results/pv_bess_hosting_capacity_ieee33.csv`.

---

## Debugging Journey

This project went through an extensive debugging session. Eight critical bugs were identified and fixed. The full report is in [`project_summary.docx`](project_summary.docx). A quick summary:

| # | Bug | Symptom | Fix |
|---|-----|---------|-----|
| 1 | **Wrong Sbase** (1 MVA instead of 10 MVA) | Min voltage ~0.20 pu (should be 0.91 pu) | Set `Sbase_kW = 10000`, recompute `Zbase = 16.034 Ω` |
| 2 | **Bilinear SOC constraints** (`E <= SOC_max × Emax`, Var×Var) | `Presolve: Infeasible` at 0 nodes | Redefine `Emax` as usable energy; `E ∈ [0, Emax]` is linear |
| 3 | **Slack bus in Q-balance** | `Qflow[branch 1] = 0` forced, contradicts 0.23 pu system Q | `Constraint.Skip` for bus 1 in both P and Q balance |
| 4 | **Vmin = 0.95 pu** (base case reaches 0.9115 pu) | Infeasible without any DER | Set `Vmin = 0.90 pu` |
| 5 | **No reactive flow modelled** | Voltage drops underestimated; X·Q term missing | Added `Qflow` variables and full LinDistFlow |
| 6 | **No PV curtailment** | Optimizer forced to absorb all PV even if harmful | Added `Pcurt[n,t]` variable |
| 7 | **Mutex binaries** (u_ch/u_dis) | 1,584 extra binaries → infeasible | Removed; SOC dynamics implicitly penalise simultaneous charge/discharge |
| 8 | **SOC plot formula** (`E/Emax` shows 0–1, not 0.20–0.90) | SOC appears to violate bounds | `SOC = 0.20 + (E/Emax) × 0.70` |

---

## Parameters Reference

| Parameter | Value | Description |
|-----------|-------|-------------|
| `Sbase_kW` | 10,000 | System MVA base (kW) |
| `Vbase_kV` | 12.66 | Nominal line voltage (kV) |
| `Zbase_ohm` | 16.034 | Impedance base (Ω) |
| `Vmin` | 0.90 pu | Minimum allowable bus voltage |
| `Vmax` | 1.05 pu | Maximum allowable bus voltage |
| `Npv_max` | 5 | Maximum number of PV installation sites |
| `Nbess_max` | 3 | Maximum number of BESS installation sites |
| `PV_cap_max_kW` | 5,000 | Per-site PV capacity ceiling (kW) |
| `BESS_p_max_kW` | 2,000 | Per-site BESS power ceiling (kW) |
| `BESS_e_max_kWh` | 5,000 | Per-site BESS usable energy ceiling (kWh) |
| `SOC_min_frac` | 0.20 | Minimum state of charge |
| `SOC_max_frac` | 0.90 | Maximum state of charge |
| `eta_ch` | 0.95 | Charging efficiency |
| `eta_dis` | 0.95 | Discharging efficiency |
| `dt` | 1 h | Time step |

---

## Citation

If you use this code or methodology in your work, please cite:

```bibtex
@misc{rai2026pvbess,
  author       = {Rai, [Your Name]},
  title        = {PV and BESS Optimal Hosting Capacity Sizing on IEEE 33-Bus Distribution System},
  year         = {2026},
  howpublished = {\url{https://github.com/<your-username>/<repo-name>}},
  note         = {Thesis project, LinDistFlow MILP, Pyomo/HiGHS}
}
```

**Reference network:**
> Baran, M. E., & Wu, F. F. (1989). *Network reconfiguration in distribution systems for loss reduction and load balancing*. IEEE Transactions on Power Delivery, 4(2), 1401–1407.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

<div align="center">
  <sub>Built with Pyomo · HiGHS · IEEE 33-bus benchmark · LinDistFlow · March 2026</sub>
</div>
