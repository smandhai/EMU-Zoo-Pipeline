# EMU Zoo: Image Production Framework

Original author: Emma Alexander
Currently maintained and updated by: Soheb Mandhai

EMU Zoo is the image-production pipeline for **Radio Galaxy Zoo 2 – EMU (RGZ-EMU)**, a [Zooniverse](https://www.zooniverse.org/projects/hongming-tang/radio-galaxy-zoo-emu) citizen-science project. The code builds sky maps of potential radio outflow sources, together with comparable optical and infrared sky maps. Volunteers classify these images on Zooniverse.

## What the code does

For each selected radio source from the ASKAP EMU survey, the pipeline produces a PNG showing:

- **Radio:** EMU contours from the ASKAP continuum image
- **Optical:** a DES (DR1 or DR2) RGB image built from the i, r and g bands. DSS is used as a fallback where DES coverage is absent or incomplete
- **Infrared:** WISE band 1 (3.4 µm)

Outputs for each source:

| Output | Description |
|---|---|
| Overlay PNG | Radio contours over optical and infrared imagery |
| `_cross_` PNG | The 6'×6' cutout with a crosshair at the target. These are the images released to Zooniverse |
| FITS cutout (optional) | Radio FITS cutout (`export_fits_cutout`) |
| Contours HDF5 (optional) | Extracted contour levels (`extract_contours`, work in progress) |
| Excluded sources | Sources with no emission at the centre or only stray single contours are written to a separate folder for inspection |

The code is currently set up for the **EMU Pilot I fields**. Using other fields requires editing the field lookup in `settings.py`.

## Repository contents

| File | Purpose |
|---|---|
| `EMUZoo_generate_overlays.py` | Main script: builds the overlays for each source |
| `settings.py` | All configuration (fields, paths, options) |
| `utils.py` | Helper functions |
| `DES_file_download.py` | Downloads DES g, i and r tiles |
| `create_DES_tiles_dims.py` | Writes the bounding box of each DES tile |
| `DES_tiles_dims.txt` | DES tile lookup table: tile name, RA min, RA max, Dec max, Dec min |
| `WISE_tiles.txt` | WISE tile table (IRSA AllWISE query result) |
| `separate_source_names.py` | Splits a subject list into one list per field |
| `verify_sources.py` | Checks produced images against a subject list |
| `cutout_validation.py` | Checks that all subjects were produced |
| `check_field_completion.py` | Reports the completion fraction of each field |
| `extract_cross_images.py` | Collects the `_cross_` images for release |
| `old_code/` | Earlier (Phase A) scripts, kept for reference |

## Requirements

- Python 3
- `numpy`, `pandas`, `matplotlib`, `astropy`, `astroquery`, `reproject`, `shapely`, `h5py`, `cmasher`
- `curl` (used by the DES download script)
- Roughly 1 TB of disk space for the full Phase 1 data set

Pin the `numpy` and `matplotlib` versions across machines. The matplotlib version changes the borders of the generated PNGs.

```bash
pip install numpy pandas matplotlib astropy astroquery reproject shapely h5py cmasher
```

## Getting started

### 1. Clone the repository

```bash
git clone https://github.com/smandhai/EMU-Zoo.git
cd EMU-Zoo
```

### 2. Arrange the pre-downloaded tiles

All input locations in `settings.py` are built from a single root folder, `prefix`. The default is `../sample_data/des/`. The example below uses that default.

```text
EMU-Zoo/                                  <- run the scripts from here
├── EMUZoo_generate_overlays.py
├── settings.py
├── DES_tiles_dims.txt                    <- DES tile bounding boxes
├── WISE_tiles.txt                        <- WISE tile table
│
└── ../sample_data/des/                   <- `prefix` in settings.py
    │
    ├── askap_data/                       <- RADIO (`dataloc`)
    │   ├── image.i.SB9351.cont.taylor.0.restored.fits
    │   ├── image.i.SB9325.cont.taylor.0.restored.fits
    │   └── catalogues/
    │       ├── AS101_Continuum_Component_Catalogue_9351_78.csv
    │       └── AS101_Continuum_Component_Catalogue_9325_76.csv
    │
    ├── DEStiles/                         <- OPTICAL (`DESfiles_dir`)
    │   ├── DES2033-5748_r2590p01_g.fits.fz
    │   ├── DES2033-5748_r2590p01_i.fits.fz
    │   ├── DES2033-5748_r2590p01_r.fits.fz
    │   └── ...                           <- g, i and r band for every tile
    │
    ├── WISEtiles/                        <- INFRARED (`WISEfiles_dir`)
    │   ├── <wise_tile>-w1-int-3.fits
    │   └── ...                           <- band 1 tiles
    │
    ├── phase1_radio_cutouts/             <- OUTPUT: overlay PNGs, one folder per field
    │   ├── SB9351/
    │   └── SB9325/
    ├── phase1_radio_cutouts_6x6/         <- OUTPUT: 6'x6' cutouts (and FITS if enabled)
    ├── excluded_radio_cutouts/           <- OUTPUT: sources rejected by the filters
    └── contours/                         <- OUTPUT: contour HDF5 (if enabled)
```

File naming rules the code relies on:

| Data | Required name |
|---|---|
| ASKAP image | `image.i.SB<field>.cont.taylor.0.restored.fits` in `askap_data/` |
| ASKAP catalogue | `catalogues/AS101_Continuum_Component_Catalogue_<field>_<component>.csv` in `askap_data/`. The component number comes from `SB_dic` in `settings.py` |
| DES tile | `<tilename>*_g.fits*`, `<tilename>*_r.fits*` and `<tilename>*_i.fits*` in `DEStiles/`. The tile name is the one listed in `DES_tiles_dims.txt` |
| WISE tile | `<wise_tile>-w1-int-3.fits` in `WISEtiles/`. The tile name is the `coadd_id` listed in `WISE_tiles.txt` |

Create the output folders yourself before the first run. The code attempts to create some of them, but the parent folders must already exist.

If you do not have the tiles yet:

- **Radio:** download the images and catalogues for each field from [CASDA](https://research.csiro.au/casda/).
- **DES:** obtain the tile names with a Data Lab query (see below), then run `python DES_file_download.py <tile_list.txt> "<prefix>/DEStiles/"`. Edit `dr` at the top of the script to choose DR1 or DR2, and run it once per release.
- **WISE:** download the AllWISE band 1 tiles into `WISEtiles/`. The table of tiles covering the survey area is `WISE_tiles.txt`.

DES tile query (run on [NOIRLab Data Lab](https://datalab.noirlab.edu/query.php?name=des_dr2.main) for DR1 and DR2):

```sql
SELECT tilename
FROM des_dr2.main
WHERE ra > 300.4493 AND ra < 360.6493
  AND dec > (dec - 0.1) AND dec < (dec + 0.1)
```

Check the RA and Dec bounds against your survey area before using this query. As written, the `dec` condition compares a column with itself and the upper RA bound exceeds 360.

### 3. Build the DES tile table

If you add or change DES tiles, regenerate the bounding boxes:

```bash
python create_DES_tiles_dims.py
```

The script reads the header of each `*_i.fits.fz` tile in a folder set at the top of the script (edit it first) and writes `new_DES_tiles_dim.txt`. Copy or merge its contents into `DES_tiles_dims.txt`.

### 4. Configure `settings.py`

At a minimum, set:

| Setting | Meaning |
|---|---|
| `prefix` | Root folder of the data layout above |
| `SB` | Field to process. It must be a key of `SB_dic` (9287, 9325, 9351, 9410, 9434, 9437, 9442, 9501, 10083, 10635) |
| `cat_type` | Catalogue type. Use `"component_new"` for Phase 1 |
| `override_src` | Sources to process: a single name, a list of names, or the path to a file with source names in the first column. **The last assignment in the file takes effect**, so remove stale assignments |
| `use_file` | Set to `True` when `override_src` is a file path |

Other options:

| Setting | Meaning |
|---|---|
| `cont_limit` | Number of radio contour levels (default 8) |
| `remove_single_contours` | Remove single contours outside the target |
| `create_cutout`, `export_fits_cutout`, `cutout_dir` | Write the 6'×6' cutouts, and optionally FITS files |
| `use_cross` | Add a crosshair to the 6'×6' cutouts |
| `selavy_convention` | Use the Selavy lettering suffix. The catalogue must contain it in the column before the component names |
| `extract_contours`, `extract_contours_dir` | Save contour levels to HDF5 |
| `skip_plotting`, `skip_cutout_plotting` | Skip the main plots or the cutout plots |
| `overwrite` | Regenerate existing files. The default is `True` |

### 5. Run the main script

From the repository root:

```bash
python EMUZoo_generate_overlays.py
```

Run it once for each field. Fields are independent and can be run in parallel, but each can take up to about a day. If tiles or outputs are missing after a run, run it again.

### 6. Check the output

```bash
python verify_sources.py          # images match the subject list
python cutout_validation.py       # every subject was produced
python check_field_completion.py  # completion fraction per field
python extract_cross_images.py    # collect the _cross_ images for release
```

The helper scripts contain hard-coded paths. Edit them to match your layout before running. Then inspect a sample of the PNGs by eye for gaps, missing images, borders and artefacts.

## Running the full Phase 1 workflow

Phase 1 selected sources by complexity and major axis (complexity ≥ 4000 and major axis ≥ 20 arcsec). The steps for each field are:

1. Set `skip_plotting = True` and run the main script to produce radio-only cutouts.
2. Run the complexity analysis on the cutouts (done outside this repository) and apply the cut. This produces a subject-list CSV whose `File` column holds the subject file names.
3. Set `override_src` to that CSV and run `python separate_source_names.py`. It writes one source list per field, named `<field>_<csv name>`.
4. Set `skip_plotting = False`, `use_file = True` and `override_src` to the per-field list. Run the main script again.
5. Validate the output as in step 6 above.

## How image selection works

1. The script loads the ASKAP image and catalogue for the field, sorted by major axis.
2. It cuts a 12 arcmin region around each source, draws the radio contours and masks the background.
3. Sources with no emission at the centre, or with only stray single contours, go to `excluded_radio_cutouts/`.
4. DES tiles covering the source are found from `DES_tiles_dims.txt`. Several tiles are mosaicked with `reproject` if needed.
5. DSS (DSS2 Red, DSS and DSS2 Blue via SkyView, up to five attempts) is used instead if no DES tile is found, a tile cannot be read, or more than 20% of the DES mosaic is NaN. These images are prefixed `dss_`.
6. WISE band 1 is read from `WISEtiles/` if the tile is present. Otherwise it is downloaded through SkyView.

## Troubleshooting

| Symptom | Action |
|---|---|
| Downloads hang or time out (DSS, WISE, astroquery) | Re-run, and process one field at a time. If astroquery keeps returning the same error, delete its cache folder |
| Blank WISE panels | The in-situ download failed silently. Download the tile into `WISEtiles/` and re-run |
| Segmentation fault on a field | A DES file is probably corrupted. Delete and re-download it (`curl -C -` resumes partial downloads) |
| Wrong sources processed | Check `override_src` in `settings.py` |
| "DES File not found" messages | The tile is not in `DEStiles/` or does not follow the naming rules above |
| PNG borders or sizes differ between runs | Use identical `matplotlib` and `numpy` versions |
| Path errors in helper scripts | Edit the hard-coded paths at the top of the script |

## Known limitations

- Limited to the EMU Pilot I fields.
- Catalogue columns are read by index, so a change in the catalogue layout will break the code.
- Several helper scripts contain hard-coded paths, and some use Windows-style backslashes.
- The `extract_contours` option is a work in progress.

## Contacts

- Pipeline: Soheb Mandhai
- Working group chair and source selection: Eleni Vardoulaki
- Zooniverse project lead: Hongming Tang

## Acknowledgements

This project uses data from ASKAP (EMU), the Dark Energy Survey, the Digitized Sky Survey and WISE. Please add the acknowledgement text required by each survey.
