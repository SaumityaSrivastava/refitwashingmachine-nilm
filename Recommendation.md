# Recommendation — what a utility can (and cannot) do with this model

## Suitable uses

- **Monthly washing-machine energy per customer** from the whole-home meter only, best done with the plain LightGBM regressor (daily error ≈0.35 kWh/day on unseen homes vs 0.51 for "always off"). It is reliable at the month level, not the day level.
- **"How many washes and when" insights.** The final ensemble finds ≈70 % of real washes, and ≈61 % of the washes it reports are real (unseen homes, RESULTS.md). That is good enough for aggregated weekly patterns, not for individual washes.
- **Targeting load-shifting campaigns** at households with a recurring evening or weekend wash pattern (e.g. House 10: ≈2 starts/week around 19:00; House 5: Saturday-heavy).

## Not suitable for

- Billing or any decision about an individual wash. Minute-level timing is only moderately accurate (F1 ≈0.5 on unseen homes), and dishwashers and tumble dryers cause false alarms.
- Homes with very different appliances or habits without checking performance there first (results vary by house — see the ± std in RESULTS.md).
- 15/30-minute smart-meter data without retraining and re-validation (see below).

## 1. Practical opportunity: shift washes out of the evening peak
**Insight:** in the six analysed homes almost all washes start between 07:00 and 21:00. Most homes wash in the morning, but some have a clear evening habit: House 10 starts ≈2 washes/week around 19:00, in the evening network peak. A washing machine is flexible, so most loads could move to midday (solar), late evening, or overnight via the machine's delay timer.

**Action:** send a personalised nudge ("Your washes usually start around 18:00; using the delay timer to start after 21:00 would move ≈X kWh/month off-peak"), optionally with a time-of-use tariff incentive.

**How to measure:** randomised A/B test. Randomly split eligible customers (≥ 1 detected evening wash/week in a 4-week baseline) into treatment (nudge) and control. Primary metric: **share of detected washing-machine energy in the 16:00–20:00 window**, compared between groups after 8 weeks (difference-in-differences against the baseline). Secondary: total WM kWh (should be unchanged — shifting, not saving), opt-out rate. Because the *same* model scores both groups, its systematic errors cancel in the comparison; a sub-sample with plug monitors validates the estimate.

## 2. Limitations

- **Overlapping signatures:** dishwasher, tumble dryer (often run right after a wash), kettle and oven produce the same ≈2 kW heating blocks. These are the main source of false alarms in our failure analysis.
- **Missing data:** REFIT has long outages (House 1: 41 days), and the clean release writes gaps > 2 min as 0 W; without handling these as missing, a model learns "machine off" from missing data. Plug dropouts add label noise.
- **Household differences:** machine model, programmes (cold vs hot), number of occupants and routines vary. Performance on a new home is lower and more variable than on a home the model has seen; hence leave-one-house-out validation.
- **UK → India:** Indian homes differ strongly — many semi-automatic or top-load machines **without internal heaters** (the 2 kW block that the model relies on most), cold-water washing, voltage fluctuations, power cuts / inverter backup, and large confounding loads (air-conditioners, geysers, water pumps, ceiling fans) with strong seasonality. Usage times also differ (e.g. morning washes, water-supply timings). The UK model should **not** be applied directly; it needs Indian labelled data (even a few dozen plug-monitored homes) for retraining and validation, and probably a lower ON threshold and different features.

## 3. What gets harder with 15- or 30-minute smart-meter data
At 1-minute resolution a wash is a 30–120-minute *shape*: a heater block, drum spikes, a spin. At 15–30 minutes it collapses into 2–8 averaged points. The sharp switch-on edges and the heater block's flat top disappear, so the washing machine becomes indistinguishable from a dishwasher, tumble dryer, oven or kettle use in the same interval; short cold or quick washes may vanish into the base load entirely. Cycle counts and start times become uncertain (± one interval), and the useful output shrinks to coarse **daily or monthly energy estimates** with wider error bars, typically relying more on behavioural patterns (time of day, weekday) than on the electrical signature. Models would need to be retrained on down-sampled data and evaluated at that resolution — the 1-minute results here would overstate their accuracy.
