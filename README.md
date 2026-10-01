# hindsight
Watch an exoplanet get discovered, archive snapshot by archive snapshot, in your browser. No server, no install.

## 1. Verdict

**GO, conditional on Gate-0.** Project: **Hindsight**. You open a browser tab and watch an exoplanet signal emerge from the archive, snapshot by snapshot.

## 2. The wow

Hindsight stores TESS light curves as an Apache Iceberg table. Each TESS sector is committed as its own snapshot. A static web page runs DuckDB-Wasm (DuckDB compiled to WebAssembly), time-travels through those snapshots, and runs a transit search in the browser at each one. You drag a slider from "sector 1" to "today" and watch the periodogram peak sharpen as the archive grows. Every view has a permalink pinned to an Iceberg snapshot ID. A paper or a forum post can link to "the data exactly as the archive held it on date D," and the link reproduces the result with no server.

**10-second demo:** Drag the slider. Noise turns into a single sharp peak, and the phase-folded light curve snaps into a clean transit dip. A label reads "detectable from snapshot 4 of 9." The network tab shows only static file fetches and no backend.

## 3. Prior art and gap

- **Lightkurve.** Lightkurve is the standard open-source Python package for Kepler and TESS time series ([Medium intro](https://medium.com/@nunorc/hunting-exoplanets-in-kepler-data-introduction-to-lightkurve-87ece24aa5f6)). It can build a periodogram with the Box Least Squares (BLS) method. Gap: it needs a Python environment, and it always fetches the current archive state.
- **lightkurve-website.** A web interface for Kepler and TESS analysis built on Lightkurve, with Lomb-Scargle and BLS periodograms ([GitHub](https://github.com/yosefmiller/lightkurve-website)). Gap: server-side, unmaintained for years, no history.
- **MAST TIKE.** A free Jupyter-based cloud platform for analyzing MAST's time-series data on AWS ([AWS registry](https://registry.opendata.aws/collab/stsci/)). Gap: needs a hosted notebook server.
- **DuckDB Iceberg in the browser.** DuckDB-Wasm can query Iceberg REST catalogs from a browser tab, with computation staying local ([DuckDB blog](https://duckdb.org/2025/12/16/iceberg-in-the-browser)). Gap: it's a generic demo tied to S3 Tables credentials, with no domain use.
- **AXS.** A Spark-based framework that stores light curves as vector columns in object catalogs ([arXiv](https://arxiv.org/pdf/1905.09034)). Gap: needs a cluster.

**Gap:** I found no tool that does serverless, browser-only light-curve analysis pinned to an archive version. I also found no Iceberg-based light-curve store. Not finding one in search does not prove none exists.

## 4. How it works

```
[Builder CLI, Python]                    [Static host]              [Browser]
TESS SPOC light curves ──► pyiceberg ──► Iceberg files     ──HTTP──► DuckDB-Wasm + iceberg ext
one append = one sector    SqlCatalog    (metadata, Avro,  Range    iceberg_scan(..., snapshot_from_id)
                           (SQLite)       Parquet)                    ──► BLS in SQL (or Pyodide)
                                                                      ──► plot + permalink
```

**Role of each ingredient:**

- **Light curves** are the core data. Schema: `tic_id, sector, time, flux, flux_err, quality`.
- **Wasm** is load-bearing. DuckDB-Wasm reads Iceberg and runs the search client-side. That makes the static host the only infrastructure, so hosting cost is near zero.
- **Iceberg** is load-bearing, with a caveat. If the only goal is "replay by sector," a plain Parquet file with `WHERE sector <= N` does the same job, and Iceberg would be decorative. Iceberg earns its place for two reasons:
  1. Snapshots capture *what the archive contained at a point in time*, including reprocessed or replaced data. A sector filter cannot show that.
  2. Snapshot IDs give reproducible permalinks.

  The MVP must demo at least one overwrite snapshot (a reprocessed sector) to justify Iceberg. Otherwise, drop it.

**Supporting facts:**
- `iceberg_scan` reads a table from a path or a metadata file without an attached catalog, and `iceberg_snapshots` lists a table's snapshots ([docs](https://duckdb.org/docs/current/core_extensions/iceberg/iceberg_functions)).
- PyIceberg needs no JVM and can use a local SQLite-backed SQL catalog ([PyIceberg](https://py.iceberg.apache.org/)).

## 5. Minimal infrastructure and prerequisites

- macOS laptop, modern Chrome or Firefox.
- `uv` (Python 3.11+). Packages: `pyiceberg[sql-sqlite,pyarrow]`, `lightkurve`, `duckdb` (native, for a sanity check).
- Node.js for `npx http-server` (a static server with CORS).
- **Accounts: none.** TESS data is public. The TESS-SPOC products sit in the public `stpubdata` bucket, readable without an AWS account ([registry](https://registry.opendata.aws/mast-tess-spoc/)). Gate-0 can also use Lightkurve's MAST download.
- MVP only: one static host with CORS and HTTP Range support, such as GitHub Pages or a public R2 bucket. [ASSUMPTION: Range support on the chosen host must be verified.]

## 6. Gate-0 spike

**Hypothesis:** A static page running DuckDB-Wasm can read a pyiceberg-written table over plain HTTP at chosen snapshots. It can then recover a known planet's period in under 5 seconds per snapshot, and the detection strength rises across snapshots.

**Steps (time box: 4 hours):**

1. **Build the table (45 min).**
   ```bash
   uv init hindsight-spike && cd hindsight-spike
   uv add "pyiceberg[sql-sqlite,pyarrow]" lightkurve duckdb
   ```
   Write `build.py`:
   - Pick one confirmed TESS planet host with at least 6 sectors of 2-minute data. Take its published period from the NASA Exoplanet Archive. [ASSUMPTION: you choose the target and look up the period.]
   - Download its SPOC light curves with Lightkurve.
   - Create the table in a `SqlCatalog` and call `table.append()` once per sector. Then overwrite one sector to simulate a reprocessing.
2. **Native sanity check (15 min).** In native DuckDB, run `SELECT * FROM iceberg_snapshots(...)`. Then run `iceberg_scan(..., snapshot_from_id=...)` and confirm the row counts grow with each snapshot.
3. **Serve over HTTP (10 min).**
   ```bash
   npx http-server ./warehouse -p 8080 --cors
   ```
   [ASSUMPTION: http-server honors Range requests.]
4. **Wasm read (80 min).** Build one HTML page that loads `@duckdb/duckdb-wasm` from jsDelivr, runs `LOAD iceberg`, and calls `iceberg_scan('http://localhost:8080/.../vN.metadata.json')`.
   - pyiceberg writes absolute `file://` paths into its manifests. If the browser cannot resolve them, map each path to its HTTP URL with `db.registerFileURL()`. [ASSUMPTION: this works for paths referenced inside manifests.]
5. **Search (60 min).** Implement BLS-lite in SQL: cross-join a period grid of about 2,000 values, bin by phase, and score the deepest bin against the noise. If it's too slow, load astropy's `BoxLeastSquares` through Pyodide. Pyodide 0.28.3 lists astropy 7.0.1 among its built packages ([Pyodide](https://pyodide.org/en/0.28.3/usage/packages-in-pyodide.html)). [ASSUMPTION: the BLS C extension is included in that build.]
6. **Record results (30 min).** Log time per snapshot, the peak period, and the peak signal-to-noise ratio (SNR).

**Pass (all three):**
- (a) Wasm reads at least 3 distinct snapshots over HTTP with no REST catalog.
- (b) Query plus search takes under 5 seconds per snapshot on Apple Silicon.
- (c) At the final snapshot, the recovered period is within 1% of the published value, and peak SNR rises monotonically or near-monotonically across snapshots.

**Fail:**
- (a) fails after 90 minutes → **NO-GO** on "static Iceberg." Fall back to a local REST catalog, which adds a server and weakens the zero-infrastructure story. Re-evaluate before continuing.
- (b) fails in both SQL and Pyodide → reduce the scope to precomputed periodograms. The wow drops, so treat this as NO-GO.

## 7. Install and distribution

- **Viewer:** open one URL (the static page plus a demo lake).
- **Build your own lake:**
  ```bash
  uvx hindsight build --tic <TIC_ID> --out ./lake
  ```
  [ASSUMPTION: the PyPI name `hindsight` is likely taken. Check availability and pick an alternative such as `hindsight-lc`.]

## 8. Risks and assumptions

**Top risks:**
1. **Static Iceberg in Wasm is off the documented path.** Browser support is documented for REST catalogs; a path-based scan over plain HTTP in Wasm is unverified. This is the main Gate-0 risk.
2. **Iceberg could look decorative.** Without a real reprocessing or overwrite story, critics on HN will say "just use Parquet." Lead the demo with an overwrite snapshot and the permalinks.
3. **Commit time is not observation time.** Snapshot timestamps record when you committed, not when TESS observed. Store the sector and release date in snapshot summary properties. [ASSUMPTION: pyiceberg `append()` accepts `snapshot_properties`.]
4. **Payload size.** The Wasm bundle plus the iceberg extension may be several MB on first load. [ASSUMPTION]
5. **Science credibility.** BLS-lite in SQL is less rigorous than astropy's BLS. Label it clearly, or use Pyodide BLS for the final view.
6. **Employer IP.** If you work at a data archive, check its IP policy before launching or commercializing. [ASSUMPTION about your situation]

**Other assumptions:**
- [ASSUMPTION] Real TESS reprocessings exist that can serve as the overwrite example. Verify with the MAST data release notes.
- [ASSUMPTION] DuckDB-Wasm's iceberg extension accepts `snapshot_from_id` in `iceberg_scan`, as native DuckDB does.

## 9. Launch angle

**HN title:** "Show HN: Watch an exoplanet get discovered, sector by sector, in your browser (DuckDB-Wasm and Iceberg time travel)"

**Communities and why they would engage:**
- **Hacker News.** Space, Wasm, and "no backend" each draw interest on their own. Combined, they make a strong screenshot.
- **r/dataengineering.** It's a real, non-dashboard use of Iceberg time travel, and it reads Iceberg from a static host with no catalog server. Expect debate about whether Iceberg beats plain Parquet here, which also drives engagement.
- **r/webassembly and the DuckDB community.** It's a domain demo of DuckDB-Wasm's Iceberg support, which the DuckDB team has promoted.
- **Astronomy software community (Astropy forum, r/exoplanets).** It offers reproducible "data as of publication" links and zero-install vetting.

## 10. If given more time

- Add a second Iceberg table for the TESS Objects of Interest (TOI) catalog, so viewers can see when a signal was flagged next to when it became detectable.
- Support multiple targets, using partition pruning so the browser fetches only one star's files.
- Load ZTF or other survey light curves as additional tables to test the pattern beyond TESS.
- Add an "embed this snapshot" iframe for papers and blogs.
- Detect regressions: flag stars whose detection got *worse* after a reprocessing.

---

**Correction to my earlier answer:** I said "jev" wasn't a widely known term. The DuckDB blog has a recent post titled "Jev and DuckDB: Plain-English Conditions in SQL," dated 2026-09-29. If that's the jev you meant, it could fit back into this idea, for example as plain-English filters over light curves. I'd want to read the post before claiming it's load-bearing.
