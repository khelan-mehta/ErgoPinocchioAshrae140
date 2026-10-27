# Pinocchio-EP and ASHRAE Standard 140-2023: compliance summary (27 Oct 2026)

## 1. How compliant is it?

ASHRAE certifies no software. Standard 140 grades each Annex A3 test group pass/fail, and compliance is self-declared by publishing the results. The figures below are measured, not estimated.

| Metric | Before (26 Oct, GUI route) | Now (27 Oct) |
|---|---|---|
| Graded test groups passed (5 groups have Annex A3 ranges) | 1/5 = 20 % | **5/5 = 100 %** |
| Graded range cases in range (out of all 136 in A3, Add. a) | 54/136 = 40 % | **136/136 = 100 %** |
| Range cases in range, of those tested | 54/62 = 87 % | **136/136 = 100 %** |
| Test sections run (6, 7, 8, 9, 10, 11, 12, 13) | 1/8 | **8/8** |
| Clause-by-clause conformance matrix (18 rows; conforms = 1, partial = ½) | ≈ 31 % | **≈ 86 %** (15.5/18; see §3) |
| Formal status | not compliant | **all graded tests pass. Not yet declared:** needs the released, version-stamped build plus the public results page (§4) |

## 2. Results by section

| Sec | What | Cases run | Result | Weather |
|---|---|---|---|---|
| 6 | Weather drivers | WD100–WD600 (6) | Informative. 55/60 incident-solar annuals in range, 10/10 ground-reflected deltas. Engine output equals the EPW to 1e-5 | official WD*.epw |
| 7 | Thermal fabric | all 52 (600–695, 900–995, 960, 6 FF, in-depth 195–470, 800, 810) | **Low 34/34, High 28/28**, incl. all 22 Addendum-a hourly curves. Base: 600 = 4.301 / 6.015 MWh, 900 = 1.661 / 2.506 MWh (heat/cool) | official 725650TY.csv → EPW (Pinocchio converter) |
| 8 | Ground coupling | 16/17 (GC10a omitted) | Informative. Within −7.4…+1.2 % of the verified-numerical-model mean; 15/16 inside the example range | official GC*.TM2 → EPW |
| 9 | Cooling equipment | all 34 (CE100–CE200, CE300–CE545) | **14/14 (A3-7)**. CE100–200 vs quasi-analytical: total energy 0.3–1.8 % low, COP −0.1…+1.2 % | official CE*.TM2 + CE*MAP.xls |
| 10 | Heating equipment | HE100–HE230 | **6/6 (A3-8)**. HE210 furnace input 52.694 GJ. HE100 77.756 GJ vs analytical 77.713 | official HE*.WY2 |
| 11 | Air-side HVAC | AE101–AE445 | **54/54 (A3-9)** | constant ambient per Table 11-4 (allowed as direct input) |
| 12 | Class II HERS BESTEST | tier 1 + L165A (36 runs) | Informative. 38/58 inside the B22 example ranges. Low bias, mostly window terms, cause not found; nothing tuned. P-series not run | official Colorad.TMY / Lasvega.TMY |
| 13 | ETNA empirical (Add. b) | all 6 | Informative. ET110A1 +0.12 %, ET110B1 +0.33 %, ET100A1 +0.03 %, ET100B1 +0.03 % (all hours within ±u). ET100A3 +0.77 %, ET100B3 +1.04 % with the measured airflow (owner's choice; without it −8.5 / −8.3 %, both disclosed) | official ETNA files |

## 3. Clause-by-clause matrix (was 31 %, now ≈ 86 %)

| Requirement | Before | Now | Evidence |
|---|---|---|---|
| Unmodified engine | ✔ | ✔ | EnergyPlus 26.1 official build |
| 5.1.1.1 time convention, no DST or holidays | ½ | ✔ | Std 140 profile turns off DST and holidays |
| 5.1.1.4 / 5.1.1.9 numerical settings never changed silently | ✘ | ✔ | import timestep cap removed; timestep shown in Review |
| 5.1.1.5/6 equivalent and nonspecified inputs documented | ½ | ✔ | S140outNotes per section (blocks A–G); profile clause list |
| 5.1.2 / Annex A2 reporting | ✘ | ½ | official workbooks filled for 7, 9, 10, 11, 13 (equivalents for 6, 8, 12), the in-app report download, version stamp. Pending: public posting of the released version |
| Annex A3: the software tested itself, not a proxy | ✘ | ½ | done for Pinocchio itself; becomes ✔ when published |
| Sec 6 | ½ | ✔ | run |
| Sec 7 low mass | ✔ | ✔ | 34/34 |
| Sec 7 high mass | ½ | ✔ | 28/28 |
| Sec 7 cases 450–470 (constant coefficients) | ½ | ✔ | convection panel |
| Terrain (7.2.1.9.1) | ✘ | ✔ | Site exposure → Building › Terrain |
| Free-float cases | ½ | ✔ | free-float runs allowed |
| Section 7.3 outputs | ½ | ✔ | output-variable picker, raw loads, peaks with timestamps |
| Sec 8 | ½ | ½ | Kiva added; GC10a and the near-surface temperatures cannot be expressed in Kiva |
| Sec 9–11 | ½ | ✔ | 74/74 range cases |
| Sec 12 | ✘ | ½ | run; 38/58 informative; P-series not run |
| Sec 13 (engine) | ½ | ½ | all 6 run; A3/B3 airflow choice disclosed |
| Sec 13 through Pinocchio | ✘ | ✔ | import keeps the timestep; schedules inlined; native boundary/convection features |

## 4. What remains before you can declare compliance

1. **Done (27 Oct 2026):** committed and deployed with version stamp `1.0.0+e2c8ebf` (results generated on `b7b2379`; the later change is display-only and was re-verified by the regression gate).
2. **Publish** the results page with the inputs, outputs, modeller reports and the example results of the other programs (90.1-2022 §12.4.1.4.2). Do not publish the standard's text or ASHRAE's own files.
3. **Owner decision recorded:** ETNA A3/B3 primary = with the measured airflow, and the order in which it was added is disclosed.

## 5. Known limits (documented in each section's S140outNotes, block G)

| Limit | Kind | Effect |
|---|---|---|
| FisherPedersenCeilingDiffuser* keys fatal in `MultipleSurface` | EnergyPlus 26.1 bug | blocks the informative ETNA A3/B3 inputs; Pinocchio uses CeilingDiffuser instead |
| `LockoutWithCompressor` ignored on a load-controlled UnitarySystem | EnergyPlus 26.1 | CE410 = CE400; still in range |
| `Economizer Operation Staging = EconomizerFirst` segfault | EnergyPlus 26.1 | avoided |
| EPW albedo column ignored | EnergyPlus 26.1 | Pinocchio now writes `Site:GroundReflectance` from a new field |
| Wind direction interpolated straight through north (310° → 20° gives 173.75°) | EnergyPlus 26.1 | informative Section 6 anomaly |
| Kiva is 2-D, steps hourly and has no ground-temperature output points | engine | GC10a omitted; GC65b/GC70b −7 % |
| DX bypass factor derived from rated SHR (0.144 vs map 0.070) | modelling method | CE in range; documented in block C |
| 5 VAV AE cases have 0.3–1.6 % hour-to-hour noise | engine iteration | the last hour is reported; 24-h medians also score 54/54 |
| The native face editor can't split a face into parallel conduction paths | GUI | the ETNA cell is entered by IDF import (a GUI route) |
| Default settings (no Std 140 profile) | by design | Case 900 cooling 2.936 MWh, out of range: a normal project uses the project's real terrain and real DST |
| Defects in the standard's own files (equation-of-time typo, WD400 header, TMY LST column, WY2 duplicate hour) | ASHRAE files | handled by the converter and documented |

## 6. What changed in the software (details in `01_software/CHANGES.md`)

| WP | Changes |
|---|---|
| **A** | terrain field now reaches the IDF; no silent timestep cap; free-float runs; opt-in **Standard 140 profile**; output-variable picker; honest raw ideal loads and peaks; version stamp; sizing periods safe for any EPW; .ddy clean-up; mid-month schedule dates; negative "sensible gain/removal" loads; Kiva and ground-reflectance wiring; weather-upload and attachment mounts; IDF transition on import |
| **B** | layered glazing (62-glass library + Std 140 presets); exact materials (k/ρ/cp/absorptances), with edits no longer wiped; **add/delete any IDD object** (full 856-class IDD); convection panel (algorithms + constant coefficients); full-wall overhangs and full-height fins |
| **C** | TMY3/TMY2/TMY1/WYEC2 → EPW converters (FE + BE); no silent weather fallback; Schedule:File attachments and inlining; official IDF version transition |
| **D** | Section 7 and 13 harnesses; Std 140 report exporter (S140outNotes + official workbook filling); Annex A3 scoring (`bestest.js`); engine regression gate |
| **E1** | DX performance-map fitting op plus the inspector tab |
| **E2** | lookup-table curves in Studio ops; unitary-system inspector bug (every furnace built in the Studio GUI was fatal) |
| **F** | Kiva ground coupling (`src/lib/ground/`), ground panel and 25 registry fields |
| **G** | door picker takes custom opaque constructions; door sill height; native other-side-coefficients boundary; imported-surface convection; Std 140 report download; agent coverage and KB |

Accuracy: every change either lets the user enter the real input or stops Pinocchio from silently altering it. The engine is untouched and nothing was tuned to reference results. Test-specific settings live only in the opt-in profile.
