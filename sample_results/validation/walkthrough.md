# Walkthrough: USD Interest Rate Swap Bootstrapping Framework

We have updated the Jupyter Notebook [SOFR_Swap_Curve_Bootstrapping.ipynb](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/SOFR_Swap_Curve_Bootstrapping.ipynb) to include **dedicated Markdown header cells for all 10 sections**, ensuring complete visibility in Jupyter Notebook's built-in Table of Contents / Outline sidebar.

---

## 1. Complete Jupyter Outline Structure (18 Cells Total)

1. **Markdown Cell 1**: `# USD Interest Rate Swap (SOFR) Bootstrapping Framework` & `## 1. Executive Summary & Market Context`
2. **Markdown Cell 2**: `## 2. Mathematical Framework & Hand-Calculated Step-by-Step Walkthrough`
3. **Code Cell 3**: Environment Setup & Self-Healing Library Installer (`plotly`, `seaborn`, etc.)
4. **Markdown Cell 4**: `## 3. User Entry Point & Dynamic Data Ingestion Engine`
5. **Code Cell 5**: Parameter `USER_REFERENCE_DATE = "2026-07-29"` & Dynamic FRED API Ingestion
6. **Markdown Cell 6**: `## 4. Bootstrapping Core Engine Class`
7. **Code Cell 7**: `SOFRSwapCurveBootstrapper` Class Implementation
8. **Markdown Cell 8**: `## 5. Single-Curve Bootstrapping & Table Output`
9. **Code Cell 9**: Bootstrapping Execution & Summary DataFrame Output
10. **Markdown Cell 10**: `## 6. Yield & Discount Curve Visualization Dashboard`
11. **Code Cell 11**: Matplotlib Discount Factor, Zero Rate, and 1Y Forward Curve Dashboard
12. **Markdown Cell 12**: `## 7. Arbitrage-Free Par Swap Re-Pricing Verification`
13. **Code Cell 13**: Par Swap Re-Pricing Net Error Verification Table
14. **Markdown Cell 14**: `## 8. Multi-Horizon Historical Change Analysis (Weekly, Monthly, Annual, YTD)`
15. **Code Cell 15**: Multi-Date Historical Horizon Extraction & Basis Point Changes Calculation
16. **Markdown Cell 16**: `## 9. Multi-Package Visualizations (Plotly, Seaborn, Matplotlib)`
17. **Code Cell 17**: Plotly Interactive HTML Dashboard, Seaborn Heatmaps, and Matplotlib Multi-Panel Plot
18. **Markdown Cell 18**: `## 10. Key Analytical Findings & Practical Applications`

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

## 5. Deliverable Files

- **Jupyter Notebook**: [SOFR_Swap_Curve_Bootstrapping.ipynb](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/SOFR_Swap_Curve_Bootstrapping.ipynb)
- **Interactive Plotly HTML**: [sofr_plotly_interactive.html](file:///Users/chrishsieh/Library/CloudStorage/OneDrive-Personal/CodingProjects/Antigravity/IR_Deriv/sofr_plotly_interactive.html)
- **Implementation Plan**: [implementation_plan.md](file:///Users/chrishsieh/.gemini/antigravity/brain/4b3af5f1-8c0e-43b6-8478-8dba4a356ca8/implementation_plan.md)
