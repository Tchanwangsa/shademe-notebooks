# ShadeMe notebooks

Working notebooks for [Tchanwangsa/shademe](https://github.com/Tchanwangsa/shademe) — one
topic per notebook, in dependency order, added to the main repo as a submodule at
`notebooks/`.

These are **not** a tour of the package. Nothing here does `from shademe...`. Every
notebook re-derives its piece from scratch, in the open, so the physics is readable rather
than imported. If a notebook and the package disagree, that is a thing to go look at, not
automatically a bug in the notebook.

The chain ends in one number: how hot a person feels standing on a given square metre of
Melbourne CBD at a given hour. Everything is built to get there — shadow → sky view →
weather → materials → surface temperature → MRT → UTCI.

## How the notebooks talk to each other

Three shared surfaces, nothing else:

| surface          | what it is                                                                                            |
| ---------------- | ----------------------------------------------------------------------------------------------------- |
| `config.json`  | the shared constants — CRS, bbox, grid geometry, plus the occasional one a later notebook has to add |
| `data/`        | raw downloads, each notebook fetches what it needs, cached by filename                                |
| `out/rasters/` | `.npy` arrays on the common grid, written by one notebook and read by the next                      |

`config.json` can grow, but the default is that it doesn't. Every notebook **reads** it at
the top:

```python
import json
CFG = json.load(open("config.json"))          # top of every notebook
```

A notebook only **writes** it back when it produces a constant that a later notebook
genuinely has to agree with it on — a leaf transmissivity two notebooks both use, a set of
material albedos. That is rare. Anything that only matters inside one notebook — loop
bounds, plot limits, the crop window, a coefficient used once — stays a literal in the cell
that uses it. Prefer leaving the file alone; a smaller `config.json` is the better outcome.

When a notebook does have to add something, it adds a key and dumps the same dict back —
never rebuild it from scratch, or you drop what an earlier notebook wrote and silently
break something three notebooks downstream:

```python
CFG["canopy"]["tau_leaf"] = 0.03
json.dump(CFG, open("config.json", "w"), indent=4)
```

Both `config.json` and `out/` are gitignored. The repo holds notebooks and nothing else,
so the set has to run top to bottom from a clean checkout — `initialise.ipynb` first, then
01, 02, … in order.

## Running

From the parent repo, once:

```bash
uv sync --group notebooks
```

```bash
uv run python -m ipykernel install --user --name shademe --display-name "shademe (uv)"
```

Then `uv run jupyter lab` from `notebooks/`, and pick the **shademe (uv)** kernel. Paths in
the notebooks are relative to `notebooks/`, so that is the working directory.

Available without installing anything else: `numpy`, `scipy`, `pandas`, `shapely`,
`rasterio`, `pyproj`, `pvlib`, `requests`, `matplotlib`, `ipywidgets`. Anything beyond
that goes in the parent `pyproject.toml` under the `notebooks` group first.

## The set

| #  | notebook                 | question it answers                                               | writes                                                     |
| -- | ------------------------ | ----------------------------------------------------------------- | ---------------------------------------------------------- |
| 00 | `initialise`           | where is the grid, and how do polygons become a 2 m height raster | `config.json`, `dsm_buildings.npy`, `dsm_canopy.npy` |
| 01 | sun and shadows          | where is the sun, and what does the city block                    | `shade_{06..20}.npy`                                     |
| 02 | trees are not buildings  | how does a trunk diameter become a crown on a stick               | `dsm_canopy_top/base.npy`, re-runs `shade_*`           |
| 03 | sky view factor          | how much sky can a pedestrian see from each cell                  | `svf_bldg.npy`, `svf_veg.npy`                          |
| 04 | weather                  | what is the air actually doing today                              | `data/weather_{date}.json`                               |
| 05 | materials                | what is the ground made of, and what does that do to heat         | `material_id.npy`, `material_props.json`               |
| 06 | surface temperature      | the energy balance march, and why it needs memory                 | `tsurf_{06..20}.npy`                                     |
| 07 | mean radiant temperature | the six-term radiation budget a body actually sees                | `mrt_{hour}.npy`                                         |
| 08 | utci and stress          | the Bröde polynomial, and what "stress" means in °C·minutes    | `utci_{hour}.npy`                                        |
