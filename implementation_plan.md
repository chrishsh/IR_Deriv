# USD Interest Rate Swap (SOFR) Bootstrapping Implementation Plan

This plan outlines the theoretical framework, step-by-step mathematical methodology, user-configurable Reference Date entry point, dynamic data binding, multi-horizon curve analytics (Weekly, Monthly, Annual, YTD), multi-library visualizations (`plotly`, `matplotlib`, `seaborn`), and Python code implementation for bootstrapping a USD Secured Overnight Financing Rate (SOFR) yield and discount curve.

---

## 1. Overview & Theoretical Research: USD SOFR Swap Bootstrapping

### 1.1 Post-LIBOR Paradigm & The SOFR Framework
Following global reference rate reform, the Secured Overnight Financing Rate (**SOFR**) established by the Federal Reserve Bank of New York replaced USD LIBOR as the primary risk-free benchmark for USD derivatives. 

### 1.2 Fixed vs. Floating Leg Valuation
A standard fixed-for-floating USD SOFR Overnight Index Swap (OIS) has two legs:
1. **Fixed Leg**: Pays a fixed swap rate $S_N$ at agreed coupon dates $T_i$ ($i = 1, \dots, N$).
   $$\text{PV}_{\text{fixed}} = S_N \sum_{i=1}^{N} \tau_{\text{fixed}, i} \cdot P(0, T_i)$$
   where $\tau_{\text{fixed}, i}$ is the day-count fraction for period $i$ (commonly **Actual/360** or **30/360**) and $P(0, T_i)$ is the discount factor at $T_i$.

2. **Floating Leg**: Pays daily compounded SOFR over each calculation period. By arbitrage-free pricing, the present value of a floating leg paying daily compounded SOFR with no spread is:
   $$\text{PV}_{\text{float}} = P(0, T_0) - P(0, T_N) = 1 - P(0, T_N) \quad (\text{assuming } T_0 = 0 \text{ and } P(0,0)=1)$$

3. **Par Swap Condition**:
   At swap inception at par, $\text{PV}_{\text{fixed}} = \text{PV}_{\text{float}}$:
   $$1 - P(0, T_N) = S_N \sum_{i=1}^{N} \tau_{\text{fixed}, i} \cdot P(0, T_i)$$

### 1.3 Exact Bootstrapping Formula
Solving for the terminal discount factor $P(0, T_N)$ recursively gives:
$$P(0, T_N) = \frac{1 - S_N \sum_{i=1}^{N-1} \tau_{\text{fixed}, i} \cdot P(0, T_i)}{1 + S_N \cdot \tau_{\text{fixed}, N}}$$

From the bootstrapped discount factors $P(0, T)$, we derive:
- **Continuously Compounded Zero Rate**:
  $$R_{\text{cont}}(0, T) = -\frac{\ln P(0, T)}{T}$$
- **Money Market (Act/360) Zero Rate**:
  $$R_{\text{mm}}(0, T) = \left( \frac{1}{P(0, T)} - 1 \right) \times \frac{360}{\text{Days}(0, T)}$$
- **Instantaneous Forward Rate / Period Forward Rate**:
  $$f(T_a, T_b) = \left( \frac{P(0, T_a)}{P(0, T_b)} - 1 \right) \times \frac{1}{\tau(T_a, T_b)}$$

---

## 2. User Entry Point Parameterization & Dynamic Data Architecture

### 2.1 User Reference Date Input (`USER_REFERENCE_DATE`)
To enable dynamic historical refresh and parameterization, the top of the Jupyter notebook will introduce a single user input parameter:
```python
USER_REFERENCE_DATE = "2026-07-29"  # e.g., "2026-07-29", "2025-10-15", or None for Latest
```
When `USER_REFERENCE_DATE` is changed, executing the notebook will dynamically:
1. Locate the closest valid trading business day on or before `USER_REFERENCE_DATE` (`REF_DATE_ACTUAL`).
2. Query FRED API for market yield quotes (`SOFR`, `DGS1`..`DGS30`) as of `REF_DATE_ACTUAL`.
3. Compute dynamic historical horizon dates relative to `REF_DATE_ACTUAL`:
   - 1-Week Ago: `REF_DATE_ACTUAL - 7 days`
   - 1-Month Ago: `REF_DATE_ACTUAL - 30 days`
   - 1-Year Ago: `REF_DATE_ACTUAL - 365 days`
   - YTD: Prior Year-End relative to `REF_DATE_ACTUAL` (e.g. `(REF_DATE_ACTUAL.year - 1)-12-31`)
4. Drive all downstream calculations, curves, tables, re-pricing checks, and plots dynamically with **zero hardcoded constants**.

---

## 3. Notebook Structure & Sequential Outline Scheme (Sections 1-10)

The notebook markdown headers will follow a clean, unbroken sequential outline:

1. **Section 1**: Executive Summary & Market Context
2. **Section 2**: Mathematical Framework & Hand-Calculated Step-by-Step Walkthrough
3. **Section 3**: User Entry Point & Dynamic Data Ingestion Engine
4. **Section 4**: Bootstrapping Core Engine Class (`SOFRSwapCurveBootstrapper`)
5. **Section 5**: Single-Curve Bootstrapping & Table Output
6. **Section 6**: Yield & Discount Curve Visualization Dashboard
7. **Section 7**: Arbitrage-Free Par Swap Re-Pricing Verification
8. **Section 8**: Multi-Horizon Historical Change Analysis (Weekly, Monthly, Annual, YTD)
9. **Section 9**: Multi-Package Visualizations (`plotly`, `seaborn`, `matplotlib`)
10. **Section 10**: Key Analytical Findings & Practical Applications

---

## 4. Verification Plan

### Automated Tests
1. **Dynamic Reference Date Test**: Test execution with multiple reference dates (e.g. `"2026-07-29"`, `"2025-10-15"`).
2. **Zero-Arbitrage Re-pricing Check**: Assert that re-pricing net error remains $< 10^{-15}$ across all reference dates.
3. **Outline Integrity Test**: Confirm markdown header section numbers are sequentially numbered 1 through 10.
4. **Documentation Sync**: Update [walkthrough.md](file:///Users/chrishsieh/.gemini/antigravity/brain/4b3af5f1-8c0e-43b6-8478-8dba4a356ca8/walkthrough.md) to reflect parameterization and outline updates.
