# mercator-model-fields-repo

Mercator ocean scalar fields for the C4PO ocean map — **the scalar fields from the same run: temperature at the surface and at 29 m, salinity, sea surface height, and ocean heat content — and, since 2026-09-02, sea ice concentration, thickness and velocity and the bottom temperature**.

**It publishes, since 2026-09-01.** `pipeline/products.toml` declares the
products, the workflow builds them four times a day, and the site draws
nine layers under names that carry the model:

| root | what it is |
| --- | --- |
| `sst-mercator.json` | temperature at 0.49 m |
| `temp29-mercator.json` | temperature at **29.445 m** — the model's level 14, its nearest to ESPC's 30 m |
| `sss-mercator.json` | salinity at 0.49 m |
| `ssh-mercator.json` | sea surface height |
| `ohc-mercator.json` | **tropical cyclone heat potential**, ρ·c_p ∫(T − 26 °C) dz from the 26 °C isotherm to the surface, in kJ/cm², off the daily profile — the same quantity and constants as ESPC's `ohc-navy`, held against ESPC's own function as the oracle |
| `sic-mercator.json` | sea ice concentration, a fraction, **zeros over open water** as ESPC's `sic-navy` publishes; the map draws it above 15% |
| `sit-mercator.json` | sea ice thickness in m, **null under 15% concentration** — this model writes 0 m over open water where ESPC writes null, so the two are made to read alike |
| `icevel-mercator.json` | sea ice velocity, a u/v **pair** in the currents' file shape at 0.001 m/s, one daily frame, null under 15% concentration — without that the whole ocean drifts at exactly zero and the particles draw it as still water |
| `bottomt-mercator.json` | the model's bottom-layer temperature, everywhere the ocean has a bottom; no depth in the header because it is not a level |

**The layer names carry the depth that is actually there**, not ESPC's
nearest: 29 m against 30 here, 47 m against 50 in the currents. Naming them
for ESPC's round numbers so the two lists lined up would say the layers are
comparable at the same depth when they are not quite; 29.445 against 30 is
small, and the rule is the rule.

`PLAN.md` carries the measurements and what is open. `DECISIONS.md` indexes
the dated one-way decisions. **Which document gets what, and what "update
docs" means across all ten repositories, is the doctrine block at the top of
`CLAUDE.md`** — the same text in all ten, held equal by the site's
`check:docs`.

## What it will publish

| | |
| --- | --- |
| source | Mercator Ocean International, via Copernicus Marine (CMEMS) |
| product | `GLOBAL_ANALYSISFORECAST_PHY_001_024` |
| dataset | `cmems_mod_glo_phy-thetao / -so / _anfc_0.083deg_P1D-m` |
| variables | `thetao` (surface, 29 m, and the profile to 300 m for heat content), `so`, `zos`; since 2026-09-02 `siconc`, `sithick`, `usi`/`vsi`, `tob` off the daily 2-D dataset |
| resolution | **0.083 deg (1/12 deg)**, 2041 x 4320 global |
| cadence | daily |
| access | the `copernicusmarine` toolbox over Zarr; account required |

## What one frame costs, measured 2026-09-01

| | |
| --- | --- |
| one global frame | **33.6 MB** float32 |
| currents transfer | 0.9 MB/s (36 s a frame) |
| scalar transfer | 3.6 MB/s (9 s a frame) |
| depths | 50 levels, 0.49 m to 5727.92 m, priced individually |
| toolbox install | 22 s |

**Every transfer figure above was measured through the toolbox's DEFAULT dask
blocks, which read about 26x the bytes a level needs** (found 2026-09-01: 50.7 s
a level that way, 1.9 s with `chunk_size_limit=0`, on the same store). They
were true of a path the build no longer takes. The fetcher opens with no dask
now and refuses a dask-backed array; the site's `PLAN.md` carries the
measurement and the runner's own numbers are in the run logs, timestamped per
line.

A build was five scalars in 63 s (run 33578824264), 17 s of it the heat
content over 29 levels, 62 tropical tiles — the four 2-D products added on
2026-09-02 cost 34 s more on a laptop, most of it tiles (`PLAN.md` has the
split; the runner's own number goes there after its first run); the "20 frames, 672 MB, 12.5
minutes" once quoted here was a currents figure on the slow path and never
described this repository. `PLAN.md` has the reasoning.

## Why a second model at all

ESPC stalls, and not in a way retrying fixes: on 2026-08-31 `tds.hycom.org`
refused every connection for hours, every ESPC tile tier was withdrawn from
the live map, and all five products were briefly held. A different model is
the only answer to a run that goes late. This one is the like-for-like
option — same 1/12 deg, currents and temperature and salinity together.

**It is a PEER, not a fallback** (the owner's call, 2026-09-01). Both models
are fetched every run and the reader chooses; the contract already carries
`source` and `modelRun`, and the map already shows them, so which ocean is on
screen is visible rather than inferred.

## Credentials

Two repository secrets, `COPERNICUSMARINE_SERVICE_USERNAME` and
`COPERNICUSMARINE_SERVICE_PASSWORD`. **There is no API key** — the toolbox
takes a username and password and nothing else, and the username is not the
account email. A Copernicus Marine account is free.
