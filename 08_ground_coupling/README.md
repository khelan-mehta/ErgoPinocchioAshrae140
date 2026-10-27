# ASHRAE 140-2023 Section 8: ground-coupled slab-on-grade (GC series) in Pinocchio-EP

**Result in brief.**
- 16 of the 17 Section 8 cases were run through Pinocchio-EP's own emitter (`exportIdf`), using the **new Kiva ground-coupling setting** (`project.ground`, `src/lib/ground/kiva.js`), on EnergyPlus 26.1.0.
- All 16 runs finished with **0 severe errors**. All converged to well under the standard's 0.1 %/year criterion.
- **15 of 16** floor-conduction results fall inside the range of the informative example results (Annex B8.2, `Std140_GC_Results.xlsx`).
- The 3-D verified numerical models (TRNSYS / FLUENT / MATLAB) are the "secondary mathematical truth standard". Against their mean, results differ by **−7.4 % … +1.2 %**. For 11 of the 16 cases the difference is within ±3.4 %.
- **GC10a is omitted.** Its boundary condition cannot be expressed in Kiva (see E below).
- Section 8 has **no Annex A3 acceptance criteria**. Every comparison here is informative.

## 1. What was built in the app (so that the GC cases are possible at all)

| Item | Before | Now |
|---|---|---|
| Ground model for on-grade floors | Monthly `Site:GroundTemperature:BuildingSurface` only | `project.ground = {method:"kiva", …}` → `Foundation:Kiva:Settings` (soil k/ρ/cp, ground solar and IR absorptivity, roughness, far-field width, deep-ground boundary ZeroFlux/GroundWater/Autoselect, deep-ground depth, mesh, Kiva timestep) + one `Foundation:Kiva` per foundation (zones, initial indoor T, wall height above grade, wall depth below slab, wall material or construction, interior/exterior horizontal/vertical insulation) + `SurfaceProperty:ExposedFoundationPerimeter` (computed from the plan: edges not shared with another on-grade floor, or a user fraction) |
| Survives geometry edits | An imported `Foundation` boundary was lost when the zone was re-derived (audit F / H-7) | The setting lives on the project and is re-applied to whatever floors the export derives. Tested by moving and scaling zones. `groundFromImport()` captures an imported Foundation:Kiva coupling into the setting. |
| Massless slab layers | n/a. Kiva rejects them, and the starter "Slab Floor" has a NoMass insulation layer. | The Kiva floor gets a copied "(Kiva)" construction in which each NoMass/AirGap layer is replaced by a 5 mm Material of the same R and negligible capacitance |
| GUI / agent | none | `lib/ground/GroundPanel.jsx` (every control aiKey'd) + 25 registry FieldDefs `ground.*` / `foundation.*` in `src/agent/fields/ground.js`. Wiring into fieldRegistry / exportIdf / EnvelopeEditor / persistence / import was requested from WP-A (`work/COORDINATION.md`). Until it lands, the harness applies the same library function to the exported text (`groundCouplingIdf`); a test shows the model-level and text-level paths give the same result. |

Tests:
- `tools/ground-kiva.test.mjs` 9/9.
- `tools/ground-panel.test.mjs` 2/2 (GUI result equals registry result).
- Real-engine smoke on an ordinary project (starter constructions, 2 on-grade zones + 1 upper zone, Denver): `harness/kiva_smoke.mjs`, 0 severe errors.

## 2. Method

**Harness.** `harness/suite.mjs` builds each case through Pinocchio:
- `newProject` → one zone B×L×2.7 m;
- walls and roof set to **Adiabatic** through GUI surface overrides;
- `p.loads = []`;
- GUI ideal-loads thermostat: heating 30 °C, cooling 60 °C (off);
- `profile = "std140"` (WP-A);
- `project.outputVariables` (WP-A's registry field) for the hourly outputs;
- `setGround(...)` (this WP) for the ground.

`exportIdf` → (`groundCouplingIdf`) → `in.idf` → `energyplus -x -r`. `harness/report.py` computes the 8.3 outputs, fills the official workbook and writes `scoring.json`.

**Weather.** `weather/GC30a|30b|65b|40a|40b|70b.epw`, written by `gc_weather.py`:
- Table A1-3 constants: dew point −56.6 °C, RH 0, 1013 mbar, no solar, sky cover 10/10, visibility, ceiling, AOD, snow, the per-file wind speed of 40 / 19.9 / 1.0 m/s, and the site 25.8 N 80.3 W, TZ −5, 2 m.
- Hourly ODB taken from the **normative GC\*.TM2** files. They are now available locally. The script asserts that every constant column of each TM2 matches Table A1-3.
- The TM2 ODB equals the A1.1.2.1 formula to within ±0.05 °C (its 0.1 °C resolution). Without the TM2s the script writes the formula itself.
- So the EPWs carry the normative data. They are equivalent to the TM2s, not substitutes.

**Model mapping** (Section 8.2 → Kiva):

| Spec | Input used |
|---|---|
| Soil & slab k / ρ / cp (1.9 or 0.5 / 0.85 ; 1490 ; 1800) | `Foundation:Kiva:Settings` soil properties. The slab is a 1 mm layer of the same soil material (8.2.1.1.2 g: least stable thickness). |
| Deep-ground T 10 °C at E (15 / 2 / 30 / 5 m) | Deep-Ground Boundary = **GroundWater** (constant at the weather file's annual-mean ODB = 10.0 °C) at Deep-Ground Depth E |
| Far-field adiabatic at F (15 / 20 / 8 m) | Far-Field Width F (Kiva: zero-flux far-field boundary) |
| Adiabatic 0.24 m wall that touches but does not penetrate the ground | Footing-wall construction 0.24 m, k 0.001, ρ 0.001, cp 100 (EnergyPlus minimum). Wall height above grade 0, depth below slab 0, so the wall is only as deep as the 1 mm slab. |
| No radiation exchange | Ground solar and IR absorptivity 0. Slab and adiabatic surfaces have emittance and absorptance 1e-5 (EnergyPlus minimum > 0). |
| hint = 100 / 7.95 W/m²K | `SurfaceProperty:ConvectionCoefficients` Inside Value on the floor. Kiva **uses** it: the reported inside h is 100.0 / 7.95 every hour. |
| a-series: direct Ti, To | hint = 5000 (8.2.2.2.1 a; HeatBalanceAlgorithm maximum raised to 5000). To is approached through the 40 m/s weather wind. |
| hext = 100 / 11.95 / ≥5000 | **Kiva computes the exterior ground convection from the weather-file wind speed.** The standard chose the wind speeds (19.9 / 1.0 / 40 m/s) so that this works (A1.1.2.3), and EnergyPlus 26.1 has no user hext for Kiva ground. |
| Ideal heating to 30 °C, convective, 100 % | GUI ideal loads (HVACTemplate IdealLoads, Std 140 profile: no dehumidification / outdoor air) |
| Multiyear to 0.1 % | `RunPeriod` 2001 → 2003 (steady), 2001 → 2008 (harmonic), 2001 → 2014 (GC80b/c). Kiva also initialises itself with a steady-state solve. |

**Numerical settings** (5.1.1.9; one set for the whole suite): Kiva minimum cell 0.5 mm, growth 1.2, hourly Kiva step, zone timestep 4/h.

**GC30b mesh / slab-thickness study** (2-year runs; last-hour qfloor):

| slab t | min cell | growth | qfloor (W) |
|---|---|---|---|
| 0.02 m | 0.02 m (E+ default) | 1.5 (default) | 2421.3 |
| 0.02 | 0.01 | 1.5 | 2427.6 |
| 0.02 | 0.02 | 1.2 | 2409.4 |
| 0.01 | 0.005 | 1.2 | 2446.9 |
| 0.005 | 0.0025 | 1.2 | 2464.4 |
| 0.002 | 0.001 | 1.2 | 2476.1 |
| **0.001** | **0.0005** | **1.2** | **2480.1 (used)** |
| 0.001 | 0.00025 | 1.1 | 2471.7 |

- **Slab thickness** matters by +2.4 % between 20 mm and 1 mm. The reason is that the depth of the adiabatic wall strip equals the slab thickness.
- **Mesh refinement** at 1 mm changes the result by −0.34 %. The 0.1 % mesh-independence recommended in 8.2.1.1.7.1 note 1 was **not** reached; the Kiva mesh uncertainty is about ±0.3 %.
- Neither choice was tuned to the reference results. The least stable thickness is what 8.2.1.1.2 g requires.

## 3. Results (informative comparison, Annex B8.2)

Floor conduction:
- Steady cases: last hour, W.
- Harmonic cases: last-year total, kWh/y.

Zone load = floor conduction in every case: |Qzone − Qfloor| < 0.01 kWh, because the above-grade surfaces are truly adiabatic. Tzone = 30.000 °C.

The "All example" range includes the verified models plus GHT, SUNREL-GC, EnergyPlus 2.0 (the 2008 slab preprocessor), VA114/ISO-13370, BASESIMP and BASECALC.

| Case | Pinocchio-EP / Kiva | Verified numerical models (TRNSYS / FLUENT / MATLAB) | VM mean | Δ vs VM mean | All example results (min – max) | In example range | Year-to-year change |
|---|---|---|---|---|---|---|---|
| GC30a | 2,541.6 W | 2,642 / 2,585 / 2,695 | 2,641 | −3.7 % | 2,457 – 2,695 | yes | 0.002 % |
| GC30b | 2,480.1 W | 2,533 / 2,504 / 2,570 | 2,536 | −2.2 % | 2,341 – 2,652 | yes | 0.001 % |
| GC30c | 2,154.8 W | 2,137 / 2,123 / 2,154 | 2,138 | +0.8 % | 1,973 – 2,308 | yes | 0.001 % |
| GC60b | 2,136.4 W | 2,113 / 2,104 / 2,128 | 2,115 | +1.0 % | 1,999 – 2,219 | yes | 0.000 % |
| GC65b | 1,848.5 W | 1,994 / 1,991 / 2,004 | 1,996 | −7.4 % | 1,616 – 2,004 | yes | 0.001 % |
| GC40a | 22,199 kWh | 23,033 / 22,761 / 23,609 | 23,134 | −4.0 % | 20,812 – 23,609 | yes | 0.080 % |
| GC40b | 21,723 kWh | 22,099 / 21,932 / 22,513 | 22,181 | −2.1 % | 20,513 – 23,204 | yes | 0.003 % |
| GC45b | 31,790 kWh | 32,758 / 32,456 / 33,483 | 32,899 | −3.4 % | 30,856 – 33,483 | yes | 0.002 % |
| GC50b | 274,401 kWh | 277,923 / 277,988 / 281,418 | 279,110 | −1.7 % | 277,923 – 324,257 | **no** (−1.3 % below min) | 0.004 % |
| GC55b | 34,447 kWh | 35,075 / 34,879 / 35,491 | 35,148 | −2.0 % | 31,601 – 39,932 | yes | 0.000 % |
| GC70b | 16,282 kWh | 17,396 / 17,434 / 17,552 | 17,461 | −6.7 % | 15,553 – 17,552 | yes | 0.006 % |
| GC80b | 6,061 kWh | 6,029 / 5,939 / 6,151 | 6,040 | +0.3 % | 5,661 – 6,151 | yes | 0.029 % |
| GC40c | 18,875 kWh | 18,649 / 18,598 / 18,873 | 18,707 | +0.9 % | 17,285 – 20,255 | yes | 0.003 % |
| GC45c | 26,842 kWh | 27,004 / 26,906 / 27,392 | 27,101 | −1.0 % | 23,849 – 28,707 | yes | 0.002 % |
| GC55c | 20,997 kWh | 20,760 / 20,714 / 20,986 | 20,820 | +0.9 % | 20,172 – 22,570 | yes | 0.000 % |
| GC80c | 9,321 kWh | 9,192 / 9,137 / 9,314 | 9,215 | +1.2 % | 8,635 – 10,073 | yes | 0.004 % |

GC10a: omitted. The analytical solution (CSIRO, Annex B11.2.1) is **2432.6 W**. The verified models give 2425 – 2432 W.

Peaks and phase (harmonic, last year). Phase = hours from the coldest hour (Jan 15, hour 4) to the floor-conduction peak.

| Case | qfloor,max (W) | at | VM peak range (W) | Δ vs VM mean | Phase (h) | VM phase range (h) |
|---|---|---|---|---|---|---|
| GC40a | 2,953.8 | 02/04 h14 | 3,042 – 3,174 | −4.7 % | 490 | 416 |
| GC40b | 2,871.3 | 02/04 h15 | 2,914 – 3,002 | −2.8 % | 491 | 417 – 465 |
| GC45b | 4,287.7 | 02/04 h15 | 4,396 – 4,551 | −3.9 % | 491 | 417 – 441 |
| GC50b | 33,905 | 02/04 h15 | 34,510 – 35,033 | −2.3 % | 491 | 465 – 489 |
| GC55b | 4,263.2 | 01/22 h15 | 4,336 – 4,427 | −2.6 % | 179 | 153 – 177 |
| GC70b | 2,073.1 | 02/17 h17 | 2,254 – 2,276 | −8.4 % | 805 | 659 – 660 |
| GC80b | 776.0 | 02/09 h20 | 763 – 794 | −0.2 % | 616 | 567 – 591 |
| GC40c | 2,474.4 | 02/09 h16 | 2,444 – 2,487 | +0.5 % | 612 | 538 – 562 |
| GC45c | 3,600.8 | 02/07 h16 | 3,618 – 3,691 | −1.3 % | 564 | 490 – 538 |
| GC55c | 2,734.4 | 02/09 h16 | 2,703 – 2,744 | +0.6 % | 612 | 490 – 538 |
| GC80c | 1,200.5 | 02/11 h18 | 1,181 – 1,207 | +0.7 % | 662 | 589 – 613 |

Sensitivities (Pinocchio-EP/Kiva vs the verified-model range, Annex B8.2-9/-10):

| Delta | Ours | VM range |
|---|---|---|
| GC45b − GC40b (kWh) | 10,067 | 10,524 – 10,971 |
| GC55b − GC40b | 12,725 | 12,947 – 12,978 |
| GC40b − GC70b | 5,440 | 4,498 – 4,961 |
| GC40b − GC80b | 15,662 | 15,993 – 16,362 |
| GC45c − GC40c | 7,968 | 8,309 – 8,519 |
| GC55c − GC40c | 2,123 | 2,111 – 2,117 |
| GC40c − GC80c | 9,554 | 9,457 – 9,559 |
| GC40a − GC40b | 476 | 829 – 1,096 |

**Reading of the results**
- **c-series** (hint 7.95 combined, surface To): within +0.8 … +1.2 % of the 3-D models, peaks within ±1.3 %.
- **b-series** with hext 100 (wind 19.9 m/s): −1.7 … −3.4 %, plus GC80b +0.3 %.
- **a-series** (h = 5000 inside, 40 m/s wind outside): −3.7 / −4.0 %.

**Likely causes** (attributed, not proven):
1. **Kiva in EnergyPlus is 2-D.** It approximates the 3-D slab by its area/perimeter ratio. The aspect-ratio case GC45b (−3.4 %) and the a-series show the largest geometric gaps.
2. **Kiva's exterior ground convection is computed from the wind**, not input. At 1 m/s (GC65b −7.4 %, GC70b −6.7 %, the largest gaps) the computed hext is evidently lower than the specified 11.95 W/m²K. At 40 m/s it is high but finite, so the "direct To" of the a/c series is only approached.
3. **The hourly Kiva step.** Peak-conduction timing is 25 – 75 h later than the 3-D models, and 145 h later for GC70b. The last-year peaks are 0.2 – 8 % lower.

**Weather minimum.** The weather file's minimum ODB is 2.0 °C (Jan 8, hour 4; 15 hours at the minimum, the same as the verified models). EnergyPlus reports 2.038 °C because its hourly output is the mean of the timestep-interpolated values; the workbook carries the engine's value.

## 4. Files

- **`Std140_GC_Output_PinocchioEP.xls`**: the official `Std140_GC_Output.xls`, filled:
  - steady rows (qfloor, qzone, Tzone, tsim);
  - harmonic rows, including peaks with dates/hours and TODB,min;
  - hour convention "Hour 1 = 0:00 to 1:00";
  - 8760-h floor conduction for GC40a/b/c.
  - Left blank: the near-surface temperatures (not available, see S140outNotes_8 E) and GC10a.
- **`scoring.json`**: every output, the convergence measures, the example results per case, the deviations, the in-range flags and the deltas.
- **`results_table.md`**: generated tables.
- **`S140outNotes_8.TXT`**: Annex A2 blocks A – G.
- **Run files**:
  - `inputs/<case>/in.idf`: the exact IDF Pinocchio produced.
  - `outputs/<case>/`: eplusout.csv (hourly, all years), eplusout.err, eplustbl.htm, run.json.
  - `weather/`: the 6 EPWs + `gc_weather.py`. ASHRAE-derived; keep local.
  - `harness/`: suite.mjs, report.py, gc_weather.py, kiva_smoke.mjs. The source lives in `pinocchio-ep/tools/std140/gc/`.
- **Re-run**:
  - `node tools/std140/gc/suite.mjs` (env `GC_JOBS` ≤ 3);
  - then `python tools/std140/gc/report.py <this folder> "<…>/Accompanying FIles/Std140_GC_Files"`.

## 5. Deviations / limits (honest list)

- **GC10a not run.** Kiva cannot impose a linearly varying perimeter surface temperature on a semi-infinite solid.
- **No near-surface ground temperatures** (Table 8-12 / Figure 8-6). EnergyPlus' Kiva has no output for ground-cell temperatures at user coordinates.
- **hext is not a user input** for the Kiva ground surface. It is driven by the weather wind, as A1.1.2.3 intends.
- **Kiva is 2-D.** There is no 3-D option in EnergyPlus 26.1.
- **Passthrough objects** were used where no GUI/registry field exists:
  - `SurfaceProperty:ConvectionCoefficients` (floor inside h);
  - the HeatBalanceAlgorithm maximum h (5000, a-series);
  - Materials / Constructions with emittance 1e-5 and the soil-property slab (the GUI construction builder has no IR emittance);
  - RunPeriod begin/end year (multiyear).
  - Every passthrough is listed in `suite.mjs` `PASSTHROUGH`.
- **Kiva module not yet wired by WP-A.** Until then the harness applies `groundCouplingIdf` to the exported text, which gives the identical result.
- **Basements are not modelled.** The Kiva module couples floors only (no below-grade walls / foundation depth from wall surfaces). Section 12's basement cases therefore used the standard's alternative method (see 12_class2_hers).
