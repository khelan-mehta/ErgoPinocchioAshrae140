# Section 13 — ETNA empirical validation (Addendum b), through Pinocchio-EP (WP-D)

> **Update 2026-10-27, after WP-G.** The guards now come in with the imported cell and are recorded as native "Other side coefficients" face boundaries. The per-path coefficients are entered through the native imported-surface convection table. "+ Add object" is now used only for the EMS airflow program (and for the fatal FisherPedersen sensitivity runs). All six cases were re-run. The exported IDFs contain the same object set as before (checked object by object for ET110A1 and ET100A3), and every Qhtr result is unchanged to the last digit.
>
> **Owner decision (2026-10-27): the PRIMARY variant imposes the measured HWD airflow** (`scoring.json › primaryVariant`). The no-airflow variant stays recorded under `withoutHwdAirflow` (ET100A3 −8.48 %, ET100B3 −8.34 %), and the disclosure below is kept.

Pinocchio-EP version stamp: `1.0.0+b0c9cf1.dirty` (unreleased working tree, branch `ashrae140-compliance`). Engine: EnergyPlus 26.1.0
(6f2e40d102), unmodified, the build bundled with the app. Section 13 has **no acceptance criteria**; every comparison here is informative.

## What was run

All six cases of Table 13-1: ET110A1, ET110B1, ET100A1, ET100B1, ET100A3, ET100B3, each from the **normative** Addendum b specification and
the normative files in `_spec/zip140/Accompanying Files/Normative Materials/` (measurement CSVs, `ET110meteo_within_Melun-071530_MY.2000_v3.epw`).
The informative EnergyPlus 23.1 inputs were **not** used to build the models (they were opened only afterwards as a sanity check, see "Notes").

### Input route (all GUI-reachable)

| Input | Spec | Pinocchio-EP route |
|---|---|---|
| Cell geometry: 16 parallel 1-D conduction paths (ceiling 2, floor 3, north wall + door, E/S/W walls, 3 window paths on each windowed wall) | Tables 13-5, 13-23; Fig 13-2 (3.5 x 4.655 x 2.565 m) | IDF authored from the spec (`tools/std140/et/build.mjs` `cellIdf`) → **Project › Import IDF**. One surface per path (strip of the face with the tabulated net area). The native face editor cannot split a face into parallel paths (COORDINATION request to WP-B). |
| Layers with imputed k, ρ, cp (Cell A, Cell B, insulated / uninsulated windows) | Tables 13-6…13-18, 13-25…13-37, 13-42…13-44 | Same authored IDF (`spec.mjs`, every value with its table). Floor paths merged to 10 layers exactly as the standard's informative notes recommend (EnergyPlus 10-layer limit). Composite R and UA reproduce the published values (test `et.test.mjs`). |
| Guards as boundary conditions: measured guard air temperature + hcomb,ext = 18 W/m²K | 13.2.2.15.1, Table 13-20 | Part of the cell brought in with **Project › Import IDF**: 6 × `SurfaceProperty:OtherSideCoefficients` (film coefficient 18, constant-temperature coefficient 1, schedule = measured guard T); every path surface carries `OtherSideCoefficients`, NoSun, NoWind. The importer records them as the faces' **Other side coefficients** boundary (`{ boundary: "othersidecoefficients", oscObject }`, WP-G), so they survive later edits. (Was: + Add object, before WP-G.) The native generated-OSC form (`oscH` / `oscSchedule`) needs native faces, and the native editor cannot split a face into the 16 parallel paths. |
| Measured hourly schedules (Tattic, Tcellar, Tnorth, Teast, Tsouth, Twest, Tcell, Qfan, HWDsafr) | Table 13-3, 13.2.2.2.1 | A schedule CSV (8784 h, 2000, GMT+1, preceding-hour stamps on the hour they end) dropped with the IDF; WP-C's importer **inlined** it as exact `Schedule:Compact`s (`gui_steps.json`: `status: inlined`). Before the data: first row held; after: last row held. |
| Fan heat, 100 % convective | 13.2.2.14 | `ElectricEquipment` 1 W × Qfan schedule, radiant/latent/lost 0 (authored IDF). |
| Ideal heater, unlimited, 100 % convective, no OA, set point = measured Tcell | 13.2.2.16 | `HVACTemplate:Zone:IdealLoadsAirSystem` + thermostat (authored); **Standard 140 profile** (`project.profile = std140`) forces dehumidification/humidification/OA = None. |
| Measured HWD supply air flow (CSV column J, usage "Input") | Table 13-3, 13.2.6.4.1 a.1.ii | **+ Add object**: EMS sensor / actuator (Ideal Loads Air System › Air Mass Flow Rate) / calling manager / program, m³/h × 1.1455 kg/m³ ÷ 3600. All six cases (5.1.1.4); see "A3/B3" below. |
| Constant hcomb,int per path ("1" cases) | Tables 13-20, 13-39, 13-46 | **Envelope › Surface heat transfer › Imported surfaces** (`convection.setModelSurfaceConvection`, registry `idfSurface.convInside`, WP-G): 16 inside constants → `SurfaceProperty:ConvectionCoefficients` (Inside, Value). (Was: + Add object, before WP-G.) **objects editor**: inside-layer Thermal Absorptance → 0.00001 (IDD minimum), per 13.2.2.10 Informative Note 4. |
| Interior algorithms ("3" cases) | 13.2.6.4, 13.2.7.4 | **Envelope › Surface heat transfer** panel (`lib/convection.setConvectionAlgorithm`): inside = CeilingDiffuser; Table 13-19 / 13-45 emittances kept. |
| Timestep | 13.2.2.5, 5.1.1.9 | **objects editor**: Timestep = 60/h (import no longer caps it, WP-A). |
| Outputs | 13.3, Tables 13-50…13-54 | **Project › Analysis options › Output variables** (`project.outputVariables`, 18 picks): ideal-load sensible heating energy, zone air T, fan energy, per-surface inside convection / net LW radiation / conduction (W/m², hourly mean = Wh/m²), inside surface T and h, schedule values. |

The exported IDF (`inputs/<case>/in.idf`) is exactly what the GUI's Run sends to `POST /api/run`; `inputs/<case>/gui_steps.json` lists each step,
`inputs/<case>/<case>_authored_for_import.idf` + `_schedules.csv` are the files that were imported.

## Timestep convergence study

Mean modelled Qhtr over the best steady-state window (Wh/h) vs timesteps per hour (change relative to 60/h):

| Case | HWD flow imposed | 4 | 6 | 10 | 15 | 20 | 30 | 60 |
|---|---|---|---|---|---|---|---|---|
| ET110A1 | yes | 635.43 (−0.014 %) | 635.45 (−0.011 %) | 634.89 (−0.100 %) | 635.48 (−0.007 %) | 635.06 (−0.072 %) | 635.34 (−0.029 %) | 635.52 |
| ET110A1 | no | 635.45 | 635.47 | 634.90 | 635.48 | 635.07 | 635.34 | 635.52 |
| ET100A1 | yes | 736.76 (−0.006 %) | 736.78 | 736.18 (−0.085 %) | 736.78 | 736.35 (−0.062 %) | 736.63 | 736.81 |
| ET100A1 | no | 736.78 | 736.78 | 736.19 | 736.78 | 736.35 | 736.63 | 736.81 |
| ET100A3 | yes | 742.20 (−0.012 %) | 742.23 | 741.64 (−0.087 %) | 742.25 | 741.82 (−0.063 %) | 742.10 | 742.29 |
| ET100A3 | no | 674.09 | 674.10 | 673.50 | 674.10 | 673.67 | 673.94 | 674.12 |

**There is no 60/h need in this model**: every timestep from 4 to 60/h is within 0.1 % of 60/h (an order of magnitude below u = 0.7–1.0 %), with or
without the imposed HWD air flow, and also with `ZoneAirHeatBalanceAlgorithm AnalyticalSolution` (ET110A1 forced flow: 635.44 at 4/h vs 635.52 at 60/h).
The small non-monotonic step at 10 and 20/h is the hourly reporting of sub-hourly values. 60/h is used for all six cases because it costs ~3 s per run
(5.1.1.4 consistency). The previous paper's +293…+1043 % at 4/h was a property of the *informative* EnergyPlus input routed through the old
import cap (which also carries a 1-minute minimum system timestep, EMS mass-flow actuation inside the HVAC iteration and a radiant-heater comment
"60 required"), not of the ETNA physics; it is not reproduced by a model built from the normative spec.

## Results vs measured Qhtr ± u (best steady-state windows)

| Case | Window (GMT+1) | n | Measured avg (Wh/h) | ± u | Model avg (Wh/h) | Diff | Hours within band | Model min…max |
|---|---|---|---|---|---|---|---|---|
| ET110A1 | 2/10 16:00 – 2/11 09:00 | 18 | 634.7 | 0.9 % | **635.5** | **+0.12 %** | **18/18** | 632.6…638.4 |
| ET110B1 | 2/10 16:00 – 2/11 09:00 | 18 | 640.4 | 1.0 % | **642.5** | **+0.33 %** | **18/18** | 637.3…646.7 |
| ET100A1 | 9/16 08:00 – 9/18 14:00 | 55 | 736.6 | 0.7 % | **736.8** | **+0.03 %** | **55/55** | 732.9…741.1 |
| ET100B1 | 9/16 08:00 – 9/18 14:00 (last 56 excl. final) | 55 | 737.6 | 0.9 % | **737.8** | **+0.03 %** | **55/55** | 733.9…743.6 |
| ET100A3 | 9/16 08:00 – 9/18 14:00 | 55 | 736.6 | 0.7 % | **742.3** | **+0.77 %** | 27/55 | 731.2…758.6 |
| ET100B3 | same as ET100B1 | 55 | 737.6 | 0.9 % | **745.2** | **+1.04 %** | 22/55 | 729.2…760.8 |

Comparison with the earlier study (`ashrae140-paper/main.tex`, `work/etna/results.json`, `compare.json`):

| Case | Old: informative IDF, direct engine | Old: same IDF via Pinocchio import (Timestep capped 4/h) | **Now: normative spec via Pinocchio GUI paths** |
|---|---|---|---|
| ET110A1 | 635.1 (+0.06 %), 14/18 | 7255 (+1043 %), 0/18 | **635.5 (+0.12 %), 18/18** |
| ET110B1 | 643.6 (+0.50 %), 5/18 | 7204 (+1025 %), 0/18 | **642.5 (+0.33 %), 18/18** |
| ET100A1 | 736.3 (−0.04 %), 37/55 | 2893 (+293 %), 0/55 | **736.8 (+0.03 %), 55/55** |
| ET100B1 | 737.3 (−0.04 %), 43/55 | 2948 (+300 %), 0/55 | **737.8 (+0.03 %), 55/55** |
| ET100A3 | fatal (FisherPedersen MultipleSurface); 794.5 (+7.86 %) with the roof override removed | not run | **742.3 (+0.77 %), 27/55** |
| ET100B3 | fatal; 797.0 (+8.05 %) with the override removed | not run | **745.2 (+1.04 %), 22/55** |

Secondary, informative — per-wall heat flow over the window, model Σ Qsurf (W) vs UA_required × (Tcell − Tguard) (W), difference:

| Case | North + door | East | South | West | Ceiling | Floor |
|---|---|---|---|---|---|---|
| ET110A1 | 150.1 / 149.9 (+0.1 %) | 169.9 / 170.3 (−0.3 %) | 139.7 / 139.8 (−0.1 %) | 87.4 / 85.9 (+1.8 %) | 67.5 / 67.4 (+0.2 %) | 157.6 / 158.1 (−0.3 %) |
| ET110B1 | 142.6 / 142.5 | 77.6 / 75.3 (+3.0 %) | 155.6 / 155.4 | 162.2 / 162.1 | 62.9 / 62.9 | 182.5 / 183.0 |
| ET100A1 | 149.5 / 149.4 | 229.4 / 229.4 | 198.2 / 198.4 | 84.6 / 84.4 | 65.0 / 65.0 | 154.8 / 154.7 |
| ET100B1 | 140.4 / 140.4 | 74.5 / 74.2 | 211.7 / 211.7 | 219.8 / 220.1 | 61.9 / 61.9 | 178.6 / 178.5 |
| ET100A3 | 148.1 / 149.4 (−0.8 %) | 230.6 / 229.4 | 199.1 / 198.4 | 85.3 / 84.4 | 66.8 / 65.0 (+2.8 %) | 157.1 / 154.7 (+1.5 %) |
| ET100B3 | 139.0 / 140.4 (−1.0 %) | 74.8 / 74.2 | 213.5 / 211.7 | 221.0 / 220.1 | 63.4 / 61.9 (+2.3 %) | 182.5 / 178.5 (+2.2 %) |

(UA_required: Tables 13-6…13-24, 13-41, 13-48; Cell B windowed walls with insulation = uninsulated − 2.44 W/K. The ET110 party walls, A west / B
east, are the heaviest, slowest paths and still release stored heat in the 18-h window.)

## A3 / B3: interior algorithm (Block B) and sensitivity

Chosen: **CeilingDiffuser** + Table 13-19 / 13-45 emittances + the measured HWD air flow imposed on the ideal heater. CeilingDiffuser is the only
EnergyPlus 26.1 inside algorithm whose coefficients depend on the supply air change rate, and the cell is mixed by ~21 ach of diffuser air
(13.2.6.4.1). Sensitivity (informative, mean Qhtr vs measured):

| Inside algorithm | ET100A3, HWD flow imposed | ET100B3, HWD flow imposed | ET100A3, heater's own flow | ET100B3, heater's own flow |
|---|---|---|---|---|
| **CeilingDiffuser** | **+0.77 %** (27/55) | **+1.04 %** (22/55) | −8.48 % | −8.34 % |
| Simple | −14.7 % | −14.9 % | −14.7 % | −14.9 % |
| TARP | −19.2 % | −19.3 % | −19.2 % | −19.3 % |
| ASTMC1340 | −21.8 % | −22.0 % | −21.9 % | −22.2 % |
| AdaptiveConvectionAlgorithm | −27.3 % | −27.6 % | −27.3 % | −27.6 % |
| MultipleSurface `FisherPedersenCeilingDiffuserWalls / Ceiling / Floor` (+ Add object) | **fatal**: `Unhandled convection coefficient algorithm` (EnergyPlus 26.1 upstream bug, not patched; Block G) | | | |


> **Owner decision (2026-10-27):** the primary reported results are the variant with the measured HWD supply-air flow imposed (Table 13-3 column J is a normative *input*): ET100A3 +0.77 %, ET100B3 +1.04 %. The variant without it (−8.5 % / −8.3 %) stays recorded in `scoring.json › withoutHwdAirflow`, and the order in which the flow was added is disclosed above.

**Disclosure (5.1.1.10).** The first A3/B3 runs used CeilingDiffuser with the ideal heater's own air flow (−8.5 %). The imposed HWD flow was added
afterwards, for all six cases, because Table 13-3 designates column J (HWDsafr) as an *input* and 13.2.6.4.1 a.1.ii ties the algorithm choice to it;
while adding it we also saw (sanity check) that the informative EnergyPlus input imposes the same measured flow through EMS. The "1" cases are
unaffected (≤ 0.01 %). Both variants are reported here and in `scoring.json` (`withoutHwdAirflow`, `algorithms`); the submitted workbook uses the
imposed-flow model throughout.

## Files

| Path | Content |
|---|---|
| `Std140_ET_Output.xlsx` | the OFFICIAL template filled with `std140Report.fillWorkbook` (template structure untouched; only data cells from row 6 + Program Info B1–B7): input checks (schedule values the model used, Tcell = zone air T, Qfan = fan energy, HWDsafr), Qhtr, per-path inside convective / radiative / total flux (Wh/m², + = cell → surface), A3/B3 inside surface T |
| `S140outNotes_13.TXT` | blocks A–G (`notesText(etNotesSpec(...))`) |
| `scoring.json` | per-case steady-state stats + per-path means, convergence study, algorithm study, the no-HWD-flow variant |
| `inputs/<case>/` | `in.idf` (exported), the authored IDF + schedule CSV that were imported, `gui_steps.json` |
| `outputs/<case>/` | `eplusout.csv`, `eplusout.err`, `eplustbl.htm`, `eplusout.eio` |
| `weather/` | the normative EPW (dummy for guarded cells, required by the engine) |
| `harness/README.md` | how to rerun |
| `et_test.log` | `node --test tools/std140/et/et.test.mjs` |

## Deviations / notes

- Warnings in every run: "Weather file location will be used rather than entered (IDF) Location object" (Table 13-4 site vs Melun weather; no effect —
  every surface is NoSun / NoWind with other-side coefficients) and, for the "1" cases, "areas*inside surface emissivities are summing to zero"
  (the intended minimum emittance). 0 severe errors in all six cases.
- Warm-up repeats the first run day with the first measured row held before the data starts; ET110A1/B1 have a 15-day lead, ET100 an 8-day lead.
  The first ET110 hours (cell heating from 15 °C) show Qhtr = 0 while the warmed-up cell is above the set point; this is outside the comparison window.
- Surface texture (Table 13-49) does not enter EnergyPlus' inside convection correlations; inside layers are "Smooth".
- Interior radiation uses EnergyPlus' area-weighted enclosure exchange (no user view factors); strip placement of the paths does not change it.
- Not done: the Annex B23 measured h_int comparison (`h_int_Summary_ET100A3_B3`, informative workbook not in the normative folder); model inside h and
  effective hcomb per path are in `scoring.json` (`paths.*.h_conv_model`, `h_comb_eff`).
