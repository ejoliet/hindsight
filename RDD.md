# Hindsight

> Watch an exoplanet get discovered, archive snapshot by archive snapshot, in your browser. No server, no install.

![CI](https://github.com/<org>/hindsight/actions/workflows/ci.yml/badge.svg)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

> ⚠️ **Status: spec only (RDD Type A).** Gate-0 has not run. Phase 0 below must pass before any other phase starts.

---

## Purpose

**Problem**: Light-curve analysis tools fetch the archive *as it is today*. Nobody can easily show what the archive held on a past date, or share a link that reproduces a result against that exact data. Existing tools also need a Python environment or a hosted notebook server.

**Solution**: Store each target's TESS light curves as an Apache Iceberg table, with one commit per sector added or reprocessed. A static web page reads any snapshot over HTTP and runs a transit search compiled to WebAssembly (Wasm). A slider moves through snapshots; a permalink pins one.

**Scope (v1)**: A public demo page with a few curated targets, plus a CLI so anyone can build their own lake and host it on any static host.

---

## Architecture

```
 BUILD (laptop, Python)                       HOST (static)                 BROWSER
 ┌───────────────────────┐                    ┌───────────────┐             ┌───────────────────────────────┐
 │ hindsight build       │  Iceberg files     │ lake/         │  HTTP GET   │ lake.js   Icebird + PathMap   │
 │  fetch  (lightkurve)  │ ─────────────────► │  <table>/     │  + Range    │           resolver            │
 │  lake   (pyiceberg)   │  + snapshots.json  │  snapshots.json│ ──────────► │ search.js bls-wasm (Rust)    │
 │  index  (manifest)    │                    │ web/ (app)    │             │ plot.js   uPlot               │
 └───────────────────────┘                    └───────────────┘             │ app.js    slider, permalink   │
                                                                            └───────────────────────────────┘
```

**Data flow**:
- MAST (TESS SPOC 2-min light curves) → `fetch` → normalized Arrow table per sector.
- Arrow table → `lake` → one Iceberg commit per sector (`append`), or per reprocessing (`overwrite` of that sector).
- Iceberg metadata → `index` → `snapshots.json` (ordered list the browser can read without directory listing).
- Browser: `snapshots.json` → pick snapshot → Icebird reads that snapshot's metadata file → rows → `bls-wasm` → periodogram + folded curve → uPlot.

**Key components**:

| Component | Responsibility |
|-----------|---------------|
| `builder/` (Python) | Fetch light curves, write Iceberg commits, emit `snapshots.json` |
| `bls/` (Rust → Wasm) | Box Least Squares (BLS) transit search; the only compute in the browser |
| `web/lake.js` | Read a snapshot with Icebird; rewrite absolute file paths to HTTP URLs |
| `web/app.js` | Slider, target picker, permalink state, error display |
| `web/plot.js` | Light curve, periodogram, phase-folded plots |

> 💡 **Why each ingredient is load-bearing.**
> - **Light curves**: the product.
> - **Iceberg**: snapshots record what the archive held at each point, *including reprocessed data*. A `WHERE sector <= N` filter on plain Parquet cannot show a reprocessing. If v1 ships without a real or clearly labeled overwrite snapshot, Iceberg is decorative and should be dropped.
> - **Wasm**: the transit search runs client-side at near-native speed, so a static host is the only infrastructure. Reading Iceberg is done in pure JavaScript (Icebird), not Wasm.

> 💡 **Design change from the original pitch.** The pitch read Iceberg with DuckDB-Wasm. The read path now uses Icebird: it reads Iceberg tables in pure JavaScript, works with file-based catalogs, and time-travels by metadata file name ([icebird](https://github.com/hyparam/icebird)). Practitioners on HN noted that DuckDB-Wasm blobs run 30–40 MB, while hyparquet is about 10 KB ([HN](https://news.ycombinator.com/item?id=43857856)). DuckDB-Wasm stays as the fallback.

---

## Recommended Stack

Versions retrieved from PyPI and npm on 2026-10-01. GitHub stars were not retrievable (API rate limit) except where noted.

| Layer | Chosen | Latest release | Why chosen | Rejected |
|-------|--------|----------------|------------|----------|
| Iceberg writer | `pyiceberg` 0.12.0 | 2026-09-01 | No JVM; local SQLite `SqlCatalog`; `append`/`overwrite` | `duckdb-iceberg` writes need a REST catalog; `iceberg-rust` has a smaller Python surface |
| Light-curve fetch | `lightkurve` 2.6.0 | 2026-04-16 | Standard TESS/Kepler package; handles SPOC files | `astroquery` (lower level), raw `stpubdata` S3 (more code) |
| CLI | `typer` 0.27.2 | 2026-08-28 | Typed, minimal boilerplate | `click` 8.5.0 (more verbose), `argparse` |
| Browser Iceberg reader | `icebird` 0.8.34 | 2026-09-29 | Pure JS, static tables, time travel, custom resolver; ~129 stars ([hyparam](https://github.com/hyparam)) | `@duckdb/duckdb-wasm` 1.33.1-dev57.0 (30–40 MB; fallback), DataFusion Wasm (no Iceberg path verified) |
| Parquet decode | `hyparquet` 1.31.2 (via Icebird) | 2026-09-27 | ~954 stars, zero deps | — |
| Transit search | Rust crate `bls` → `wasm-pack --target web` | — | Small Wasm, testable against astropy | Pyodide + astropy 7.0.1 (tens of MB), SQL BLS in DuckDB-Wasm, `transitleastsquares` 1.32 (Python only, last release 2024-04-05) |
| Test oracle | `astropy` 8.0.1 `BoxLeastSquares` | 2026-07-05 | Reference results for `bls` tests | — |
| Plots | `uPlot` 1.6.32 | 2025-03-14 | Fast for 10⁴–10⁵ points, small | `@observablehq/plot` 0.6.17 (slower at this size), `plotly.js` 4.1.1 (heavy) |
| Python tooling | `uv` | — | Reproducible env, `uvx` distribution | pip + venv |
| Hosting (v1) | GitHub Pages | — | Free, static, CI deploy | Cloudflare R2 public bucket (for lakes > Pages limits) |

> ⚠️ **Override round.** Review this table before Phase 1. The main open choice is Rust port vs. Emscripten build of astropy's C BLS (see Open Questions).

---

## Repository Layout

```
hindsight/
├── builder/                     # Python package, published as `hindsight-lc` [name TBD in Open Questions]
│   ├── pyproject.toml
│   ├── src/hindsight/
│   │   ├── cli.py               # Typer app: build, index, verify
│   │   ├── config.py            # Settings from env vars
│   │   ├── fetch.py             # lightkurve → normalized Arrow table per sector
│   │   ├── lake.py              # pyiceberg catalog, append/overwrite commits
│   │   ├── index.py             # Emit snapshots.json from table metadata
│   │   └── errors.py            # Named exceptions
│   └── tests/
│       ├── conftest.py          # Synthetic light-curve fixtures
│       ├── test_fetch.py
│       ├── test_lake.py
│       └── test_index.py
├── bls/                         # Rust crate → Wasm
│   ├── Cargo.toml
│   ├── src/lib.rs               # bls_search(), fold()
│   └── tests/oracle.rs          # Compare against astropy fixtures
├── web/                         # Static app, no bundler
│   ├── index.html
│   ├── app.js                   # State, slider, permalink
│   ├── lake.js                  # Icebird read + PathMap resolver
│   ├── search.js                # Wasm loader + typed-array marshalling
│   ├── plot.js                  # uPlot wrappers
│   ├── vendor/                  # Pinned icebird, hyparquet, uPlot (committed)
│   └── pkg/                     # wasm-pack output (built in CI, gitignored)
├── fixtures/                    # astropy BLS reference outputs (JSON)
├── spike/GATE0.md               # Gate-0 log: commands, timings, pass/fail
├── .github/workflows/
│   ├── ci.yml                   # lint + tests (Python, Rust, JS)
│   └── pages.yml                # Build Wasm, deploy web/ + demo lake
├── Makefile
├── LICENSE                      # MIT
└── README.md
```

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| macOS or Linux | — | Developed on macOS (Apple Silicon) |
| Python | 3.11+ | pyiceberg requires ≥ 3.10; astropy 8 requires ≥ 3.11 |
| uv | latest | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Rust | stable | `rustup target add wasm32-unknown-unknown` |
| wasm-pack | latest | `cargo install wasm-pack` |
| Node.js | 20+ | Only for `npx http-server` and JS tests |
| Browser | current Chrome / Firefox / Safari | Wasm + ES modules |

**Accounts and permissions**: none for local work. TESS data is public. GitHub Pages deploy uses the repo's built-in `GITHUB_TOKEN` in Actions; no personal tokens.

---

## Quick Start

> ⚠️ Commands below are the target interface. They do not exist until Phases 1–4 are built.

```bash
# 1. Clone
git clone https://github.com/<org>/hindsight.git && cd hindsight

# 2. Build a lake for one target
uv run --directory builder hindsight build --tic <TIC_ID> --out ../lake

# 3. Build the Wasm search
make wasm            # wasm-pack build bls --target web --out-dir ../web/pkg

# 4. Serve app + lake locally
make serve           # npx http-server . -p 8080 --cors

# 5. Open
open "http://localhost:8080/web/#lake=/lake&t=<TIC_ID>"
```

Published use (after v1): open the demo URL, or run `uvx hindsight-lc build --tic <TIC_ID> --out ./lake` and host `./lake` anywhere static.

---

## Configuration Reference

Builder only. The web app takes no config beyond URL hash parameters. No secrets exist in this project.

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `HINDSIGHT_WAREHOUSE` | `path` | `./lake` | | Local warehouse root. Overridden by `--out`. |
| `HINDSIGHT_CATALOG_DB` | `path` | `<warehouse>/catalog.db` | | SQLite file for pyiceberg `SqlCatalog`. Not published. |
| `HINDSIGHT_PUBLIC_BASE` | `url` | `""` | | Base URL written into `snapshots.json` as `path_map.to`. Empty means relative to the lake root. |
| `HINDSIGHT_CACHE_DIR` | `path` | lightkurve default | | Download cache for FITS files. |
| `LOG_LEVEL` | `str` | `INFO` | | `DEBUG`, `INFO`, `WARNING`, `ERROR` |

**URL hash parameters (web)**:

| Param | Example | Meaning |
|-------|---------|---------|
| `lake` | `/lake` or `https://…/lake` | Lake root containing `snapshots.json` files |
| `t` | `<TIC_ID>` | Target |
| `s` | `"5124…"` | Snapshot ID (string). Omitted = latest. This is the permalink. |

---

## API / Interface Contract

### CLI

```
Usage: hindsight [OPTIONS] COMMAND [ARGS]...

Commands:
  build    Fetch light curves for a target and commit one snapshot per sector.
  reprocess  Overwrite one sector with a newer product version (new snapshot).
  index    Regenerate snapshots.json from table metadata.
  verify   Read every snapshot back and check row counts and path mapping.

build options:
  --tic INTEGER          TESS Input Catalog ID. Required.
  --out PATH             Warehouse root. Default: $HINDSIGHT_WAREHOUSE.
  --sectors TEXT         Comma list or range, e.g. "1-13". Default: all available.
  --cadence [120]        Seconds. v1 supports 120 only.
  --published-period FLOAT  Optional; stored in snapshots.json for the demo label.
  --dry-run              List sectors that would be committed, then exit.

reprocess options:
  --tic INTEGER --sector INTEGER --source PATH   FITS file of the newer product.
  --note TEXT            Human label, e.g. "SPOC reprocessing r2".
```

### Iceberg table (one table per target)

Namespace `hindsight`, table `lc_tic<TIC_ID>`, format version 2, unpartitioned.

| Column | Type | Notes |
|--------|------|-------|
| `tic_id` | `long` | Constant per table |
| `sector` | `int` | TESS sector |
| `time_btjd` | `double` | Barycentric TESS Julian Date |
| `flux` | `float` | PDCSAP flux, normalized to median 1.0 per sector |
| `flux_err` | `float` | Normalized with the same factor |
| `quality` | `int` | SPOC quality bitmask; rows with non-zero quality are dropped at build |
| `product_version` | `string` | From FITS header; changes on reprocessing |

Every commit sets snapshot summary properties [ASSUMPTION: pyiceberg `append(..., snapshot_properties=...)` accepts these; verify in Phase 1]:

| Property | Example |
|----------|---------|
| `hindsight.sector` | `"7"` |
| `hindsight.operation` | `"add"` or `"reprocess"` |
| `hindsight.note` | `"Sector 7 added"` |

### `snapshots.json` (one per target, at `lake/lc_tic<TIC_ID>/snapshots.json`)

> 💡 Needed because static hosts cannot list directories, and pyiceberg metadata file names are not sequential.

```json
{
  "schema_version": 1,
  "tic_id": 123456789,
  "table_path": "lc_tic123456789",
  "published_period_days": null,
  "path_map": { "from": "file:///Users/me/hindsight/lake/", "to": "" },
  "snapshots": [
    {
      "snapshot_id": "5124012345678901234",
      "sequence_number": 1,
      "metadata_file": "metadata/00001-<uuid>.metadata.json",
      "operation": "add",
      "sector": 1,
      "note": "Sector 1 added",
      "committed_at": "2026-10-02T17:04:11Z",
      "row_count": 18210
    }
  ]
}
```

- `snapshot_id` is a **string**: 64-bit IDs exceed JavaScript's safe integer range.
- `path_map.to` empty means "relative to the URL of `snapshots.json`'s parent".

### Wasm search (`bls/src/lib.rs`, exported via `wasm-bindgen`)

```rust
#[wasm_bindgen]
pub fn bls_search(
    time: &[f64],          // BTJD, sorted ascending
    flux: &[f32],          // normalized
    flux_err: &[f32],
    periods: &[f64],       // days, caller-built grid
    durations: &[f64],     // days; each < min(periods)
    oversample: u32,       // phase bins per duration, default 10
) -> Result<BlsResult, JsError>;

#[wasm_bindgen]
pub struct BlsResult {
    // getters return copies as Float64Array
    power: Vec<f64>,       // len == periods.len(); log-likelihood objective
    best_period: f64,
    best_t0: f64,
    best_duration: f64,
    depth: f64,
    snr: f64,              // depth / depth_err at best peak
}

#[wasm_bindgen]
pub fn fold(time: &[f64], period: f64, t0: f64) -> Vec<f64>; // phase in [-0.5, 0.5)
```

Default period grid (built in `search.js`): 2,000 log-spaced periods from 0.5 days to half the snapshot's time baseline; durations `[0.04, 0.08, 0.12, 0.2]` days.

### Web module contracts

```js
// lake.js
export async function loadIndex(lakeUrl, ticId): Promise<SnapshotIndex>
export async function readSnapshot(index, snapshotId): Promise<{time: Float64Array, flux: Float32Array, flux_err: Float32Array, sector: Int32Array}>

// search.js
export async function initSearch(): Promise<void>        // loads web/pkg/bls_bg.wasm
export function runSearch(rows, grid?): BlsResultJS      // sync, in a Web Worker

// app.js — state lives in location.hash; changing the slider updates `s`.
```

`readSnapshot` calls Icebird with `metadataFileName` from the index and a resolver that rewrites `path_map.from` → resolved `path_map.to` for every file path [ASSUMPTION: Icebird's resolver sees manifest and data file paths; verify in Phase 0].

---

## Error Handling

### Builder (Python)

| Error class | When raised | Exit code | Retry? |
|-------------|-------------|-----------|--------|
| `ConfigError` | Bad path or env var | 1 | No |
| `TargetNotFound` | No SPOC 2-min products for the TIC ID | 2 | No |
| `ArchiveUnavailable` | MAST download fails | 3 | Yes, 3× exponential backoff (1 s, 4 s, 16 s) |
| `CommitConflict` | Iceberg commit rejected | 4 | Yes, once after reload |
| `PathMapError` | `verify` finds a file path not covered by `path_map` | 5 | No |

Logs go to stderr as JSON lines: `{"level":"ERROR","error":"TargetNotFound","context":{"tic":…}}`.

### Web

| Error | Cause | User-facing message |
|-------|-------|---------------------|
| `IndexFetchError` | `snapshots.json` 404 or CORS | "Lake not found or blocks cross-origin reads." |
| `RangeUnsupported` | Host ignores HTTP Range | "This host does not support partial reads." |
| `SnapshotReadError` | Icebird fails on a metadata or data file | Shows failing URL |
| `WasmLoadError` | Wasm fetch or instantiate fails | "Search engine failed to load." |
| `SearchInputError` | Empty or unsorted time array | "This snapshot has no usable data." |

No silent fallbacks: every error renders in the status bar with the snapshot ID.

---

## Testing

```bash
make test          # all suites
make test-py       # builder: pytest, synthetic fixtures, no network
make test-rs       # bls: cargo test, astropy oracle fixtures
make test-js       # web: node --test, lake.js against a local fixture lake
make test-e2e      # build fixture lake → serve → headless browser reads 3 snapshots
make lint          # ruff, mypy --strict, cargo clippy -D warnings, eslint
```

| Suite | Covers | Network? |
|-------|--------|----------|
| `test_fetch.py` | Normalization, quality filtering, dtype casts | No (FITS fixtures) |
| `test_lake.py` | One snapshot per sector; overwrite creates a new snapshot; properties set | No |
| `test_index.py` | `snapshots.json` order, string IDs, path_map | No |
| `bls/tests/oracle.rs` | Peak period within 0.1% and power correlation ≥ 0.99 vs. astropy on 3 fixtures | No |
| `web/*.test.js` | Path rewriting, hash state round-trip | No |
| `e2e` | Full read + search in headless Chromium | Localhost only |

Tests never call MAST. A separate manual `make smoke TIC=<id>` hits the real archive.

---

## Non-Goals (v1)

- Multi-target catalog table or cross-target queries.
- Partitioning, compaction, or snapshot expiry.
- Kepler, K2, ZTF, or full-frame-image (FFI) light curves.
- 20-second cadence data.
- Any server: no REST catalog, no API, no auth, no private lakes.
- Writing to the lake from the browser.
- Detrending beyond SPOC PDCSAP; no Gaussian-process or spline detrending.
- Transit Least Squares, multi-planet search, or vetting statistics.
- Mobile-optimized layout (it must work on mobile, not be designed for it).
- Accounts, analytics, or telemetry.

---

## Open Questions

Resolve before the phase that needs them. Do not guess.

- [ ] **Q1 (Phase 0)**: Does Icebird read pyiceberg-written tables when every file path is rewritten by a custom resolver? If not: fall back to DuckDB-Wasm `iceberg_scan` with `registerFileURL`, then re-decide the Wasm story.
- [ ] **Q2 (Phase 0)**: Does the chosen host serve HTTP Range requests with CORS? Test GitHub Pages and `npx http-server`.
- [ ] **Q3 (Phase 1)**: Is there a real SPOC reprocessing of a demo target's sector, with both versions still downloadable? If not, the demo uses a *labeled, simulated* reprocessing, and the launch copy must say so.
- [ ] **Q4 (Phase 2)**: Rust port of BLS vs. Emscripten build of astropy's C implementation (BSD-3). Rust is simpler to test and toolchain-light on macOS; Emscripten reuses proven code. Default: Rust.
- [ ] **Q5 (Phase 1)**: Demo targets. Need 2–3 confirmed TESS planet hosts with ≥ 6 sectors of 2-min data and a published period from the NASA Exoplanet Archive.
- [ ] **Q6 (Phase 5)**: PyPI and repo name. `hindsight` is likely taken; check `hindsight-lc` and alternatives.
- [ ] **Q7 (Phase 5)**: Lake size per target vs. GitHub Pages limits. If too large, host lakes on a public R2 bucket.
- [ ] **Q8 (Phase 3)**: Search runtime per snapshot. Target < 5 s for ~13 sectors on Apple Silicon. If exceeded, reduce the grid or add progressive refinement.

---

## Agent Build Instructions

> This section is the build specification. Implement using only this README. Anything ambiguous is listed in Open Questions; stop and ask rather than guess.

### Build Order

| Phase | Deliverable | Done when |
|-------|-------------|-----------|
| 0 | **Gate-0 spike** in `spike/` (throwaway code allowed) | `spike/GATE0.md` records: Q1 and Q2 answered; ≥ 3 snapshots read in browser; recovered period within 1% of published; time per snapshot. Fail → stop and report. |
| 1 | Builder: `fetch`, `lake`, `index`, CLI `build`/`reprocess`/`index` | `make test-py` passes; `hindsight build --dry-run` lists sectors |
| 2 | `bls` crate + astropy oracle fixtures | `make test-rs` passes; `wasm-pack build` output < 300 KB |
| 3 | Web: `lake.js`, `search.js` (Web Worker), `plot.js`, `app.js` | `make test-js` and `make test-e2e` pass |
| 4 | `verify` command + error surfaces | Each error class in Error Handling has a test |
| 5 | CI + Pages deploy + demo lakes | Public URL loads a demo target and permalink round-trips |

### File Map

| File | Purpose | Key symbols |
|------|---------|-------------|
| `builder/src/hindsight/config.py` | Env settings | `class Settings(BaseSettings)` |
| `builder/src/hindsight/fetch.py` | Download + normalize | `fetch_sectors(tic, sectors) -> dict[int, pa.Table]`, `normalize(lc) -> pa.Table` |
| `builder/src/hindsight/lake.py` | Iceberg writes | `open_catalog(settings)`, `ensure_table(catalog, tic)`, `commit_sector(table, sector, data, note)`, `reprocess_sector(...)` |
| `builder/src/hindsight/index.py` | Index emit | `build_index(table, path_map) -> dict`, `write_index(path, index)` |
| `builder/src/hindsight/cli.py` | Typer CLI | `app`, `build`, `reprocess`, `index`, `verify` |
| `builder/src/hindsight/errors.py` | Exceptions | classes in Error Handling |
| `bls/src/lib.rs` | Search | `bls_search`, `fold`, `BlsResult` |
| `web/lake.js` | Read | `loadIndex`, `readSnapshot`, `makePathMapResolver` |
| `web/search.js` | Wasm wrapper | `initSearch`, `runSearch`, `defaultGrid` |
| `web/worker.js` | Off-main-thread search | message protocol `{type:'search', rows}` |
| `web/app.js` | UI state | `parseHash`, `writeHash`, `onSlide` |
| `Makefile` | Dev commands | `wasm`, `serve`, `test*`, `lint`, `smoke` |

### Constraints

- Python 3.11+, typed signatures, `ruff` + `mypy --strict` clean.
- Rust stable, `cargo clippy -D warnings` clean, no `unsafe`.
- Web: no bundler, no framework, ES modules only. Vendored dependencies are pinned and committed with their licenses.
- No network calls in tests. No telemetry anywhere.
- No secrets, account IDs, or absolute personal paths in committed files. `path_map.from` is written at build time and must not be committed for demo lakes built on a personal machine; CI rebuilds demo lakes.
- Snapshot IDs handled as strings end-to-end in JS.
- Search runs in a Web Worker; the main thread never blocks > 50 ms.
- Grep-friendly notes use `AIDEV-NOTE:`, `AIDEV-TODO:`, `AIDEV-QUESTION:`.

### Acceptance Criteria

- [ ] Phase 0 passed and logged in `spike/GATE0.md`.
- [ ] `make lint` and `make test` pass in CI.
- [ ] On a demo target, the latest snapshot recovers the published period within 1%.
- [ ] Peak SNR is non-decreasing across `add` snapshots for at least one demo target (small dips allowed and explained in the UI note).
- [ ] At least one `reprocess` snapshot exists in the demo and is labeled real or simulated (Q3).
- [ ] A permalink opened in a fresh browser shows the same snapshot and same best period.
- [ ] Network tab shows only static GETs: no POSTs, no third-party calls at runtime.
- [ ] Wasm module < 300 KB; first meaningful plot < 3 s on a warm cache [ASSUMPTION: target, to be measured].
- [ ] All Open Questions resolved or moved to v2.

---

## Next Steps

1. [ ] Run Phase 0 (Gate-0), time-boxed to 4 hours. Answer Q1 and Q2.
2. [ ] Review the Recommended Stack table; decide Q4.
3. [ ] Pick demo targets (Q5) and check for a real reprocessing (Q3).
4. [ ] Agent implements Phases 1–4.
5. [ ] Human review: science check of normalization and BLS output vs. astropy.
6. [ ] Phase 5: CI, Pages deploy, name check (Q6), size check (Q7).
7. [ ] Write `DECISIONS.md` with the Icebird-over-DuckDB-Wasm decision and the Q4 outcome.

---

## References

- Icebird (JS Iceberg client): https://github.com/hyparam/icebird
- DuckDB, "Iceberg in the Browser": https://duckdb.org/2025/12/16/iceberg-in-the-browser
- DuckDB Iceberg functions: https://duckdb.org/docs/current/core_extensions/iceberg/iceberg_functions
- PyIceberg: https://py.iceberg.apache.org/
- TESS-SPOC on AWS Open Data: https://registry.opendata.aws/mast-tess-spoc/
- Pyodide packages (fallback path): https://pyodide.org/en/0.28.3/usage/packages-in-pyodide.html
- Hyperparam Show HN thread: https://news.ycombinator.com/item?id=43857856
