# Walkthrough: USD Interest Rate Swap Bootstrapping Framework

We have updated the Jupyter Notebook [SOFR_Swap_Curve_Bootstrapping.ipynb](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/SOFR_Swap_Curve_Bootstrapping.ipynb) with:
1. **Unbroken Sequential Outline (Sections 1-10)**: Clean, logical section numbering across all markdown headers and code cells.
2. **User Entry Point Parameter (`USER_REFERENCE_DATE`)**: Added a top-level reference date parameter (`USER_REFERENCE_DATE = "2026-07-29"`, or any historical date like `"2025-10-15"`).
3. **Dynamic Variable Data Ingestion**: Replaced hardcoded constants with dynamic FRED API yield extraction and dynamic historical date offset calculations relative to the reference date.
4. **Multi-Horizon Tracking & Multi-Package Visualizations**: Computes Weekly, Monthly, Annual, and YTD changes (in basis points `bps`) and renders charts using `plotly`, `seaborn`, and `matplotlib`.

---

## 1. Sequential Notebook Outline (Sections 1 to 10)

- **Section 1**: Executive Summary & Market Context
- **Section 2**: Mathematical Framework & Hand-Calculated Step-by-Step Walkthrough
- **Section 3**: User Entry Point & Dynamic Data Ingestion Engine
- **Section 4**: Bootstrapping Core Engine Class (`SOFRSwapCurveBootstrapper`)
- **Section 5**: Single-Curve Bootstrapping & Table Output
- **Section 6**: Yield & Discount Curve Visualization Dashboard
- **Section 7**: Arbitrage-Free Par Swap Re-Pricing Verification
- **Section 8**: Multi-Horizon Historical Change Analysis (Weekly, Monthly, Annual, YTD)
- **Section 9**: Multi-Package Visualizations (`plotly`, `seaborn`, `matplotlib`)
- **Section 10**: Key Analytical Findings & Practical Applications

---

## 2. Dynamic Reference Date Entry Point (`USER_REFERENCE_DATE`)

Located in **Section 3** of the Jupyter notebook:

```python
# USER ENTRY POINT: Reference Date for FRED Data Retrieval
USER_REFERENCE_DATE = "2026-07-29"  # e.g., "2026-07-29", "2025-10-15", or None for Latest
```

When `USER_REFERENCE_DATE` is changed and the notebook is executed:
- The engine resolves the actual trading business day on or before `USER_REFERENCE_DATE` (e.g. `2026-07-28`).
- Yield quotes (`SOFR`, `DGS1`..`DGS30`) as of that exact date are fetched dynamically from FRED.
- Historical horizons (1W Ago, 1M Ago, 1Y Ago, YTD) relative to that reference date are dynamically constructed.
- All downstream discount factors, zero rates, forward rates, re-pricing checks, and plots update dynamically.

---

## 3. Dynamically Generated Multi-Horizon Change Tables

### Par Swap Rates (%) & Multi-Horizon Shifts (bps) [Ref Date: 2026-07-28]

| Tenor | Reference Date | 1W Ago | 1M Ago | 1Y Ago | YTD | 1W Chg (bps) | 1M Chg (bps) | 1Y Chg (bps) | YTD Chg (bps) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **ON** | 3.6500% | 3.6100% | 3.6200% | 4.3600% | 3.8700% | **+4.0 bps** | **+3.0 bps** | **-71.0 bps** | **-22.0 bps** |
| **1Y** | 4.1400% | 4.0800% | 3.9400% | 4.0900% | 3.4800% | **+6.0 bps** | **+20.0 bps** | **+5.0 bps** | **+66.0 bps** |
| **2Y** | 4.3100% | 4.2600% | 4.0700% | 3.9100% | 3.4700% | **+5.0 bps** | **+24.0 bps** | **+40.0 bps** | **+84.0 bps** |
| **3Y** | 4.3500% | 4.3100% | 4.0900% | 3.8700% | 3.5500% | **+4.0 bps** | **+26.0 bps** | **+48.0 bps** | **+80.0 bps** |
| **5Y** | 4.4000% | 4.3700% | 4.1200% | 3.9600% | 3.7300% | **+3.0 bps** | **+28.0 bps** | **+44.0 bps** | **+67.0 bps** |
| **7Y** | 4.5200% | 4.5000% | 4.2300% | 4.1800% | 3.9400% | **+2.0 bps** | **+29.0 bps** | **+34.0 bps** | **+58.0 bps** |
| **10Y** | 4.6500% | 4.6300% | 4.3800% | 4.4200% | 4.1800% | **+2.0 bps** | **+27.0 bps** | **+23.0 bps** | **+47.0 bps** |
| **20Y** | 5.1500% | 5.1400% | 4.8700% | 4.9500% | 4.7900% | **+1.0 bps** | **+28.0 bps** | **+20.0 bps** | **+36.0 bps** |
| **30Y** | 5.1200% | 5.1300% | 4.8700% | 4.9600% | 4.8400% | **-1.0 bps** | **+25.0 bps** | **+16.0 bps** | **+28.0 bps** |

---

## 4. Verification Results: Zero-PV Re-Pricing Sanity Check

| Tenor | Par Rate (%) | Fixed Leg PV | Floating Leg PV | Net PV (Error) | Status |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **1Y** | 4.1400% | 0.04028408 | 0.04028408 | `0.00e+00` | **PASSED (PV=0)** |
| **2Y** | 4.3100% | 0.08205133 | 0.08205133 | `0.00e+00` | **PASSED (PV=0)** |
| **3Y** | 4.3500% | 0.12155588 | 0.12155588 | `0.00e+00` | **PASSED (PV=0)** |
| **5Y** | 4.4000% | 0.19629169 | 0.19629169 | `0.00e+00` | **PASSED (PV=0)** |
| **7Y** | 4.5200% | 0.27018880 | 0.27018880 | `0.00e+00` | **PASSED (PV=0)** |
| **10Y** | 4.6500% | 0.37147360 | 0.37147360 | `0.00e+00` | **PASSED (PV=0)** |
| **20Y** | 5.1500% | 0.65195453 | 0.65195453 | `0.00e+00` | **PASSED (PV=0)** |
| **30Y** | 5.1200% | 0.78733414 | 0.78733414 | `0.00e+00` | **PASSED (PV=0)** |

**Maximum Net Error**: $2.22 \times 10^{-16}$ (Exact Zero to Floating Point Precision).

---

## 5. Visualizations

- **Plotly Interactive HTML**: [sofr_plotly_interactive.html](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/sofr_plotly_interactive.html)
- **Seaborn Heatmap**: `sofr_term_structure_changes_seaborn.png`
- **Matplotlib Dashboard**: `sofr_term_structure_changes_matplotlib.png`

---

## 6. Deliverable Files

- **Jupyter Notebook**: [SOFR_Swap_Curve_Bootstrapping.ipynb](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/SOFR_Swap_Curve_Bootstrapping.ipynb)
- **Interactive Plotly HTML**: [sofr_plotly_interactive.html](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/sofr_plotly_interactive.html)
- **Implementation Plan**: [implementation_plan.md](file:///Users/chrishsieh/.gemini/antigravity/brain/4b3af5f1-8c0e-43b6-8478-8dba4a356ca8/implementation_plan.md)
