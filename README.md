# Marketing Budget Allocation Optimization (LP + MIP, Gurobi)

Allocate a fixed marketing budget across multiple channels to **maximize total return** when each channel has **tiered (piecewise) ROI curves**, while satisfying real-world business constraints. The project implements:

- **Linear Programming (LP)** when ROI tiers are **concave / non-increasing**
- **Mixed Integer Programming (MIP)** when ROI tiers are **non-concave** (tiers can improve later)
- **Robustness checks** when ROI assumptions change (cross-scenario evaluation)
- **Minimum investment (activation) rules** per channel
- **Multi-period monthly planning** with **reinvestment dynamics**
- A **stability analysis** of month-to-month spend volatility

---

## Problem Summary

Given a marketing budget (default: **$10M**) and ROI curves for each platform (given as spend tiers), compute the spend allocation that maximizes total return subject to:

- Total spend ≤ budget
- Per-platform spend cap (≤ **$3M**)
- Business constraints, including:
  - **Print + TV ≤ Facebook + Email**
  - **Social (FB, LinkedIn, Instagram, Snapchat, Twitter) ≥ 2 × (SEO + AdWords)**

The project repeats the optimization under different ROI sources and extends it with minimum-spend and monthly reinvestment logic.

---

## Data Files (CSV Inputs)

- `roi_company1.csv` — ROI tiers where ROI is non-increasing (concave case → LP)
- `roi_company2.csv` — ROI tiers that can be non-concave (requires MIP)
- `min_amount.csv` — minimum spend per platform if activated (0 or ≥ minimum)
- `roi_monthly.csv` — month-by-month ROI tiers for multi-period planning

---

## Approach

### 1) LP Model (Concave ROI)
When ROIs are non-increasing across tiers, the return function is concave and can be solved with an LP:
- Decision variables: spend per (platform, tier)
- Concavity implies “fill earlier tiers first” behavior without binaries
- Solved using `gurobipy`

### 2) MIP Model (Non-Concave ROI)
When later tiers may have higher ROI than earlier tiers, the LP can incorrectly “skip” tiers.
To enforce valid tier activation, we formulate a MIP:
- Binary variables select the active tier per platform
- Continuous variables determine the spend within the selected tier (convex interpolation)
- Same business constraints as the LP case

### 3) Robustness (Cross-Scenario Evaluation)
Evaluate how the *allocation from one ROI source* performs under the *other ROI source* to quantify downside risk if the ROI assumptions are wrong.

### 4) Minimum Investment Rule
Add an activation constraint per platform:
- either spend = 0
- or spend ≥ minimum threshold from `min_amount.csv`

### 5) Monthly Planning with Reinvestment
For each month:
- Solve the (monthly) optimization using that month’s ROI tiers
- Update the next month’s budget using reinvestment:
  - `Budget_next = 10M + 0.5 * Return_current` (as specified)

### 6) Stability Check (Post-Analysis)
A plan is “stable” if each platform changes by at most **$1M** month-to-month.
This project checks stability and outlines how to enforce it via additional linear constraints.

---

## Setup

### Prerequisites
- Python 3.9+
- A working **Gurobi** installation + license
- Core Python packages: `gurobipy`, `pandas`, `numpy`, `matplotlib` (optional: `seaborn` for heatmaps)

### Install (example)
```bash
pip install pandas numpy matplotlib seaborn
# Install gurobipy following Gurobi’s instructions for your OS / license
````

---

## Run

Open the notebook and run all cells top-to-bottom. Ensure the first cell reads the CSVs from `data/`.

---

## Outputs

The code prints:

* Platform-level allocation (spend, return, ROI)
* Tier-level allocation details
* Constraint feasibility checks

And generates plots such as:

* LP vs MIP platform allocation comparison
* Monthly allocation heatmap
* Month-to-month spend change (Δ) heatmap for stability checking

---

### Report-backed numeric results 
- **LP (Company 1, concave ROI) optimal allocation**: TV $3M, Instagram $3M, Email $3M, AdWords $1M; **Total Return $543,640; Overall ROI 5.44%**. 
- **MIP (Company 2, non-concave ROI) optimal allocation**: Print $3M, AdWords $2.333M, Facebook $3M, LinkedIn $1.667M; **Total Return $452,827; Overall ROI 4.53%**. 
- **Cross-scenario loss** (using the “wrong” allocation under the other ROI model) shows large degradations in expected return (table in report).  
- **Minimum investment rule (Part 6)**: adding platform minimums leaves the optimal MIP allocation unchanged and objective unchanged (still **$0.453M return, 4.53% ROI** under Company 2). 
- **Monthly reinvestment planning (Part 7)**: yearly total return and ROI summary is reported (including relaxed-vs-minimum comparison).
- **Stability check (Part 8)**: allocation is **not stable**, with max monthly change **$3.000M** and **33** month-platform violations; report also outlines how to linearize the stability constraint. 
