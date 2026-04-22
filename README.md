# CPG Demand Forecasting & Safety Stock Dashboard

An end-to-end supply chain analytics tool for Canadian CPG sales teams — forecasting weekly SKU demand 16 weeks forward, calculating statistically optimal safety stock, and surfacing stockout risk before it becomes a retailer chargeback.

Built to answer the question every supply chain team faces every Monday: *"Do we have enough stock, and when do we need to order?"*

---

## What It Does

- Generates **2 years of synthetic weekly CPG sales data** (1,560 rows) with realistic Canadian market seasonality, trend, and embedded business scenarios
- Applies **Prophet multiplicative forecasting** (trend × seasonality × holidays) to produce 16-week forward forecasts with 90% confidence intervals
- Calculates **safety stock per SKU** using the industry-standard formula: `Z × σ_demand × √(lead_time)`
- Derives **reorder points** that account for both average demand and lead time uncertainty
- Builds a **stockout risk register** (🔴 High / 🟡 Medium / 🟢 Low) with calculated order-by dates
- Visualises the **S&OP demand vs. supply gap** across all accounts with a demand-gap waterfall

---

## Dashboard Tabs

| Tab | What You See |
|---|---|
| **📈 Demand Forecast** | 2-year historical sales, Prophet 16-week forecast with CI ribbon, OOS lost-demand markers, promo week indicators, annual seasonality decomposition |
| **🛡️ Safety Stock Optimizer** | Recommended SS and ROP by SKU at selected account, service level sensitivity chart (90% → 99%) |
| **🚨 Stockout Risk Monitor** | Traffic light risk register, weeks-of-cover bar chart, order-by dates, OOS forensics with root cause |
| **📋 S&OP Planner** | Aggregated demand plan vs. conservative supply plan, demand gap waterfall, 16-week shortfall forecast, account share pie |

---

## Dataset at a Glance

| Metric | Value |
|---|---|
| Date Range | Jan 2, 2023 → Dec 23, 2024 (104 weeks) |
| Total Rows | 1,560 (5 SKUs × 3 accounts × 104 weeks) |
| Total 2-Year Revenue | $6,011,283 |
| Avg Weekly Revenue | $57,801 |
| Year-over-Year Growth | +7.3% (Year 1: $2.90M → Year 2: $3.11M) |
| Weekly Revenue Trend | +$201/week |
| SKUs | 5 across Skincare, Personal Care, Haircare |
| Accounts | 3 (Shoppers Drug Mart, Walmart Canada, Loblaws) |
| Promo Events | 106 week-SKU-account rows across 8 events |
| OOS Events | 3 (9 total week-SKU-account rows) |

---

## Canadian Market Seasonality (Verified from Data)

| Period | Months | Seasonal Multiplier |
|---|---|---|
| Post-holiday dip | January | **0.80x** (−20% vs. baseline) |
| Spring recovery | Feb–Mar | 0.98–0.99x |
| Spring promo | April | 1.09x |
| Victoria Day | May | 0.95x |
| Early summer | June | 1.13x |
| Summer / Canada Day | Jul–Aug | **1.18–1.21x** (+18–21%) |
| Back-to-School | Sep | 1.02x |
| Q4 build-up | October | 1.22x |
| Q4 peak | November | **1.60x** (+60%) |
| Holiday | December | **1.45x** (+45%) |

**Q4 vs. baseline peak factor: 1.53x** (Nov–Dec avg vs. Feb/Mar/Sep avg)

---

## Promo Calendar & Observed Lift

| Promo Event | Weeks | SKUs Affected | Observed Lift |
|---|---|---|---|
| Victoria Day (2023 & 2024) | May wk 2 | Skincare, Personal Care | +22% |
| Canada Day (2023 & 2024) | Jun wk 4 | All SKUs @ Walmart | +35% |
| Back-to-School (2023 & 2024) | Aug wk 4 | Skincare, Haircare | +18% |
| Black Friday (2023 & 2024) | Nov wks 3–4 | All SKUs, all accounts | +75% / +40% |

**Average promo lift across all SKUs: +61%** (vs. non-promo, non-OOS baseline — event lifts range from +18% to +75% by event)

---

## OOS Events Embedded

Three deliberate stockout scenarios that the model must detect and forecast through:

| Event | SKU | Account | Period | Lost Units | Lost Revenue | Root Cause |
|---|---|---|---|---|---|---|
| Valentine's OOS | ColorPop Lip Balm | Loblaws | Feb 6–20, 2023 | 800 | $4,392 | Seasonal ROP too low; post-holiday replenishment delayed |
| Summer Stockout | FreshGuard Deodorant | Walmart Canada | Jul 17–31, 2023 | 1,569 | $14,105 | Canada Day promo vol not pre-built at DC |
| Q4 Early OOS | HydraBoost Moisturizer | Shoppers Drug Mart | Nov 20–Dec 4, 2023 | 1,899 | $36,062 | Black Friday demand spike not shared with supply chain |

**Total implied lost revenue from OOS events: $54,559**

---

## SKU Performance Summary

| SKU | Category | 2-Year Revenue | Avg Weekly Units | Demand CV |
|---|---|---|---|---|
| ShineBoost Shampoo 400ml | Haircare | $1,475,573 | 364 | 43% |
| HydraBoost Moisturizer 100ml | Skincare | $1,428,504 | 243 | 49% |
| FreshGuard Deodorant 100ml | Personal Care | $1,178,931 | 424 | 45% |
| NourishPro Hair Mask 250ml | Haircare | $967,443 | 194 | 51% |
| ColorPop Lip Balm SPF20 4g | Personal Care | $960,832 | 566 | 52% |

*CV = Coefficient of Variation (demand volatility). Higher CV → larger safety stock requirement.*

---

## Account Performance Summary

| Account | 2-Year Revenue | Share | Avg Weekly Units |
|---|---|---|---|
| Shoppers Drug Mart | $2,871,839 | 47.8% | 511 |
| Walmart Canada | $2,026,600 | 33.7% | 359 |
| Loblaws | $1,112,843 | 18.5% | 198 |

---

## Forecasting Methodology

Uses **Prophet multiplicative decomposition** — the same foundation used at Meta for business time series forecasting:

```
demand = trend × seasonality × holiday_effects × noise
```

- **Trend**: linear growth fit with automatic changepoint detection (`changepoint_prior_scale=0.05`)
- **Seasonality**: Fourier-series annual pattern, `seasonality_mode="multiplicative"` (correct for CPG spikes)
- **Holidays**: Canadian statutory holidays with ±1 day windows
- **OOS handling**: zero-sale weeks replaced with interpolated values before fitting so the model learns true demand, not supply failures
- **Confidence intervals**: 90% interval width (Prophet posterior sampling)

*Fallback: when Prophet is unavailable, a pure numpy/pandas 13-week centred moving-average decomposition is used — same mathematical structure, runs instantly.*

---

## Safety Stock Formula

```
Safety Stock  =  Z  ×  σ_demand  ×  √(lead_time)

Reorder Point  =  avg_weekly_demand  ×  lead_time  +  Safety_Stock
```

| Service Level | Z-Score | Use Case |
|---|---|---|
| 90% | 1.28 | Standard shelf replenishment |
| 95% | 1.65 | High-priority SKUs |
| **98%** | **2.05** | **Walmart OTIF compliance target (default)** |
| 99% | 2.33 | Can't-stockout SKUs |

Walmart Canada's OTIF threshold is 98%. Accounts that fall below 95% face financial chargebacks — which is why the dashboard defaults to the 98% service level.

---

## Business Questions Answered

- **Which SKUs will stockout before the next replenishment cycle?** → Stockout Risk Monitor tab
- **How much safety stock do I need to hit 98% OTIF at Walmart?** → Safety Stock Optimizer tab
- **When exactly do I need to place the next purchase order?** → Order-by date in risk register
- **What does the demand gap between our forecast and supply plan look like over the next 16 weeks?** → S&OP Planner tab
- **How much revenue did we lose to the three OOS events last year?** → OOS Forensics panel ($54,559)

