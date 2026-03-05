# Production Spec (v2): Risk Decomposition with Separate Equity-Barra + FX + Commodity Futures Sleeves  
*(Designed for production use, explicit conventions, and reconciliation with Barra-style equity risk reports.)*

> **Core idea**:  
> 1) Compute **Equity sleeve** risk using a Barra-style factor model (CFR/SR/Total) with strict alignment.  
> 2) Compute **FX sleeve** risk separately (currency-factor model or realized sleeve covariance).  
> 3) Compute **Commodity futures sleeve** risk separately (commodity-factor model or realized sleeve covariance).  
> 4) Combine sleeves at the **variance** level using **sleeve covariances** (recommended) rather than summing volatilities.

---

## 0) Why Separate Sleeves (and What You Lose)

### Why this is production-safe (phase 1)
- Equity Barra models often **do not** have correct exposures/covariance for FX + commodities.
- Avoids **bad proxies** (e.g., mapping WTI futures into “OILGAS” equity industry).
- Enables near-exact reconciliation with a vendor “Portfolio Manager” equity report.

### What separation can miss
- Cross-sleeve hedging/diversification (equity vs FX hedges; equity vs oil correlation).
- “Total risk” becomes ambiguous unless you add sleeve-level covariance terms.

**Production requirement**: always document whether “Portfolio Total” includes sleeve covariances.

---

## 1) Global Definitions

There are **three sleeves**:

1. **Equity sleeve (EQ)**: Barra-style multi-factor model (style/industry/country/currency if in your model).
2. **FX sleeve (FX)**: FX spot/forward swaps, FX futures, or currency overlay.
3. **Commodity sleeve (CMD)**: commodity futures (energy, metals, ags) and related derivatives.

Let `s ∈ {EQ, FX, CMD}` denote a sleeve.

Portfolio total variance (recommended):
\[
Var_{total} = \sum_s Var_s + 2\sum_{s<t} Cov(s,t)
\]

---

## 2) Inputs & Data Contracts

### 2.1 Common inputs (all sleeves)
- `asof` timestamp
- `base_ccy` (e.g., USD)
- `ann_factor` (default 252; configurable)
- Portfolio metadata (strategy name, book, etc.)

### 2.2 Equity sleeve inputs (Barra-style)
Required per asset `i` in EQ:
- `id` (unique)
- `qty`, `px`, `tradeFactor`, `fxRate` (local→base)
- `B_eq`: exposures (`N_eq × K_eq`)
- `F_eq`: factor covariance (`K_eq × K_eq`), **daily** unless specified
- `sigma_spec_eq`: specific volatility per asset (daily or annual, must specify)

### 2.3 FX sleeve inputs (choose one approach)

#### Option A (preferred for control): Currency-factor model
- Currency exposure vector by currency (e.g., USD, EUR, JPY…): `c` (dimension `K_fx`)
- Currency covariance matrix `F_fx` (daily)
- Optional: residual/specific FX risk (often 0 if factors fully span FX)

#### Option B (fastest): Realized sleeve return covariance
- Time series of FX sleeve returns or PnL normalized by AUM:
  - `r_fx[t]` for t in trailing window (e.g., 60–252 days)
- Then `Var_fx` is realized variance (daily) and `Cov(EQ,FX)` realized covariance

**Production requirement**: store window length and dates used.

### 2.4 Commodity sleeve inputs (choose one approach)

#### Option A: Commodity-factor model
- Exposure vector to commodity return factors (WTI, Brent, Gold, Copper…): `g` (dimension `K_cmd`)
- Commodity factor covariance `F_cmd` (daily)

#### Option B: Realized sleeve return covariance
- Time series of commodity sleeve returns: `r_cmd[t]`

---

## 3) Normalization & Units (Must Be Explicit)

### 3.1 Define position exposure in base currency
For any instrument:
\[
dlrPos_i = qty_i \cdot px_i \cdot tradeFactor_i \cdot fxRate_i
\]

**For futures**: `px` is futures price; `tradeFactor` must include contract multiplier.  
Do NOT use margin as exposure.

### 3.2 Define reporting denominator
For multi-asset portfolios, **use AUM/NAV** as the primary denominator:

- `AUM` provided externally for the as-of date
- sleeve weights: `w_i = dlrPos_i / AUM`

**If you must match an equity Barra report** that uses gross/2 normalization, do so **only inside EQ sleeve**, and report it separately:

- `NF_eq = (|Long_eq| + |Short_eq|)/2`
- `w_eq_i = dlrPos_i / NF_eq`

**Production outputs must include**:
- `AUM`, `NF_eq`, `Long_eq`, `Short_eq`, `Gross_eq`, `Net_eq`

---

## 4) Equity Sleeve (EQ) — Barra-Style Computation

### 4.1 Alignment requirements (hard)
- Exposure factor ordering must match covariance ordering exactly:
  - `B_eq.columns == F_eq.index == F_eq.columns`
- Asset alignment:
  - `B_eq.index` aligns to positions index after filtering to EQ universe

### 4.2 Compute EQ weights
Two modes:
- `EQ_NORMALIZE = "AUM"`: `w_eq = dlrPos_eq / AUM`
- `EQ_NORMALIZE = "GROSS_OVER_2"`: `w_eq = dlrPos_eq / NF_eq`  *(for vendor reconciliation)*

### 4.3 Portfolio factor exposure
\[
x_{eq} = B_{eq}^T w_{eq}
\]

### 4.4 Factor variance
\[
Var_{eq,factor} = x_{eq}^T F_{eq} x_{eq}
\]

### 4.5 Specific variance
If `sigma_spec_eq` is annual, convert to daily:
\[
\sigma_{spec,daily} = \sigma_{spec,annual}/\sqrt{A}
\]
Then:
\[
Var_{eq,spec} = \sum_i w_{eq,i}^2 \sigma_{spec,i}^2
\]

### 4.6 EQ total variance
\[
Var_{eq} = Var_{eq,factor} + Var_{eq,spec}
\]

### 4.7 Annualization
\[
\sigma_{eq,annual} = \sqrt{Var_{eq}} \cdot \sqrt{A}
\]

Also compute:
- `CFR_eq_annual = sqrt(Var_eq,factor)*sqrt(A)`
- `SR_eq_annual = sqrt(Var_eq,spec)*sqrt(A)`

---

## 5) FX Sleeve (FX)

### 5.1 Option A: currency-factor model
Define currency factor exposure vector (dimension `K_fx`):
- `c = exposures_by_currency / AUM` (recommended)
Then:
\[
Var_{fx} = c^T F_{fx} c
\]

### 5.2 Option B: realized sleeve risk
Given daily return series `r_fx[t]`:
\[
Var_{fx} = Var(r_{fx})
\]

---

## 6) Commodity Sleeve (CMD)

### 6.1 Option A: commodity-factor model
Exposure vector:
- `g = exposures_to_commodities / AUM`
Then:
\[
Var_{cmd} = g^T F_{cmd} g
\]

### 6.2 Option B: realized sleeve risk
Given daily return series `r_cmd[t]`:
\[
Var_{cmd} = Var(r_{cmd})
\]

---

## 7) Sleeve Covariances (Recommended for Portfolio Total)

### 7.1 Realized covariance method (practical)
Compute daily return series for each sleeve on the same dates:
- `r_eq[t]`: equity sleeve return (or modeled return proxy)
- `r_fx[t]`: FX sleeve return
- `r_cmd[t]`: commodity sleeve return

Then:
- `Cov(eq,fx) = Cov(r_eq, r_fx)`
- `Cov(eq,cmd) = Cov(r_eq, r_cmd)`
- `Cov(fx,cmd) = Cov(r_fx, r_cmd)`

### 7.2 Portfolio total variance (with covariance)
\[
Var_{total} = Var_{eq} + Var_{fx} + Var_{cmd} + 2Cov(eq,fx) + 2Cov(eq,cmd) + 2Cov(fx,cmd)
\]

If covariances are not available (phase 0), document assumption:
- `Cov=0` (conservative-ish but can be wrong)

---

## 8) Outputs (What Production Must Return)

### 8.1 Summary
Return a dict with:
- global:
  - `asof`, `base_ccy`, `ann_factor`, `AUM`
- equity normalization diagnostics:
  - `EQ_NORMALIZE`, `NF_eq`, `Long_eq`, `Short_eq`, `Gross_eq`, `Net_eq`
- sleeve risks (daily and annual):
  - `Var_eq`, `Var_fx`, `Var_cmd`
  - `Vol_eq_annual`, `Vol_fx_annual`, `Vol_cmd_annual`
  - `CFR_eq_annual`, `SR_eq_annual`
- portfolio total:
  - `Var_total` (with/without cov terms)
  - `Vol_total_annual`

### 8.2 EQ factor exposure tables
- dollar factor exposure: `df_eq = B_eqᵀ dlrPos_eq`
- normalized factor exposure: `x_eq = df_eq / denom` (AUM or NF_eq)
Include factor group labels (style/industry/country/currency).

### 8.3 Industry exposure table (EQ)
For industry factor subset:
- `industry_dollar = B_indᵀ dlrPos_eq`

### 8.4 Coverage diagnostics
- assets missing exposures
- assets missing spec risk
- %MV covered
- policy applied (error/drop/proxy)

### 8.5 Sleeve return diagnostics (if using realized cov)
- window length
- start/end dates
- data completeness

---

## 9) Engineering Requirements

### 9.1 API surface
```python
compute_multi_sleeve_risk(
    positions_df,
    equity_exposure_df,
    equity_cov_df,
    equity_spec_risk,
    *,
    aum,
    eq_normalize="GROSS_OVER_2|AUM",
    fx_mode="factor|realized",
    cmd_mode="factor|realized",
    fx_inputs=...,
    cmd_inputs=...,
    realized_window=126,
    ann_factor=252,
) -> dict
```

### 9.2 Strict alignment & safety
- enforce factor ordering
- enforce covariance symmetry checks
- clip tiny negative variances before sqrt

### 9.3 Auditability
Log:
- model version and factor list hash
- normalization mode + denominators
- sleeve definitions and membership rules
- final risk numbers + key intermediates

---

## 10) Reconciliation Guidance (Vendor vs Multi-Asset “Truth”)

### If your goal is **vendor parity** for EQ sleeve:
- use `eq_normalize="GROSS_OVER_2"`
- ensure `F_eq` and `sigma_spec_eq` frequency matches vendor
- match `ann_factor` (252 vs 250)
- ensure factor set matches vendor report (including currency if vendor includes it)

### If your goal is **portfolio truth** across sleeves:
- prefer AUM normalization
- include sleeve covariances via realized returns
- treat EQ Barra as one sleeve, not “total truth”

---

## 11) Hidden Assumptions Removed (Now Explicit)
This spec forces explicit choices for:
- denominators (NF_eq vs AUM)
- FX/commodity modeling approach (factor vs realized)
- inclusion of sleeve covariances
- futures exposure definition (notional vs MV)
- asset coverage policy (error/drop/proxy)
- annualization constant

---

## 12) Deliverable Request to a Coding AI

Implement production-quality Python that:
- computes EQ Barra CFR/SR/Total with strict checks
- computes FX and CMD sleeve variances with chosen mode(s)
- optionally computes sleeve covariances using realized returns
- produces a complete output dict + diagnostics
- supports reconciliation mode returning intermediate vectors (`w_eq`, `x_eq`, `Var_eq,factor`, `Var_eq,spec`, denominators)

