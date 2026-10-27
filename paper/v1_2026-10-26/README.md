# Pinocchio-EP vs ANSI/ASHRAE Standard 140-2023 (with Addenda a and b)

- `main.tex` is the paper. It is self-contained (tables inlined) and uses the figures in `figures/`.
- Compile it with any TeX distribution by running `pdflatex main.tex` twice, for longtable widths. Alternatively, upload the folder to Overleaf.
- `harness/` holds the scripts that produced every number:
  - `suite.mjs` builds the Section 7 cases through Pinocchio's `exportIdf`.
  - `roundtrip.mjs` runs the IDF-import route.
  - `score.py` scores against Annex A3 as revised by Addendum a.
  - `run_etna.py` and `via_pinocchio.mjs` run Section 13 (ETNA).
  - `figures.py` draws the figures.
- `data/` holds the raw scored results as JSON.

Script paths point at the original scratch directory, so adjust them before re-running.
