# Pinocchio-EP weather converter (WP-C): TMY3 / TMY2 / TMY / WYEC2 → EPW

Pure ES modules, no dependencies. The same code runs in the browser and on the server.

| Where | Path |
|---|---|
| Frontend (canonical) | `pinocchio-ep/src/lib/weather/` |
| Backend byte-twin | `pinocchio-ep-api/src/wxconvert/` (test `test/wxconvert.test.js` fails if the two differ) |

## API

```js
import { convertToEpw, detectWeatherFormat, epwToTmy3, parseEpw, writeEpw } from ".../lib/weather/index.js";

detectWeatherFormat(fileName, text)          // "epw" | "tmy3" | "tmy2" | "tmy" | "wyec2" | null  (content-based)
convertToEpw(text, { fileName, format?, site?, startDayOfWeek = "Sunday", groundTemps = true })
  // → { epw, format, location, records, warnings, provenance, timeShift? (TMY only), filled }
epwToTmy3(epwText)                           // EPW → TMY3 CSV (round trip / programs that need TMY3)
parseEpw(text) / writeEpw({ location, records, ground, comments, startDayOfWeek })
```

Each format also has its own module:

| Module | Functions |
|---|---|
| `tmy3.js` | `parseTmy3`, `tmy3ToEpw` |
| `tmy2.js` | `parseTmy2`, `tmy2ToEpw` |
| `tmy1.js` | `parseTmy1`, `tmy1ToEpw` (solar time) |
| `wyec2.js` | `parseWyec2`, `wyec2ToEpw` |
| `epw.js` | `horizontalIR`, `rhFrom`, `dewPointFrom`, `satPress`, `stdPressure`, `groundTempsFromMonthly`, `EPW_FIELDS`, `EPW_MISSING` |
| `solar.js` | `equationOfTime`, `solarMinusStandardMin`, `cosZenith*`, `hourlyExtraterrestrial` |
| `stations.js` | `STD140_STATIONS`, `stationFor(wban)` |

TMY and WYEC2 files carry no site header. The converter then needs `site: { lat, lon (east +), tz (h), elevation (m) }`, or a station from `stations.js`. That table holds the Std-140 stations with the values from Tables A1-5/6, A1-9, A1-13 and A1-14: Miami 12839, Ottawa 04772, Colorado Springs 93037, Las Vegas 23169.

### CLI

```
node ashrae140/results/06_weather_drivers/converter/harness/convert.mjs <file> [out.epw] [--lat 38.8 --lon -104.7 --tz -7 --elev 1873]
node ashrae140/results/06_weather_drivers/converter/harness/validate.mjs     # the validation below; writes validation.json + epw/
```

### In the product

- **Server.** `POST /api/weather {name, data_base64, site?}` accepts `.epw .csv(TMY3) .tmy3 .tm2 .tmy .wy2`.
  - Non-EPW files are converted on the server.
  - It stores `wx/<uuid>/<stem>.epw`, the original file, and `provenance.json`. The provenance records the source name and format, converter version, SHA-256 of the source and of the EPW, record count, site, warnings and, for TMY, the time-shift note.
  - The response returns `format`, `converted`, `provenance` and `warnings`.
- **Frontend.** `product/WeatherFileUpload.jsx` handles the upload. It detects the format in the browser and asks for the site of a header-less file from a non-Std-140 station.
  - It records `project.weather.source`.
  - Mounting it in `ProjectStep.jsx` was requested from WP-A (see COORDINATION).

## Time conventions (Std 140 5.1.1.1, A1.5–A1.10)

| Format | Stamp | Solar / illuminance | Met values | Converter |
|---|---|---|---|---|
| EPW | LST, hour 1 = 00–01 | sum over the 60 min ending at the stamp | at the stamp | — |
| TMY3 (.csv) | LST, `01:00`…`24:00` | same as EPW | same as EPW | **no shift** |
| TMY2 (.tm2) | LST, hh 1…24 | same as EPW | same as EPW | **no shift** |
| WYEC2 (.wy2) | LST, hh 1…24 | integrated over the hour (kJ/m²) | end of hour | **no shift**; kJ → Wh ÷ 3.6 |
| TMY (.tmy, TD-9734) | **solar time**, end of the solar hour | kJ/m² over the solar hour | TD-1440 obs at the LST hour in cols 71-72 | **shifted** (below) |

The TMY solar time is converted to standard time as follows:

1. **Equation of time.** Solar time = LST + 4·(L_st − L_loc) + E, with E from Spencer: `E = 229.2[0.000075 + 0.001868 cos B − 0.032077 sin B − 0.014615 cos 2B − 0.04089 sin 2B]`, using **B = (n−1)·360/365**.
   - Std 140-2023 A1.5 prints `B = 360(n − 81)/365` with these coefficients. That is a typo: it gives E(1 Jan) = +13.9 min, while the true value is about −3 min.
   - The standard's own files confirm this. The LST column of CE100.TMY (Miami) agrees with the Spencer form to a median of **−0.2 min**.
2. **Tape LST check.** The tape's LST column (cols 16-19) is compared with the computed LST.
   - Colorad.TMY and Lasvega.TMY are **−30.0 / −30.1 min** off. These are the "add 30 minutes" stations of Table A1-15 field 004: their tape carries the hour midpoint.
   - The converter reports this and always uses the computed LST.
3. **Radiation.** Each solar-hour value is spread minute by minute over its LST interval and summed into LST hours.
   - GHI, ETR and DHI are weighted by max(cos θz, 0). DNI is spread uniformly over the minutes when the sun is up.
   - Energy is conserved (CE100.TMY: GHI 376 188 → 376 185 Wh/m², DNI 374 698 → 374 703; the difference is rounding), and nothing lands after sunset.
4. **Diffuse.** TMY tapes carry no diffuse (field 103, all 99999). It is derived as DHI = GHI − DNI·⟨cos θz⟩ over the solar hour, clipped to [0, GHI].
   - GHI is field 108, the "standard year corrected" value that the table recommends.
5. **Meteorological values.** Each value is an observation at the LST hour of field 201 and is placed on that EPW hour. EPW hours without an observation are interpolated linearly; wind direction is interpolated as a vector. Colorad.TMY has 2 duplicate observations.

**Independent check.** CE100.TMY (solar time, 1962) and CE100A.TM2 (LST) are different data sets of the same Miami station. After conversion, their hourly GHI correlates best at **zero lag**: r = 0.936, against 0.897 at −1 h and 0.904 at +1 h. Sunrise and sunset fall in the same hours.

## EPW field mapping

| EPW field | TMY3 col → | TMY2 cols → | TMY cols → | WYEC2 cols → |
|---|---|---|---|---|
| Dry bulb °C | 32 | 68-71 ÷10 | 104-107 ÷10 | 90-93 ÷10 |
| Dew point °C | 35 | 74-77 ÷10 | 108-111 ÷10 | 95-98 ÷10 |
| RH % | 38 | 80-82 | derived | derived |
| Station pressure Pa | 41 mbar ×100 | 85-88 mbar ×100 | 99-103 kPa·100 ×10 | 84-88 ×10 |
| ETR / ETRN Wh/m² | 3 / 4 | 10-13 / 14-17 | 20-23 kJ ÷3.6 / – | 15-18 kJ ÷3.6 / – |
| Horizontal IR Wh/m² | derived* | derived* | derived* | derived* |
| GHI / DNI / DHI Wh/m² | 5 / 8 / 11 | 18-21 / 24-27 / 30-33 | 55-58 / 25-28 kJ ÷3.6 / derived | 19-22 / 25-28 / 31-34 kJ ÷3.6 |
| GH / DN / DH illum lx | 14 / 17 / 20 | 36-39 / 42-45 / 48-51 ×100 | – | 37-40 / 42-45 / 47-50 ×100 |
| Zenith luminance Cd/m² | 23 | 54-57 ×10 | – | 52-55 ×100 |
| Wind dir ° / speed m/s | 44 / 47 | 91-93 / 96-98 ÷10 | 112-114 / 115-118 ÷10 | 100-102 / 104-107 ÷10 |
| Total / opaque sky tenths | 26 / 29 | 60-61 / 64-65 | 119-120 / 121-122 | 109-110 / 112-113 |
| Visibility km | 50 m ÷1000 | 101-104 ÷10 (7777 → missing) | 82-85 hm ÷10 | 70-73 ÷10 |
| Ceiling m | 53 (77777/88888 kept) | 107-111 | 73-76 dam ×10 (7777→77777) | 60-63 ×10 |
| Present weather obs / codes | 9 / missing | 114 / 115-123 | 9 / missing | 9 / missing |
| Precipitable water mm | 56 cm ×10 | 124-126 | – | – |
| AOD | 59 | 129-131 ÷1000 | – | – |
| Snow depth cm / days since | 0 / 88 (as the NREL EPWs) | 134-136 / 139-140 | snow indicator 123 → 0/1 cm | 115 → 0/1 cm |
| Albedo / liquid precip mm, h | 62 / 65, 66 | – | – | – |

\* Horizontal IR = ε·σ·T_db⁴, with the Clark–Allen sky emissivity `(0.787 + 0.764 ln(T_dp/273))·(1 + 0.0224N − 0.0035N² + 0.00028N³)`, N = opaque sky cover. This is the EnergyPlus Weather Converter method, and it reproduces WD100.epw (181 and 375 Wh/m² in the test rows).

Other fields:

- **RH** is derived with Hyland–Wexler saturation pressure.
- **Ground temperatures** in the header are computed with the Kusuda formula (α = 0.0557 m²/day, at 0.5 / 2 / 4 m) from the monthly mean dry bulb, and omitted for files shorter than a year.
  - The reference EPWs' ground rows are **not** derived from their own data (WD100–WD600 carry rows copied from other station files), so they cannot serve as a check.
  - EnergyPlus does not use EPW ground temperatures for building surfaces.
- **Missing values** (TMY3 −9900, TMY2/TMY/WYEC2 all-9s) are written as the EPW missing codes, and EnergyPlus substitutes for them.
- **The flags field** gets the standard NREL string `?9?9?9?9E0?9…`. EnergyPlus does not read it.

## Validation results (`validation.json`, `validation_log.txt`)

**1. WD100–WD600.tmy3 → EPW vs the normative EPW equivalents WD100–WD600.epw** (Std 140 Table A1-26), field by field over 8760 h, all time-aligned.

- **Exact match, every hour, all six files:** dry bulb, dew point, RH, pressure, GHI, DNI, DHI, wind direction and speed, total and opaque sky cover, ceiling, snow and days since snow. These are all the fields EnergyPlus reads for heat balance and solar.
- **ETRN** matches exactly. **ETR** differs in ≤ 0.8 % of hours, by at most 31 Wh/m²; EnergyPlus recomputes it.
- **Horizontal IR** is within ±1 Wh/m² (rounding) in WD100/200/400/600. WD300 and WD500 reach up to 20 and 39 Wh/m², because their reference EPWs carry the IR of the ISD/TMYx source file.
- **Illuminance and zenith luminance** differ.
  - The WD*.tmy3 files store illuminance about 100× too small (e.g. 984 lx at noon in June).
  - The reference EPWs recomputed illuminance (Perez) instead of reading those columns.
  - Our converter passes through what the TMY3 says. EnergyPlus daylighting computes its own sky luminance and does not use these EPW fields.
- **WD300 and WD500** also differ in precipitable water, AOD and albedo. The reference EPWs keep their TMYx values where the TMY3 has 0. EnergyPlus does not use these: ground reflectance comes from `Site:GroundReflectance` (see WP-F's Section 6 note).
- **Site header (a finding).** WD400.tmy3's header says 71.320 N / 156.620 W. The normative WD400.epw and the standard's text (6.2.4) say 71.286 N / 156.767 W. The converter uses the TMY3 header as published. For Section 6, WP-F should either use WD400.epw or override `site`.

**2. 725650TY.CSV (Section 7) → EPW.**

- **Against WD100.epw (same Denver data):** every heat-balance field is identical. The albedo differs because WD100 sets it to 0, which WD100.tmy3 does too.
- **Against the climate.onebuilding.org Denver TMY3 EPW (`ashrae140/work/wx/`):** dry bulb, pressure, GHI, DHI and wind are identical in every hour.
  - Dew point and RH differ in 111 h (1.3 %), and DNI in 219 h (2.5 %). The onebuilding file is the 2015 NREL re-release, with updated precipitation and some corrected values.
  - Horizontal IR differs: onebuilding uses a different IR model, about 18 Wh/m² mean.
- **Conclusion.** The official 725650TY.CSV, converted, is the correct Section-7 EPW. The onebuilding EPW used in earlier work (`work/wx`) is a close but **not identical** substitute.

**3. EPW → TMY3 → EPW round trip** (onebuilding Denver EPW): lossless for all 24 TMY3-carried fields over 8760 h. IR is derived, so it is excluded.

**4. All 27 other Std-140 weather files convert.** The EPWs are in `epw/`.

| Section | Files | Hours | Notes |
|---|---|---|---|
| 8 | GC30a, GC30b, GC40a, GC40b, GC65b, GC70b (TM2) | 8760 | GHI 0; DB 10 °C constant, or 2–18 °C for the harmonic files |
| 9 | CE100A, CE110A, CE165A, CE200A, CE300 (TM2) | 8760 | |
| 9 | CE100, CE110, CE165, CE200 (TMY) | 2160 | quarter-year, as published |
| 10 | HE100W–HE210W (WY2) | 8760 | **Defect in the files:** HE100W, HE120W, HE130W and HE140W stamp record 40 as "01/02 hour 15" a second time; it is hour 16. Without a fix, EnergyPlus stops with a Fatal error. The converter re-stamps that record from the hourly sequence and warns. HE140W also has one stray short line, which is ignored. |
| 11 | AE101–AE106 (TM2) | 8760 | |
| 12 | Colorad, Lasvega (TMY) | 8760 | |

Annual GHI / DNI / DHI in kWh/m² is listed in `validation.json`, e.g. Colorad 1842 / 2395 / 478 and Lasvega 2167 / 2751 / 493.

### Tests

- Backend: `pinocchio-ep-api/test/wxconvert.test.js`, **18/18**. It covers:
  - format records built from the published column tables;
  - the real WD files, every field EnergyPlus reads;
  - all 27 Std-140 files;
  - upload, provenance and the resolver;
  - re-stamping of an out-of-sequence record.
- Frontend: `pinocchio-ep/tools/schedule-file.test.mjs` (browser-side conversion and upload helpers), **7/7**.

**5. EnergyPlus 26.1 runs the converted files** (`harness/eplus_accept.mjs` → `eplus_accept.json`). The test runs 1ZoneUncontrolled.idf for a full year on nine EPWs: 725650TY, WD500, GC40a, CE300, Lasvega, Colorad, HE140W, HE210W, and CE100 (Jan–Mar). All nine end **Completed Successfully** with 0 severe errors. The only weather message is the usual "weather file location used rather than IDF Location".
