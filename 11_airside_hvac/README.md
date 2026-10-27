# Section 11 — Air-side HVAC equipment (AE101–AE445), run through Pinocchio-EP

**Verdict: PASS. 54 of the 54 Table A3-9 range cases are in range on the last-hour values (48 needed).**
- The 24-hour-median values also give 54/54.
- The ambient conditions are entered directly, as constant values (11.2.1.1.7), so no substitute weather is involved.

Every A3-9 check (range case, quantity, value, limits) is in `scoring.json` and in the `A3-9 ranges` sheet of `Std140_AE_Output_equivalent.xlsx`.

| System | Ranged checks | In range |
|---|---|---|
| SZ (AE201–AE245) | 9 | 9 |
| CV terminal reheat (AE301–AE345, incl. CV−VAV deltas) | 30 | 30 |
| VAV terminal reheat (AE401–AE445) | 15 | 15 |
| **Total** | **54** | **54** |

## What was run and how

- **Software:** Pinocchio-EP `1.0.0+b0c9cf1.dirty` running unmodified EnergyPlus 26.1.0-6f2e40d102. The harness is `tools/std140/ae/`; copies are in `harness/`.

### Building part (Pinocchio project)

- **Zones:** one zone (FC/SZ) or two unattached zones (CV/VAV), 8 × 6 × 2.7 m. Every face uses the Adiabatic boundary override, with no-capacitance constructions. This is the adiabatic-zone method of 11.2.1.1.4.
- **Zone loads:**
  - Cooling sensible loads are Pinocchio interior-equipment loads with Fraction Radiant 0.
  - Latent loads are interior-equipment loads with Fraction Latent 1, so they are applied at the zone temperature.
  - The heating cases' negative sensible gain (−2931 W / −2345 W) is an `OtherEquipment` passthrough, because Pinocchio loads cannot be negative.
- **Thermostats:**
  - Zone 1 uses Pinocchio's Heating/Cooling Setpoint schedules, both equal to the specified zone temperature (no deadband).
  - Zone 2 uses the Studio's zone-thermostat edit (`Zone 2 Tstat`).
- **Outputs:** requested through the Project step's Output:Variable picker.

### HVAC (Pinocchio Studio op log, `inputs/<case>/studio_ops.json`)

- **Air loop:** AirLoopHVAC with the following on its supply branch, in order (Figures 11-1/3/4/5):
  1. return fan (SZ, CV, VAV);
  2. outdoor-air system;
  3. electric heating coil (FC/SZ) or preheat coil (CV/VAV);
  4. chilled-water cooling coil;
  5. supply fan.
- **Outdoor air:** `Controller:OutdoorAir`. The minimum OA flow equals the zone exhaust flow, so OA mass equals exhaust mass, since both volumetric inputs use EnergyPlus' standard density. The maximum OA flow is the supply flow, with fixed minimum and no lockout. The economizer is NoEconomizer, DifferentialDryBulb (x26) or DifferentialEnthalpy (x45).
- **Fans:**
  - Supply fan: 498 Pa, total efficiency 0.7, motor efficiency 1, motor outside the airstream. Fan heat to the air is therefore the shaft power Hᵢ = Hₒ/0.7, which is what the QAS uses: its supply-fan ΔT is 0.58–0.61 K.
  - VAV supply and return fans: `Fan:VariableVolume` with power ∝ flow³, i.e. pressure ∝ flow².
  - Zone exhaust: `Fan:ZoneExhaust`, 0 Pa.
- **Cooling coil:** `Coil:Cooling:Water`, oversized (design approach 2 K at 0.01 m³/s, inlet water 2 °C) on a district-cooling loop. This is the "enlarged hydronic coil" of 11.2.1.1.3 note 2. The wet-coil leaving RH is 99.3–100 %, so BF ≈ 0.
- **Control:**
  - FC/SZ: `SetpointManager:SingleZone:Reheat` on the supply outlet. The coils modulate to the zone load.
  - CV/VAV: `SetpointManager:Scheduled` holds 12.78 °C at the supply-fan outlet and 7.22 °C at the preheat-coil outlet. The cooling-coil and mixed-air set points are set by the Studio's automatic `SetpointManager:MixedAir` (supply set point − fan rise).
- **Terminals:**
  - FC/SZ: CV no-reheat.
  - CV: `AirTerminal:SingleDuct:ConstantVolume:Reheat` at 283.17 / 330.36 L/s.
  - VAV: `AirTerminal:SingleDuct:VAV:Reheat`. The minimum is a FixedFlowRate equal to the zone exhaust, with Normal damper action (reheat only at minimum flow).
  - All reheat coils are electric.

### Ambient, run length and reported values

- **Ambient:** constant EPWs written from Tables 11-4, 11-5, 11-7, 11-14 and 11-16 (`weather/`): exact dry-bulb and dew point, 101 325 Pa, 4.3 m/s, no solar.
- **Run length:** 5 days of constant weather after warm-up.
- **Reported values:** last hour. Node psychrometrics come from EnergyPlus node T, W, h and m. Specific volume (dry-air basis) is computed from node T, W and P (Block B).

## Results, last hour (kWh/h)

QAS is the quasi-analytical solution from the informative Annex B16.7 example results, for context. For heating cases it is the heating or preheat load; for cooling cases it is the total cooling-coil load.

| Case | QH | QHpre | QCtot | QCsens | QClat | Reheat z1 / z2 | RHcco % | Tz (°C) | Steady | QAS |
|---|---|---|---|---|---|---|---|---|---|---|
| AE101 | 8.454 | – | 0.000 | 0.000 | 0.000 | – | 6.5 | 21.122 | yes | 8.420 |
| AE103 | 0.000 | – | 0.763 | 0.763 | 0.000 | – | 45.6 | 23.328 | yes | 0.783 |
| AE104 | 0.000 | – | 5.683 | 3.499 | 2.184 | – | 99.3 | 23.878 | yes | 5.551 |
| AE201 | 8.387 | – | 0.000 | 0.000 | 0.000 | – | 6.5 | 21.122 | yes | 8.360 |
| AE203 | 0.000 | – | 0.830 | 0.830 | 0.000 | – | 45.6 | 23.328 | yes | 0.856 |
| AE204 | 0.000 | – | 5.750 | 3.566 | 2.184 | – | 99.3 | 23.878 | yes | 5.628 |
| AE205 | 0.000 | – | 1.913 | 1.913 | 0.000 | – | 44.1 | 23.328 | yes | 1.916 |
| AE206 | 0.000 | – | 2.720 | 1.697 | 1.023 | – | 99.5 | 23.328 | yes | 2.677 |
| AE226 | 0.000 | – | 3.429 | 1.556 | 1.873 | – | 99.7 | 23.328 | yes | 3.362 |
| AE245 | 0.000 | – | 2.208 | 2.208 | 0.000 | – | 34.0 | 23.328 | yes | 2.208 |
| AE301 | – | 3.550 | 0.000 | 0.000 | 0.000 | 7.509 / 8.136 | 24.1 | 21.139 / 22.261 | yes | 2.947 |
| AE303 | – | 0.000 | 6.517 | 6.517 | 0.000 | 2.196 / 2.377 | 67.3 | 23.341 / 24.455 | yes | 6.735 |
| AE304 | – | 0.000 | 16.793 | 10.177 | 6.616 | 0.942 / 1.456 | 99.3 | 23.892 / 25.007 | yes | 16.578 |
| AE305 | – | 0.000 | 9.221 | 9.221 | 0.000 | 2.194 / 2.376 | 65.0 | 23.341 / 24.455 | yes | 9.383 |
| AE306 | – | 0.000 | 14.513 | 8.765 | 5.747 | 2.215 / 2.402 | 99.4 | 23.341 / 24.455 | yes | 14.364 |
| AE326 | – | 0.000 | 20.769 | 8.248 | 12.521 | 2.215 / 2.403 | 100.0 | 23.341 / 24.455 | yes | 20.763 |
| AE345 | – | 0.000 | 9.510 | 9.510 | 0.000 | 2.186 / 2.365 | 51.0 | 23.341 / 24.455 | yes | 9.703 |
| AE401 | – | 10.348 | 0.000 | 0.000 | 0.000 | 4.502 / 4.900 | 4.1 | 21.153 / 22.263 | yes | 10.260 |
| AE403 | – | 0.000 | 1.401 | 1.401 | 0.000 | 0 / 0 | 55.0 | 23.329 / 24.436 | no (≤1.1 %) | 1.458 |
| AE404 | – | 0.000 | 13.664 | 7.408 | 6.256 | 0 / 0 | 99.6 | 23.885 / 25.131 | no (≤1.0 %) | 13.323 |
| AE405 | – | 0.000 | 4.106 | 4.106 | 0.000 | 0 / 0 | 52.8 | 23.327 / 24.434 | no (≤0.3 %) | 4.102 |
| AE406 | – | 0.000 | 8.429 | 3.595 | 4.834 | 0 / 0 | 99.9 | 23.328 / 24.435 | no (≤0.3 %) | 8.223 |
| AE426 | – | 0.000 | 8.949 | 3.548 | 5.401 | 0 / 0 | 99.9 | 23.327 / 24.434 | yes | 8.830 |
| AE445 | – | 0.000 | 4.155 | 4.155 | 0.000 | 0 / 0 | 49.5 | 23.327 / 24.437 | no (≤1.6 %) | 4.157 |

The zone loads are the specified values to within 1 W in every case (`results.json` › `zones`). The node outputs (oa, ma, hco/pco, cco, sa, z1s/z2s, z1/z2, rfi/rfo or ra) are in `Std140_AE_Output.xlsx` (the official form, filled) and in the `Nodes` sheet of the equivalent workbook.

## Deviations, anomalies and limits (factual)

1. **VAV steady state.** In AE403–AE406 and AE445, EnergyPlus' VAV terminal and water-coil controller iterations leave hour-to-hour noise of 0.3–1.6 % that never fully settles, so "last hour = previous hour" is not met exactly.
   - The last hour is still reported, as the Standard asks.
   - The 24-hour median and spread of each coil load are in `scoring.json`; the medians also score 54/54.
   - Tighter convergence (temperature/loads tolerance 0.001, 50 HVAC iterations, 6 or 12 timesteps per hour, a controller tolerance of 1E-2 to 1E-4) did not remove it. These were tried and are not used.
2. **The return fan carries the supply flow.**
   - EnergyPlus models a fan upstream of the outdoor-air system as carrying the branch (= supply) flow. The return-stream node therefore shows the supply mass flow, and the mixer relieves the surplus, so the relief flow equals the OA flow.
   - Its heat is computed at that flow, and only the (supply − exhaust)/supply share reaches the mixed air. With the constant 249 Pa used for SZ/CV, the heat into the mixed air equals the Standard's Q_return × 249 Pa / 0.7. The return-fan temperature rise (rfi→rfo) equals the specified one.
   - For VAV, pressure ∝ (supply flow fraction)² approximates the Standard's (return flow fraction)².
   - A maximum flow equal to the design return flow would cap the supply flow, as found in a first run where SZ supply was capped at 0.227 kg/s. The maximum is therefore set to the supply flow.
   - The reported rfi/rfo mass flows are EnergyPlus' node values (the supply flow), not the physical return flow (supply − exhaust).
3. **Volumetric flows use standard air.** EnergyPlus converts volumetric flow inputs to mass with standard air density, not at fan-inlet or zone conditions. 11.2.x.1.1.4 permits these predetermined assumptions.
   - The largest consequence is AE301's preheat: 3.550 kWh/h against QAS 2.947, which is +20 %. The range is 2.80–4.20, and the informative EnergyPlus example gives 3.575. The OA (exhaust) mass is 1.7 % above the QAS and the supply mass is 3.6 % below it.
   - All other coil loads are within 5 % of the QAS.
4. **Zone temperature offset.**
   - The zone air temperature is 0.005–0.04 K above set point: AE301 zone 2 is 22.261 against 22.222.
   - Set-point adjustment to remove the offset is permitted (11.2.1.1.6.3) but was not applied.
   - The effect on coil loads is below 0.02 kWh/h.
5. **FC system modelled as a single-zone air loop.** The FC system is a single-zone air loop with the fan downstream of the coils and no return fan, which 11.2.1.1 note 1 allows (most similar model).
   - The Studio's `ZoneHVAC:FourPipeFanCoil` puts its fan upstream of the coils in EnergyPlus, so it would contradict Figure 11-1.
   - The FC cases have no ranges.
6. **Sizing runs first.** Studio-HVAC exports always run design-day sizing on the weather file's extreme periods, so the constant EPWs carry two extreme weeks. Everything that matters is hard-sized.
7. **Passthrough:** `OtherEquipment` for the negative sensible gains of AE101/201/301/401. No other passthrough is used; all HVAC is Studio ops.

## App fixes and coordination

See `../01_software/WP-E2.md`. The Section 10 fixes (Table:Lookup call ops, the unitary-system coil-method inspector fix) are Studio-wide. Section 11 needed no further code change: every element is expressible with existing Studio ops, including the return fan, economizers, VAV minimums and the zone exhaust fan.

## Files
- `scoring.json`: the 54 A3-9 checks, last hour and 24-h median; not-steady flags; all results.
- `Std140_AE_Output.xlsx`: the official form, filled (summary and detailed node outputs).
- `Std140_AE_Output_equivalent.xlsx`: the summary, nodes, A3-9 checks, and the informative QAS / EnergyPlus-example comparison.
- `S140outNotes_11.TXT`
- `results.json`, `examples_informative.json`
- `inputs/`, `outputs/`, `weather/`, `harness/`
- Rerun:
  ```
  node tools/std140/ae/run.mjs
  python tools/std140/ae/report.py
  node --test tools/std140/ae/ae.test.mjs
  ```
