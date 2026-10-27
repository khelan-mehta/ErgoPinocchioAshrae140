# Section 9: space-cooling equipment (CE100–CE200, CE300–CE545), run through Pinocchio-EP

**Result: Annex A3 Table A3-7 — 14 of 14 range cases within range (12 required) → the cooling-equipment test group PASSES.**
CE100–CE200 agree with the quasi-analytical solutions to within −0.3 % … −1.8 % (total energy), −0.1 % … −1.8 % (compressor),
latent coil loads within 0.03 %. No case was omitted; all 34 runs completed with 0 severe errors.

Software: Pinocchio-EP 1.0.0+b0c9cf1.dirty (branch `ashrae140-compliance`, working tree 2026-10-27), HVAC built in Pinocchio
Studio, IDF written by Pinocchio's own exporter (`exportIdf`), engine EnergyPlus 26.1.0-6f2e40d102 (unmodified, the deployed build).

## What was run, and how

| Item | How (all through Pinocchio) |
|---|---|
| Weather | The **normative** files `CE100A/CE110A/CE165A/CE200A.TM2` and `CE300.TM2` (Std 140-2023 accompanying files, ASHRAE public download) converted to EPW with **Pinocchio's own weather converter** (`src/lib/weather/tmy2.js`, the Project-step import). No substitute weather was needed. CE300 check: mean ODB 19.91 °C (A1-7: 19.9), max 34.8 °C, mean W 0.01161 (0.0116). |
| Performance maps | The normative electronic maps `CE100MAP.xls` (Table 9-7, gross) and `CE300MAP.xls` (Table 9-16) → `src/lib/std140/dxMapData.js` (`harness/gen_mapdata.py`; a hand transcription of the printed Table 9-16 agrees to 0.005 kW). Fitted inside the app by the new Studio op **`fit_dx_coil_map`** (DX coil inspector › *Performance map* tab): rated gross capacity / SHR / COP at the AHRI point + biquadratic CapFT, EIRFT(EWB, ODB) + PLF = CDF. Fit: CE100 capacity rms 0.11 % (max 0.20 %), power rms 0.22 %; CE300 capacity rms 0.64 % (max 2.9 %), power rms 0.66 %. |
| Building | Pinocchio project per case (`harness/build.mjs`): one zone 8×6×2.7 m (CE100) / 14×14×3 m (CE300), custom constructions of two Material:NoMass layers (R 100 resp. 325 m²K/W total; solar abs. 0.1 out / 0.6 in), floor exposed to outdoor air, **Standard 140 profile** on (no DST/holidays, no auto loads). Sensible + latent gains = two Interior-equipment loads (fraction latent 0 / 1, 100 % convective) on dated Schedule:Compact schedules (Tables 9-15, 9-20…9-23). Infiltration = Flow/Zone on a schedule (CE320, CE340). Cooling set point via the Schedules step (CE350 dated). Heating off (−60 °C set point, no heater). |
| HVAC | Pinocchio Studio ops (the ops the Studio GUI writes): air loop → [OA system: Controller:OutdoorAir, min = max flow 1.888 m³/s × OA-fraction schedule] → **AirLoopHVAC:UnitarySystem** (Load control, no dehumidification control) with **Fan:OnOff** draw-through (230 W @ 0.425 m³/s; 1242 W @ 1.888 m³/s, motor in airstream) + **Coil:Cooling:DX:SingleSpeed**; fan operating-mode schedule 0 = cycling (CE100–CE200, CE500–CE545), 1 = continuous (CE300–CE440); CV terminal. Economizers: DifferentialDryBulb (CE400), + LockoutWithCompressor (CE410), + 20 °C limit (CE420), DifferentialEnthalpy (CE430), + 47 250 J/kg limit (CE440); mixed-air reference from SetpointManager:SingleZone:Cooling. Outdoor fan power folded into the coil EIR and split back out in post-processing (= P_od × run-time fraction). |
| Runs | `harness/run.mjs`: `energyplus -x -r -w <epw>` (bundled 26.1), full year, timestep 4/h (app default). CE100 series reported for February of the annual run. |
| Outputs | `harness/post.py` → the **official** `Std140_CE_a_Output.xls` and `Std140_CE_b_Output.xls` (normative templates filled with xlutils: every 9.3 quantity incl. maxima with dates/hours, CE300 June 28 hourly, CE500/CE510 May–Sep, CE500/CE530 Apr 30 / Jun 25), `scoring.json`, and the comparison with the Annex B16 example / quasi-analytical results (`harness/example_results_extract.json`). |

Reproduce (from `pinocchio-ep/`): `node tools/std140/ce/weather.mjs "<Accompanying FIles>" <results>/weather` →
`PINOCCHIO_APP_VERSION=… node tools/std140/ce/build.mjs <results>` → `node tools/std140/ce/run.mjs <results>` →
`python tools/std140/ce/post.py <results> "<Accompanying FIles>"` → `python tools/std140/ce/report.py <results>`.
Canonical scripts: `pinocchio-ep/tools/std140/ce/` (copied in `harness/` with `src/lib/std140/*.js`).

## Results

### Annex A3 Table A3-7 (annual total cooling energy, kWh/yr): 14 of 14 within range (12 required) -> PASS

| Range case | Pinocchio-EP | Lower | Upper | Within |
|---|---:|---:|---:|:-:|
| CE300 | 35039 | 33497 | 37023 | yes |
| CE310-CE300 | 4327 | 2671 | 6197 | yes |
| CE320-CE300 | 4381 | 2610 | 6136 | yes |
| CE330-CE320 | 1108 | -766 | 2760 | yes |
| CE330-CE300 | 5490 | 3697 | 7223 | yes |
| CE340-CE300 | 5085 | 3172 | 6698 | yes |
| CE330-CE340 | 404 | -1393 | 2133 | yes |
| CE350-CE300 | -3639 | -5354 | -1828 | yes |
| CE360-CE300 | 20002 | 18084 | 21610 | yes |
| CE400-CE300 | -3719 | -5783 | -2179 | yes |
| CE410-CE300 | -3719 | -5929 | -371 | yes |
| CE420-CE300 | -2051 | -3945 | -419 | yes |
| CE430-CE300 | -2928 | -4762 | -1236 | yes |
| CE440-CE300 | -1746 | -3694 | -168 | yes |

### CE100-CE200 (February) vs the quasi-analytical solutions (Annex B16.5.1: TUD, HTAL1, HTAL2)

| Case | Total kWh | QAS range | dev vs QAS mean | Compressor kWh | QAS range | Indoor fan kWh | Outdoor fan kWh | Coil total kWh | Coil latent kWh | QAS latent | Mean COP | QAS COP | Mean IDB C | Mean W kg/kg | QAS W |
|---|---:|---|---:|---:|---|---:|---:|---:|---:|---:|---:|---|---:|---:|---|
| CE100 | 1516.3 | 1530.5-1530.8 | -0.94% | 1306.8 | 1318.9-1319.0 | 142.6 | 67.0 | 3796.3 | -0.0 | 0.0 | 2.410 | 2.389-2.390 | 22.21 | 0.00737 | 0.00734-0.00743 |
| CE110 | 1062.8 | 1076.2-1077.4 | -1.30% | 876.8 | 887.9-889.4 | 126.6 | 59.4 | 3762.2 | 0.0 | 0.0 | 3.421 | 3.379-3.380 | 22.20 | 0.00646 | 0.00640-0.00652 |
| CE120 | 1004.3 | 1011.0-1012.7 | -0.74% | 834.2 | 839.1-840.9 | 115.8 | 54.4 | 3746.2 | -0.0 | 0.0 | 3.615 | 3.587-3.590 | 26.70 | 0.00790 | 0.00786-0.00788 |
| CE130 | 108.0 | 109.5-110.6 | -1.84% | 93.1 | 94.3-95.3 | 10.2 | 4.8 | 216.6 | -0.0 | 0.0 | 1.911 | 1.890-1.910 | 22.20 | 0.00737 | 0.00734-0.00743 |
| CE140 | 67.4 | 68.3-69.0 | -1.78% | 55.6 | 56.4-56.9 | 8.0 | 3.8 | 196.3 | 0.0 | 0.0 | 2.792 | 2.750-2.770 | 22.20 | 0.00647 | 0.00640-0.00652 |
| CE150 | 1192.6 | 1206.5-1206.5 | -1.15% | 987.9 | 999.0-999.2 | 139.3 | 65.4 | 4507.7 | 739.2 | 739.3 | 3.663 | 3.627-3.630 | 22.20 | 0.00833 | 0.00820-0.00823 |
| CE160 | 1130.5 | 1138.9-1140.4 | -0.80% | 943.1 | 949.1-950.4 | 127.6 | 59.9 | 4490.1 | 739.2 | 739.2 | 3.859 | 3.833-3.840 | 26.70 | 0.01016 | 0.00994-0.00997 |
| CE165 | 1488.6 | 1497.8-1499.7 | -0.68% | 1271.5 | 1278.7-1280.2 | 147.7 | 69.4 | 4527.5 | 739.2 | 739.2 | 2.942 | 2.930-2.930 | 23.31 | 0.00930 | 0.00920-0.00927 |
| CE170 | 633.5 | 637.7-641.1 | -0.92% | 527.0 | 530.0-533.0 | 72.5 | 34.0 | 2224.3 | 739.2 | 739.3 | 3.397 | 3.367-3.390 | 22.20 | 0.01050 | 0.01037-0.01045 |
| CE180 | 1078.9 | 1081.9-1082.7 | -0.31% | 906.6 | 907.7-908.1 | 117.3 | 55.1 | 4479.9 | 2956.8 | 2956.8 | 4.044 | 4.040-4.042 | 22.20 | 0.01620 | 0.01619-0.01623 |
| CE185 | 1533.7 | 1542.9-1544.6 | -0.65% | 1331.2 | 1338.2-1339.8 | 137.8 | 64.7 | 4520.7 | 2956.8 | 2956.9 | 2.858 | 2.846-2.850 | 22.20 | 0.01608 | 0.01605-0.01606 |
| CE190 | 163.6 | 164.1-165.2 | -0.60% | 137.4 | 137.6-138.5 | 17.8 | 8.4 | 574.1 | 369.6 | 369.7 | 3.399 | 3.386-3.410 | 22.20 | 0.01586 | 0.01580-0.01590 |
| CE195 | 248.7 | 250.0-252.2 | -0.96% | 215.7 | 216.8-218.6 | 22.4 | 10.5 | 597.1 | 369.6 | 369.7 | 2.311 | 2.294-2.310 | 22.20 | 0.01545 | 0.01537-0.01544 |
| CE200 | 1463.6 | 1475.8-1477.4 | -0.88% | 1239.4 | 1248.8-1250.2 | 152.5 | 71.6 | 5483.3 | 1221.0 | 1221.1 | 3.642 | 3.620-3.621 | 26.70 | 0.01135 | 0.01109-0.01113 |

### CE300-CE545 annual results vs the informative example programs (Annex B16.5.2)

| Case | Total kWh | Examples min-max | Compressor kWh | Cond fan kWh | Indoor fan kWh | Coil total kWh | Coil sens kWh | Coil lat kWh | COP2 | Mean IDB C | Mean W | Mean RH % |
|---|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| CE300 | 35039 | 34746-35634 | 21843 | 2316 | 10880 | 78548 | 55181 | 23367 | 3.251 | 24.09 | 0.00914 | 47.8 |
| CE310 | 39367 | 39290-39973 | 25812 | 2675 | 10880 | 97551 | 55068 | 42483 | 3.424 | 24.10 | 0.01113 | 57.8 |
| CE320 | 39421 | 38745-40060 | 25880 | 2661 | 10880 | 97981 | 61834 | 36147 | 3.433 | 24.23 | 0.01002 | 51.4 |
| CE330 | 40529 | 39438-40963 | 26932 | 2717 | 10880 | 104594 | 63157 | 41437 | 3.528 | 24.29 | 0.00986 | 50.5 |
| CE340 | 40125 | 39265-40619 | 26541 | 2704 | 10880 | 101808 | 62485 | 39323 | 3.481 | 24.31 | 0.00988 | 50.6 |
| CE350 | 31401 | 30547-32237 | 18582 | 1939 | 10880 | 66833 | 48353 | 18480 | 3.257 | 26.24 | 0.00977 | 44.5 |
| CE360 | 55042 | 54016-55299 | 39893 | 4269 | 10880 | 162835 | 134867 | 27968 | 3.687 | 25.39 | 0.00854 | 41.2 |
| CE400 | 31320 | 30846-32045 | 18548 | 1893 | 10880 | 66973 | 40604 | 26369 | 3.276 | 24.09 | 0.01004 | 52.2 |
| CE410 | 31320 | 31503-32078 | 18548 | 1893 | 10880 | 66973 | 40604 | 26369 | 3.276 | 24.09 | 0.01004 | 52.2 |
| CE420 | 32988 | 32530-33387 | 20020 | 2088 | 10880 | 71443 | 49452 | 21991 | 3.232 | 24.09 | 0.00932 | 48.7 |
| CE430 | 32111 | 31772-32538 | 19234 | 1998 | 10880 | 68535 | 46955 | 21579 | 3.228 | 24.09 | 0.00936 | 48.9 |
| CE440 | 33293 | 32973-33691 | 20291 | 2123 | 10880 | 72561 | 50006 | 22555 | 3.237 | 24.09 | 0.00921 | 48.1 |
| CE500 | 23022 | 22323-23138 | 18435 | 1964 | 2623 | 65581 | 47351 | 18230 | 3.215 | 20.69 | 0.00916 | 57.1 |
| CE510 | 43036 | - | 34419 | 3689 | 4927 | 136581 | 98459 | 38121 | 3.584 | 20.69 | 0.00917 | 57.2 |
| CE520 | 26172 | 24987-25781 | 20501 | 2428 | 3243 | 66281 | 48069 | 18213 | 2.891 | 13.88 | 0.00604 | 60.2 |
| CE522 | 24502 | 23544-24360 | 19418 | 2177 | 2907 | 65925 | 47695 | 18230 | 3.053 | 17.31 | 0.00751 | 58.9 |
| CE525 | 20627 | 20242-21323 | 16823 | 1629 | 2175 | 65010 | 46779 | 18230 | 3.523 | 27.42 | 0.01332 | 52.9 |
| CE530 | 18615 | 17281-17875 | 14688 | 1681 | 2245 | 47113 | 47113 | 0 | 2.878 | 20.69 | 0.00287 | 21.1 |
| CE540 | 20802 | 19061-20164 | 16098 | 2014 | 2690 | 47652 | 47652 | -0 | 2.631 | 13.87 | 0.00287 | 29.8 |
| CE545 | 16987 | 15687-16636 | 13597 | 1451 | 1938 | 46711 | 46711 | 0 | 3.104 | 27.42 | 0.00287 | 16.6 |
| CE500 May-Sep | 17991 | 17391-18051 | 14450 | 1516 | 2025 | 50349 | 36359 | 13989 | 3.153 | 25.00 | 0.01094 | 55.3 |
| CE510 May-Sep | 35719 | 34609-35971 | 28628 | 3036 | 4055 | 112784 | 81305 | 31479 | 3.562 | 25.00 | 0.01096 | 55.4 |

CE300 weather check (CE300.TM2 via Pinocchio's converter): mean ODB 19.91 C (Table A1-7: 19.9), max ODB 34.8 C at 20-Jul 15 (35.0), mean outdoor W 0.01161 (0.0116).

CE100–CE200 quantities not shown (zone loads, max/min COP, IDB, W) are in `Std140_CE_a_Output.xls` / `scoring.json`.
The example results are informative only (no ranges exist for CE100–CE200 or CE500–CE545).

## Observations and deviations

1. **CE410 = CE400 exactly.** `LockoutWithCompressor` on the load-controlled AirLoopHVAC:UnitarySystem has no effect in
   EnergyPlus 26.1 here (identical results), so the non-integrated economizer of CE410 behaves as integrated. CE410−CE300 is
   still within its range (−3719 in [−5929, −371]) but ~180 kWh below the example programs' CE410. Setting
   *Economizer Operation Staging = EconomizerFirst* (meant for multi-speed units) **crashes** EnergyPlus 26.1 (segfault) with
   this single-speed system — engine issue, recorded, not patched.
2. **CE520, CE522, CE530, CE540, CE545 run 1.5–4 % above the example maximum.** They exercise the dry coil and EDB 15 °C, where
   the program's ADP/BF dry-coil adjustment and the (EWB, ODB)-only curves differ from map interpolation. No ranges apply.
3. **Bypass factor.** Derived by the program from the rated SHR: CE100 ≈ 0.077 (spec 0.049–0.080), CE300 ≈ 0.144 (manufacturer
   0.070). The map's AHRI-point SHR was kept as the primary data (no tuning).
4. **Sizing pass.** The Studio export always adds a design-day sizing pass; all components are hard-sized, so results are
   unaffected. Design days are a project `.ddy` (CE300: New Orleans 722310 0.4 % / 99.6 %; CE100 series: the constant file
   condition). The station `.ddy`'s `SizingPeriod:WeatherFileConditionType` objects were left out (they name TMY3 weeks and are
   fatal with CE300.TM2).
5. Zone IDB above the set point occurs in CE360 (undersized by design) and in a few humid economizer hours of CE400–CE440.

## App gaps found and what was done

| Gap | Status |
|---|---|
| No way to enter a DX performance map (curve coefficients had to be fitted outside the app) | **Fixed**: DX coil inspector *Performance map* tab + Studio op `fit_dx_coil_map` (`src/lib/std140/dxMap.js`, `dxMapOp.js`, `src/osapp/inspector/Inspector.jsx`, `src/osapp/kernelModule.js`). The op log stores the map; replay re-fits. Tests `tools/std140/ce/dxMap.test.mjs` 6/6. |
| Economizer on a load-controlled unitary with SZ:Reheat: in the deadband it targets ODB → 100 % OA all winter (CE400 mean IDB 20.8 °C) | Use the kernel's existing `OS:SetpointManager:SingleZone:Cooling` (CE400 mean IDB 24.1 °C, economizer off when IDB ≤ 25 °C). No code change. |
| Studio export without a `.ddy` adds `WeatherFileConditionType` extremes → FATAL on EPWs without TYPICAL/EXTREME PERIODS (every converted TMY2/TMY/measured file) | `idfProject.js` is not WP-E1's; logged in `work/COORDINATION.md`. Worked around with a `.ddy`. |
| A `.ddy` with `SizingPeriod:WeatherFileConditionType` is injected verbatim → FATAL with another station's EPW | Logged; harness keeps only `SizingPeriod:DesignDay`. |
| Annual schedule editor is month-granular (Std 140 schedules change on Mar 11, Apr 11/21, Oct 13/19, Nov 6) | Logged (`scheduleAnnual.js` not WP-E1's); schedules created with the Schedules step's create helper + dated Schedule:Compact data. |
| Non-integrated economizer ineffective with UnitarySystem | Engine behaviour; documented in S140outNotes block G. |

## Files

- `inputs/<case>/in.idf` (the exact Pinocchio export) + `project.json` (persisted Pinocchio project incl. the Studio op log); `inputs/coil_fits.json`
- `outputs/<case>/` eplusout.csv / .err / eplustbl.htm / .eio
- `weather/` the five converted EPWs + `conversion_provenance.json`, the New Orleans `.ddy`
- `Std140_CE_a_Output.xls`, `Std140_CE_b_Output.xls` (official templates, filled), `S140outNotes_9.TXT`, `scoring.json`
- `harness/` scripts, `tables.md`, `example_results_extract.json`; `run.log`, `run_status.txt`
