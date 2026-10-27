# Pinocchio-EP: ANSI/ASHRAE Standard 140-2023 test results

**ERGO Energy LLP**: Standard 140 conformance evidence for **Pinocchio-EP**, the web building energy modelling application at [www.ergoenergy.in/pinocchio](https://www.ergoenergy.in/pinocchio).

| | |
|---|---|
| Software | Pinocchio-EP, release `1.0.0+e2c8ebf` (results generated on build `1.0.0+b7b2379`; the later change only affects one Review display card, and the on-engine regression gate and all tests were re-run on it) |
| Engine | EnergyPlus 26.1.0-6f2e40d102, official NREL build, unmodified |
| Standard | ANSI/ASHRAE Standard 140-2023 with Addendum a (2025) and Addendum b (2026) |
| Test inputs | ASHRAE's official Standard 140-2023 accompanying files, with Addendum b's ETNA files |
| Date | 27 October 2026 |

Every case was built as a Pinocchio-EP project, or as an IDF imported into it. Each was exported with Pinocchio-EP's own IDF writer, the code path behind the application's Run button, and run on the engine above. No input or code was adjusted to match reference results.

## Results: Annex A3 graded test groups

| Test group | In range | Needed | Result |
|---|---|---|---|
| Thermal fabric, low mass (Section 7) | 34/34 | 30 | **Pass** |
| Thermal fabric, high mass (Section 7) | 28/28 | 25 | **Pass** |
| Cooling equipment (Section 9) | 14/14 | 12 | **Pass** |
| Heating equipment (Section 10) | 6/6 | 5 | **Pass** |
| Air-side HVAC (Section 11) | 54/54 | 48 | **Pass** |
| **All graded range cases** | **136/136** | | **5/5 groups** |

The thermal-fabric results include the Addendum a hourly-load criteria (Pearson correlation and RMSE on 1 February and 14 July).

## Informative sections (the standard sets no pass/fail)

| Section | Result |
|---|---|
| 6 Weather drivers | 6/6 cases. The engine reproduces the weather file to 1e-5. 55 of 60 annual incident-solar values fall inside the example-program range |
| 8 Ground coupling (Kiva) | 16 of 17 cases, within −7.4 % … +1.2 % of the verified numerical models. GC10a cannot be represented in Kiva |
| 12 Class II (HERS BESTEST) | Tier 1 + L165A. 38 of 58 results inside the informative example ranges |
| 13 ETNA empirical validation | Heater energy vs measured: ET110A1 +0.12 %, ET110B1 +0.33 %, ET100A1 +0.03 %, ET100B1 +0.03 %, ET100A3 +0.77 %, ET100B3 +1.04 % |

## Repository contents

| Path | Contents |
|---|---|
| `report/main.pdf` | technical report (IEEE format) with interface screenshots |
| `paper/` | detailed paper (v2), plus the first assessment (v1, 26 Oct 2026) that found the issues fixed before testing |
| `00_summary/` | compliance summary and machine-readable metrics |
| `NN_<section>/README.md` | method, results tables, deviations |
| `NN_<section>/S140outNotes_N.TXT` | Annex A2 modeller report (blocks A–G) |
| `NN_<section>/Std140_*_Output*.xls[x]` | the official ASHRAE output workbooks, filled with Pinocchio-EP results |
| `NN_<section>/scoring.json` | every result and range check |
| `NN_<section>/inputs/<case>/in.idf` | the exact EnergyPlus input Pinocchio-EP produced for each case |
| `NN_<section>/outputs.zip` | the EnergyPlus outputs for each case (`eplusout.csv`, `eplusout.err`, `eplustbl.htm`) |
| `01_software/` | test logs of the tested build: 992 front-end and 210 back-end tests passing |

**Not included:**
- **ASHRAE's own accompanying files** (weather files, measured ETNA data, example results of other programs), because they are ASHRAE's copyrighted material. ASHRAE publishes them free of charge at [data.ashrae.org/standard140](https://data.ashrae.org/standard140/).
- **Pinocchio-EP's source code.**

## Notes
- ASHRAE does not certify or approve software. These results document testing to the method of test in ANSI/ASHRAE Standard 140-2023 and do not imply ASHRAE endorsement.
- The results apply to the version stated above. Later releases are re-tested with the same harness.
- Known limits, and defects found in EnergyPlus 26.1 and in the standard's own files, are listed in `00_summary/COMPLIANCE_SUMMARY.md` and in each section's S140outNotes, block G.

Contact: ERGO Energy LLP, [www.ergoenergy.in](https://www.ergoenergy.in)
