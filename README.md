# mercator-model-fields-repo

Mercator ocean scalar fields for the C4PO ocean map — **the scalar fields from the same run — temperature, salinity and sea surface height**.

**It publishes, since 2026-09-01.** `pipeline/products.toml` declares the
products, the workflow builds them four times a day, and the site draws
three surface scalars — temperature, salinity and sea surface height under names that carry the model.

**The layer names carry the depth that is actually there**, not ESPC's
nearest: 47 m against 50, 186 against 200, 380 against 350, 1062 against
1000. Naming them for ESPC's round numbers so the two lists lined up would
say the layers are comparable at the same depth when 380 against 350 is 9%
deeper water.

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
| variables | `thetao`, `so`, `zos` |
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

An ESPC-shaped set — five depths x two leads x (u,v) — is 20 frames, about
**672 MB**; the 12.5 minutes once quoted for it was the slow path's price.
The set count is still a choice with a price on it rather than a copy.
`PLAN.md` has the reasoning.

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
