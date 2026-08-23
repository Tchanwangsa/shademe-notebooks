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

| surface | what it is |
|---|---|
| `config.json` | the shared constants — CRS, bbox, grid geometry, plus the occasional one a later notebook has to add |
| `data/` | raw downloads, each notebook fetches what it needs, cached by filename |
| `out/rasters/` | `.npy` arrays on the common grid, written by one notebook and read by the next |

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

`initialise.ipynb` is already written and is the shape to copy.

| # | notebook | question it answers | writes |
|---|---|---|---|
| 00 | `initialise` | where is the grid, and how do polygons become a 2 m height raster | `config.json`, `dsm_buildings.npy`, `dsm_canopy.npy` |
| 01 | sun and shadows | where is the sun, and what does the city block | `shade_{06..20}.npy` |
| 02 | trees are not buildings | how does a trunk diameter become a crown on a stick | `dsm_canopy_top/base.npy`, re-runs `shade_*` |
| 03 | sky view factor | how much sky can a pedestrian see from each cell | `svf_bldg.npy`, `svf_veg.npy` |
| 04 | weather | what is the air actually doing today | `data/weather_{date}.json` |
| 05 | materials | what is the ground made of, and what does that do to heat | `material_id.npy`, `material_props.json` |
| 06 | surface temperature | the energy balance march, and why it needs memory | `tsurf_{06..20}.npy` |
| 07 | mean radiant temperature | the six-term radiation budget a body actually sees | `mrt_{hour}.npy` |
| 08 | utci and stress | the Bröde polynomial, and what "stress" means in °C·minutes | `utci_{hour}.npy` |

Phase 2, only if routing gets its own write-up: 09 the walking graph · 10 edge cost ·
11 A\* with the frontier animated · 12 the two objectives and why they disagree.

### What each one has to cover

The beats, in order. Turn these into a cell list when you write the notebook.

**01 — sun and shadows.** `pvlib.solarposition` for az/elev, print the 06→20 table (the
azimuth swings through north at noon — southern hemisphere). `h / tan(el)` for one
building at three hours, which is what justifies the 500 m buffer. A 1-D toy: a 16-cell
row, looped, printed as `#` and `.`. The same thing as `max(shift(dsm, d) - d·cell·tan(el))`
— the shift trick, and why the loop inverts. Then the 2-D march, `dr = -k·cos(az)`,
`dc = +k·sin(az)`, quarter-cell steps, deduped. Figures: shade at 09/13/17 over the DSM,
translucent blue on magma; then the 06→20 loop as a 5×3 grid on one colour scale.

**02 — trees are not buildings.** Download `trees-with-species-and-dimensions-urban-forest`
(~82 k points with DBH and genus). Print the DBH distribution — there is a tree with a
26,027 cm trunk — and clip to 1–250 cm. Genus → five form classes, Chapman–Richards
`H = 1.3 + a(1-e^{-bD})^c`, plotted as five curves over the DBH histogram. Crown base
linear in H, drawn as one tree in profile. Join heights to the canopy polygons (STRtree
nearest), rasterise top and base. The slab test: a crown blocks iff
`base(d) ≤ d·tan(el) < top(d)`. Figure: 07:00 shade, flat 8 m vs crown slab, side by side —
the low beam now reaches under the trees.

**03 — sky view factor.** What SVF is (1 = open park, 0.1 = laneway floor). One cell, one
azimuth: march out, take the max blocked angle → horizon angle, shown as a polar plot for
a laneway cell and a park cell. 16 azimuths, integrated, vectorised over the grid with the
shift trick from 01. Canopy blocks partially — blend by `TAU_LEAF = 0.03`. Figure: the SVF
raster in viridis, plus a profile along one street centreline.

**04 — weather.** Open-Meteo forecast/archive JSON for one date: `temperature_2m`,
`relative_humidity_2m`, `direct_radiation`, `diffuse_radiation`, `wind_speed_10m`,
`cloud_cover`. Six diurnal curves on a shared x-axis. Direct vs diffuse as a stacked area —
that ratio is what makes shade worth anything, and diffuse never goes to zero. One cell on
the units trap: wind arrives in km/h, the physics wants m/s.

**05 — materials.** Download `road-segments-with-surface-type` from CoM and show what
actually turns up: 56,519 polygons, 54 distinct values in `material`, and they are
asset-management labels rather than physical surfaces. Count them by row *and* by area —
the two orderings disagree completely (11,898 Dressed Bluestone rows are 11.6 ha; 8,693
HMA rows are 386 ha), because over half the dataset is kerb and channel slivers. Then the
collapse to ~8 physical classes: every `N Row Bluestone Pitcher`, 1 through 15, is the same
stone; `Cast in Situ Off Form`, `Pre Cast Exposed Agg` and `Precast Concrete` are all
concrete; `Timber`, `Steel`, `Wrought Iron` and `Plants` are not a walking surface at all.

The junk gets named rather than quietly dropped — 18 rows with `material` null, plus
`Other` (135), `Other Paver` (266), `Unmade` (49), `Not Known` (4), `Mixture of Materials`
(3) — and the notebook says what it does with them and why (they are ~0.4% of rows, they
sit in the road reserve, so they take the asphalt default).

Then the coverage gap, which is bigger than the unclassified one: this dataset is the road
reserve only. Parks, plazas, private land and the ground under buildings have no polygon at
all, so OSM landcover via Overpass fills grass/park/water, and whatever is still unpainted
takes a stated default. Finish with the class table — albedo, emissivity, `ρc·d` — and the
class raster in a categorical colormap with a legend, where the parks come out grass and
Swanston St comes out bluestone.

**06 — surface temperature.** One equation:
`absorbed_SW + L_down − L_up − H_conv − storage = 0`. Each term separately for one cell and
one hour, printed in W/m² and drawn as a horizontal bar — shortwave dominates in sun,
longwave in shade. Then why it needs memory: asphalt at 14:00 depends on 12:00, so march
hourly from a cold start with sub-stepping for stability. Figures: `Ts(t)` for four cells
(sunlit asphalt, shaded asphalt, sunlit grass, laneway) with air temp dashed — asphalt runs
~20 °C above air; then the 14:00 raster in inferno, where the road network lights up.

**07 — mean radiant temperature.** MRT is the temperature of an imaginary uniform enclosure
that would radiate the same at you as the real street does — not air temperature, and up to
25 °C above it. Six terms, one cell first, printed in W/m²: `K_dir` (gated by shade, scaled
by the projected-area factor `f_p(elev)`), `K_dif` (gated by SVF), `K_ref_g` (ground albedo),
`K_ref_w` (walls, the crudest and smallest), `L_sky` through SVF, `L_wall` through 1−SVF,
`L_gnd` from Ts. Figure: six small rasters on a shared scale — `K_dir` is the shade map,
`L_gnd` is the material map, `L_sky` is the SVF map. Combine, fourth-root back to a
temperature, then MRT at 14:00 in turbo and MRT − air temp on a diverging map so the shade
benefit reads directly in °C.

**08 — UTCI and stress.** A 6th-order polynomial in (ta, va, mrt−ta, vapour pressure), 210
terms, fitted to a full physiological model — transcribe it, don't derive it. Plot UTCI vs
MRT at fixed air temp, one line per wind speed
(wind beats shade on some days). The stress band: 9–26 °C is no stress, stress is degrees
outside it — plot the piecewise function, then the UTCI raster in the ten official category
colours. Close on UTCI at 10:00 vs 17:00 on one scale, which is the figure explaining why
the app quotes °C·minutes and not a percentage.

## House style

What makes `initialise.ipynb` work, and what a new notebook has to keep:

- **One idea per cell.** A cell does a thing, prints or plots the result, and moves on. No
  cell both downloads data and derives a constant.
- **Small before big.** Do it once for one cell, one tree, one hour, and print the numbers.
  Then vectorise the same thing over the grid. The scalar version is the explanation.
- **No test cells.** No asserts, no sanity checks, no "confirming the grid lines up",
  no re-verifying something the previous cell just did. Write the code that builds the
  thing and draw the figure. Print a number only when the number is itself the point —
  the azimuth table, the DBH range that motivates the clip, the six radiation terms in
  W/m² — not to prove the cell above worked.
- **End on a picture.** Every notebook closes on the figure that makes its point, and
  usually has two or three along the way.
- **Retype constants where they are used.** `CELL = 2.0` at the top of the cell that needs
  it beats an import. Repetition is cheaper than indirection here.
- **Downloads are cached and idempotent.** Check for the file, print `cached`, skip. Re-running
  a whole notebook must not re-hit the City of Melbourne API.
- **Show the data as it arrived.** Right after a download, print what actually came back —
  the field names, the value counts, the nulls, the categories nobody can use. Open data is
  messy and the mess is half the story: a tree with a 26,027 cm trunk, 54 material labels
  that mean 8 surfaces, 18 rows with no material at all. Say what you do with each pile and
  why, in the cell that does it. Never silently drop rows.
- **Crop the slow ones.** 03, 06, 07 and 08 at full grid (2438 × 2485 ≈ 6.06 M cells) are
  minutes per pass. Define one `CROP` near the top — a 1500 × 1500 window over the CBD —
  and run the physics inside it. Full grid only for the final poster raster, in its own
  clearly marked cell at the end.

## Writing a notebook

Plan first, then emit. Before writing any code, lay out the cell list — for each cell, one
line saying what it does and what it prints or plots. Get that agreed. Then write the
notebook in groups of about five cells, one coherent step at a time (the setup, the toy
version, the vectorised version, the figure), so each group can be run and looked at before
the next one is written. Never dump twenty cells at once.

## Loose ends

`nbtools.py` is a leftover — it holds a pasted copy of the old `config.py` constants and
nothing imports it. Either it becomes the one place shared plotting helpers live, or it
gets deleted; right now it is neither.
