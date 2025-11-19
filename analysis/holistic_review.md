# Capstone Data & Notebook Review

## 1. Notebook structure and assumptions
- **Environment setup** – The first two cells import `pandas`, `numpy`, scikit-learn regressors, XGBoost, LightGBM, CatBoost, Matplotlib, and Seaborn, and configure an FBI Crime Data API key that is embedded directly in the notebook (`MaD4GVXCynpoocKq1c3wEuEdyFDw1jZ7VNu5JYxq`).【F:ML-Contextual.io - 1-2.ipynb†L1-L88】 This hard-coded key should be externalized before sharing.
- **Data ingestion** – Cell 3 reads `Insight Requests Report (Sanitized).xlsx`, reports 25,093 rows × 26 columns, and prints the column list, establishing that the Insight Request export is the primary labeled dataset for `vendorWageRate`.【F:ML-Contextual.io - 1-2.ipynb†L90-L145】
- **Feature engineering** – Multiple helper functions map ZIP codes to U.S. states, attach state-level crime aggregates, and merge vendor density data via ZIP-level joins. The pipeline now expects 24 engineered features, adding normalized vendor scale metrics (`VendorHoursTotal`, `VendorHoursAvg`, `VendorCount`) on top of the guard-level metadata, schedule descriptors, FBI crime rates, and competition metrics derived from the Vendor Density workbook.【F:ML-Contextual.io - 1-2.ipynb†L154-L800】【F:ML-Contextual.io - 1-2.ipynb†L960-L1050】
- **Vendor Density processing** – Cell 6B loads the “5 Digit Zip Codes” sheet from `Vendor Density 9.30.25.xlsx`, aggregates 24,543 vendor-ZIP rows into 6,569 ZIP codes, and calculates bucketed service-hour ratios plus multi-location vendor metrics used downstream (e.g., `SrvcHoursBk1-4`, `WO2MVendorRatio`, `AllAvailableVendors`, `Coverage`).【F:ML-Contextual.io - 1-2.ipynb†L688-L800】

## 2. Raw data review
### Insight Request export
- After excluding the header, the Insight workbook contains 25,093 request rows. `vendorWageRate` is populated for 21,208 rows (3,885 missing), `vendorRate` for 22,336 rows, and ZIP codes for 25,004 rows (89 missing). Guard levels are nearly complete (390 missing), but schedule details (`hoursPerWeek`, `daysPerWeek`) have ~4.5k gaps each. ZIP entries cover 4,206 unique codes, of which roughly 361 appear to be non-U.S. postal codes because they begin with letters.【7b76f8†L1-L14】
- Wage labels range from $0–$85 (mean ≈ $21.53). Guard level mix is skewed toward `3a` (13,274 instances), with `2c`, `2b`, `2a`, and `3c` forming the next largest groups, suggesting that most data reflects mid- to high-tier guard posts.【7b76f8†L9-L14】

### Vendor Density workbook
- Sheet **“5 Digit Zip Codes”** supplies 24,543 vendor-ZIP-hour observations spanning 6,569 ZIP codes and 1,658 unique vendors. The mean logged hours per vendor-ZIP pair is ~554, with a long tail up to ~77k hours and 702 zero-hour entries that can signal inactive listings needing filtering. Sheet **“3 Digit Zip Codes”** adds 8,917 aggregated rows covering 995 ZIP3 prefixes for coarse coverage mapping.【7b76f8†L14-L20】
- Sheet **“Vendor County Hours Worked”** contributes 8,701 county-level hour totals (useful when ZIP data is missing), while **“Vendor Country CAN Work List”** adds 41,350 Canadian county-level entries that should be excluded when training the U.S.-only wage model unless Canadian jobs are explicitly targeted.【7b76f8†L18-L21】

## 3. Data cleaning outcomes
- The notebook’s dedicated cleaning cell removes 466 non-U.S. postal codes, 3,867 rows with missing wages, 24 rows with non-positive wages, and 4,645 rows lacking critical fields before reporting a 16,091-row modeling dataset (9,002 rows removed, 35.9% of the original). Post-cleaning diagnostics confirm no NaN/inf in `vendorWageRate`, a wage range of $1–$85, and complete coverage for `guardLevel`, `hoursPerWeek`, `daysPerWeek`, and `zipCode`.【F:ML-Contextual.io - 1-2.ipynb†L1033-L1184】
- A pre-training validation cell highlights that the 24-feature matrix initially contained 13,620 NaNs (primarily in schedule-derived features) and that 3,885 wage targets were blank; filtering down to the 16,700 valid samples resolves these issues before the train/test split.【F:ML-Contextual.io - 1-2.ipynb†L1187-L1291】

## 4. Modeling results
- After an 80/20 split (13,360 train / 3,340 test), four regressors are trained with tuned hyperparameters: Random Forest (R²≈0.517), XGBoost (0.527), LightGBM (0.464), and CatBoost (0.427).【F:ML-Contextual.io - 1-2.ipynb†L1300-L1658】
- A weighted ensemble of the four models yields R²=0.5127, MAE=$1.73, and RMSE=$2.88, with most predictions within $2 (73.3%) or within 10% (75.4%) on the hold-out set. The top drivers are violent/property crime rates (`murder`, `arson`, `vehicleTheft`, `drugCrimes`, `identityTheft`), competition metrics (`AllAvailableVendors`, `WO2MVendorRatio`, service-hour buckets), and guard-level attributes (`GuardLevel`, `SchLen`, `HPD`).【F:ML-Contextual.io - 1-2.ipynb†L1670-L1868】

## 5. Recommended next steps
1. **Externalize sensitive configuration** – Move the FBI API key to environment variables or a config file ignored by version control.
2. **Augment data coverage** – Backfill `hoursPerWeek`/`daysPerWeek` gaps (4.4–4.5k rows) or engineer proxies (e.g., schedule templates) to avoid dropping potentially valuable labeled examples.
3. **Handle zero-hour vendor density entries** – Decide whether 702 zero-hour vendor-ZIP rows represent dormant relationships and exclude or impute them before deriving competition buckets.
4. **Canadian vendor list segregation** – Ensure the `Vendor Country CAN Work List` sheet is only used for Canadian modeling; mixing it with U.S. data introduces inconsistent wage baselines.
5. **Error analysis** – Investigate the 90th–95th percentile absolute errors ($3.66–$5.09) and the worst-case $40 outliers to determine whether certain industries, customer types, or geographies require bespoke sub-models.
