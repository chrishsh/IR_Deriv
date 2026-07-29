# USD Interest Rate Swap (SOFR) Bootstrapping Implementation Plan

This plan outlines the theoretical framework, step-by-step mathematical methodology, user-configurable Reference Date entry point, dynamic data binding, multi-horizon curve analytics (Weekly, Monthly, Annual, YTD), multi-library visualizations (`plotly`, `matplotlib`, `seaborn`), and Jupyter Notebook layout structure for bootstrapping a USD Secured Overnight Financing Rate (SOFR) yield and discount curve.

---

## 1. Overview & Theoretical Research: USD SOFR Swap Bootstrapping

### 1.1 Post-LIBOR Paradigm & The SOFR Framework
Following global reference rate reform, the Secured Overnight Financing Rate (**SOFR**) established by the Federal Reserve Bank of New York replaced USD LIBOR as the primary risk-free benchmark for USD derivatives. 
- **Single-Curve vs. Multi-Curve**: In the legacy LIBOR era, dual-curve bootstrapping was mandatory (LIBOR for floating leg projections, OIS/Fed Funds for discounting). In the SOFR ecosystem, SOFR OIS swaps pay fixed rates versus daily compounded SOFR. Consequently, the SOFR curve serves as **both** the discounting curve and the projection curve.

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
- **Continuously Compounded Zero Rate**: $R_{\text{cont}}(0, T) = -\frac{\ln P(0, T)}{T}$
- **Money Market (Act/360) Zero Rate**: $R_{\text{mm}}(0, T) = \left( \frac{1}{P(0, T)} - 1 \right) \times \frac{360}{\text{Days}(0, T)}$
- **Instantaneous Forward Rate / Period Forward Rate**: $f(T_a, T_b) = \left( \frac{P(0, T_a)}{P(0, T_b)} - 1 \right) \times \frac{1}{\tau(T_a, T_b)}$

---

## 2. Explicit Markdown Outline Structure (Sections 1 through 10)

To ensure that Jupyter Notebook's built-in **Table of Contents / Outline Sidebar** renders every section cleanly, each section will have a dedicated **Markdown Cell** (`## Section Number. Section Name`) directly preceding its corresponding code cell:

1. **Markdown Cell 1**: `# USD Interest Rate Swap (SOFR) Bootstrapping Framework` & `## 1. Executive Summary & Market Context`
2. **Markdown Cell 2**: `## 2. Mathematical Framework & Hand-Calculated Step-by-Step Walkthrough`
3. **Code Cell 3**: Environment Setup & Self-Healing Library Auto-Installer
4. **Markdown Cell 4**: `## 3. User Entry Point & Dynamic Data Ingestion Engine`
5. **Code Cell 5**: Reference Date Input (`USER_REFERENCE_DATE`) & FRED API Extraction
6. **Markdown Cell 6**: `## 4. Bootstrapping Core Engine Class`
7. **Code Cell 7**: `SOFRSwapCurveBootstrapper` Class Implementation
8. **Markdown Cell 8**: `## 5. Single-Curve Bootstrapping & Table Output`
9. **Code Cell 9**: Bootstrapping Execution & DataFrame Summary Output
10. **Markdown Cell 10**: `## 6. Yield & Discount Curve Visualization Dashboard`
11. **Code Cell 11**: Matplotlib Discount Factor, Zero Rate, and 1Y Forward Curve Dashboard
12. **Markdown Cell 12**: `## 7. Arbitrage-Free Par Swap Re-Pricing Verification`
13. **Code Cell 13**: Par Swap Re-Pricing Net Error Verification Table
14. **Markdown Cell 14**: `## 8. Multi-Horizon Historical Change Analysis (Weekly, Monthly, Annual, YTD)`
15. **Code Cell 15**: Multi-Date Historical Horizon Extraction & Basis Point Changes Calculation
16. **Markdown Cell 16**: `## 9. Multi-Package Visualizations (Plotly, Seaborn, Matplotlib)`
17. **Code Cell 17**: Plotly Interactive Dashboard, Seaborn Heatmaps, and Matplotlib Multi-Panel Plot
18. **Markdown Cell 18**: `## 10. Key Analytical Findings & Practical Applications`

---

## 3. Verification Plan

### Automated Tests
1. **Jupyter Outline Verification**: Inspect generated `.ipynb` file to confirm all 10 Markdown header cells exist and display in Jupyter Table of Contents.
2. **Top-to-Bottom Cell Execution**: Execute all 18 cells (9 Markdown cells + 9 Code cells) in Python.
3. **Documentation Sync**: Update [walkthrough.md](file:///Users/chrishsieh/.gemini/antigravity/brain/4b3af5f1-8c0e-43b6-8478-8dba4a356ca8/walkthrough.md) with the new cell layout.
