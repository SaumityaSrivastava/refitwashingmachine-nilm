# RESULTS.md

All numbers come from `notebooks/REFIT_full_pipeline.ipynb` (official CLEAN REFIT data, first 365 days of each house, 1-minute resolution). Figures are in `figures/`.

## 1. Houses and periods

The README lists a washing machine in 19 homes. Homes with **a second washing appliance** were excluded — the other machine has the same signature in the aggregate but is labelled "off", which is label noise for training and a source of false alarms for testing:

| Excluded | Reason (README) |
|---|---|
| 1, 8, 9 | Washer Dryer + Washing Machine |
| 4 | Washing Machine (1) + Washing Machine (2) |
| 18 | Washer Dryer (garage) + Washing Machine |

Used: **Houses 2, 3, 5, 6, 7, 10** for EDA and as unseen test houses; **Houses 11, 13, 15** as extra training-only houses.

**Data quality**

| House | WM channel | Period (365 days) | Missing % | Outage periods | Longest outage (h) | WM off % |
|---:|:--|:--|--:|--:|--:|--:|
| 2 | Appliance2 | 2013-09-17 → 2014-09-17 | 34.3 | 429 | 1488.6 | 95.3 |
| 3 | Appliance6 | 2013-09-25 → 2014-09-25 | 13.2 | 91 | 998.4 | 94.4 |
| 5 | Appliance3 | 2013-09-26 → 2014-09-26 | 8.7 | 770 | 40.8 | 93.6 |
| 6 | Appliance2 | 2013-11-28 → 2014-11-28 | 14.1 | 97 | 805.3 | 98.0 |
| 7 | Appliance5 | 2013-11-01 → 2014-11-01 | 12.5 | 95 | 998.4 | 92.0 |
| 10 | Appliance5 | 2013-11-20 → 2014-11-20 | 9.9 | 777 | 40.8 | 90.7 |

**Detected cycles**

| House | Cycles | Cycles/week | Median duration (min) | Median kWh/cycle | Median peak (W) | WM share of house energy % |
|---:|--:|--:|--:|--:|--:|--:|
| 2 | 164 | 4.78 | 94.5 | 0.574 | 2095 | 4.67 |
| 3 | 386 | 8.53 | 64 | 0.525 | 1934 | 4.16 |
| 5 | 396 | 8.32 | 42 | 0.263 | 2070 | 4.04 |
| 6 | 79 | 1.76 | 131 | 0.670 | 2064 | 1.52 |
| 7 | 473 | 10.37 | 59 | 0.519 | 2091 | 6.43 |
| 10 | 395 | 8.40 | 120 | 0.692 | 2080 | 4.32 |

Missing % includes minutes the official release wrote as 0 W (gaps > 2 min), which we treat as missing.

## 2. Washing-machine EDA (Part 2)

**Cycle rule (plain language):** the machine is *running* when its plug reads more than **20 W**. Pauses of up to **10 min** (soaking, drum rest) belong to the same wash. A wash must last at least **15 min** and use at least **0.05 kWh**, which removes standby blips and door-open spikes.

**Usage by hour and weekday** (`11_usage_hour_weekday.png`). Washing is almost entirely a daytime activity: nothing starts between 00:00 and 05:00. Most homes start washes between 07:00 and 12:00. House 7 has a sharp early peak (07–08 h, ≈2.2 starts/week), House 3 a late-morning one (08–11 h), and House 10 is clearly **evening-heavy** (≈2.1 starts/week at 19:00). By weekday, House 5 concentrates on **Saturday** (≈3.2 cycles/week vs ≈0.5 midweek), House 7 rises Friday–Saturday, and Houses 2 and 6 wash rarely and evenly (< 1 per weekday).

**Cross-house comparison** (`12_cycle_comparison_across_houses.png`). Peak power is similar everywhere (≈2.0–2.1 kW heater; House 3 a little lower at ≈1.93 kW), but programmes differ strongly. House 10 almost always runs one ≈120-min programme. House 6 runs long cycles (median 131 min, 0.67 kWh). House 5 mostly runs short, low-energy washes (median 42 min, 0.26 kWh), i.e. cold or quick programmes. These differences are why performance varies from home to home.

**Traces** (`13_representative_24h.png`, `14_detected_cycles_per_house.png`). A typical cycle is a ≈2 kW heater block (10–25 min), then 100–500 W drum and rinse activity, then a spin. The heater block stands out in the aggregate; the drum phase is buried under fridge, lighting and cooking loads.

## 3. Model and validation (Part 3)

**Task:** estimate washing-machine watts every minute from the aggregate only.

**Validation: leave-one-house-out.** Each of the 6 houses is the test house once. The next house is the validation house, used for early stopping, feature selection, the ON threshold τ and the model choice. The remaining 4 houses plus Houses 11, 13 and 15 are used for training. The test house plays no part in any choice, so the scores estimate accuracy for a **new customer**. A random minute split would put neighbouring minutes of the same wash in both training and test.

**Metrics:**

- MAE (W): overall watt error.
- MAE when ON: error during washes. Plain MAE rewards "always off", because the machine is off 91–98 % of the time.
- ON/OFF precision, recall and F1: is the timing right?
- Energy error % and daily-energy MAE: what a bill would show.
- Cycle recall and precision: share of real washes found, and share of predicted washes that are real.

### Results — mean ± std over the 6 unseen houses

| Model | MAE (W) | MAE when ON (W) | Precision | Recall | **F1** | Energy err % | Daily MAE (kWh) | Cycle recall | Cycle precision |
|:--|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| B0 Always-off | 24.8 ± 10.7 | 404 ± 51 | 0.00 | 0.00 | 0.00 | −100 | 0.511 ± 0.246 | 0.00 | – |
| B1 Hour-of-week profile | 41.4 ± 8.4 | 374 ± 51 | 0.09 | 0.73 | 0.16 ± 0.06 | +16 ± 96 | 0.484 ± 0.130 | 0.79 | 0.35 |
| A LightGBM regressor | 35.2 ± 11.0 | 297 ± 81 | 0.24 | 0.69 | 0.35 ± 0.11 | +19 ± 51 | **0.346 ± 0.117** | **0.87** | 0.37 |
| B Gated LightGBM (soft) | 37.9 ± 17.2 | 327 ± 80 | 0.23 | 0.65 | 0.33 ± 0.04 | +2 ± 44 | 0.434 ± 0.271 | 0.80 | 0.39 |
| B+ Gated LightGBM + cycle rule | 48.8 ± 34.1 | 295 ± 104 | 0.37 | 0.56 | 0.34 ± 0.21 | +67 ± 124 | 0.765 ± 0.533 | 0.66 | 0.39 |
| C+ Gated CNN + cycle rule | 34.8 ± 26.8 | 258 ± 91 | 0.50 | 0.45 | 0.42 ± 0.18 | +24 ± 87 | 0.484 ± 0.437 | 0.72 | **0.61** |
| E Ensemble LGBM+CNN + cycle rule | **30.2 ± 21.2** | **255 ± 103** | **0.57** | 0.61 | **0.50 ± 0.24** | +9 ± 79 | 0.433 ± 0.354 | 0.69 | **0.61** |
| **S Selected on validation house (final)** | 30.2 ± 21.2 | 255 ± 103 | 0.57 | 0.61 | **0.50 ± 0.24** | +9 ± 79 | 0.433 ± 0.354 | 0.69 | 0.61 |

In every fold the validation house picked the ensemble (E), so S = E.

**F1 per unseen house (final model):** House 2: 0.26 · House 3: 0.57 · House 5: 0.16 · House 6: 0.74 · House 7: 0.73 · House 10: 0.54.

Figures: `15_loho_ablation.png` (ablation), `16_actual_vs_predicted_unseen_house.png` (House 3, the median house by F1, not the best one), `17_daily_energy_all_houses.png`.

### Interpretation

- **Always-off has the lowest overall MAE but finds no washes.** This is why MAE alone is misleading here. On the metrics that matter (timing and cycles), the final model is clearly better than both baselines: F1 0.50 vs 0.16 for the usage profile, and 61 % of predicted cycles are real.
- **Ablation.**
  - Splitting into "is it running?" × "how many watts?" helps most when combined with the CNN. The ensemble beats each part on F1 (0.50 vs 0.34 for B+ and 0.42 for C+) and on MAE when ON (255 W).
  - The cycle rule helps the CNN and the ensemble (precision 0.57). On LightGBM alone it sometimes merges false alarms into long false cycles: B+ has the worst energy error and its spread across houses is large.
- **Energy:** across houses the final model's total energy error is small on average (+9 %) but varies a lot by house (± 79 %). For **daily energy the plain LightGBM regressor (A) is the most reliable** (0.35 kWh/day vs 0.51 for always-off). For a bill-style monthly estimate, A is the safer choice; for detecting *when* washes happen, the ensemble is.
- **Effect of excluding second-machine homes:** in an earlier run on Houses 1–6 (including House 1 with a washer-dryer and House 4 with two machines, without extra training houses) the ensemble reached F1 only 0.28 ± 0.20, and those two houses were the worst. The test houses differ between the two runs, so this is indicative, not a controlled comparison.
- `16_actual_vs_predicted`: on House 3 (unseen) the heater blocks and most washes are found, even on days with 7–9 washes. Daily predicted energy is 3.04 / 3.48 / 2.21 kWh vs actual 3.51 / 3.46 / 2.85 kWh. Misses happen when a wash overlaps a large load (e.g. 15:00 on 2013-11-25 under a ≈4 kW block).

## 4. Failure analysis
During false-alarm minutes, these appliances were running much more often than in normal OFF time (lift = ratio of the two):

| Appliance | ON during false alarms | ON during normal OFF time | Lift |
|---|--:|--:|--:|
| Tumble Dryer | 26.2 % | 3.7 % | 7.0 |
| Dishwasher | 32.6 % | 6.8 % | 4.9 |
| Magimix (blender) | 79.4 % | 25.5 % | 3.1 |
| Kettle | 3.0 % | 1.1 % | 2.8 |
| Microwave | 3.2 % | 1.6 % | 2.0 |

1. **Overlapping heating loads.** Tumble dryers (often run right after a wash) and dishwashers draw the same ≈2 kW heating blocks. They are the main cause of false alarms.
2. **Short / cold washes.** Of 1,893 true test cycles, 1,333 (70 %) were detected. Missed cycles are shorter (median 59 vs 80 min) and use less energy (0.35 vs 0.59 kWh) at a similar peak. In other words, the low-power drum phase is not recognised once the heater block is short or hidden under other loads. House 5 (mostly short, low-energy washes) has the lowest F1 (0.16).
3. **Sensor artefacts / label noise.** In the daily-energy plot House 5 has days with 6–28 kWh of "washing-machine" energy, which is physically impossible for a washing machine and points to stuck plug readings. House 10 has days with ≈10 kWh predicted but 0 kWh recorded, which looks like a plug dropout or a long confounding load. These days dominate the daily-energy error, and better cleaning of the plug channels (stuck-value detection) is a next step.
4. **House-to-house variation.** F1 ranges from 0.16 to 0.74. A new customer's accuracy is uncertain, and per-home calibration (a few labelled weeks) would help.

## 5. Single-house check (raw House 1, time split)
From an earlier single-house run on raw House 1 (chronological 70/15/15 split with a 2-day purge): gated LightGBM + cycle rule reached F1 0.80, cycle recall/precision 0.84/0.84 and daily-energy MAE 0.075 kWh (vs 0.186 kWh for always-off). The same-house score is much higher than the unseen-house score (F1 0.50), which shows how much of the difficulty is *generalising to a new home*.
