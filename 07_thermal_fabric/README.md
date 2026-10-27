# Section 7: building thermal envelope and fabric (Class I), through Pinocchio-EP's GUI path

**Result: both Annex A3 test groups PASS.** These results come from GUI-reachable inputs on the normative weather (725650TY.CSV), scored against the criteria of Addendum a.

| Test group | In range | Minimum to pass | Verdict |
|---|---|---|---|
| Thermal fabric, low mass | **34 / 34** (21 single values + 13 day curves) | 30 | PASS |
| Thermal fabric, high mass | **28 / 28** (19 single values + 9 day curves) | 25 | PASS |

The 2026-10 paper's GUI-reachable route scored 33/34 and **21/28**, so high mass failed. Every range case that was out of range then is now in range. The detailed tables are in `tables.md`.

- Tested build: `Pinocchio-EP 1.0.0+b0c9cf1.dirty`. This is the `ashrae140-compliance` working tree, which is not yet released (see S140outNotes Block F).
- Engine: EnergyPlus 26.1.0-6f2e40d102, unmodified.
- Run date: 2026-10-27.

## What was run

**Cases.** All 52 Section 7 cases were run:
- 600–695, 900–995 and 960;
- the free-float cases 600FF, 650FF, 680FF, 900FF, 950FF and 980FF;
- the in-depth cases 195, 200, 210, 215, 220, 230, 240, 250, 270, 280, 290, 300, 310, 320, 395, 400, 410, 420, 430, 440, 450, 460, 470, 800 and 810.

Each case ran for a full year at 4 time steps per hour, the program default and the same for every case. All 52 runs completed with **0 severe errors**; each had 1–4 warnings (see Block G and `runs.json`).

The **Pinocchio-EP default variant** (a normal user, no Standard 140 profile) was also run for Cases 600 and 900.

**Weather.** The normative file `Std140_TF_Files/Normative Materials/725650TY.csv` from the official 140-2023 accompanying files was used.
- It was converted to EPW by Pinocchio-EP's own converter (`src/lib/weather`, pinocchio-wx 1.0.0). This is the same code the server runs when a user uploads a TMY3 CSV.
- The converter applies no time shift, because TMY3 and EPW are both LST hour-ending.
- The result is `weather/725650TY.epw` (sha256 `e7404680efbd0aab…`), byte-identical to WP-C's `06_weather_drivers/converter/epw/725650TY_from_tmy3.epw`. Provenance is in `weather/provenance.json`.
- **No substitute weather was used.** The paper used the climate.onebuilding.org Denver TMY3 EPW as a substitute.

**Build path: GUI data only.** `pinocchio-ep/tools/std140/tf/cases.mjs` builds each case as a Pinocchio-EP project. It uses the same data structures and pure helpers the GUI controls call, and nothing is pushed into `project.model` by hand. The export is `exportIdf`, which produces the exact text the GUI Run posts to `/api/run`.

| Std 140 input | How it was entered (GUI) |
|---|---|
| Geometry: 8 × 6 × 2.7 m zone, windows, raised floor | Geometry: zone footprint, windows (sill 0.2 m, or 0.5 m for the 960 sun zone), floor face boundary Outdoors with wind exposure off (zone editor / Surfaces sheet) |
| Materials (Tables 7-2, 7-25, 7-27, 7-29, 7-31) | Envelope › + New construction › **Exact material** layers (k, ρ, cp, thickness, roughness, solar/thermal/visible absorptance; WP-B). Floor insulation is a **No-mass** layer, R = 25.075 / 25.175 m²K/W |
| Windows (Tables 7-10, 7-17, 7-21) | Envelope › Window › **Layered: pane by pane** › Std 140 presets 600 / 660 / 670 (WP-B). No SimpleGlazing |
| 610 / 630 / 290 / 310 shading | Inspector › window › Overhang + **↔ whole wall · roof** / Fins + **↕ floor → roof** (`ovhL/ovhR/finTop/finBot`, WP-B) |
| 450 / 460 / 470 constant coefficients | Envelope › Surface heat transfer › **Fill Std 140 Table 7-46** (convective) + emittance 0.00001 on the affected faces (layer ⚙ / pane ⚙) |
| In-depth radiative properties (0.1 / 0.9 absorptance, 0.1 emittance) | construction layer ⚙ (outer / inner face absorptances) |
| High-conductance wall element (220 series, 400–430, 800) | Envelope › + New construction **"Std140 HC Element"** (exact layers: panel / 0.0625 W/mK constant-k gap / panel, VerySmooth), picked for two **opaque doors** 3 × 2 m in the door construction select (Opaque (custom)), with door **sill 0.2 m** (`door.glass` / `door.y`, WP-G) |
| Infiltration 0.5 / 1.0 ach, gains 200 W (60 % radiative) | Internal loads: Infiltration (ACH) and Interior equipment (design level, radiant fraction 0.6, latent 0) |
| 650 / 950 night vent 1700 m³/h | Internal loads › a second Infiltration load (design flow 0.4722 m³/s) + an hourly fan schedule (Schedules › Add a schedule) |
| Thermostats 20/27, 20/20, 650 vent | HVAC › Quick ideal-loads system › zone thermostat (fixed values or setpoint schedules from Schedules › Add a schedule) |
| 640 / 940 15-min setback ramp | + Add object `Schedule:Compact` with `Interpolate: Linear` (the bar editor is hourly) |
| DST/holidays off, Country terrain, FullInteriorAndExterior, ideal loads sensible-only, no auto loads, hourly outputs | Project › Analysis options › **Simulation profile = Standard 140 / validation** (`project.profile = "std140"`, WP-A) |
| Solar, sky and transmitted-solar outputs | Project › Analysis options › **Output variables** (`project.outputVariables`, WP-A) |
| Case 960 sun zone | 2 zones; common wall = face boundary **Surface** + a custom construction; sun zone `conditioned: false` |

The only object created with **+ Add object** (GUI per the WP-D brief) is the 640/940 15-minute ramp schedule, reported in S140outNotes Block C. Before WP-G closed the door gaps, the high-conductance element's construction was also entered that way (as `Ext Door`), with the element 0.17 m too low. Re-running with the native door construction and sill changed only the 12 element cases, by at most 0.01 MWh (≤ 0.3 %; e.g. 220 heating 6.447 → 6.450, 250 cooling 3.203 → 3.193). None of those cases is ranged, and all range cases are unchanged.

## Results

Full tables are in `tables.md`: every range case with lower/upper limits and the paper's P and G values next to the new value; the 22 day-curves; all 46 conditioned cases with peaks; free float; solar and sky. Highlights (annual MWh):

| | 600 heat | 600 cool | 900 heat | 900 cool |
|---|---|---|---|---|
| Range | 3.75–4.98 | 5.00–6.83 | 1.04–2.28 | 2.35–2.60 |
| Paper, faithful passthrough (substitute weather) | 4.379 | 5.891 | 1.706 | 2.377 |
| Paper, GUI-reachable (SimpleGlazing etc., substitute weather) | 3.948 | 6.369 | 1.333 | 2.811 † |
| **Now, GUI-reachable, normative weather** | **4.301** | **6.015** | **1.661** | **2.506** |
| Now, Pinocchio-EP **defaults** (no profile) | 4.269 | 6.696 | 1.482 | **2.936 †** |

† = out of range.

The paper's seven GUI failures were 900 cooling, 950−900, 960−900 (heating and cooling), 980−900 cooling, 940 Feb 1 (r = 0.869) and 980 Feb 1 (RMSE). All seven are now in range, as is 695 Feb 1 (paper G RMSE 0.332, now 0.174). The fixes came from the WP-A and WP-B GUI features: layered glazing, exact materials, and shading extents.

**Default variant (informative).** Without the Standard 140 profile, a normal user gets:
- City terrain;
- FullExterior solar distribution;
- weather-file DST and holidays;
- ideal loads with SHR 0.7.

Case 900 cooling then falls out of range, and both 600 and 900 Jul 14 fail on RMSE (`scoring.json › defaultVariant`). The profile is therefore needed. Terrain remains the dominant default, as the paper found.

**Informative comparisons (no acceptance ranges).**
- **Example-results spread.** Annual heating and cooling of every conditioned case lies inside the Annex B8 example-results spread (`Std140_TF_Results.xlsx`, Tables B8-1/B8-2) except two results on the boundary: in-depth 290 cooling is 5.743 against a maximum of 5.743 (rounding), and 695 cooling is 9.175 against a maximum of 9.172. The 25 in-depth cases have no acceptance ranges.
- **Free float.** 600FF ranges from −12.4 to 63.1 °C (mean 25.0); 900FF from 1.4 to 43.9 °C (mean 25.2).
- **Solar.** Annual incident on the horizontal is 1664.6 kWh/m², equal to the file's GHI of 1670 kWh/m². South is 1370.7 kWh/m². Transmitted through the south windows is 804.2 kWh/m² for 600, 436.1 for 660 and 1024.4 for 670.
- **Sky temperature.** Mean −1.98 °C; range −38.0 to 24.6 °C.

## Deliverables in this folder

| File | Content |
|---|---|
| `Std140_TF_Output.xlsx` | The **official** template, copied and filled by `std140Report.fillWorkbook` + `std140ReportTF.tfCellMap`. Only the data-entry ranges of its instructions (rows 40–52) are written, and styles and formulas are kept. It contains annual loads and peaks for all 46 conditioned cases, free-float rows, solar, sky, Case 600/900 monthly values, the three hourly tables, and the 900FF 8760 h temperatures in `TMPBIN!E11:E8770`. Cached bin counts are written into the template's formula cells, and the workbook recalculates on open |
| `S140outNotes_7.TXT` | Blocks A–G, generated by `std140Report.notesText(tfNotesSpec(...))`. Version stamp `__APP_VERSION__`; Block B lists `PROFILE_140_SETTINGS` |
| `scoring.json` | Every range case (value, limits, pass), Pearson/RMSE per day curve, group counts, the default variant, and run status |
| `results.json` | Annual, peaks, monthly, free float, solar, sky, hourly day tables, and the 900FF 8760 h series |
| `tables.md` | README tables, including the comparison with the paper and the example results |
| `inputs/<case>/in.idf`, `project.json` | The exact IDF Pinocchio-EP exported, and the project data (the model's object list is summarised). `default_600` and `default_900` are the default variant |
| `outputs/<case>/` | `eplusout.csv`, `eplusout.err`, `eplustbl.htm` |
| `runs.json` | Per-run status: completed, severe count, warning count |
| `weather/` | `725650TY.epw` and `provenance.json` |
| `harness/` | Copies of `pinocchio-ep/tools/std140/tf/*.mjs`, plus the commands below |

## How to reproduce

```
cd pinocchio/pinocchio-ep
node tools/std140/tf/run.mjs all                     # 52 cases, std140 profile, ≤ 3 EnergyPlus at once (~1 min)
node tools/std140/tf/run.mjs all --variant default   # Cases 600 / 900 with Pinocchio-EP defaults
node tools/std140/tf/report.mjs                      # results / scoring / workbook / notes / tables
node --test tools/ashrae140-wpd.test.mjs             # unit tests + the 600/900 regression gate (skips without the engine)
```

## Deviations and GUI gaps (honest list)

1. **High-conductance wall element.** It is now fully native: a custom opaque door construction at the specified 0.2 m sill (WP-G closed both gaps logged earlier). It is still an opaque door subsurface, which is exactly what the element is.
2. **Night ventilation.** It is entered as a scheduled infiltration load, which is the same heat-balance term as a zero-heat vent fan. Block C justifies this.
3. **Combined coefficients (450–470).**
   - The coefficients are entered as convective, with emittance 1e-5. The forms require a value above 0, and the standard's informative note allows this approach.
   - FisherPedersen keys are not involved.
4. **Visible-band glass properties.** They are set equal to the solar-band values. The standard gives none, and they do not affect the loads.
5. **Unreleased build.** The tested build carries unreleased changes (Block F).
6. **Engine warnings.** There are a few psychrometric warnings (wet-bulb not converged) at isolated time steps (Block G).
7. **No tuning.** No input was tuned. The only changes from the paper's tier G are the new GUI features and the normative weather.
