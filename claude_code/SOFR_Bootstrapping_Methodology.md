# USD SOFR Swap Curve Bootstrapping — Methodology, Worked Example & Implementation Plan

**Scope:** Research notes, a hand-calculated step-by-step example, and a detailed implementation
plan for bootstrapping a USD SOFR discount/zero curve from market data, with the short end taken
live from FRED. Companion notebook: `SOFR_Swap_Curve_Bootstrapping_claude_code.ipynb`.

---

## 1. Research: the USD SOFR swap market

### 1.1 SOFR replaced LIBOR as the USD benchmark

The Secured Overnight Financing Rate (**SOFR**), published daily by the Federal Reserve Bank of
New York, is a volume-weighted median of overnight Treasury repo transactions (~$2 trillion/day
of underlying volume). After USD LIBOR's cessation (final panel June 2023), SOFR became the
standard floating benchmark for USD interest rate swaps.

The standard instrument is the **SOFR Overnight Index Swap (OIS)**: a fixed rate is exchanged
against **daily compounded SOFR** paid in arrears.

### 1.2 Single-curve framework

This is the key structural simplification versus the LIBOR era:

| Era | Projection curve | Discount curve | Bootstrapping |
|---|---|---|---|
| LIBOR (pre-2022) | LIBOR 3M curve | OIS (Fed Funds) curve | Dual-curve, joint |
| SOFR (today) | SOFR curve | SOFR curve | **Single-curve, self-consistent** |

Because cleared SOFR swaps are discounted at SOFR (CME/LCH PAI) *and* their floating leg pays
compounded SOFR, one curve serves both roles. The bootstrap is therefore self-contained: the
same discount factors that discount the cash flows also determine the floating-leg value.

### 1.3 Market conventions for USD SOFR OIS (standard cleared swaps)

| Item | Convention |
|---|---|
| Fixed leg | **Annual** payments, **Act/360** day count |
| Floating leg | Daily compounded SOFR in arrears, annual payments, Act/360 |
| Settlement | T+2 (spot-starting) |
| Business days | SIFMA/US government securities calendar, modified following |
| Compounding | Geometric daily: $\left[\prod_d (1 + r_d \cdot n_d/360) - 1\right]\cdot \frac{360}{D}$ |

The notebook simplifies two of these (spot lag ignored; weekend-only calendar) — see §5.5.

### 1.4 What FRED does and does not provide (verified via the API)

A `fred/series/search?search_text=SOFR` query returns exactly these usable series:

| FRED series | Content | Use in this project |
|---|---|---|
| `SOFR` | Overnight SOFR fixing (%) | Anchors the overnight node |
| `SOFR30DAYAVG` | 30-day compounded average (%) | 1M node (approximation, see below) |
| `SOFR90DAYAVG` | 90-day compounded average (%) | 3M node (approximation) |
| `SOFR180DAYAVG` | 180-day compounded average (%) | 6M node (approximation) |
| `SOFRINDEX` | Cumulative compounded index | (not needed here) |

**FRED does *not* publish SOFR OIS par swap rates** (1Y–30Y). Those are distributed by
CME/Bloomberg/LSEG under license. Two honest ways to complete the long end, both supported by
the notebook via a `SWAP_QUOTE_SOURCE` switch:

1. **`MANUAL`** — paste true SOFR OIS par quotes from a licensed source into a dictionary.
   This is the *correct* production path.
2. **`TREASURY_PROXY`** — fetch Treasury constant-maturity yields (`DGS1`…`DGS30`) from FRED and
   apply a user-configurable per-tenor **swap-spread adjustment** (SOFR swaps trade *below*
   Treasury yields at most maturities in recent years; the spread widens with maturity). This
   keeps the notebook fully live/self-updating, at the cost of a documented approximation.

Two approximations to keep in mind when using FRED-only data:

- The 30/90/180-day averages are **backward-looking** (realized SOFR over the *past* window),
  while curve nodes need **forward-looking** expectations. They coincide only when policy
  expectations are flat; near FOMC moves they can differ by tens of basis points.
- Treasury CMT yields are semiannual bond-equivalent yields on a different (govt credit/repo
  specialness) basis than Act/360 annual OIS — the spread adjustment absorbs this only roughly.

---

## 2. The mathematics of the bootstrap

### 2.1 Pricing a SOFR OIS off a discount curve

Let $P(0,T)$ be the discount factor to time $T$, with $P(0,0)=1$. For a spot-starting swap with
fixed rate $S_N$, annual fixed payment dates $T_1 < T_2 < \dots < T_N$ and Act/360 accruals
$\tau_i = \frac{\text{days}(T_{i-1},T_i)}{360}$ on notional 1:

**Fixed leg**
$$\text{PV}_\text{fixed} = S_N \sum_{i=1}^{N} \tau_i \, P(0,T_i)$$

**Floating leg** — the telescoping property of compounded overnight rates makes each period's
expected payoff equal to $P(0,T_{i-1}) - P(0,T_i)$ in the single-curve framework, so the whole
leg collapses to
$$\text{PV}_\text{float} = P(0,T_0) - P(0,T_N) = 1 - P(0,T_N).$$

Intuition: receiving compounded overnight SOFR is economically identical to rolling $1 at the
overnight rate — worth $1 today — and returning the notional at maturity.

**Par condition** — a swap quoted at par has zero initial value:
$$S_N \sum_{i=1}^{N} \tau_i \, P(0,T_i) \;=\; 1 - P(0,T_N)$$

### 2.2 The bootstrap recursion

Process instruments in order of increasing maturity. If all earlier discount factors
$P(0,T_1)\dots P(0,T_{N-1})$ are already known, the par condition solves in closed form for the
new one:

$$\boxed{\;P(0,T_N) \;=\; \frac{1 - S_N \sum_{i=1}^{N-1} \tau_i \, P(0,T_i)}{1 + S_N\,\tau_N}\;}$$

For **money-market instruments** (the ON node and the sub-1Y average-rate nodes) there is a
single payment and the formula degenerates to the deposit formula
$P(0,T) = \frac{1}{1 + r\,\tau}$ with $\tau = \text{days}/360$.

### 2.3 Gap tenors need interpolation + root-finding

Quoted tenors are typically ON, 1M, 3M, 6M, 1Y, 2Y, 3Y, 5Y, 7Y, 10Y, 20Y, 30Y. A 5Y swap pays
annual coupons at 1,2,3,4,5Y — but **4Y is not a quoted node**, so $P(0,4Y)$ is unknown at that
point and the closed form above no longer applies. The standard resolution:

1. Choose an **interpolation scheme** on the curve. We use **log-linear interpolation of
   discount factors** (equivalent to piecewise-constant overnight forward rates):
   $$\ln P(0,t) = \frac{t_{k+1}-t}{t_{k+1}-t_k}\ln P(0,t_k) + \frac{t-t_k}{t_{k+1}-t_k}\ln P(0,t_{k+1})$$
2. Treat the new node's DF as the single unknown $x = P(0,T_N)$. Every interior coupon DF is a
   deterministic function of $x$ via interpolation.
3. Solve the one-dimensional root-finding problem
   $$g(x) = S_N \sum_{i=1}^{N} \tau_i \, P_x(0,T_i) - \bigl(1 - x\bigr) = 0$$
   with Brent's method. $g$ is strictly increasing in $x$ (fixed leg grows, floating leg
   shrinks), so the root is unique and bracketing is trivial.

This "interp + Brent" treatment is the general case; when there is no gap it reproduces the
closed-form recursion to machine precision, so the implementation uses it uniformly.

### 2.4 Outputs derived from the discount factors

| Quantity | Formula |
|---|---|
| Zero rate (continuous, Act/365) | $z(T) = -\ln P(0,T)\,/\,T$ |
| Money-market zero (simple, Act/360) | $r_{mm}(T) = \bigl(1/P(0,T) - 1\bigr)\cdot 360/\text{days}$ |
| Forward rate $T_a \to T_b$ | $f = \bigl(P(0,T_a)/P(0,T_b) - 1\bigr)\,/\,\tau(T_a,T_b)$ |

---

## 3. Hand-calculated worked example

Simplifications for hand arithmetic: every year is exactly 365 days (so every annual accrual is
$\tau = 365/360 = 1.0138\overline{8}$), no business-day adjustments, valuation at $t=0$.

Market inputs (FRED data as of 2026-07-29/31, Treasury yields used directly as swap quotes for
this example):

| Instrument | Rate |
|---|---|
| Overnight SOFR | 3.65% |
| 1Y par swap | 4.04% |
| 2Y par swap | 4.22% |
| 3Y par swap | 4.29% |
| 5Y par swap | 4.37% |

**Step 0 — overnight node.** One-day deposit at 3.65%:
$$P(0,1d) = \frac{1}{1 + 0.0365 \cdot \tfrac{1}{360}} = 0.999898621$$

**Step 1 — 1Y node.** One fixed payment; par condition $S_1\tau P(0,1) = 1 - P(0,1)$:
$$P(0,1Y) = \frac{1}{1 + 0.0404 \times 1.013889} = \frac{1}{1.040961} = 0.960650681$$
Zero rate: $z(1) = -\ln(0.960650681)/1 = 4.014443\%$.

**Step 2 — 2Y node.** Recursion with the known $P(0,1Y)$:
$$P(0,2Y) = \frac{1 - 0.0422 \times 1.013889 \times 0.960650681}{1 + 0.0422 \times 1.013889}
          = \frac{0.958897493}{1.042786111} = 0.919553380$$
Zero rate: $z(2) = 4.193359\%$.

**Step 3 — 3Y node.** The known-coupon annuity is
$\tau\,(P_1 + P_2) = 1.013889 \times (0.960650681 + 0.919553380) = 1.906318006$, so
$$P(0,3Y) = \frac{1 - 0.0429 \times 1.906318006}{1 + 0.0429 \times 1.013889} = 0.879945016
\qquad z(3) = 4.263195\%$$

**Step 4 — 5Y node (the gap case).** The 5Y swap pays coupons at 1..5Y but $P(0,4Y)$ is not yet
known. Let $x = P(0,5Y)$ and interpolate log-linearly between the 3Y node and the candidate:
$$P_x(0,4Y) = \exp\!\bigl(\tfrac{1}{2}\ln 0.879945016 + \tfrac{1}{2}\ln x\bigr) = \sqrt{0.879945016\,x}$$
Solve with Brent's method:
$$g(x) = 0.0437 \times 1.013889 \times \bigl(P_1 + P_2 + P_3 + \sqrt{0.879945016\,x} + x\bigr) - (1 - x) = 0$$
Converged solution:
$$P(0,5Y) = 0.804764582, \qquad P(0,4Y) = 0.841515646$$
$$z(4) = 4.313767\%, \qquad z(5) = 4.344110\%, \qquad |g(x^\*)| \approx 4\times10^{-16}$$

**Verification.** Re-price the 5Y swap off the finished curve:
$$S_5^{\text{implied}} = \frac{1 - P(0,5Y)}{\tau\,(P_1+P_2+P_3+P_4+P_5)} = 4.3700000000\%$$
— it recovers the input quote exactly. This repricing test, applied to *every* input instrument
with a tolerance near machine epsilon (the prototype achieves worst-case error ≈ 9×10⁻¹⁷),
is the standard correctness certificate for a bootstrap.

---

## 4. Detailed implementation plan

### 4.1 Architecture

```
┌────────────────────────────────────────────────────────────┐
│ CONFIG   reference date · API key · quote source switch    │
│          manual quotes · proxy swap-spread table           │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────┐
│ DATA LAYER (FRED REST API, JSON)                           │
│  fred_observations(series_id, ...)  → pandas Series        │
│  · as-of logic: last published value ≤ reference date      │
│  · offline fallback to a hardcoded snapshot                │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────┐
│ INSTRUMENT SET                                             │
│  deposits: ON (SOFR), 1M/3M/6M (SOFR averages)             │
│  swaps 1Y…30Y: MANUAL quotes  or  DGS + spread proxy       │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────┐
│ CURVE ENGINE  (class SOFRCurveBootstrapper)                │
│  schedule generation → add_deposit → add_swap (Brent)      │
│  log-linear DF interpolation · flat-forward extrapolation  │
└──────────────────────────┬─────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────┐
│ OUTPUTS & VALIDATION                                       │
│  node table (DF, zero, forward) · repricing test (assert)  │
│  matplotlib dashboard                                      │
└────────────────────────────────────────────────────────────┘
```

### 4.2 Data layer

- Endpoint: `https://api.stlouisfed.org/fred/series/observations` with
  `series_id`, `api_key`, `file_type=json`, `observation_end=<ref date>`, `sort_order=desc`,
  `limit≈10` — one lightweight call per series.
- **As-of rule:** use the most recent non-missing observation on or before the reference date
  (FRED reports `"."` for holidays; publication lags differ per series — `SOFR` lags one
  business day, `DGS*` similar). Record the actual observation date used per series.
- **Resilience:** wrap in `try/except`; on any network/API failure fall back to a hardcoded
  snapshot (values + snapshot date printed loudly) so the notebook always runs end-to-end.

### 4.3 Schedule generation and day counts

- Fixed-leg coupon dates for an *n*-year swap: calendar anniversaries of the valuation date,
  rolled **following** if they land on a weekend (simplification: no holiday calendar).
- Accruals: Act/360 between consecutive *adjusted* dates.
- Time axis for interpolation: Act/365 year fractions from valuation date.

### 4.4 Curve engine

State: parallel arrays `dates[]`, `dfs[]` starting at `(t0, 1.0)`, kept sorted by maturity.

| Method | Contract |
|---|---|
| `df(date, trial_node=None)` | Log-linear interpolation in $\ln P$; optional trial node lets the root-finder query the curve *as if* the candidate node were already added; flat-forward extrapolation beyond the last node |
| `add_deposit(maturity, rate)` | Append node with $P = 1/(1+r\tau)$ |
| `add_swap(years, rate)` | Build schedule; `brentq` on $g(x)$ over $x\in[10^{-6}, 2]$, `xtol=1e-15`; append root |
| `par_rate(years)` | $(1-P(T_N))/\sum\tau_i P(T_i)$ — used by the verification step |
| `zero_rate(date)` / `forward_rate(d1,d2)` | Output transforms per §2.4 |

Instruments **must be added in increasing maturity order**; the engine asserts this.

### 4.5 Simplifications vs. production practice (documented, deliberate)

| This project | Production desk practice |
|---|---|
| Valuation = reference date, no spot lag | T+2 spot start, curve anchored at settlement |
| Weekend-only roll | SIFMA holiday calendar, modified following |
| 1M/3M/6M nodes from backward-looking SOFR averages | SOFR futures (1M/3M) or OIS quotes; FOMC-date jump modeling |
| Long-end quotes manual or Treasury-proxy | Live cleared OIS quotes (CME/Bloomberg) |
| Log-linear DF interpolation | Same, or monotone convex / tension splines |
| Single-curve | Single-curve (same!) — plus separate credit/xVA curves |

### 4.6 Validation plan

1. **Repricing test (hard assert):** every input swap repriced off the finished curve must
   recover its quote to < 0.01 bp (achieved: ~1e-14 bp).
2. **Sanity checks:** all DFs in (0,1] and strictly decreasing; zero curve continuous;
   forwards positive (in the current rate regime).
3. **Cross-check:** short-end zeros should sit within a few bp of the input SOFR averages;
   the ON zero equals the SOFR fixing by construction.

### 4.7 Notebook layout

| # | Cell | Content |
|---|---|---|
| 1 | md | Title, data sources, honest caveats |
| 2 | md | Methodology recap (formulas §2) |
| 3 | code | Imports + **CONFIG** (single user entry point) |
| 4 | code | FRED data layer + instrument assembly + input table |
| 5 | code | `SOFRCurveBootstrapper` engine (heavily commented) |
| 6 | code | Bootstrap execution + node/zero/forward table |
| 7 | code | Repricing verification table + assert |
| 8 | code | Matplotlib dashboard (DF, zero, forward curves) |
| 9 | md | Limitations & production-hardening notes |

---

## 5. References

- NY Fed — SOFR overview and publication: https://www.newyorkfed.org/markets/reference-rates/sofr
- FRED series: [SOFR](https://fred.stlouisfed.org/series/SOFR),
  [SOFR30DAYAVG](https://fred.stlouisfed.org/series/SOFR30DAYAVG),
  [SOFR90DAYAVG](https://fred.stlouisfed.org/series/SOFR90DAYAVG),
  [SOFR180DAYAVG](https://fred.stlouisfed.org/series/SOFR180DAYAVG),
  [DGS10](https://fred.stlouisfed.org/series/DGS10) et al.
- FRED API documentation: https://fred.stlouisfed.org/docs/api/fred/
- CME SOFR OIS conventions: https://www.cmegroup.com/trading/interest-rates/cleared-otc.html
- Andersen & Piterbarg, *Interest Rate Modeling*, Vol. 1 (curve construction);
  Hagan & West (2006), *Interpolation Methods for Curve Construction*.
