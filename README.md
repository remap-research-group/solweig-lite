# SOLWEIG-light

SOLWEIG-light runs SOLWEIG urban radiation and thermal-comfort workflows on
CPU using NumPy, SciPy, Numba and GDAL. It calculates mean radiant temperature
(TMRT), UTCI and optional radiation, shadow, SVF, WBGT, air-temperature and
wind outputs. The local meteorological-file/TIFF workflow needs no Torch, CUDA
or network services, and keeps the inputs, arguments and outputs of
[SOLWEIG-GPU](https://github.com/nvnsudharsan/SOLWEIG-GPU).

## Quick start

```sh
# 1. Environment for a local met file (ERA5/WRF/wind need more; see Installation)
conda create -n solweig-light --override-channels -c conda-forge \
    python=3.11 gdal=3.13 numpy=2.4 scipy=1.17 numba=0.67 pytz "timezonefinder=9"
conda activate solweig-light
git clone https://github.com/Synn-Arch/Solweig-Lite.git
cd Solweig-Lite
python -m pip install .
```

```python
# 2. Run one scene on 16 CPU threads
from solweig_light import RuntimeOptions, runtime_options, thermal_comfort

with runtime_options(RuntimeOptions(
    cpu_budget=16, workers=1, threads_per_worker=16, block_pixels=1024,
)):
    thermal_comfort(
        base_path="/absolute/path/to/scene",
        selected_date_str="2023-06-25",
        own_met_file="/absolute/path/to/scene/met.txt",
        landcover_filename="Landcover.tif",   # optional
        ERA_5_z0_find=False,                  # local met file, no ERA5 download
    )
```

Results appear in `/absolute/path/to/scene/output_folder/<tile>/`.

## Installation

Create a new conda-forge environment, separate from upstream SOLWEIG-GPU, and
install every compiled dependency (GDAL, rasterio, netCDF4, ...) in that one
`conda create` command. pip then adds only `solweig-light` itself. Most install
errors come from letting pip fetch GDAL-linked packages: those wheels bundle
their own GDAL, PROJ or netCDF libraries, which then conflict with conda's.

### 1. Choose an environment

| What you run | Extra | Environment |
| --- | --- | --- |
| Local met file (`own_met_file=...`) with `ERA_5_z0_find=False`, as in the Quick start | none | A |
| ERA5 or WRF NetCDF forcing (`use_own_met=False`, `data_source_type="ERA5"` or `"wrfout"`) | `forcing` | A with the forcing packages, or B |
| Wind coefficients from ERA5 roughness (`ERA_5_z0_find=True`, `build_wind_ext_coeff`) | `wind` | B |
| Static input construction (`build_inputs`) | `inputs` | B with the input packages |

`ERA_5_z0_find` defaults to `True` in Python. If the `wind` packages or the
ERA5 file are missing, the run continues without wind coefficients and prints
only `Could not find ERA-5 file with roughness length`, even when the cause is a
missing package. Pass `ERA_5_z0_find=False` unless you supply ERA5 roughness.

**A. Python 3.11, the tested configuration**

```sh
conda create -n solweig-light --override-channels -c conda-forge \
    python=3.11 gdal=3.13 numpy=2.4 scipy=1.17 numba=0.67 pytz "timezonefinder=9"
```

For ERA5/WRF forcing, add these to the same command:
`"xarray>=2026.7,<2027" "pandas>=3,<4" "netcdf4>=1.7,<2" "matplotlib-base>=3.11,<4" "shapely>=2.1,<3"`

**B. Python 3.12, all optional workflows**

conda-forge builds rasterio against GDAL 3.13 only for Python 3.12 and newer,
so the `wind` and `inputs` extras cannot be installed in environment A.

```sh
conda create -n solweig-light-full --override-channels -c conda-forge \
    python=3.12 gdal=3.13 numpy=2.4 scipy=1.17 numba=0.67 pytz "timezonefinder=9" \
    "xarray>=2026.7,<2027" "pandas>=3,<4" "netcdf4>=1.7,<2" "matplotlib-base>=3.11,<4" \
    "shapely>=2.1,<3" "rasterio>=1.5,<2"
```

For `build_inputs`, add these to the same command:
`"geopandas>=1.1,<2" "fiona>=1.10,<2" "pyproj>=3.7,<4" "geopy>=2.5,<3" "requests>=2.34,<3" "cftime>=1.6,<2" osmnx earthengine-api geemap`

Environment B was checked on macOS ARM64, where the reference-pipeline,
forcing and wind tests pass. The `build_inputs` packages were only checked to
resolve on conda-forge, and the measurements below used environment A on Linux.

### 2. Install solweig-light

```sh
conda activate solweig-light          # or solweig-light-full
git clone https://github.com/Synn-Arch/Solweig-Lite.git
cd Solweig-Lite
python -m pip install .               # environment B: '.[forcing,wind]'
```

Name the extras you installed (for example `'.[forcing]'` or
`'.[forcing,wind,inputs]'`) so pip checks their versions. pip should install
only `solweig-light`; `python -m pip install --dry-run '.[forcing,wind]'` shows
this in advance. If it wants to download GDAL, rasterio, netCDF4, NumPy or
another dependency, that package is missing from the conda environment or has
the wrong version. Add it with conda instead and rerun.

Activate the environment before running. Calling its `python` directly leaves
`PROJ_DATA` unset, and GDAL then fails CRS lookups with
`PROJ: proj_create_from_database: Open of .../share/proj failed`. Set
`PROJ_DATA=$CONDA_PREFIX/share/proj` if you cannot activate it.

### HPC clusters (for example PACE)

- Run `module purge` and load only the module that provides conda. GDAL, PROJ
  or Python modules can put their own libraries and `GDAL_DATA`/`PROJ_DATA`
  ahead of the environment's.
- If the solve takes very long, the cluster's conda probably uses the classic
  solver (conda before 23.10). Install
  [Miniforge](https://conda-forge.org/download/) in your own directory instead;
  it uses the faster libmamba solver.
- If your home quota is small, create the environment elsewhere with
  `-p /path/to/envs/solweig-light` in place of `-n solweig-light`, and activate
  it by that path.

### Other platforms

- **System GDAL:** install native GDAL 3.13.x first, then
  `python3.11 -m venv .venv && .venv/bin/python -m pip install .`
- **macOS ARM64:** a pinned environment is in
  [requirements/macos-arm64-py311.txt](requirements/macos-arm64-py311.txt).
- **Windows (conda-forge, untested):** see [Windows](#windows).

### Windows

Windows support is new and has not been validated against the reference
fixtures; Linux is the tested platform. Install
[Miniforge](https://conda-forge.org/download/), open a Miniforge Prompt, and
enter each command on one line (cmd does not accept the `\` continuations used
elsewhere in this README):

```bat
conda create -n solweig-light --override-channels -c conda-forge python=3.11 gdal=3.13 numpy=2.4 scipy=1.17 numba=0.67 pytz "timezonefinder=9"
conda activate solweig-light
python -m pip install https://github.com/Synn-Arch/Solweig-Lite/archive/refs/heads/master.zip
```

This is environment A. For ERA5/WRF forcing, wind or `build_inputs`, use the
packages of [environment B](#1-choose-an-environment) on one line instead;
conda-forge has the same Python 3.12 requirement for rasterio on Windows.

Install from the archive, not from `git clone` or `pip install git+https://...`.
`tests/` contains two WRF fixtures named `..._13:00:00`, and Windows cannot
create a file with `:` in its name, so a clone fails to check out. The archive
leaves `tests/` out; the package itself does not need it.

- Write paths as raw strings or with forward slashes:
  `base_path=r"C:\data\scene"` or `base_path="C:/data/scene"`.
- Put the run code under `if __name__ == "__main__":`. With `workers` above 1,
  SVF export starts spawn processes that re-import the calling script (this
  applies on every platform).
- If GDAL cannot find PROJ without an activated environment, set
  `PROJ_DATA=%CONDA_PREFIX%\Library\share\proj`.
- Name WRF files `wrfout_d0X_YYYY-MM-DD_HH_MM_SS`. The `HH:MM:SS` form WRF
  writes by default is not a valid Windows file name; the underscore and `HH`
  forms are accepted.

## Input data

Put aligned GeoTIFFs and a meteorological file in one scene directory:

| File | Meaning |
| --- | --- |
| `Building_DSM.tif` | Terrain plus buildings, in metres above the elevation datum. |
| `DEM.tif` | Bare-earth terrain, in metres above the same datum. |
| `Trees.tif` | Canopy height above ground, in metres; zero without trees. |
| `Landcover.tif` | Optional land cover, passed with `landcover_filename`. |
| `met.txt` | Local UMEP-format meteorological text file. |

Rasters must share dimensions, CRS and geotransform. Timesteps come from the met
file. `selected_date_str` (`YYYY-MM-DD`) sets the scene's UTC offset, including
daylight saving time, so use a date the met file covers. Pass the met file as an
absolute path: a relative path resolves against the working directory, not
`base_path`.

## Running

### Python API (multi-threaded)

Thread use is set with `RuntimeOptions` inside a `runtime_options(...)` block,
as in the Quick start. **Without it, a run uses one worker and one thread.**

| Option | Default | Meaning |
| --- | --- | --- |
| `cpu_budget` | `1` | CPU slots available to the run. Must not exceed the machine's CPUs. |
| `workers` | `1` | Upper bound on logical tiles processed in parallel (separate processes). |
| `threads_per_worker` | `1` | Native threads inside each tile worker; at most `cpu_budget`. |
| `block_pixels` | `128` | Internal buffer size. Changes speed and memory, not results. |
| `resident_lw_masks` | `True` | Reuses static longwave predicates across timesteps; `False` saves memory. |

A scene smaller than `tile_size` (default 3600 px) is one logical tile, so use
`workers=1` and spend the budget on `threads_per_worker`. The measurements below
use `cpu_budget=16, workers=1, threads_per_worker=16, block_pixels=1024`; other
thread counts have not been benchmarked. [Runtime](docs/runtime.md) covers
memory admission, caches and restart.

### Command line (single-threaded)

```sh
solweig-light --base_path /absolute/path/to/scene \
    --date 2023-06-25 --own_metfile /absolute/path/to/scene/met.txt \
    --landcover Landcover.tif --era5_z0_find False
```

The CLI has no thread options and always runs with the defaults (one worker,
one thread). Use the Python API to run on more cores.

### Outputs

Each logical tile writes to `<base_path>/output_folder/<tile>/`, for example
`output_folder/0_0/TMRT_0_0.tif` and `UTCI_0_0.tif`, with one timestamped band
per timestep. UTCI is always written and TMRT by default; `save_kup`,
`save_kdown`, `save_lup`, `save_ldown`, `save_shadow`, `save_wbgt`, `save_ta`
and `save_wind` add fields.

Intermediate files go to `<base_path>/processed_inputs/`. By default this
includes upstream-compatible SVF exports (`processed_inputs/SVF/*.tif|zip|npz`).
`export_svf=False` (CLI: `--export_svf False`) skips writing them without
changing the model computation. `save_svf=True` requests a single-band
`SVF_<tile>.tif` only when no complete legacy SVF export set already exists.

### Re-running after editing inputs

Re-run in the same `base_path` after changing a raster, for example
`Trees.tif`. Sky-view geometry is cached under
`processed_inputs/.solweig-light/cache` by a hash of the raster contents:

- Unchanged inputs reuse the cached geometry.
- Changed inputs miss the cache and are recomputed automatically. The whole
  geometry is recomputed, not just the edited area.

TMRT/UTCI from such a re-run were bitwise identical to a fresh run with the
edited trees. However, existing SVF exports in `processed_inputs/SVF/` are not
rewritten and still describe the old inputs. Do not analyse them after an edit;
delete that folder or use `export_svf=False` for edit-and-rerun work. Upstream
SOLWEIG-GPU behaves differently: it reuses those files whenever they exist, so
it must start from a clean `processed_inputs/` after any input change.

## Measured performance

One scene, one trial per configuration, measured on 2026-10-22. These numbers
describe this workload on this host. They are not a general speedup claim:
scene shape, thread count, first-use compilation and exports all affect runtime.

**Workload**

| Item | Value |
| --- | --- |
| Scene | Atlanta (Georgia Tech campus), one tile of 2394 m × 2394 m at three resolutions |
| Forcing | 19 hourly steps of 2023-06-25 from a local UMEP met file |
| Inputs and outputs | Land cover included; TMRT and UTCI written; SVF exports on (default) |
| Tiling and geometry | Single logical tile, 153 sky patches, `ERA_5_z0_find=False` |

**Settings**

| Item | Value |
| --- | --- |
| Host | Intel Xeon w9-3475X (36 cores / 72 threads), shared Linux server, 16 threads per run |
| solweig-light | Python 3.11, NumPy 2.4.6, Numba 0.67.0, GDAL 3.13.3. Compiled kernels warm, geometry cold (fresh scene directory). |
| SOLWEIG-GPU | 1.2.22 from a local checkout, not the pinned reference commits. Torch 2.14.1: the CPU build, or CUDA 12.6 on one NVIDIA RTX A6000. |

**Total wall time (seconds, lower is better)**

| Resolution (pixels) | solweig-light, CPU | SOLWEIG-GPU, CPU | SOLWEIG-GPU, RTX A6000 |
| --- | ---: | ---: | ---: |
| 1 m (2400 × 2400) | **568** | 2794 | 1031 |
| 3 m (800 × 800) | **36.1** | 177.7 | 123.4 |
| 4 m (600 × 600) | **18.2** | 112.4 | 81.3 |

**Stages at 1 m (2400 × 2400), seconds**

| Stage | solweig-light, CPU | SOLWEIG-GPU, CPU | SOLWEIG-GPU, RTX A6000 |
| --- | ---: | ---: | ---: |
| Walls and aspect | 5.6 | 193.7 | 208.5 |
| SVF / sky geometry | 332.4 | 1758.1 | 482.3 |
| TMRT / UTCI (19 steps) | 230.2 | 841.7 | 339.3 |
| Peak memory (largest process RSS) | 4.2 GiB | 24.8 GiB | 14.3 GiB + 16.5 GiB GPU |

**Other observations**

- **CPU use:** solweig-light kept on average 8.0, 7.2 and 5.9 of its 16
  threads busy at 1 m, 3 m and 4 m.
- **First run in a new environment:** compiling kernels added about 24 s. On a
  400 × 400 test crop, the first run took 34.1 s and a run with cached kernels
  took 10.1 s.
- **`block_pixels`:** at 4 m, 1024 took 18.2 s and 20.2 s in two runs, and the
  default 128 took 21.1 s. The difference is within run-to-run variation, and
  outputs were bitwise identical.
- **Re-runs in the same folder (4 m):**
  - Unchanged inputs: 10.0 s.
  - After editing 220 tree pixels: 16.2 s, and its TMRT/UTCI were bitwise
    identical to a fresh 20.1 s run with the edited trees.
  - For comparison, a fresh SOLWEIG-GPU run on the A6000 took 82.2 s.
- **Agreement with SOLWEIG-GPU on CPU:**
  - TMRT: maximum absolute difference 0.26 °C, and at most 0.0002 °C for 99.9 %
    of pixels.
  - UTCI: maximum absolute difference 0.08 °C.
  - Against SOLWEIG-GPU on CUDA, the maxima were 0.61 °C (TMRT) and 0.15 °C
    (UTCI).

## How it differs from SOLWEIG-GPU

| Area | SOLWEIG-light |
| --- | --- |
| Numerical execution | CPU arrays and compiled kernels replace Torch tensor execution. |
| Geometry | Lossless packed visibility, compiled rays and SVF count/replay avoid repeated dense patch work. |
| Radiation | Pixel-owned, ordered LW/SW and anisotropic reductions use bounded blocks and guarded lane kernels. |
| Repeated timesteps | Static geometry and longwave visibility predicates are reused; thermal state stays chronological. |
| Resources and outputs | Explicit CPU/memory admission, controlled workers, streamed bands and resumable transactions. |
| Cache validity | Content identities and checksums replace reliance on legacy artifact existence. |
| GPU support | Not implemented. Existing numerical boundaries leave room for a future backend; no CUDA execution is supplied. |

[Architecture](docs/architecture.md) explains the optimization points. The
source comparison uses upstream commit
`1fec79a78002f0c7f3f4af817ce34d1952d8b8f3`; numerical references remain frozen
at `0d7fe742abeeddd890dd58fc76ed7f78bd47faec`. These serve different purposes
and are not a claim that the latest upstream revision has been revalidated. See
[provenance](docs/provenance.md).

## Documentation and verification

- [Architecture](docs/architecture.md): source comparison and optimization mechanisms.
- [Runtime](docs/runtime.md): CPU controls, memory, cache validity and recovery.
- [Compatibility](docs/compatibility.md): seven APIs, CLI/import compatibility, extras and numerical policy.
- [Model deviations](docs/model_deviations.md): inherited limitations, explicit repair and reference-profile boundaries.
- [Provenance](docs/provenance.md): upstream pins, reference evidence and attribution.
- [Development supplements](development/README.md): validation sources and performance measurement.

A focused local differential check is:

```sh
python -m pip install '.[test]'
python -m pytest tests/differential/test_pipeline_reference.py -q
```

It executes real TIFF workflows against retained upstream CPU artifacts.
Source-inspection tests elsewhere also need the frozen upstream checkout.
Existing tests and manifests are evidence for their fixtures, not proof for
every scene or platform. Four inherited independent scientific checks remain
failed under a documented compatibility exception, and full release
qualification is incomplete. There is no shipped GPU backend. Historical
patched-upstream CUDA references are separate evidence, and authenticated
acquisition remains unverified. See the [ledger](docs/model_deviations.md).

## Credits and license

SOLWEIG originates with Fredrik Lindberg, Björn Holmer and Sofia Thorsson,
[Lindberg et al. (2008)](https://doi.org/10.1007/s00484-008-0162-7), and
[UMEP](https://github.com/UMEP-dev/UMEP). This implementation derives directly
from [nvnsudharsan/SOLWEIG-GPU](https://github.com/nvnsudharsan/SOLWEIG-GPU).
GPL-3.0-or-later; see [LICENSE](LICENSE). Original source notices and the
vendored [SLEEF Boost license](src/solweig_light/radiation/SLEEF_LICENSE.txt)
are retained.
