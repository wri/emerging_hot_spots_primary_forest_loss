# emerging_hot_spots_primary_forest_loss
Methods for the emerging hot spot analysis from Harris et al. (2017).

# Emerging Hot Spot Analysis of Primary Forest Loss

Identifies emerging hot spots of primary forest loss across tropical
countries using ArcGIS Pro's space time pattern mining tools. Journal publication 
available at Harris et al. (2017).

For each country, the workflow extracts annual primary forest loss,
aggregates it to points, builds a space time cube, and runs emerging hot
spot analysis using a per-country neighborhood distance.

## Requirements

- ArcGIS Pro with the Spatial Analyst extension
- The Pro Python environment (arcpy, pandas)
- `gdal_translate` on the system PATH

## Setup

Edit the paths at the top of `config.py` to match your machine. That is the
only file that needs changing.

## Running

### Annual run

Most of the time this is all you need. When a new year of loss data is
available:

1. Update `DATA_YEAR` and `LAST_SHORT_YEAR` in `config.py`.
2. Point `LOSS_MOSAIC` and `AREA_MOSAIC` at the new mosaics.
3. Run `main.py`.

Results go to `OUTPUT_ROOT\ehs_<DATA_YEAR>`, so each year's output is kept
separate.

The per-country masks do not need rebuilding for a new year of loss data.

### Rebuilding the primary forest masks

Only needed when the primary forest dataset changes, or when the country
list changes. Both stages live in `build_primary_masks.py`.

New primary forest dataset:

1. Point `PRIMARY_RASTER_TILES` at the new raster tiles.
2. Set `VECTORIZE_TILES = True`.
3. Run `build_primary_masks.py`. The script vectorizes the raster tiles into
   `PRIMARY_VECTOR_GDB`, then clips, merges, and simplifies them into
   per-country masks in `MASK_GDB`.
4. Set `VECTORIZE_TILES` back to `False`.

Country list changed, same primary forest data:

1. Update the country list variable.
2. Leave `VECTORIZE_TILES = False`.
3. Run `build_primary_masks.py`. The existing vector tiles are reused and
   only the masks are rebuilt.

The vectorize stage skips tiles that already exist, so a long run can be
stopped and resumed.

`main.py` never triggers the mask build. The two scripts are run
independently, and a new year of loss data does not require rebuilding
masks.

## Files

| File | Purpose |
|------|---------|
| `config.py` | All paths and settings. The only file to edit. |
| `main.py` | Annual EHS run. Loops over countries. |
| `emerging_hotspot_factor.py` | Per-country workflow, called by `main.py`. |
| `utilities.py` | Shared functions. |
| `build_primary_masks.py` | Builds the primary forest masks. Run separately. |

## Inputs

| Config variable | What it is |
|-----------------|------------|
| `COUNTRY_EHS` | Country boundaries with an ISO field. What the EHS run loops over. |
| `COUNTRY_EHS_INT` | Same countries pre-split into smaller pieces, so the extract step can handle large countries. |
| `COUNTRY_MASKS_INPUT` | Countries to build masks for. Usually the full list. |
| `LOSS_MOSAIC` | Tree cover loss year mosaic, masked to primary forest. |
| `AREA_MOSAIC` | Pixel area mosaic. Used as the snap raster. |
| `SNAP_RASTER` | Raster the aggregate step snaps to. |
| `DISTANCE_TABLE` | Spreadsheet of per-country EHS neighborhood distances. Needs an `iso` column and a `distance_m` column, in meters. |
| `TILE_GRID` | Footprint grid whose `Name` field ends in the tile id. |
| `PRIMARY_RASTER_TILES` | 10-degree primary forest rasters, 1 = primary, 0 = not primary. |

## Outputs

Written to `OUTPUT_ROOT\ehs_<DATA_YEAR>`:

| Output | What it is |
|--------|------------|
| `results.gdb\<ISO>_ehs` | Emerging hot spot results. The main product. Classification is in the `PATTERN` field. |
| `results.gdb\<ISO>_all_points` | Aggregated annual loss points that fed the cube. |
| `<ISO>_cube.nc` | Space time cube. |
| `<ISO>_ehs_output.txt` | Tool messages and the neighborhood distance used. Appends across runs. |
| `scratch.gdb` | Intermediates. Not cleaned up automatically. |

Masks go to `MASK_GDB` as `<ISO>_tcd_merged_mask`. Vector tiles go to
`PRIMARY_VECTOR_GDB` as `tile_<tile_id>`.

## Data archive

Input data and results are archived on S3. See the table below for where
each input lives.

| Config variable | S3 location |
|-----------------|-------------|
| `COUNTRY_EHS` | *TBD* |
| `LOSS_MOSAIC` / `AREA_MOSAIC` | *TBD* |
| `DISTANCE_TABLE` | *TBD* |
| `PRIMARY_RASTER_TILES` | *TBD* |
| `MASK_GDB` | *TBD* |
| Results | *TBD* |

Geodatabases must be zipped before upload. Syncing them as loose files
corrupts them.

## Notes and known issues

- A country with no mask in `MASK_GDB`, or no row in the distance
  spreadsheet, raises an error and is skipped. The run continues to the
  next country.
- `gdal_translate` may print `Can't load requested DLL` warnings about
  `ogr_Arrow.dll` and `ogr_Parquet.dll`. These are harmless. Look for the
  progress line and `done` after them.
- `scratch.gdb` accumulates `<ISO>_extract` mosaics across a run and is not
  cleaned up. Watch disk space on long runs.
- The EHS tool clamps neighborhood distances below the cube cell size
  (2226.3898 m). The script prints a note when this happens rather than
  failing.
- Emerging hot spot output has no default symbology. Open `<ISO>_ehs` in
  Pro and symbolize on `PATTERN`.
