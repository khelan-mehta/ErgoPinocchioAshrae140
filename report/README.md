# IEEE-format report: Pinocchio-EP and ASHRAE Standard 140-2023

- `main.tex` uses the IEEEtran conference class, two-column, with the ERGO logo in the title block.
- Compile with `pdflatex main.tex` run twice, or upload this folder to Overleaf. IEEEtran and subfig ship with TeX Live, MiKTeX and Overleaf.
- `figures/` holds the ERGO logo, the results charts, and `ui/` with 12 screenshots taken from a local run of the tested build.
  - `CAPTIONS.md` gives each screenshot a factual caption and the Standard 140 clause it serves.
  - `ui_version_stamp.png` is not used: it came from the local dev build, which shows `.dirty`. The released stamp is `1.0.0+b7b2379`.
- `harness/` holds the screenshot scripts; run `bash start-stack.sh` first, then `capture*.mjs`.
