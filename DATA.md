# DATA.md — Data coverage, quality and cleaning

## 1. Dataset
**REFIT Electrical Load Measurements** (University of Strathclyde; Murray, Stankovic & Stankovic, *Scientific Data* 2017, CC BY 4.0).
20 households in Loughborough, UK, Oct 2013 – Jun 2015. Each house has one whole-home **aggregate** channel and nine plug-level **appliance** channels (`Appliance1–9`), active power in **watts**, logged roughly every **8 s**. Appliance names per channel come from the official README (`CLEAN_READ_ME_081116.txt`); the CSVs only contain `Appliance1..9`.

| Use | Files | Notes |
|---|---|---|
| Part 1 – raw cleaning | `RAW_House1_Part1.csv` + `RAW_House1_Part2.csv` (REFIT_RAW_081116), concatenated | House 1, 2013-10-09 → 2015-07-10 (638.9 days) |
| Parts 2–3 – EDA & model | `CLEAN_House{N}.csv` (CLEAN_REFIT_081116) | houses listed in RESULTS.md |

House 1 channels (README): 1 Fridge, 2 Freezer (1), 3 Freezer (2), 4 Washer Dryer, **5 Washing Machine**, 6 Dishwasher, 7 Computer, 8 Television Site, 9 Electric Heater. The mapping was cross-checked visually: `Appliance5` shows the expected washing-machine signature (≈2.3 kW heater block followed by 100–500 W drum activity).

## 2. Raw House 1 — quality findings (before cleaning)

| Check | Finding |
|---|---|
| Rows | 8,518,763 (Part1 4,377,877 + Part2 4,140,886) |
| Period | 2013-10-09 14:06 → 2015-07-10 11:56 UTC (638.9 days) |
| Unparseable timestamps | 0 (Unix seconds, UTC) |
| Out-of-order steps | 2 |
| **Duplicate timestamps** | **1,211,048 (14.2 %)** — 884,707 exact duplicate rows; ≈326k duplicates carry *different* values |
| Sampling interval | median 7 s, 95th percentile 13 s |
| Intervals > 60 s / > 5 min / > 1 h | 4,040 / 390 / 21 |
| Longest gap | **998 h (≈41.6 days, from 2014-01-24)** |
| Missing values | aggregate 188,973 rows; appliances 232,752–510,963 per channel (Appliance9 1,911,811) |
| Negative power | none |
| Aggregate > 20 kW | 127 readings |
| Appliance > 4 kW (above plug-monitor rating) | 36–2,085 per channel (most on Appliance4 washer-dryer: 2,085; Appliance1: 846) |
| Sub-meters sum > aggregate | 50,534 rows |

## 3. Cleaning pipeline (documented rules)
1. Parse `Unix` as UTC (avoids the repeated hour at the end of British Summer Time); local time (`Europe/London`) only for hour/day features.
2. **Stable sort** by timestamp.
3. **De-duplicate: keep the first reading** for each timestamp (later copies treated as logger re-transmissions).
4. **Impossible values → missing:** negative power; aggregate > 20 kW (UK domestic supply 60–100 A × 230 V ≈ 14–23 kW); single appliance > 4 kW (plug monitors are rated ≈ 3 kW at 13 A; the official clean release uses the same 4 kW limit).
5. **Resample to 1-minute means.**
6. **Short gaps (≤ 5 consecutive minutes) interpolated in time** — too short to invent or hide an appliance cycle.
7. **Longer gaps flagged `outage = True` and left missing**; they are excluded from modelling, never filled.

| Result | Value |
|---|---|
| Duplicates removed | 1,211,048 |
| Values set to missing (impossible) | aggregate 104; appliances 33–1,541 per channel |
| 1-min grid | 920,031 minutes |
| Missing minutes before filling | 115,143 |
| Minutes filled (≤ 5 min gaps) | 770–3,119 per channel (aggregate 859) |
| **Minutes flagged as outage** | **114,284 (12.4 %)** in 376 periods, total 1,905 h; longest 998 h (from 2014-01-24), then 208 h (2015-05-10), 180 h (2015-06-10), 105 h (2015-04-03) |

Figures: `figures/01_intervals.png`, `02_daily_coverage.png`, `03_before_after.png` (aggregate + two appliances, raw 8-s vs clean 1-min).

## 4. Official CLEAN data — important property
Per the dataset page, the official cleaning merged duplicate timestamps, set plug readings > 4 kW to 0 W, **forward-filled gaps < 2 min and wrote gaps > 2 min as 0 W**. A zero in the clean files therefore often means *no data*, not *appliance off*. Our loader treats minutes whose **aggregate is 0 W** as missing (a home never draws exactly 0 W), then applies the same ≤ 5-min interpolation / outage flag as above.

## 5. Remaining concerns

- ≈326k duplicate timestamps had *different* values; "keep first" is a convention, not a correction.
- **Stuck (frozen) readings are not caught by the rules above.** The before/after plot (`03_before_after.png`, 2015-02-08) shows the aggregate frozen at ≈12.5 kW and the washing machine at ≈2.25 kW for ≈3 h (13:20–16:15) — physically implausible, a logger freeze. A next step is to flag runs of identical values longer than e.g. 30 min. The same artefact appears in the clean plug data (RESULTS.md §4).
- Plug readings > 4 kW were removed; genuine short inrush peaks above 4 kW are lost.
- Long outages (41 days in House 1, several 4–9-day gaps in spring 2015) leave seasonal holes; usage statistics are per *observed* week.
- House 1 has a **washer-dryer** (Appliance4) next to the washing machine (Appliance5): two machines with near-identical signatures in one aggregate. Such homes were excluded from the multi-house model (see RESULTS.md).
- Washing-machine plug dropouts (plug switched off at the wall while the aggregate still records) look like "machine off" and add label noise.
- Appliance-to-channel mapping relies on the README; appliances changed in some houses during the study (documented there).
- Unmetered loads (lighting, oven, hob, electric shower) are only in the aggregate and are the main source of confusion for disaggregation.
