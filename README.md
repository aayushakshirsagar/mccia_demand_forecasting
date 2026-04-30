# 🌅 AI Demand Forecasting System

A demand forecasting and inventory optimization system built for Sunrise Consumer Goods, covering 140 SKUs across multiple outlets. Produces 6-week forward forecasts, automated reorder recommendations, and business intelligence reports.

---

## Project Structure

```
sunrise_demand/
├── data/
│   ├── sales_history.csv
│   ├── inventory_snapshot.csv
│   ├── sku_master.csv
│   ├── outlet_master.csv
│   ├── promotions_calendar.csv
│   └── festive_calendar.csv
├── 01_eda_and_zeroclassification.ipynb
├── 02_forecasting.ipynb
├── 03_reorder_engine.ipynb
├── 04_monday_report.ipynb
├── outputs/
│   ├── D1_forecast_6weeks.csv
│   ├── D2_zero_classification.csv
│   ├── D3_reorder_recommendations.csv
│   ├── D4_monday_morning_report.html
│   ├── D5_sku_classification.csv
│   └── D6_diwali_retrospective.csv
└── README.md
```

---

## Deliverables

| File | Description |
|------|-------------|
| `D1_forecast_6weeks.csv` | 6-week forward demand forecast for all SKUs |
| `D2_zero_classification.csv` | True zero vs. missing data classification for every outlet-SKU-week |
| `D3_reorder_recommendations.csv` | Reorder quantities with MOQ, shelf-life, and safety stock constraints |
| `D4_monday_morning_report.html` | Auto-generated weekly HTML report for operations teams |
| `D5_sku_classification.csv` | SKU segmentation: Fast Mover, Slow Mover, Seasonal, Dead Stock |
| `D6_diwali_retrospective.csv` | Diwali 2023 retrospective — stockout identification across 14 SKUs |

---

## Methodology

### Forecasting Model — LightGBM (Global Model)
A single LightGBM model is trained across all 140 SKUs rather than fitting individual models per SKU. This approach:
- Avoids overfitting on sparse or short individual series
- Captures cross-SKU demand patterns
- Runs in seconds vs. 20–40 minutes for per-SKU SARIMA

**Features used:**
- Lag features: lag-1, 2, 3, 4, 8, 12 weeks
- Rolling statistics: 4-week and 12-week mean and standard deviation
- Calendar features: week of year, month, quarter
- Festive features: Diwali proximity flags (pre-Diwali weeks 39–41, Diwali weeks 42–44)
- Promotion flag from `promotions_calendar.csv`

Forecasting is done recursively — predictions for each week are fed back as lag features for the next step.

---

### True Zero Classification (D2)
A missing row in the sales data is **not** automatically a zero — it could mean the outlet doesn't carry that SKU, or simply didn't report that week. The classification uses a 3-rule decision tree:

```
Is the row missing?
├── No  → ACTUAL_SALE
└── Yes → Does the outlet carry this SKU? (has prior sales history)
          ├── No  → MISSING_DATA_NOT_LISTED
          └── Yes → Did the outlet report any sales that week?
                    ├── No  → MISSING_DATA_NO_REPORT
                    └── Yes → TRUE_ZERO
```

Only `TRUE_ZERO` rows are filled with `0` for model training. All other missing values are excluded, preventing the model from learning false demand suppression.

---

### Reorder Engine (D3)
Recommended order quantity is calculated as:

```
available_stock     = warehouse_stock + in_transit_qty − committed_qty
safety_stock        = avg_weekly_sales × lead_time_weeks × 1.5
units_needed        = total_forecast_6wk + safety_stock − available_stock
shelf_life_cap      = avg_weekly_sales × (shelf_life_days / 7) × 0.8
order_qty           = ceil(min(units_needed, shelf_life_cap) / MOQ) × MOQ
```

The shelf-life cap is applied **before** rounding to MOQ, ensuring no order can exceed what can be sold within the product's shelf life.

Each SKU is also flagged for:
- `stockout_risk`: available stock below safety stock threshold
- `overstock_risk`: available stock exceeds 1.5× the 6-week forecast
- `shelf_life_violation`: order quantity would breach shelf-life limits

---

### Diwali 2023 Retrospective (D6)
Expected Diwali 2023 demand is estimated using Diwali 2022 uplift ratios applied to pre-Diwali 2023 baselines:

```
uplift_2022           = diwali_2022_sales / pre_diwali_2022_avg
expected_diwali_2023  = pre_diwali_2023_avg × uplift_2022
demand_gap            = expected − actual
stockout_probability  = demand_gap / expected  (clipped 0–1)
```

The 14 SKUs with the highest `stockout_probability` are flagged as likely Diwali stockout candidates.

---

### SKU Classification (D5)

| Class | Criteria |
|-------|----------|
| `DEAD_STOCK` | Zero-sale rate > 60% |
| `FAST_MOVER` | Avg weekly sales > 75th percentile |
| `SLOW_MOVER` | Avg weekly sales < 25th percentile |
| `SEASONAL` | Coefficient of variation > 0.5 |
| `REGULAR` | Everything else |

---

## Setup & Requirements

```bash
pip install pandas numpy lightgbm scikit-learn
```

Run notebooks in order:
1. `01_eda_and_zeroclassification.ipynb`
2. `02_forecasting.ipynb`
3. `03_reorder_engine.ipynb`
4. `04_monday_report.ipynb`

All outputs are written to the `outputs/` directory.

---

## Key Design Decisions

- **LightGBM over SARIMA**: Per-SKU SARIMA fitting for 140 SKUs is prohibitively slow and fails silently on sparse series. A global LightGBM model is faster, more accurate, and generalizes better across SKUs with limited history.
- **Shelf-life before MOQ**: Capping order quantity by shelf-life *before* rounding to MOQ prevents over-ordering on short-shelf-life products.
- **Zero classification before training**: Feeding unclassified missing data as zeros would bias the model toward under-forecasting. Classification is a prerequisite, not an afterthought.
