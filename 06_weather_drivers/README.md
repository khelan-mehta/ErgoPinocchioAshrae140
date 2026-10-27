# Section 6: Weather drivers (WD100–WD600) run through Pinocchio-EP

**Status:** all six cases ran, all Section 6.3.1 outputs were produced, and the filled output workbook is here. Annex A3 has **no acceptance ranges** for Section 6, so every comparison below is informative.

| Item | Value |
|---|---|
| Software | Pinocchio-EP `1.0.0+b0c9cf1.dirty` (branch `ashrae140-compliance`, unreleased), with the bundled EnergyPlus `26.1.0-6f2e40d102`, unmodified |
| Weather | The standard's **normative EPW alternates** `WD100.epw`–`WD600.epw` (Table A1-26; 6.2.1.1.1 note 4), from the ASHRAE accompanying files. These are **not substitutes**. They are used unchanged and copied to `weather/` (ASHRAE files: local only, do not publish). |
| Site | Latitude, longitude, time zone and elevation are read from each EPW LOCATION record. There is no Site:Location in the IDF. They match Tables 6-1 and 6-3 to 6-6, including **TZ +5.5 for WD500**, which EnergyPlus accepted (eio: `Site:Location,…,28.57,77.10,5.50,236.80`). |
| Time step | 4 per hour, the application default. The Std-140 profile does not change it. Outputs are hourly. |
| Runs | 6 cases plus a WD100 run with 15-minute outputs for the time-stamp check. 0 severe errors. Warnings: unused starter constructions, and in WD100/WD600 two engine psychrometric warnings (S140outNotes G-3). |

## How the model is built (`harness/wd_suite.mjs`)
The model goes through Pinocchio's own project model and IDF emitter (`newProject` → `exportIdf`), as described in audit section L.

**From Pinocchio GUI/registry inputs:**
- `project.profile = "std140"`: no DST or holidays, Country terrain, FullInteriorAndExterior.
- The Fig 6-1 zone: an 8 × 6 m hexagon with the south corners cut 2 × 2 m at 45°, and 2.7 m walls. Pinocchio's walls give **6 of the 10 Table 6-2 surfaces**: S, E, N and W at 90°, 45° E of S and 45° W of S. Wall-0001…0006 follow the footprint edges.
- `project.outputVariables` (registry key `project.outputVariables`, WP-A) carries every 6.3 output: Site dry-bulb, RH, dew point, humidity ratio, wet-bulb, wind speed and direction, barometric pressure, total and opaque sky cover, sky temperature, plus Surface Outside Face Incident Solar / Beam / Sky Diffuse / Ground Diffuse for the 10 surfaces. **No passthrough Output:Variable objects were needed.**

**Passthrough** (objects pushed into `project.model.objects`, which the export emits verbatim; documented in S140outNotes C and F):
- **The Fig 6-2 roof.** It has a horizontal top at 2.7 + 2·tan 30° = 3.855 m, 30° planes facing E, S and W, two corner triangles, and a north gable band. These planes enclose a separate "WD Roof Space" zone with an adiabatic floor. The Pinocchio zone's own flat roof is set Adiabatic. Pinocchio's roof library has no "flat top with 30° hips on three sides" type: its mansard top cannot be flat (minimum 2°) and its hip slopes every side.
- **`Site:GroundReflectance`**: 0 for WD100–WD500, 0.2 for WD600. There is no GUI field for it yet; the request is in `COORDINATION.md`.
  - EnergyPlus 26.1 **ignores the EPW albedo column**. That column is 0.2 in WD300.epw and WD500.epw, yet ground-diffuse is 0 in those runs, as specified.

## Results

### 1. Incident solar vs an independent calculation (`harness/wd_analyze.py`; pvlib is not installed, so numpy is used)
The independent model has these parts:
- Michalsky solar position.
- Beam = DNI·cos θ.
- Perez 1990 sky diffuse (all-sites composite coefficients).
- Ground diffuse = ρ·GHI·(1−cos β)/2.

It is evaluated with two timings:
- **(a) midpoint:** hour-average EPW solar with the sun at the middle of the hour.
- **(b) E+ timing:** four 15-minute steps with DNI/DHI interpolated between hour midpoints, then averaged.

Annual totals on the 10 surfaces, Pinocchio/E+ vs independent:

| Case | vs (a) midpoint, % | vs (b) E+ timing, % | hourly RMSE vs (b), W/m² | Horizontal kWh/m², E+ vs EPW GHI |
|---|---|---|---|---|
| WD100 | −0.38 … +2.74 | −0.07 … +1.91 | ≤ 8.9 | 1664.6 vs 1670.2 |
| WD200 | −0.51 … +0.35 | −0.52 … +0.06 | ≤ 8.8 | 1676.8 vs 1685.4 |
| WD300 (S hemisphere) | −0.46 … +1.12 | −0.22 … +0.18 | ≤ 9.7 | 1901.6 vs 1910.3 |
| WD400 (71° N) | −0.61 … −0.02 | −0.54 … −0.29 | ≤ 5.6 | 710.1 vs 713.9 |
| WD500 (TZ +5.5) | −0.44 … +0.89 | −0.30 … +0.10 | ≤ 9.5 | 1918.5 vs 1927.0 |
| WD600 (ρ = 0.2) | −0.38 … +1.53 | −0.07 … +1.14 | ≤ 8.9 | 1664.6 vs 1670.2 |

- The largest relative gaps are on the **north wall**, which has small totals (WD100: 271.6 vs 266.5 kWh/m²). The two Perez implementations differ in detail there: horizon band and circumsolar treatment.
- An **isotropic** sky gives −12.9 % on N 90 and +5 to +8 % on the south and east/west walls. The engine is clearly anisotropic.

Per-surface tables are in `analysis_tables.md`; machine-readable results are in `scoring.json`.

### 2. Comparison with the informative example results (Annex B8.3, `Std140_WD_Annual_Results.xlsx`, 8 programs)
- **55 of 60** case×surface annual-average incident-solar values fall inside the range of the example programs.
- The 5 outside are all **below the minimum by 0.03–0.34 %**: WD200 W30 −0.03 %, WD400 N90 −0.34 %, WD400 W90 −0.23 %, WD400 45W90 −0.22 %, WD400 W30 −0.11 %. In each of those, the minimum is EnergyPlus 9.5 itself.
- Pinocchio/E+ 26.1 vs the EnergyPlus 9.5 example result: **max |Δ| = 0.48 %** over all 60 values.
- **WD600 − WD100** (ground-reflected): **10 of 10 inside the range**. The vertical value is 19.001 W/m² (E+ 9.5: 18.999); the 30° value is 2.546 W/m² (range 2.529–2.560).
- **Weather quantities** (annual means): agree with the EPW and E+ 9.5 to ≤ 0.001 in RH, pressure, wind speed and sky cover.
  - Some "outside range" flags occur because the example range collapses to its 4th decimal. Examples: total sky cover 5.0889 vs 5.0889–5.0893, and WD100 wet-bulb 4.8876 vs minimum 4.8883.
  - **Wind direction** annual mean: 173.6° vs 179.3° in the EPW (WD100). See below.

### 3. EPW input vs engine output consistency (hourly)
- The engine's hourly value equals **0.375·X(h−1) + 0.625·X(h)** of the EPW to within 1e-5 for every case. This holds for dry-bulb, RH, pressure, wind speed, and total and opaque sky cover. It is the mean of the four linearly interpolated 15-minute samples.
- Against the raw EPW hour value, annual-mean biases are ≤ 0.001 °C, 0.002 % RH and 0.1 Pa.
- **Dew point** is recomputed psychrometrically. Most hours match within 0.02 °C. In WD400, 115 hours near saturation below about −35 °C differ by 0.05–0.1 °C, and the maximum difference is 1.36 °C. In WD500, 3 hours differ.
- **Wind direction** is interpolated as a **scalar**. When it crosses north (e.g. 310° → 20°), the hourly mean points the opposite way (173.75°). WD100 has 933 such hours. This is an engine anomaly (S140outNotes G-2); E+ 9.5's example result shows the same.

### 4. Time stamping, 6.3 (`timestamps.json`, WD100 15-minute run)
- **Instantaneous** EPW quantities are placed at the **end** of their hour. The 15-minute dry-bulb at hh:00 equals the EPW value exactly (max |Δ| = 0), and values are linear in between.
- **Solar** EPW values, which are averages over the preceding hour, are placed at the **middle** of their hour. With the sun up, the 15-minute DHI and DNI at hh−1:30 equal the EPW hour-hh value in **100 %** of 4383 hours, versus 1 % at hh:00.
- So EnergyPlus applies the TMY3/EPW conventions of 6.2.1.1.1 note 2 correctly.
- **Hourly reports** are means of the samples at :15, :30, :45 and :00. For temperatures that equals the interpolated value at hh−0:22:30. For solar it is a smoothed hour average, about −0.1 to −0.6 % on annual horizontal.
- The workbook stamps all hourly values at the **hour midpoint** (hh−0.5), both "Time of Year" and "Solar Time of Year", following 6.3 ("average value … midpoint").

## Files
- `inputs/<case>/in.idf`: the exact IDF Pinocchio emitted. The first line carries the app version stamp.
- `outputs/<case>/`: eplusout.csv, err, eio, eplustbl.htm. All files are under 20 MB. `WD100_timestep` holds the time-stamp run.
- `weather/`: WD100–WD600.epw, the normative ASHRAE files (local only).
- `Std140_WD_Output_PinocchioEP.xlsx`: the **official** `Std140_WD_Output.xlsx` filled with openpyxl. It has Program Info and 8760 hourly rows × 43 columns per case. Units: °C, %, kg/kg, m/s, ° from N, **mbar** (Pa/100), tenths, and Wh/m² (hourly mean W/m² × 1 h). Diffuse = sky diffuse + ground diffuse.
- `S140outNotes_6.TXT`: blocks A–G.
- `scoring.json`, `analysis_tables.md`, `timestamps.json`.
- `harness/`: `wd_suite.mjs` (build and run), `wd_analyze.py` (comparisons, scoring, workbook), `wd_timestamps.py`. The sources live in `pinocchio-ep/tools/std140/wd/`.
- Re-run with `node tools/std140/wd/wd_suite.mjs --ts && python tools/std140/wd/wd_analyze.py && python tools/std140/wd/wd_timestamps.py` (needs `PINOCCHIO_APP_VERSION` for the stamp).

## Deviations and gaps (honest)
- **Passthrough:** the roof planes, the roof-space zone and Site:GroundReflectance. **4 of the 10 test surfaces** (horizontal, E30, S30, W30) cannot be drawn in Pinocchio's GUI today. A user would have to use the IDF objects editor or an imported IDF.
- **Not covered:**
  - Sub-hourly results are not in the workbook; they are in the WD100 time-stamp run only.
  - The official Python comparison script (data.ashrae.org) was not run; the annual comparison uses `Std140_WD_Annual_Results.xlsx` directly.
  - The informative hourly `ResultsFiles_WD` were not extracted (disk space).
