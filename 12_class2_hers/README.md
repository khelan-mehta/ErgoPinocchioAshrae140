# ASHRAE 140-2023 Section 12 — Class II (HERS BESTEST) through Pinocchio-EP

Pinocchio-EP 1.0.0+b0c9cf1.dirty (branch `ashrae140-compliance`), EnergyPlus 26.1.0-6f2e40d102 (the bundled engine, run locally
with `energyplus -x -r`). Section 12 has **no normative acceptance criteria**; the comparison below uses the Informative Annex B20
example results (BLAST 3.0, DOE-2.1E, SERIRES 5.7) and the Informative Annex B22 example range procedure. It is informative only.

## What was run

| Item | Status |
|---|---|
| Tier 1 L100A–L202A (11 cases × Colorado heating + Las Vegas cooling) | run, 22 results |
| L302A / L304A (slab, ASHRAE perimeter method — the normative case definition) | run |
| L322A / L324A (conditioned basement, ASHRAE below-grade method) | run (basement = 2nd zone, "A2"-type) |
| L302K / L304K — the same slabs with Pinocchio's new **Kiva** ground model (`src/lib/ground/kiva.js`, B18 soil) | run, compared with the L302B/L304B ("more detailed ground model") example rows |
| Tier 2 L165A (E/W overhang + 10 fins) | run, both climates |
| Tier 2 P100A, P105A, P110A, P140A, P150A | **omitted** (see Deviations) |
| Weather time-convention sensitivity (L100A, L140A, L170A without the solar-time shift) | run (`runs_noshift.json`) |

Every input IDF is produced by Pinocchio's own `exportIdf` from a project JSON (`harness/hers.mjs`), with `project.profile = "std140"`
(no DST/holidays, Country terrain, ideal loads with no latent removal). Files: `inputs/<case><C|L>/in.idf`,
`outputs/<case><C|L>/{eplusout.csv, eplusout.err, eplustbl.htm}`, `runs.json`, `scoring.json`,
`Std140_TF_Class2_Results_PinocchioEP.xlsx` (the official workbook, sheet `Sec7-2out` = TF_Class2_Output, filled), `S140outNotes_12.TXT`.

## Weather (normative files, converted)

`Colorad.TMY` and `Lasvega.TMY` are the standard's own accompanying files (local copy, `_spec/base140`; **not for publication**).
No converter from WP-C had landed, so `harness/tmy1_to_epw.py` (TMY1 / TD-9734 fixed-width → EPW) was written:

* **Time convention.** TMY is solar time (5.1.1.1, informative). Verified in the data: the global-irradiance centroid of each day sits
  at hour-midpoint 11.92 (Colorado) / 11.98 (Las Vegas) — Las Vegas solar noon is 11:41 LST, so the radiation records are true-solar-time
  hours. The converter re-bins radiation onto local-standard-time hours with D = 4(Lstd − Lsite) min + equation of time (annual
  totals preserved: GHI 1842 / 2167 kWh/m²; DNI −0.3 % / −0.2 % from end-of-day overlap). Effect on loads (`--noshift` run):
  L100AC 52.09 → 52.13, L100AL 47.79 → 47.76, L140AL 22.66 → 22.60 MBtu — **< 0.3 %**.
* Diffuse horizontal is missing in both files → DHI = GHI − DNI·cos z. Sky cover is 3-hourly (2446 / 3872 hours are "99") →
  linearly interpolated. RH from dry bulb / dew point. Station pressure from the file (EnergyPlus therefore applies the altitude
  air-density correction itself → the "automatic altitude adjustment" infiltration row, 0.67 ACH / attic 2.4 ACH).
* Site: Colorado Springs 38.8 N, 104.7 W, 1873 m (6145 ft), TZ −7; Las Vegas 36.1 N, 115.2 W, 664 m (2178 ft), TZ −8.

## Model (what is GUI / registry level, what is passthrough)

GUI-level (fields the GUI and the agent write): zone 57 × 27 × 8 ft, windows as 3 × 5 ft modules (glass + meeting-rail opening
2.542 × 4.542 ft at the Fig 12-2/12-11/12-15 positions), doors, per-zone constructions built from Material / Construction objects,
raised floor = boundary Outdoors with no sun / no wind, the attic = Pinocchio **gable roof** (`project.roofs`, 4:12 = 18.43°, no eave
overhang) whose unconditioned roof-space zone gets 2.4 ACH (roof `ach` field) and whose floor pairs with the ceiling, internal gains
(one ElectricEquipment: Table 12-9 profile, 70 % of sensible radiative, latent fraction 1036/5818), infiltration 0.67 / 1.5 ACH,
ideal loads (heat < 68 °F Colorado run, cool > 78 °F Las Vegas run, no capacity limit), `project.ground` (Kiva) for L302K/L304K.

Equivalent inputs (5.2.1.5): framed/insulated parallel paths are represented by one homogeneous cavity layer whose R makes the
assembly's **composite surface-to-surface R equal the table value** (walls 10.901, floor 12.928, ceiling 16.922 h·ft²·°F/Btu for L100A;
L120A 22.723 / 55.962; L200A 3.981 / 3.017 / 10.224); its mass is area-weighted. EnergyPlus calculates surface coefficients itself
(allowed by 12.2.1.9); E+-reported U with films: wall 0.0851, floor 0.0713, ceiling 0.0542, glass 1.038 Btu/h·ft²·°F (spec 0.085 /
0.071 / 0.054 / 1.064 COG).

PASSTHROUGH objects (the GUI has no form for them; emitted verbatim from `project.model` or appended to the export text):
* `InternalMass` × 2 — interior walls 921.6 ft² unframed + 102.4 ft² framed, both faces (area × 2, half construction).
* `WindowProperty:FrameAndDivider` — 2.75 in frame + 0.23 ft horizontal meeting rail, frame conductance 51.65 W/m²K (L100A) /
  4.295 W/m²K (L130A wood), edge-of-glass ratio 1.0 / 1.153; attached to every window by editing the exported text.
* `SurfaceProperty:ConvectionCoefficients` — raised floor: `SimpleCombined` (EnergyPlus' ASHRAE combined coefficient at zero wind
  for a Rough surface = 12.49 W/m²K = 2.2 Btu/h·ft²·°F exactly, 12.2.1.5); ASHRAE-method soil faces: `Value 1000` (film already in the
  table resistance) with thermal emittance 1e-4 (no IR, as the DOE-2 example inputs).
* `Shading:Building:Detailed` — L155A south overhang (whole wall, 2.5 ft at the eave); L165A E/W overhangs + 20 fins.
* L322A/L324A: each exported 8 ft basement wall is split (text post-processing) into rim-joist/sill (0.75 ft), above-grade concrete
  (0.667 ft) and below-grade (6.583 ft, no sun / no wind) bands; the GUI can only give one construction per wall face.

## Results (MBtu; heating = Colorado, sensible cooling = Las Vegas)

| Case | Pinocchio-EP | Examples min–max | B22 example range | In range |
|---|---|---|---|---|
| L100AC | 52.09 | 58.00–72.39 | 48.76–79.46 | yes |
| L110AC | 77.77 | 81.39–96.53 | 71.92–103.99 | yes |
| L120AC | 39.67 | 45.10–57.82 | 37.85–64.27 | yes |
| L130AC | 42.29 | 45.84–49.98 | 41.84–53.98 | yes |
| L140AC | 49.39 | 47.25–52.48 | 43.25–56.48 | yes |
| L150AC | 44.05 | 49.48–64.03 | 40.97–71.32 | yes |
| L155AC | 46.85 | 52.30–66.90 | 43.56–74.17 | yes |
| L160AC | 52.81 | 58.29–73.51 | 48.78–81.02 | yes |
| L170AC | 62.96 | 71.65–85.46 | 61.03–92.41 | yes |
| L200AC | 131.29 | 133.97–168.34 | 106.39–185.90 | yes |
| L202AC | 136.97 | 137.47–172.55 | 111.33–190.06 | yes |
| L302AC (ASHRAE slab) | 60.75 | 67.44–82.92 | 56.69–90.55 | yes |
| L304AC | 50.02 | 56.64–69.16 | 48.59–75.31 | yes |
| L322AC (vs A2, 2-zone) | 70.67 | 86.32–107.68 | 72.80–118.19 | **no** |
| L324AC (vs A2) | 52.30 | 60.31–73.47 | 52.49–80.03 | **no** (−0.2) |
| L302KC Kiva (vs L302BC) | 43.14 | 60.12–73.10 | 52.65–79.65 | **no** |
| L304KC Kiva (vs L304BC) | 39.16 | 50.12–61.59 | 43.92–67.61 | **no** |
| L165AC (Tier 2) | 56.87 | 64.73–78.05 | 55.09–84.66 | yes |
| L100AL | 47.79 | 54.66–60.80 | 50.66–64.88 | **no** |
| L110AL | 51.09 | 57.71–63.82 | 53.71–68.50 | **no** |
| L120AL | 43.76 | 51.36–56.14 | 47.36–60.14 | **no** |
| L130AL | 35.24 | 36.96–41.25 | 32.96–45.25 | yes |
| L140AL | 22.66 | 23.52–26.54 | 19.52–30.54 | yes |
| L150AL | 59.90 | 67.73–77.35 | 62.42–82.32 | **no** |
| L155AL | 45.09 | 54.09–59.06 | 50.09–63.06 | **no** |
| L160AL | 57.13 | 62.62–68.69 | 58.62–73.00 | **no** |
| L170AL | 38.72 | 45.83–49.30 | 41.83–53.30 | **no** |
| L200AL | 59.88 | 65.71–76.71 | 60.26–83.42 | **no** (−0.4) |
| L202AL | 50.17 | 59.61–70.57 | 52.33–75.95 | **no** |
| L165AL (Tier 2) | 48.69 | 52.87–59.58 | 48.60–63.58 | yes |

Delta cases (Tables B20-2/-4/-6/-8, all in `scoring.json`): heating deltas in range except L140AC−L100AC (−2.70 vs −10.75…−19.91),
the basement deltas and the Kiva-vs-B deltas; cooling deltas in range except L130AL−L100AL (−12.55 vs −17.70…−20.39) and
L140AL−L100AL (−25.13 vs −31.14…−34.68).

**Score (informative, B22 example ranges, A-method absolute + delta rows, P-series and B-variant rows excluded): 38 of 58 in range.**
All 11 Tier-1 Colorado heating results, L302A/L304A and both L165A results are in range; 9 of 11 Tier-1 Las Vegas cooling results are below the range (L130AL, L140AL in).

## Findings (honest reading)

1. **Systematic low loads.** All Pinocchio-EP/EnergyPlus absolute loads except L140AC are below the minimum of the three 1990s example
   programs (heating −0.4 to −12 %, cooling −4 to −17 %); the heating results still fall inside the B22 ranges (the ±4 MBtu buffer + wide
   example spread), most cooling results do not. No input was tuned toward the examples (5.1.1.10).
2. The disagreement is concentrated in the **windows**: removing all windows (L140A) changes Colorado heating by only −2.7 MBtu in
   EnergyPlus vs −10.8…−19.9 in the examples, and Las Vegas cooling by −25.1 vs −31.1…−34.7; the low-e case shows the same pattern.
   Opaque-envelope deltas (L110A, L120A, L170A, L200A, L202A, L302A−L304A) agree. Window inputs were checked in the engine report
   (glass 1.02 m² + frame 0.32 m² + rail 0.05 m² per 15 ft² module; glass U 5.894 W/m²K, SHGC 0.861; frame 51.65 W/m²K). The likely
   causes are model differences (EnergyPlus' angular glass optics, dynamic inside/outside films and detailed frame/edge model vs the
   examples' fixed combined coefficients and shading-coefficient windows) — not verified further here.
3. **Basement (L322A/L324A)** below the A-range by ~16 MBtu: Pinocchio has no single-zone "well-mixed main floor + basement" option, so
   the basement is a 2nd zone coupled through an interzone floor (12.2.2.12.1.3 allows this); main-floor solar/internal gains reaching
   the basement (basement mean 21.9 °C, max 36.8 °C) offset part of the basement loss. Inputs were checked: basement UA by element
   in the engine report = rim 25, above-grade 84, below-grade 183, floor 37 Btu/h·°F (spec 25.1 / 83.7 / 188.4 / 37.2).
4. **Kiva (L302K/L304K)** — Pinocchio's detailed ground model gives 43.1 / 39.2 MBtu, well below both the ASHRAE perimeter method
   (A rows) and the annex's 1-D soil-mass variant (B rows); the insulation benefit (L302K−L304K 4.0) is below the B range (9.7–11.5).
   A 2-D finite-volume soil domain with real storage and a 0.2 m above-grade wall is expected to lose less than the steady-state
   perimeter method; the B rows are not a verified truth (Annex B18: "this more detailed method is not verified"). Foundation-wall
   depth below grade is not given by Section 12 (Fig 12-20 is schematic) — 2 ft assumed and documented.

## Deviations / omissions

* **P100A–P150A omitted**: they need the passive-solar geometry of Figs 12-28…12-35 (new window layout, interior brick mass wall,
  massive raised floor, mass-wall locations) and Tables 12-47…12-62; not built in this pass (time). L165A was built.
* Slab and basement are modelled with the spec's own steady-state resistances to ambient air (A method). Kiva was not used for the
  basement: `lib/ground/kiva.js` couples floors only (no below-grade walls with a Foundation boundary), so a Kiva basement would be
  modelled as a slab at grade — not attempted.
* Unspecified inputs: carpet density 12 lb/ft³ and 0.5 in thickness (Kiva slab only; Kiva rejects massless layers), main-floor /
  basement-ceiling joist cavity R 1.0 for the 90 % air path, low-e outer-pane back reflectance = front (0.34), foundation wall
  8 in concrete to 2 ft below grade (Kiva only).
* One harmless engine warning in Colorado runs (PsyTwbFnTdbWPb input −100.48 °C once, 02/12 22:00) — weather row is normal
  (6.1 °C / −11.7 °C dew point); EnergyPlus' psychrometric iteration, no effect on loads.

## Reproduce

```
cd pinocchio-ep
python tools/std140/class2/tmy1_to_epw.py "<_spec>/Colorad.TMY" ../ashrae140/results/12_class2_hers/weather/Colorad_TMY.epw CO
python tools/std140/class2/tmy1_to_epw.py "<_spec>/Lasvega.TMY" ../ashrae140/results/12_class2_hers/weather/Lasvega_TMY.epw LV
PINOCCHIO_APP_VERSION=1.0.0+b0c9cf1.dirty node tools/std140/class2/hers.mjs            # all cases (one E+ process at a time)
node tools/std140/class2/hers.mjs L100A --noshift                                       # sensitivity
python tools/std140/class2/score.py                                                     # scoring.json + workbook
```
The converted EPWs in `weather/` are derived from ASHRAE accompanying files — local only, do not publish.
