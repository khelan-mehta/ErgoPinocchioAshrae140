# Section 10 — Space-heating equipment (HE100–HE230), run through Pinocchio-EP

**Verdict: PASS. All 6 of the 6 Table A3-8 range cases are in range (5 needed).** The weather is the official HE*W.WY2 files, so the result is normative, not indicative.

| Range case | Quantity | Pinocchio-EP | Table A3-8 range | In range |
|---|---|---|---|---|
| HE210 | Furnace input (GJ) | **52.694** | 49.86 – 55.11 | yes |
| HE210 | Fan energy, both fans (kWh) | **293.87** | 279.55 – 308.98 | yes |
| HE220 − HE210 | Furnace input (GJ) | **−2.860** | −5.48 – −0.23 | yes |
| HE220 − HE210 | Fan energy (kWh) | **−15.68** | −30.21 – −0.78 | yes |
| HE230 − HE220 | Furnace input (GJ) | **−6.528** | −9.10 – −3.86 | yes |
| HE230 − HE220 | Fan energy (kWh) | **+203.06** | 186.29 – 216.78 | yes |

Machine-readable results are in `scoring.json`.

## What was run and how

- **Software:** Pinocchio-EP `1.0.0+b0c9cf1.dirty` (the `ashrae140-compliance` working tree), running unmodified EnergyPlus 26.1.0-6f2e40d102 (the same build as production).
- **Building part:** a Pinocchio project built with `product/idfProject.js`.
  - The zone is 8 × 6 × 2.7 m, suspended (floor boundary set to Outdoors).
  - The Table 10-1 constructions are Pinocchio materials and constructions (the Envelope model objects): `Material:NoMass`, so there is no capacitance. Walls and floor are R 100; the roof is R 0.14.
  - Thermal and solar absorptance are set to the smallest legal values (1E-5 and 0).
  - The thermostat uses Pinocchio's Heating/Cooling Setpoint schedules: heat below 20 °C, no cooling. HE220/230 set back to 15 °C from 23:00 to 06:00.
  - Explicit zero loads replace Pinocchio's automatic people, lights, equipment and infiltration.
  - The run period is Jan 1 – Mar 31 with no DST and no holidays.
- **Outputs:** requested through the Project step's Output:Variable picker (`project.outputVariables`).
- **HVAC:** built as a Pinocchio Studio op log, the ops the Studio GUI records (`inputs/<case>/studio_ops.json`):
  - `AirLoopHVAC` + `AirLoopHVAC:UnitarySystem` (load control on the zone);
  - `Fan:OnOff`: 0.355 m³/s, efficiency 1, motor in the airstream, pressure rise = W / Q;
  - `Coil:Heating:Gas` → `Coil:Heating:Fuel`: capacity 10 kW (5 kW in HE230), efficiency 1.0 / 0.8, off-cycle parasitic fuel 0;
  - a CV no-reheat terminal.
- **Part-load curve:** PLF = PLR / HIR(PLR) (10.2.1.10.4) is not a polynomial. It is an `OS:Table:Lookup` built with Studio ops: PLR 0–1 in steps of 0.01, linear interpolation.
- **Circulating fan:** continuous in HE150/170 (operating-mode schedule = 1) and cycling with the burner in HE160 and HE210–230 (= 0). Cycling run time = PLR / PLF.
- **Draft fan (50 W, HE170–HE230):** the coil's *On Cycle Parasitic Electric Load*. Its electricity is reported and multiplied by the burner run-time fraction, and none of its heat enters the air. This is exactly the 10.2.2.7 c requirement.
- **IDF:** `exportIdf(project)`, the text the GUI's Run and Export send. It is stored in `inputs/<case>/in.idf`, and outputs are in `outputs/<case>/`.
- **Weather:** the official `HE100W/HE120W/HE130W/HE140W/HE210W.WY2` (Std 140-2023 accompanying files). They are converted to EPW by `harness/weather.mjs` (`wy2ToEpw`, the Table A1-18 column layout); the EPWs are in `weather/`.
  - The four synthetic files carry one repeated hour label each (for example HE100W 01/02 hour 15 appears twice and hour 16 is missing). The records are used in file order.
  - A generator for the text-specified files (`SYNTH`) is kept as a cross-check.
  - The Ottawa CWEC substitute was not needed.

## Results (Jan 1 – Mar 31)

Columns:
- **Load** is the total furnace load in GJ, at the coil, without fan heat.
- **Input** is the total furnace input in GJ (fuel HHV energy).
- **Fan** is the energy of both fans in kWh.
- **Examples** are the informative Annex B16.6 example results (analytical/QAS, and the min–max of ESP-r/HOT3000, EnergyPlus 1.0 and DOE-2.1E). They are shown for context only; they are not acceptance criteria.

| Case | Load | Analytical | Examples | Input | Analytical | Examples | Fuel (m³/s) | Fan | Analytical (examples) |
|---|---|---|---|---|---|---|---|---|---|
| HE100 | 77.760 | 77.74 | 77.75–77.94 | 77.756 | 77.713 | 77.713–78.417 | 2.631E-4 | – | – |
| HE110 | 77.760 | 77.74 | 77.75–77.94 | 97.195 | 97.216 | 96.920–98.020 | 3.289E-4 | – | – |
| HE120 | 31.269 | 31.10 | 31.10–31.25 | 38.491 | 38.266 | 38.266–38.556 | 1.303E-4 | – | – |
| HE130 | 0.009 | 0.00 | 0.00–0.15 | 0.015 | 0.000 | 0.000–0.142 | 5.1E-8 | – | – |
| HE140 | 31.269 | 31.10 | 31.10–31.26 | 39.079 | 39.004 | 38.764–39.004 | 1.323E-4 | – | – |
| HE150 | 29.765 | 29.65 | 29.57–29.88 | 37.157 | 37.025 | 36.820–37.231 | 1.258E-4 | 432.00 | 432.00 (432.0–433.3) |
| HE160 | 30.653 | 31.10 | 30.46–31.26 | 38.296 | 38.088 | 37.962–38.118 | 1.296E-4 | 171.16 | 172.76 (170.2–172.4) |
| HE170 | 29.765 | 29.65 | 29.57–29.88 | 37.157 | 37.025 | 36.820–37.231 | 1.258E-4 | 473.29 | 473.18 (473.1–473.4) |
| HE210 | 42.265 | – | 41.36–42.06 | 52.693 | – | 50.528–52.371 | 1.783E-4 | 293.87 | – (281.6–298.9) |
| HE220 | 40.033 | – | 39.41–39.87 | 49.833 | – | 47.869–49.466 | 1.686E-4 | 278.19 | – (268.3–281.2) |
| HE230 | 34.587 | – | 34.32–34.59 | 43.305 | – | 41.368–43.219 | 1.466E-4 | 481.25 | – (431.4–478.4) |

Zone temperature (mean / max / min, °C):

| Case | Pinocchio-EP | Examples |
|---|---|---|
| HE210 | 20.00 / 20.00 / 20.00 | mean 19.98–20.01 |
| HE220 | 18.54 / 20.00 / 15.00 | mean 18.53–18.75 |
| HE230 | 15.40 / 20.00 / 1.87 | mean 15.17–15.64, min 1.45–4.48 |

## Deviations and sensitivity (factual)

- **The residual UA is 1.03 W/K higher than specified.**
  - EnergyPlus' smallest user convection coefficient is 0.1 W/m²K, against the ≈ 0 specified for walls and floor.
  - The total UA is therefore 201.0 W/K rather than 199.95, so the HE120/HE140 loads are +0.5 % over the analytical 31.10 GJ (31.27 GJ). The effect carries into every case.
- **Timestep.** Pinocchio's default of 4 per hour is used. At 1 per hour (`results_ts1.json`) all six range values move by less than 0.3 % (HE230 input 43.04 GJ, fans 478.3 kWh) and all stay in range.
- **A cubic PLF fit would not work.** A Studio user without the table would fit a `Curve:Cubic`; the least-squares cubic over PLR 0.01–1 is in `results_cubic.json`. That cubic gives PLF(1) = 1.034 and an HE100 input 3.3 % low (75.21 GJ), so the Table:Lookup is used. Every range case still passes with the cubic.
- **Sizing periods run first.** A Studio-HVAC export always turns on design-day sizing on the weather file's extreme periods, so the converted EPWs carry two extreme weeks in the header. The first rows of `eplusout.csv` are sizing periods; the harness reads the last 2160 hourly rows (checked: 01/01 01:00 – 03/31 24:00).
- **No passthrough objects.** Every input reaches the IDF through a Pinocchio GUI path or helper:
  - the zone, constructions and materials (the Envelope model objects);
  - the constant convection coefficients through the Envelope step's *convection by surface type* helper (`lib/convection.setTypeCoefficient`: roof 20 W/m²K, walls and floor 0.1 W/m²K, inside and outside);
  - the setpoint schedules, the zero loads, the run period, and the Output:Variable picker;
  - all HVAC, including the part-load table, as Studio ops.

## App fix made for this section

Each fix is documented in `../01_software/WP-E2.md`:
1. **Studio `call` op.** It can now run `addIndependentVariable` / `addOutputValue` / `addValue`, so a Table:Lookup curve (independent-variable list + values) can be built from Studio ops (`src/lib/osstd/ops/index.js`).
2. **Studio inspector:** picking a heating or cooling coil for an `AirLoopHVAC:UnitarySystem` left the flow-rate method at "None". EnergyPlus treats that as fatal: "Input for Heating Supply Air Flow Rate Method = None". The inspector now also sets the method to SupplyAirFlowRate / Autosize, as OpenStudio's `setHeatingCoil` does (`src/lib/osstd/viewApi.js`).

## Files
- `scoring.json`: A3-8 checks, all results, sensitivity runs.
- `Std140_HE_Output.xls`: the official output form, filled.
- `Std140_HE_Output_equivalent.xlsx`: the same cells, plus range checks and the informative comparison.
- `S140outNotes_10.TXT`: Annex A2 blocks A–G.
- `results*.json`: raw harness output.
- `examples_informative.json`
- `inputs/`, `outputs/`, `weather/`
- `harness/`: copies of `tools/std140/he/*`. Run them from the repo:
  ```
  node tools/std140/he/run.mjs
  HE_VARIANT=ts1 node tools/std140/he/run.mjs
  HE_VARIANT=cubic node tools/std140/he/run.mjs
  python tools/std140/he/report.py
  node --test tools/std140/he/he.test.mjs
  ```
