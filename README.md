# ShadeMe notebooks

Working notebooks for [Tchanwangsa/shademe](https://github.com/Tchanwangsa/shademe) —
one topic per notebook, in dependency order. Added to the main repo as a submodule at
`notebooks/`.

They import the real package (`from shademe.physics import shadow`) rather than
re-implementing anything, so a notebook that disagrees with the API is a bug in one of
them, not a difference of opinion.

## Running

From the parent repo, once:

```bash
uv sync --group notebooks
uv run python -m ipykernel install --user --name shademe --display-name "shademe (uv)"
uv run python -m shademe.pipeline.build_all
```

Then `uv run jupyter lab` and pick the **shademe (uv)** kernel.

## The set

| # | notebook | question it answers |
|---|---|---|
| 00 | orientation | what files exist, which stage wrote each, what are the knobs |
| 01 | source data | what did City of Melbourne and OSM actually hand us |
| 02 | dsm rasterisation | how do polygons become a 2 m height raster |
| 03 | tree allometry | how does a trunk diameter become a crown on a stick |
| 04 | sun and shadows | how does a ray march turn that into hourly shade |
| 05 | sky view factor | how much sky can a pedestrian see from each cell |
| 06 | materials | what is the ground made of, and what does that do to heat |
| 07 | weather | today's forecast, and why it gets bias-corrected |
| 08 | surface temperature | the energy balance march, and why it needs memory |
| 09 | mrt | the SOLWEIG six-direction radiation budget |
| 10 | utci and stress | the Brode polynomial, and what "stress" means |
| 11 | the graph | OSM ways to a 61k-edge walkable network |
| 12 | edge pricing | every street, priced, for one hour |
| 13 | a-star | watching the search expand, twice, for two objectives |
| 14 | end to end | lat/lon in, route cards out, dose in degC-minutes |
