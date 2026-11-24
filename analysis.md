# Fashion Sales — Full Analysis

*Source file: `fashion_sales.csv` · 240 rows × 14 columns · covers January 2024 – December 2025*

---

## 1. What this dataset is (for someone who has never seen it)

This is a **two-year monthly performance report for a fashion retailer**. It does not contain
individual transactions; instead each row is a **summary** of how one product category performed,
through one sales channel, in one month.

Think of it as a management report: every month, for every product line, someone asked
*"how much did we sell online versus in stores, at what price, with how much advertising, how did
customers rate it, and how much came back?"* — and wrote those answers into one row.

The business sells **five product categories** — Dresses, Denim, Outerwear, Footwear and
Accessories — through **two channels** — Online and Store.

---

## 2. How the file is organised

| Property | Value |
|---|---|
| Data rows | **240** (plus 1 header row) |
| Columns | **14** |
| Time span | 24 months: `2024-01` … `2025-12` |
| Years | 2 (2024, 2025) |
| Categories | 5 (Dresses, Denim, Outerwear, Footwear, Accessories) |
| Channels | 2 (Online, Store) |
| Grain (what one row is) | one **month × category × channel** |

The grid is **perfectly balanced**: 24 months × 5 categories × 2 channels = 240 rows exactly.

- Every month contains exactly **10 rows** (5 categories × 2 channels).
- Every category has exactly **48 rows** (24 months × 2 channels).
- Every channel has exactly **120 rows** (24 months × 5 categories).
- Every month × category × channel combination appears **exactly once** — no duplicates, no gaps.

---

## 3. Column dictionary

| Column | Type | Meaning | Example |
|---|---|---|---|
| `month` | text | Month label `YYYY-MM` | `2024-01` |
| `year` | number | Year (2024 or 2025) | `2024` |
| `month_num` | number | Month number 1–12 (for sorting) | `1` |
| `category` | text | Product category | `Dresses` |
| `channel` | text | Sales channel | `Online` |
| `avg_temp_F` | number | Average monthly temperature, °F | `36.2` |
| `list_price` | number | Full/non-discounted price of the category ($) | `85` |
| `discount_pct` | number | Discount applied that month (%) | `30` |
| `sale_price` | number | Actual average selling price per unit ($) | `54.31` |
| `ad_spend` | number | Advertising spend for that category/channel/month ($) | `7380` |
| `avg_rating` | number | Average customer rating, out of 5 | `3.7` |
| `units_sold` | number | Units sold that month | `609` |
| `revenue` | number | Total revenue that month ($) | `33074.79` |
| `return_rate` | number | Share of units returned (0–1) | `0.311` (= 31.1%) |

---

## 4. How the columns depend on each other

This is the most important structural section, because several columns are **computed from others**,
and one pairing is a **trap** if you assume it works.

### 4.1 `revenue` is exactly `sale_price × units_sold` — always

Verified for **240 / 240 rows**, with no rounding drift beyond a cent:

```
revenue = sale_price × units_sold      (exact identity)
```

Example (row 1): `54.31 × 609 = 33,074.79` ✓

**Consequence:** never treat `revenue`, `sale_price` and `units_sold` as three independent signals.
If you know any two, you know the third. Adding `revenue` and `sale_price` to a model together as
"separate drivers" would be double-counting.

### 4.2 `sale_price` broadly follows `list_price × (1 − discount_pct/100)` — but not exactly

The naive expectation is `sale_price = list_price × (1 − discount_pct/100)`. Taken literally this
**fails for all 240 rows** — it is never exactly equal. But it is right *on average*, with noise:

- Median ratio `sale_price / [list_price × (1 − discount/100)]` = **1.004** (i.e. almost exactly 1).
- **All 240 ratios** fall within ±10% of that; **114 of 240** are within ±5%.
- Average absolute gap between actual and formula price: **$3.88**; largest: **$17.18**.

So the correct statement is: **sale price is approximately list price minus the discount, plus
per-row variation.** The average sale/list ratio tracks the discount neatly:

| Discount | 0% | 10% | 15% | 20% | 25% | 30% | 40% |
|---|---|---|---|---|---|---|---|
| Avg sale ÷ list | 100% | 89% | 86% | 80% | 74% | 69% | 60% |
| Expected (1−d) | 100% | 90% | 85% | 80% | 75% | 70% | 60% |

**Surprise:** because of that noise, **21 of 240 rows have a `sale_price` higher than the `list_price`**
(e.g. Feb 2024, Denim, Store: list $70, sale $75.95 despite a 30% discount being recorded). And even
at **0% discount**, no row has sale price equal to list price — 22 are below it, 21 above.

### 4.3 `list_price` is fixed per category (only 5 distinct values)

| Category | List price |
|---|---|
| Dresses | $85 |
| Denim | $70 |
| Outerwear | $180 |
| Footwear | $110 |
| Accessories | $35 |

### 4.4 `avg_temp_F` is really a *month* attribute, not a row attribute

- It takes only **24 distinct values** — exactly one per month.
- Every one of the 10 rows in a given month shares the **identical** temperature.
- 2024 and 2025 have **different** temperatures for the same calendar month (e.g. April 2024 = 56.9°F,
  April 2025 = 55.5°F), so it is a genuine monthly weather reading rather than a fixed season label.

**Consequence:** `avg_temp_F` is effectively a proxy for "which month is this". You cannot cleanly
separate the effect of weather from every other monthly effect (holidays, campaigns, paydays) — they
move together.

### 4.5 `discount_pct` is set in discrete tiers, not continuously

Only seven values exist: **0, 10, 15, 20, 25, 30, 40**. And they are used unevenly:

| Discount | 0% | 10% | 15% | 20% | 25% | 30% | 40% |
|---|---|---|---|---|---|---|---|
| Rows | 43 | 12 | 13 | 45 | 28 | 46 | 53 |

Most months run at **0%, 20%, 30% or 40%**; the 10% and 15% tiers are rare (25 rows combined), so any
conclusion drawn about those two tiers rests on thin data.

### 4.6 Dependency summary

```
list_price (fixed per category)
      │  ≈ × (1 − discount/100)
      ▼
sale_price ────────── × units_sold ──────────► revenue   (exact)
      ▲                                              │
discount_pct (7 tiers)                               │
                                                     ▼
avg_rating, return_rate   (outcomes, not inputs)   (the value chain)
avg_temp_F = monthly weather signal (drives timing, not price)
ad_spend = independent monthly input
```

---

## 5. Data quality and surprises

| Check | Result |
|---|---|
| Missing / blank / null values | **None** in any column |
| Non-numeric values in numeric columns | **None** |
| Negative values | **None** |
| Duplicate rows | **None** |
| Duplicate month × category × channel keys | **None** |
| Value ranges plausible | discount 0–40%, rating 3.4–5.0, return 1.0–35.6% ✓ |
| `revenue = sale_price × units_sold` | ✅ 240/240 |
| `sale_price = list_price × (1 − d)` exactly | ❌ 0/240 (holds only approximately) |

**Things that will surprise a first-time reader**

1. **The revenue column is not independent information.** It is price × units, exactly, everywhere.
2. **A recorded discount does not guarantee the price went down.** 21 rows have a sale price *above*
   the list price. Treat `discount_pct` as a *recorded label* and `sale_price` as the *realised price*.
3. **Temperature isn't a per-row variable.** All 10 rows in a month share one temperature.
4. **The file looks generated rather than collected.** Round list prices ($85, $70, $180…), a perfect
   24×5×2 grid, zero missing values, a deterministic revenue identity, and a smooth seasonal
   temperature curve are all hallmarks of a **simulated dataset**. Findings here describe *this file*;
   treat it as realistic-looking practice data rather than a real retailer's ledger.
5. **Bigger ≠ busier.** The category with the most units (Accessories) earns the least revenue, and
   the category with the fewest units (Outerwear) earns the most. See §9.
6. **The overall temperature-sales link is near zero (−0.08) yet two categories are strongly weather-driven
   in opposite directions** (+0.62 and −0.77). Aggregating hides the story completely.

---

## 6. Headline numbers (whole two years)

| Metric | Value |
|---|---|
| Total revenue | **$13,220,481.53** |
| Total units sold | **202,500** |
| Total advertising spend | **$2,337,530** |
| Blended return on ad spend (ROAS) | **5.66×** |
| Advertising as share of revenue | **17.7%** |
| Overall average selling price (AOV) | **$65.29** |
| Average monthly return rate (unweighted) | **17.3%** (units-weighted 17.6%) |
| Average customer rating | **4.22 / 5** |

### Summary statistics for the numeric columns

| Column | Min | Max | Mean | Median | Std dev |
|---|---:|---:|---:|---:|---:|
| avg_temp_F | 31.3 | 80.4 | 55.6 | 56.2 | 15.60 |
| list_price | 35.0 | 180.0 | 96.0 | 85.0 | 48.52 |
| discount_pct | 0.0 | 40.0 | 22.6 | 25.0 | 13.49 |
| sale_price | 19.57 | 197.18 | 73.71 | 68.31 | 39.38 |
| ad_spend | 2,050 | 19,270 | 9,740 | 9,430 | 4,606 |
| avg_rating | 3.40 | 5.00 | 4.22 | 4.20 | 0.28 |
| units_sold | 220 | 2,488 | 843.8 | 819.5 | 348.7 |
| revenue | 17,257 | 154,512 | 55,085 | 48,838 | 27,308 |
| return_rate | 0.010 | 0.356 | 0.173 | 0.175 | 0.086 |

**Reading the spread:** for revenue, the mean ($55,085) sits above the median ($48,838), so revenue is
**right-skewed** — a minority of very strong months pulls the average up. The top single cell
(Dec 2025, Outerwear, Online: **$154,512**) is ~9× the weakest
(Oct 2024, Accessories, Online: **$17,257**).

### Key correlations

| Pair | r | Reading |
|---|---:|---|
| `list_price` ↔ `sale_price` | **+0.91** | Expensive categories sell at higher prices |
| `list_price` ↔ `revenue` | **+0.67** | Price/assortment drives revenue more than volume |
| `units_sold` ↔ `revenue` | **+0.32** | Weak! Volume alone explains little of revenue |
| `sale_price` ↔ `units_sold` | **−0.52** | Higher prices → fewer units (demand curve) |
| `discount_pct` ↔ `units_sold` | **+0.51** | Deeper discounts → more units |
| `ad_spend` ↔ `revenue` | **+0.21** | Advertising only weakly tracks revenue |
| `avg_rating` ↔ `return_rate` | **−0.35** | The single strongest driver of returns |
| `avg_temp_F` ↔ `units_sold` | **−0.08** | Looks like weather doesn't matter — it does, but only per category (§8) |

*(r = Pearson correlation, −1 to +1. Correlation is not causation; see §13.)*

---

## 7. Seasonal patterns

Units and revenue rise and fall in a clear annual rhythm, repeated in both years.

**Monthly totals across both years:**

| Month | Temp °F | Units | Revenue |
|---|---:|---:|---:|
| Jan | 33.8 | 17,232 | $1,067,846 |
| Feb | 36.3 | **12,898** (lowest) | $987,565 |
| Mar | 42.5 | 16,220 | $1,067,880 |
| Apr | 56.2 | 14,617 | $1,003,886 |
| May | 63.2 | 15,140 | $1,059,010 |
| Jun | 72.8 | 14,651 | $977,241 |
| Jul | 79.2 | 20,948 | $1,089,663 |
| Aug | 75.2 | 16,210 | $1,034,485 |
| Sep | 68.5 | 13,693 | $987,816 |
| Oct | 56.2 | 15,157 | $1,051,272 |
| Nov | 45.8 | 21,874 | $1,438,159 |
| Dec | 38.0 | **23,860** (peak) | **$1,455,661** (peak) |

- **Peak-to-trough swing:** December (23,860 units) vs February (12,898) = **1.85×**.
- **Holiday concentration:** November + December alone = **22.6%** of all units sold in two years,
  and **21.9%** of all revenue.
- **The three peak months** — July, November, December — together = **32.9%** of all units.
- **Summer bump:** July is a secondary peak (20,948 units) driven by warm-weather categories.

**Different categories peak in different seasons** (revenue by calendar month, both years combined):

| Category | Peak months | Trough months |
|---|---|---|
| Dresses | Jul ($421,312), Aug, Sep | Feb ($128,218), Jan |
| Outerwear | Dec ($570,260), Nov, Jan | Jul ($176,821), Jun |
| Denim | Dec, Nov, May | Apr ($129,990), Sep |
| Footwear | Dec, Nov, Aug | Feb ($184,498), Jan |
| Accessories | Nov, Dec, Apr | May ($91,589), Oct |

So there are really **two seasonal engines**: a **holiday engine** (Nov–Dec) that lifts every category,
and a **weather engine** where Dresses peak in July and Outerwear peaks in December.

**Consistency check by year** — the holiday shape repeats in both:

| | Nov 2024 | Dec 2024 | Nov 2025 | Dec 2025 |
|---|---|---|---|---|
| Revenue | $736,471 | $700,719 | $701,687 | $754,943 |

---

## 8. How temperature affects each category

Temperature's effect is invisible in aggregate (overall r = **−0.08**) but dramatic per category:

| Category | Corr(units, temp) | Slope | Cold (<45°F) avg units | Warm (>70°F) avg units |
|---|---:|---:|---:|---:|
| **Dresses** | **+0.62** | +18.6 units per °F | 727 | **1,427** |
| Denim | −0.27 | −4.4 | 901 | 784 |
| **Outerwear** | **−0.77** | **−16.3** | **957** | **352** |
| Footwear | −0.22 | −3.2 | 786 | 680 |
| Accessories | −0.25 | −3.9 | 1,129 | 958 |

- **Dresses are strongly warm-weather:** roughly **doubling** from ~727 units/month in the cold to
  ~1,427 in the warm months.
- **Outerwear is strongly cold-weather:** ~**2.7×** more units in the cold (957) than in the warm
  (352) — the strongest weather response of any category.
- Denim, Footwear and Accessories have only mild weather sensitivity.

**Why the aggregate looks flat:** the +0.62 of Dresses and the −0.77 of Outerwear (plus the smaller
negatives) largely cancel out, leaving a near-zero overall correlation. This is a textbook
aggregation (Simpson-style) trap — *average across categories and you'd wrongly conclude weather
doesn't matter.*

---

## 9. Do bigger discounts lead to more units sold?

**Yes — clearly and consistently.**

- **Overall:** r = **+0.51**, slope ≈ **+13.3 units per 1 percentage point** of discount.
- **Holds in every category** (r between **+0.55** and **+0.75**).
- **Holds in every channel** (Online **+0.59**, Store **+0.45**).

Mean units at the shallowest vs deepest discount:

| Category | Avg units at ≤10% discount | Avg units at 40% discount | Change |
|---|---:|---:|---:|
| Dresses | 665 | 1,465 | +120% |
| Denim | 565 | 1,092 | +93% |
| Outerwear | 387 | 906 | +134% |
| Footwear | 496 | 953 | +92% |
| Accessories | 801 | 1,220 | +52% |

**But volume is not the same as revenue.** Because price falls as discount rises
(r between `discount_pct` and `sale_price` = **−0.27**), extra units only convert into modestly more
revenue: r(`discount_pct`, `revenue`) = **+0.20**, far weaker than the +0.51 seen for units.

**Caveat:** deeper discounts are also applied in high-demand months (Nov/Dec, July), so part of the
extra volume belongs to seasonality rather than the discount itself. Still, the relationship survives
*within* every category and channel, so the discount effect is real, not just seasonal.

---

## 10. Does advertising pay off?

**Yes, but with sharply diminishing returns, and very unevenly by category.**

**ROAS (revenue ÷ ad spend) by category:**

| Category | Ad spend | Revenue | ROAS |
|---|---:|---:|---:|
| **Outerwear** | $474,890 | $4,045,992 | **8.52×** |
| Dresses | $456,100 | $2,995,076 | 6.57× |
| Footwear | $452,080 | $2,794,483 | 6.18× |
| Denim | $489,100 | $2,078,650 | 4.25× |
| **Accessories** | $465,360 | $1,306,281 | **2.81×** |

**ROAS by channel:**

| Channel | Ad spend | Revenue | ROAS | Avg spend per row |
|---|---:|---:|---:|---:|
| Store | $997,330 | $6,254,329 | **6.27×** | $8,311 |
| Online | $1,340,200 | $6,966,152 | 5.20× | $11,168 |

**Diminishing returns across spend levels:**

| Monthly ad spend | Rows | Avg revenue | Avg ad spend | ROAS |
|---|---:|---:|---:|---:|
| $2–6k | 60 | $49,150 | $3,996 | **12.30×** |
| $6–10k | 68 | $49,683 | $7,972 | 6.23× |
| $10–14k | 65 | $60,483 | $11,888 | 5.09× |
| $14–18k | 36 | $61,428 | $16,048 | 3.83× |
| $18–22k | 11 | $68,202 | $18,654 | **3.66×** |

Key points:

- Every extra dollar of spend still *adds* revenue (avg revenue rises from ~$49k to ~$68k), **but the
  return collapses from ~12× to ~3.7×**. The first dollars are far more productive than the last.
- **Ad spend explains only a little of revenue** (r = **+0.21**), because revenue is dominated by
  category price. Advertising mostly rides on top of what the product mix already earns.
- **Accessories are the weakest use of money** (2.81×) and Outerwear the best (8.52×) — i.e. the
  cheapest product is the least efficient to advertise, the most expensive is the most efficient.
- **Stores spend less and return more per dollar** than Online.

---

## 11. What is related to returns?

The average return rate is **17.3%** of units (units-weighted: 17.6%). The single biggest factor is
**channel**, and the second is **category**.

**By channel (the dominant driver):**

| Channel | Average return rate |
|---|---:|
| Online | **24.7%** |
| Store | **10.0%** |

Online returns run **~2.5×** higher than store returns.

**By category:**

| Category | Average return rate |
|---|---:|
| Footwear | **22.4%** |
| Dresses | **21.9%** |
| Denim | 14.5% |
| Outerwear | 14.1% |
| Accessories | **13.9%** |

**Category × channel (the extremes are here):**

| Channel | Dresses | Denim | Outerwear | Footwear | Accessories |
|---|---:|---:|---:|---:|---:|
| Online | **28.9%** | 22.3% | 20.9% | **29.9%** | 21.3% |
| Store | 14.8% | 6.7% | 7.2% | 14.8% | **6.4%** |

- **Worst:** Footwear online (**29.9%**) and Dresses online (28.9%) — roughly **1 in 3 units**.
- **Best:** Accessories in store (**6.4%**) and Denim in store (6.7%).
- Categories where **fit matters** (Footwear, Dresses) are exactly the ones with the worst online
  returns — consistent with customers returning items that don't fit as expected when they can't try
  them on.

**Other relationships with returns:**

| Pair | r | Comment |
|---|---:|---|
| rating ↔ return_rate | **−0.35** | Higher-rated products are returned less; strongest link found. Holds within channel too (−0.27 Online, −0.29 Store). |
| ad_spend ↔ return_rate | +0.23 | **Misleading** — mainly a channel artefact: within Online it's −0.13 and within Store −0.01. Online simply has both more ad spend *and* more returns. |
| discount_pct ↔ return_rate | −0.10 | Very weak; deeper discounts do **not** meaningfully raise returns here. |
| sale_price ↔ return_rate | +0.05 | Essentially none. |
| temp ↔ return_rate | ≈ 0.00 | No weather effect on returns. |

---

## 12. What is related to ratings?

Ratings are **high and tightly clustered**: mean **4.22**, median 4.20, range **3.4–5.0**
(standard deviation only ±0.28). This narrow spread means ratings are a weak differentiator — no
category is badly rated.

**By channel:** Store **4.29** vs Online **4.15** — stores are rated higher across every category.

**By category:** Accessories **4.25** (best) and Denim **4.18** (worst); Dresses and Outerwear 4.23,
Footwear 4.20 — a spread of only 0.07.

**Category × channel:**

| Channel | Dresses | Denim | Outerwear | Footwear | Accessories |
|---|---:|---:|---:|---:|---:|
| Online | 4.15 | 4.11 | 4.16 | 4.15 | 4.18 |
| Store | 4.31 | 4.26 | 4.30 | 4.25 | 4.33 |

**What ratings correlate with:**

| Pair | r | Reading |
|---|---:|---|
| rating ↔ return_rate | **−0.35** | The one meaningful relationship: better-rated products come back less. |
| rating ↔ units_sold | +0.13 | Mild: slightly better-rated items sell a few more units. |
| rating ↔ ad_spend | −0.08 | Nothing. |
| rating ↔ discount_pct | −0.01 | Nothing — discounting neither helps nor hurts ratings. |
| rating ↔ sale_price | −0.01 | Nothing. |
| rating ↔ temp | ≈ 0.00 | Nothing. |

**Bottom line:** ratings in this dataset behave almost like a **fixed quality score** that barely moves
with price, discount, advertising or weather. The only thing it tracks is returns.

---

## 13. 2024 vs 2025

| Metric | 2024 | 2025 | Change |
|---|---:|---:|---:|
| Revenue | $6,406,734 | $6,813,748 | **+6.3%** |
| Units sold | 97,712 | 104,788 | **+7.2%** |
| Ad spend | $1,098,180 | $1,239,350 | **+12.9%** |
| ROAS | 5.83× | 5.50× | **−0.33** |
| Average selling price (AOV) | $65.57 | $65.02 | −0.8% |
| Average rating | 4.24 | 4.20 | −0.04 |
| Average return rate | 17.2% | 17.4% | +0.2 pts |
| Avg discount | 22.9% | 22.2% | −0.7 pts |

**Revenue grew in every category:**

| Category | 2024 revenue | 2025 revenue | Change |
|---|---:|---:|---:|
| Dresses | $1,435,049 | $1,560,027 | +8.7% |
| Denim | $1,008,615 | $1,070,034 | +6.1% |
| Outerwear | $1,968,818 | $2,077,174 | +5.5% |
| Footwear | $1,362,279 | $1,432,204 | +5.1% |
| Accessories | $631,974 | $674,307 | +6.7% |

**How to read this:** 2025 was a year of **growth bought with more advertising**. Units grew 7.2%, but
ad spend grew almost twice as fast (12.9%), so efficiency fell (ROAS 5.83 → 5.50) and average selling
price slipped slightly. Quality perception (rating) softened marginally and return rate was flat.
In short: **more volume, slightly less profitable per dollar spent** — consistent with the
diminishing-returns pattern in §10.

---

## 14. What this means for the business (plain language)

1. **Seasonality is the biggest lever.** Nearly a quarter of all sales happen in November–December,
   and a third happen in just three months. Staffing, stock and ad budgets should be heavily weighted
   to Nov–Dec (and July for summer lines).
2. **Outerwear is the profit engine.** It earns the most revenue ($4.05M) from the *fewest* units, has
   the best ROAS (8.5×) and the cleanest returns among major lines — but it lives or dies by the
   weather (2.7× more sales when cold) and by the holiday quarter.
3. **Accessories are a volume trap.** Most units sold (49,086) but least revenue ($1.31M) and worst
   ROAS (2.81×). They move product but not much money.
4. **Discounting works on volume, not revenue.** Deeper discounts reliably sell ~50–130% more units,
   but because price falls, revenue only inches up (+0.20 correlation). Discounts are a unit-driving
   tool, not a revenue-maximising one.
5. **Advertising pays early and fades.** The first dollars return ~12×; the last return ~3.7×. Shifting
   budget from Accessories (2.8×) toward Outerwear (8.5×) and Store (6.3×) would likely improve return.
6. **Online returns are the biggest hidden cost.** 24.7% online vs 10.0% in store, peaking at ~30% for
   online Footwear and Dresses. Sizing/fit guidance, better photos and tighter size charts are the
   obvious fixes — this is the largest single improvement opportunity in the data.
7. **Ratings barely move**, so they are a weak management lever here; the one thing they do track is
   returns (better-rated → fewer returns).

---

## 15. Method notes and caveats

- **Correlation ≠ causation.** All relationships above are statistical associations in a 240-row
  dataset. For example, low ROAS at high spend could mean "spending more is wasteful" *or* "we spend
  more precisely because a month is hard to sell" — the data cannot tell these apart.
- **Confounded monthly effects.** Because `avg_temp_F` is constant per month, weather cannot be
  separated from holidays, campaigns and other monthly events that move together with it.
- **Cross-sectional vs within-group.** Several aggregate correlations (e.g. temperature ↔ units = −0.08)
  reverse or strengthen once you split by category or channel. Always check within-group.
- **Thin cells.** The 10% and 15% discount tiers (25 rows) and the 80–90°F temperature band (a single
  month) are sparsely populated; treat conclusions about them cautiously.
- **The file appears synthetically generated** (perfectly balanced grid, no missing values, round list
  prices, deterministic revenue identity, smooth seasonal temperature curve). The analysis is valid
  for *this* data; real-world figures would likely be noisier.

---

*Companion charts are in the `charts/` folder: revenue by month and category, Online vs Store revenue,
units vs discount, units vs temperature, revenue vs ad spend, return rate by category and channel,
plus total revenue/units by category and ratings by category and channel.*
