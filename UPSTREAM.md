# Upstream Provenance

| | Repository | Commit |
|---|---|---|
| Eurorack | https://github.com/pichenettes/eurorack | `f4cfa7697095a28d292bff758beb51deb49b5a86` |
| stmlib | https://github.com/pichenettes/stmlib | `e3bd7c9cc00e4364166f9905c0509b6ffd0535ec` |

stmlib is a Git submodule of the Eurorack repository; the stmlib commit is the
one Eurorack `f4cfa76` pins. Every module in this repository is imported from
these two commits, so all modules agree on one stmlib.

## Policy

- One curated fork for every Mutable Instruments port, with a single shared
  `stmlib/`.
- Import the exact dependency closure of each module, not the whole Eurorack
  repository. Adding a module adds the stmlib files it needs, from the same
  stmlib commit.
- Keep the original directory layout and copyright headers.
- The first commit of a module imports its files unchanged. Every later change
  is a separate commit, recorded in `PATCHES.md`.
- `UPSTREAM_FILES.txt` lists every imported file, so the closure can be
  re-checked against the upstream commits.
- Moving to a newer upstream commit moves every module and stmlib together.

## Modules

| Module | Imported | Notes |
|---|---|---|
| clouds | DSP, resources | From CloudsDSP (`github.com/BrianMndz/CloudsDSP`), which imported Eurorack `08460a6`. All its files are identical in `f4cfa76`, whose only change is the new `beads/` directory. |
