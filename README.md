# REFIT washing-machine disaggregation (NILM)

Reproducible pipeline that (1) cleans raw REFIT House 1 data, (2) explores washing-machine use across several households, and (3) estimates washing-machine power from the whole-home aggregate signal at 1-minute resolution, evaluated on **households the model has never seen**.

## Repository
```
notebooks/
  REFIT_full_pipeline.ipynb         end-to-end: download → Part 1 (raw House 1) → Parts 2–3 (multi-house, LOHO models)
figures/                            all plots referenced in the markdown files
docs/REFIT_NILM_Report.pdf          full written report (all parts, figures, references)
DATA.md                             data coverage, quality findings, cleaning decisions
RESULTS.md                          EDA, model results, validation, failure analysis
Recommendation.md                   one-page utility summary + Section 4 answers
requirements.txt
```

## How to run (Google Colab)
1. Open `notebooks/REFIT_full_pipeline.ipynb` in Colab, select **Runtime → Change runtime type → T4 GPU**, then **Runtime → Run all**.
2. The notebook downloads the official archives (`CLEAN_REFIT_081116.7z`, `REFIT_RAW_081116.7z`) and the clean README itself, extracts only the needed CSVs, and selects the houses whose README lists a washing machine (up to 6). Archives already on Drive under `MyDrive/refit/` are reused.
3. Outputs (cleaned parquet, metrics CSVs, figures, auto-generated markdown tables and a zip of everything) are written to `MyDrive/refit_outputs/`. Finished folds are cached, so a disconnected run resumes with **Run all**.

Data are not committed (≈1 GB). Settings (`HOUSES`, `MAX_DAYS_PER_HOUSE`, `RUN_PART1`, …) are in the single config cell. Runtime with defaults (6 houses × first 365 days, T4 GPU) ≈ 60–90 min.

## What was built
| Step | Choice | Why |
|---|---|---|
| Timestamps | `Unix` (UTC); local time only for calendar features | avoids the duplicated BST hour |
| Duplicates | keep first reading | simple, stated rule |
| Invalid values | <0 W, aggregate > 20 kW, appliance > 4 kW → missing | physical limits of supply / plug monitor |
| Resampling | 1-min mean | regular grid for features and models |
| Gaps | ≤ 5 min interpolated; longer flagged and excluded | never invent appliance activity |
| Clean-data zeros | aggregate = 0 W treated as missing | official cleaning writes gaps > 2 min as 0 W |
| Cycle rule | > 20 W, pauses ≤ 10 min merged, ≥ 15 min, ≥ 0.05 kWh | removes standby / blips; follows Kelly & Knottenbelt (2015) |
| Features | aggregate only: multi-scale centred windows (5–121 min), 24-h base-load removal, per-window normalisation, heater-block counts, calendar | in deployment only the aggregate exists |
| Feature reduction | variance → correlation (|r| > 0.97) → permutation importance on validation house | removes redundant window statistics |
| Model | **Gated LightGBM**: classifier P(ON) × regressor W∣ON, threshold tuned on validation house, then the EDA cycle rule on the predicted ON mask | the machine is off > 95 % of the time; separating "is it running?" from "how many watts?" removes the low-level false power a single regressor spreads everywhere |
| Comparison | always-off, hour-of-week profile, single LightGBM, gated Seq2Point CNN, LGBM+CNN ensemble | baselines + ablation |
| House selection | houses whose README lists exactly one washing appliance; homes with a washer-dryer or two machines (e.g. Houses 1, 4) excluded | a second machine's signature in the aggregate would be labelled *off* (label noise) |
| Validation | **leave-one-house-out** over the test houses: test house unseen, next house = validation, rest + 3 extra training-only houses = training; final model = candidate with best F1 on the validation house | estimates performance for a new customer; no minute of the test house influences training, feature selection, thresholds or model choice |
| Metrics | MAE (W), MAE when ON, ON/OFF precision/recall/F1, energy error %, daily-energy MAE (kWh/day), cycle recall/precision | MAE alone rewards predicting zero; F1 and cycle metrics measure *when*, energy metrics measure what a bill shows |

## Literature used (what was adopted)
| Ref. | Idea adopted in this work |
|---|---|
| [1] Murray et al. 2017 | dataset description: sampling (~8 s), units (W), known gaps and limitations |
| [2] Hart 1992 | step/edge features (`d1`, `abs_d1`, step counts) for ON/OFF switching events |
| [3] Kelly & Knottenbelt 2015 | ON threshold and minimum-duration rule for washing-machine activations; unseen-house evaluation |
| [4] Zhang et al. 2018 | centred sliding window predicting the midpoint (Seq2Point) — used for the window features and the CNN |
| [5] Shin et al. 2019 | splitting the task into "is it ON?" and "how many watts?" sub-tasks (gating) — in LightGBM and the CNN |
| [6] Petralia et al. 2025 | per-window normalisation against non-stationarity — adapted as window z-scores / window-min removal |
| [7] Makonin & Popowich 2015 | choice of NILM metrics (MAE, F1, energy error) |

Large models (transformers, diffusion) were considered but not used: with a handful of training homes they are hard to validate, and the assignment values simple, justified solutions. Their key ideas were adapted into a lightweight model.

## Assumptions & trade-offs

- Centred windows use up to 60 min of *future* aggregate: fine for disaggregating historical data (bills, reports); a real-time version would need trailing windows.
- Training minutes far from any wash and without extra load are sub-sampled (15 %) and re-weighted, to keep runtime reasonable on 2 years × several homes.
- The appliance-to-channel mapping relies on the README and was checked visually.
- One validation house per fold is small; thresholds may not be optimal for every home.

## AI tools used
Claude (Anthropic) was used to discuss the approach, write and debug the notebook code, and draft the markdown files and report. All code was run on Google Colab, and the results, figures and text were checked and edited by me.

## Possible improvements

- More houses (all REFIT homes with a washing machine) and nested cross-validation for the threshold.
- Hard-negative mining for dishwasher / tumble-dryer periods; multi-appliance joint models.
- Probabilistic outputs (energy with uncertainty bands) for customer-facing reports.
- Re-evaluate at 15/30-min resolution to match typical smart-meter data.

## References
1. Murray, D., Stankovic, L., & Stankovic, V. (2017). An electrical load measurements dataset of United Kingdom households from a two-year longitudinal study. *Scientific Data*, 4, 160122. https://doi.org/10.1038/sdata.2016.122
2. Hart, G. W. (1992). Nonintrusive appliance load monitoring. *Proceedings of the IEEE*, 80(12), 1870–1891. https://doi.org/10.1109/5.192069
3. Kelly, J., & Knottenbelt, W. (2015). Neural NILM: Deep neural networks applied to energy disaggregation. In *Proc. 2nd ACM International Conference on Embedded Systems for Energy-Efficient Built Environments (BuildSys '15)*, 55–64. https://doi.org/10.1145/2821650.2821672
4. Zhang, C., Zhong, M., Wang, Z., Goddard, N., & Sutton, C. (2018). Sequence-to-point learning with neural networks for non-intrusive load monitoring. *Proceedings of the AAAI Conference on Artificial Intelligence*, 32(1). https://doi.org/10.1609/aaai.v32i1.11873
5. Shin, C., Joo, S., Yim, J., Lee, H., Moon, T., & Rhee, W. (2019). Subtask gated networks for non-intrusive load monitoring. *Proceedings of the AAAI Conference on Artificial Intelligence*, 33(01), 1150–1157. https://doi.org/10.1609/aaai.v33i01.33011150
6. Petralia, A., Charpentier, P., Kadhi, Y., & Palpanas, T. (2025). NILMFormer: Non-intrusive load monitoring that accounts for non-stationarity. In *Proc. 31st ACM SIGKDD Conference on Knowledge Discovery and Data Mining (KDD '25)*. https://doi.org/10.1145/3711896.3737251
7. Makonin, S., & Popowich, F. (2015). Nonintrusive load monitoring (NILM) performance evaluation. *Energy Efficiency*, 8(4), 809–814. https://doi.org/10.1007/s12053-014-9306-2

**Data:** REFIT: Electrical Load Measurements (raw and cleaned), University of Strathclyde, CC BY 4.0 — https://pureportal.strath.ac.uk/en/datasets/refit-electrical-load-measurements/
