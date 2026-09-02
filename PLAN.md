# mercator-model-fields-repo — the founding plan and running record

The fields half of a Mercator Ocean peer to ESPC. **It publishes**: this
records what was measured, what was built, and what is still open.

## Where it stands

**It publishes, since 2026-09-01.** `pipeline/products.toml` declares the
products, the workflow builds them, and the schedule is on at
`53 2,8,14,20` -- four runs a day.

The fetch path is `scripts/fetch-mercator.py` in the site repository, and it
was validated against the live service before any of this was switched on:
a scalar frame came back at 360x171, 1.0 deg, -2.093 to 35.76 degrees C, and a
currents frame at +/-1.75 m/s with wet counts falling and speeds weakening
with depth. Numbers that would not survive a transform fault.

**The probe is what makes a schedule affordable.** A full run here is five
scalars in 63 s (run 33578824264), 17 s of it the heat content; the probe reads the upstream's time axis -- metadata, about 5 s --
and compares it against the `refTime` on disk, so a run with nothing new costs
seconds. It also rebuilds when this script would publish something the disk
does not have, because a probe watching only the upstream cannot see a change
in us.

Created 2026-09-01 with its four documents and its credentials. The deploy key that lets a pipeline here read the private site
repository is on that repository as `mercator-model-fields-repo-checkout`,
read-only; the Copernicus pair and `PIPELINES_SSH_KEY` are secrets here.

**The set is decided and no longer blocking**: five scalars -- SST, SSS,
SSH, temperature at 29 m and ocean heat content -- daily, at lead 0, and
since 2026-09-02 four more off the daily 2-D dataset: ice concentration,
thickness and velocity, and the bottom temperature (the record below).

## Why a second model, and why this one

ESPC is flaky in two unrelated ways and only one is handled. Per-request
failures are covered by retries. The other is that **the run itself goes
late** — and on 2026-08-31 it went further than late: `tds.hycom.org` refused
every connection for hours, all five ESPC products were briefly held, and
every ESPC tile tier was withdrawn from the live map, so a reader panning
west of -100 saw the 0.96 deg globe instead of 0.08 deg detail. No amount of
retrying fixes that. Only a different model does.

Mercator is the only like-for-like option: the same 1/12 deg, with currents,
temperature and salinity from one run. The alternatives have thinned — NOAA
retired NOMADS' OPeNDAP in 2025, and OSCAR on CoastWatch ERDDAP has been
stale since 2014-10-06. The site's `PLAN.md` carries that survey.

## The upstream, verified 2026-09-01

| | |
| --- | --- |
| product | `GLOBAL_ANALYSISFORECAST_PHY_001_024` |
| dataset | `cmems_mod_glo_phy-thetao / -so / _anfc_0.083deg_P1D-m` |
| variables | `thetao`, `so`, `zos` |
| grid | 2041 x 4320 at 0.083 deg |
| cadence | daily |
| depths | 50 levels, 0.49 m to 5727.92 m |
| access | `copernicusmarine` 2.4.1 over Zarr, account required |

**There is no API key.** The toolbox takes a username and password; every
documented form is that same pair in a different wrapper. OPeNDAP, ERDDAP,
MOTU, FTP and WMS were all retired in April 2024, so there is no route the
stdlib could take — which makes the dependency unavoidable rather than a
preference. It installs in 22 s and imports in 1.8 s, so the cost the site's
plan worried about in August is small, and it is no longer a first: three
pipelines `pip install` already.

## What was measured, 2026-09-01

| | measured |
| --- | --- |
| one global frame | **33.6 MB** float32 |
| transfer | 3.6 MB/s (9 s a frame) |
| five depths against one | about 5x — **linear** |
| toolbox install | 22 s |

**Every transfer figure above was measured through the toolbox's DEFAULT dask
blocks, which read about 26x the bytes a level needs** (found 2026-09-01: 50.7 s
a level that way, 1.9 s with `chunk_size_limit=0`, on the same store). They
were true of a path the build no longer takes. The fetcher opens with no dask
now and refuses a dask-backed array; the site's `PLAN.md` carries the
measurement and the runner's own numbers are in the run logs, timestamped per
line.

**Depths are priced individually**, which the `depth: 1` chunking predicts
and the timing confirms: currents spent about 4 minutes on five levels
against 36 s for one, temperature 65 s against 9 s. So a set costs what its
levels cost, and ESPC's per-depth instinct carries over even though its
access method does not.

### The chunk shape decides usability, and it is free to ask

The most useful thing learned, and it cost two wrong answers.

| dataset | chunks | one global frame |
| --- | --- | --- |
| currents, 6-hourly | `time 1, lat 512, lon 2048` | 8 chunks |
| temperature / salinity, daily | `time 1, lat 512, lon 2048` | 8 chunks |
| all-variables, **daily** (`zos`) | `time 1, lat 512, lon 2048` | 8 chunks |
| sea level, merged, **hourly** | `time 3648, lat 16, lon 16` | **34,442 chunks** |
| all-variables, **hourly** (`zos`) | `time 3216, lat 16, lon 16` | **34,442 chunks** |

**In this family the hourly products are chunked as time series and the daily
and 6-hourly ones as maps.** A 16x16 pixel tile holding 3,648 time steps is
built for pulling a long series at one point; a global frame off it reads
about 34,442 chunks and discards nearly all of them. That is a property of
the store, not a performance problem.

It was learned twice, expensively: `merged-sl` took **866 s** for one frame,
and the hourly replacement reached for next — by reasoning from the other
datasets rather than asking — ran **21 minutes** and was still going when the
30-minute job cap killed the run. Both times the answer was sitting in
`preferred_chunks`. `scripts/probe-mercator-chunks.py` in the site repository
now asks it in **27 seconds**, and carries the known-good datasets as a
positive control, because a probe that only ever prints "bad" cannot be
trusted to notice.

**Sea surface height therefore comes from the DAILY all-variables product.**

## The tile tier

**20 degree boxes at native 1/12 deg** -- exactly 240 cells a side, which is
why the box is 20 and not 15 -- matching the ESPC currents so the two models
tile alike and a reader zooming in makes the same shape of request of either.

**It costs nothing extra upstream.** The fetch already pulls the full native
global frame in order to decimate it for the 1 degree overview; a tile is a
slice of that same array. What it costs is published bytes and the time to
serialise them.

**One tier per depth AND per lead.** A forecast frame pointing at lead 0's
tiles would draw the present at 1/12 deg and call it the forecast -- sharp,
plausible and wrong, which is the failure that looks most like success. The
contract refuses a lead file whose `tileIndex` is not its own set.

Three faults were found building it, each by a check rather than by a reader:

- **Rows overlapped by one cell.** A row holds the cells whose center lies in
  [s, s+20) and latitude descends, so the start is `floor(..) + 1`, not
  `ceil` -- which returns the boundary itself when the division is exact, and
  it is exact at every 20 degree edge on a 1/12 degree grid. 2049 rows
  covering a 2041-row grid.
- **Two lattice mutations SURVIVED the first test**, both for one reason: with
  this grid's `lo1 = 0` every box edge divides exactly, so `floor` and `ceil`
  agree everywhere. The lattice now derives its origin from the grid rather
  than from a constant, and the test carries a second geometry whose origin is
  not on a box edge.
- **The grid-step guard read float32 noise as a re-grid.** Coordinates are
  stored as float32, whose precision near 180 degrees is about 1.5e-5, so
  differencing two adjacent longitudes returns 0.0833282470703125 for a grid
  that is exactly 1/12. Derived from the whole span the error divides by 4,319
  intervals and the step comes back to 1.2e-9. Reproduced locally before the
  tolerance was touched.

And one that was not the tier at all: **the `--tile-key` probe runs in `plan`,
which had no Copernicus credentials**, so every product reported "tiles
unplanned this run" -- a message naming the tiles rather than the cause.

Caches are per product. The key is computed per product, so products sharing
one cache name would each save over the others and every restore would be
somebody else's tier.

## 2026-09-01 — a 29 m temperature and heat content, the owner's ask

Two more products off the same daily temperature dataset. **`temp29-mercator`**
is level index 14, 29.445 m -- the owner asked for "30 m (z14)", and the root
is named for the depth that is actually there, as the currents' `-47m` is;
one string to change if "30" is preferred. **`ohc-mercator`** is tropical
cyclone heat potential, the quantity ESPC's `ohc-navy` publishes: every level
to 300 m plus one (29 reads, to 318.127 m), integrated by a streamed
vectorized form of ESPC's `heat_potential` -- same constants, same three
answers (absent where the surface is at or below 26 C, a number where the
column crosses 26 C or hits the seafloor, absent and COUNTED where it is still
warm at the bottom of the read). The site's fetcher self-test imports ESPC's
function and uses it as the oracle on 400 random columns; five mutations of
the integral each fail it. Predicted: about 33 level reads a build, two
minutes.

**Measured, run 33578824264, 2026-09-02 01:18Z, a 4-cpu runner:** all five
products fetched and written in **63 s** -- the surface scalars about 11 s
each, the 29 m temperature 10 s, and the heat content **17 s** for its 29
levels, with **0 columns** still above 26 C at 318 m. It deployed: the fields
origin served both roots and their tiers within a minute, `temp29-mercator`
at `depth 29.44` and 158 tiles, `ohc-mercator` in kJ/cm2 with **62 tiles**
-- the tropics, and nothing where the surface is below 26 C, which is what
"not applicable" should look like. Half the two-minute prediction, on the
safe side again.

## 2026-09-02 — ice concentration, thickness and velocity, and the bottom temperature, the owner's ask

Four products off `cmems_mod_glo_phy_anfc_0.083deg_P1D-m`, the daily 2-D
dataset the sea surface height already reads -- map-shaped chunks, eight a
frame, so each is one more eight-chunk read. `sic-mercator` is `siconc`,
`sit-mercator` is `sithick`, `icevel-mercator` is the `usi`/`vsi` pair in the
currents' file shape at 0.001 m/s, and `bottomt-mercator` is `tob`, the
model's bottom-layer temperature.

**Zero is not missing in this model's ice.** Measured 2026-09-02 over a
Greenland Sea box (lat 55-85, lon -70 to 0): 79.5% of the wet cells carried a
thickness of exactly 0 m and 78.9% a drift of exactly 0 m/s; under 15%
concentration, 96.1% of the cells had `usi == 0` and 96.8% `sithick == 0`.
ESPC's `sit-navy` is null over open water (86.5% of its cells, read off the
live root the same day), and a particle layer draws a zero velocity as still
water. So thickness and velocity are nulled where `siconc < 0.15` -- the
map's own floor for concentration, `FIELDS.sic.drawAbove`, which the
fetcher's self-test reads off the site's `scalar-layer.ts` and refuses to
drift from -- and the concentration itself publishes unmasked, zeros and all,
as ESPC's does. Globally that nulled 83.3% of the wet cells in each of the
three ice products. Five mutations of the mask and its wiring each failed the
self-test before it was believed; a sixth was found to be a no-op edit and
redone.

**The ice velocity publishes here, not with the currents** -- D4. Same
dataset, same daily hour as every scalar here; the currents are 6-hourly
bracketing pairs with their own probe and their own fault domain.

**Measured, locally, 2026-09-02:** the four products in about 34 s end to end
-- concentration 3 s of fetch and 159 tiles (37 MB), thickness 3 s and 58
tiles (14 MB), velocity 7 s and 58 tiles (24 MB), bottom temperature 4 s and
159 tiles (48 MB); roots 273, 307, 585 and 349 KB. Under the pack the 99th
percentile of ice speed was 0.24 m/s against the surface current's 1.01, so
the map's ramp for the drift tops out at 0.5 m/s with the currents' own
headroom. Bottom temperature ran -2.4 to 35.5 degrees C globally and draws on
the surface temperature's scale so the two read against each other. The
contract passed all four roots and their tiers in isolation.

**Measured on the runner, run 33594265757, 2026-09-02 05:20Z, 4 cpus:** the
four in **34.3 s** -- concentration 9.1 s, thickness 7.9 s, velocity 7.3 s,
bottom temperature 10.0 s, each about a third fetch and two thirds tiles --
inside a nine-product fetch of **54 s**; 83.3% of the wet cells nulled in
each ice product, the same as locally, and all nine fates `fresh` on the
first run. The tile-cache Save steps were skipped, all nine, which is the
known inert Save and not this change.

## Open

1. **More leads.** The upstream carries forecast to +9 days and this
   publishes two steps; leads are the cheap axis (a lead is one more pass at
   about a minute) and the map's ladder already lists whatever is published.
   *(The set count, the published resolution and the layer names were all
   settled on 2026-09-01 -- D3 and its amendment, the tile tier record above,
   and the site's switcher -- and are no longer open.)*
2. **Whether the two halves share a cadence.** Currents are 6-hourly and the
   scalars daily upstream, so a shared publish hour is a decision, not a
   given.

## Method note

Every number above has a run behind it, taken against the live service on
2026-09-01 from `scripts/measure-mercator.py` and
`scripts/probe-mercator-chunks.py` in the site repository. The transfer rates
are one sample each on a GitHub runner and will move with the day; the chunk
shapes are properties of the store and will not.
