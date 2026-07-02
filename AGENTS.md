# AGENTS.md

## Cursor Cloud specific instructions

This repo is `dvm-dos-tem` (`dvmdostem`): a C++ bio-geo-chemical ecosystem model
(the primary product) plus a large set of Python pre/post-processing and
calibration tools under `scripts/`, `calibration/`, and `mads_calibration/`.

Upstream, the intended dev environment is the Docker `dvmdostem-dev` image
(Ubuntu jammy + pyenv Python 3.8, see `Dockerfile` / `docker-compose.yml` /
README). In Cursor Cloud we instead build and run **natively** on the VM's
Ubuntu 24.04 / g++ 13 / system Python 3.12. The C++ model (the only thing the
GitHub Actions CI actually exercises, see `.github/workflows/checkout_build_run.yml`)
builds and runs cleanly this way. Some Python doctests are coupled to the
Docker/Python-3.8 setup and do not all pass natively (details below).

### Building the C++ model (the app)

The build uses the top-level `Makefile`. Two env vars must be set so the
compiler finds the `jsoncpp` and `mpi.h` headers on Ubuntu 24.04:

```bash
export SITE_SPECIFIC_INCLUDES="-I/usr/include/jsoncpp -I/usr/lib/x86_64-linux-gnu/openmpi/include"
export SITE_SPECIFIC_LIBS="-I/usr/lib"
make -j4          # produces ./dvmdostem
```

Non-obvious gotchas:
- The sources `#include <mpi.h>` and `#include <netcdf_par.h>` **unconditionally**
  (even though `USEMPI=false`). On jammy those came with `libnetcdf-dev`; on
  noble they don't. This env provides them via `libnetcdf-mpi-dev` (installed)
  plus a symlink `/usr/include/netcdf_par.h -> /usr/lib/x86_64-linux-gnu/netcdf/mpi/include/netcdf_par.h`
  and the openmpi include path above. `mpi.h` itself is pulled in by
  `libboost-all-dev` (openmpi) at `/usr/lib/x86_64-linux-gnu/openmpi/include`.
- Do NOT add the MPI netcdf include dir directly to `SITE_SPECIFIC_INCLUDES`
  (it would shadow the serial `netcdf.h`). Only the single `netcdf_par.h`
  symlink is used, so linking stays against serial `-lnetcdf`.
- The `Makefile`'s `-Werror` acts as the effective C++ "lint" — a clean `make`
  means no warnings.
- After pulling new C++ source, rebuild manually with the `make` command above
  (the build is intentionally not part of the startup update script).

### Running the model (hello world)

Scripts assume the repo is at `/work`; a symlink `/work -> /workspace` exists so
the doc/tooling paths (`/work/dvmdostem`, etc.) resolve. Typical run:

```bash
# 1. bootstrap a run directory from the shipped demo data
python scripts/util/setup_working_directory.py \
  --input-data-path demo-data/cru-ts40_ar5_rcp85_ncar-ccsm4_toolik_field_station_10x10 \
  /data/workflows/sample_run

# 2. run the model from inside that directory (config paths are relative)
cd /data/workflows/sample_run
/work/dvmdostem -l monitor -p 100 -e 200 -s 50 -t 115 -n 85

# 3. visualize an output variable
MPLBACKEND=Agg python /work/scripts/viewers/plot_output_var.py \
  --yx 0 0 --file output/GPP_yearly_tr.nc     # writes GPP_plot_output_var.png
```

The demo run mask only enables ~2 pixels, so most cells print "Skipping cell".
Output netCDF files land in `<workdir>/output/` (e.g. `GPP_yearly_tr.nc`).

### Python tooling / tests

Python deps are installed into the system interpreter (the pinned versions in
`requirements_general_dev.txt` are for Python 3.8 and won't build on 3.12, so
compatible current versions are installed instead). `python` is aliased to
`python3` via `python-is-python3`.

Doctests are the test suite (there is no configured Python linter). Run them
from the repo root with the tooling on `PYTHONPATH` (see
`docs_src/sphinx/source/software_development_info.rst`):

```bash
export PYTHONPATH="/work:/work/scripts:/work/scripts/util:/work/calibration"
python -m doctest scripts/tests/doctests/doctests_param_util.md   # silent = pass
```

Known-non-passing natively (pre-existing, NOT caused by env setup):
- `doctests_setup_working_directory.md` – expects Python 3.8 `argparse.Namespace`
  repr ordering and a `testing-data/` fixture that isn't in the repo.
- `doctests_runmask_util.md`, `doctests_Sensitivity*.md` – depend on the exact
  `netCDF4`/`pandas` versions pinned for Python 3.8 (output-format differences).
- `doctests_MadsTEMDriver*.rst` – require Julia + MADS (the separate
  `dvmdostem-autocal` image); not installed here.
`doctests_outspec_utils.md`, `doctests_param_util.md`, `doctests_qcal.md`, and
`doctests_Sensitivity_bounds.md` pass natively.
