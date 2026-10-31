# Google Health Pipeline Implementation Plan

Status: proposed work order. This is not an elapsed-time commitment.
Date: 2026-10-27.
Read first: [intent](intent.md) and [design](design.md).

## 1. Delivery Strategy

First prove that your own Google account returns useful data and that a token can survive unattended runs. Then build one end-to-end path for a single data type (local DuckDB → MotherDuck → one chart) before adding more types. Each milestone ends in a working, re-runnable state.

## 2. Milestones And Exit Checks

### M0: API Feasibility Spike

**Outcome:** evidence that the Google Health API serves your data and allows a daily unattended job.

- Create a Google Cloud project, enable the Google Health API, and configure the OAuth consent screen (External, yourself as user). Create a Desktop OAuth client.
- Write `scripts/consent.py` (`google-auth-oauthlib`, `access_type=offline`, the 4 read-only scopes, no `include_granted_scopes`). Store the refresh token in the macOS Keychain with `keyring` (design section 9).
- Call `users/me/identity`, then `list` for each core data type over the last 7 days. Record which types return data, from which `dataSource` platforms/devices, and the exact filter syntax per kind. Save **synthetic** copies of the response shapes as fixtures (do not commit real values).
- Before calling the API, wear the Fitbit Air for at least 2 days and nights (including one tagged workout) and sync the Google Health app, so every core type has data.
- Mark each "Verify" type in design section 5 as present or absent for the Air. Note whether Premium-only metrics appear, and whether the `dataSource` device names identify the Air. Also check whether data from your Android apps (via Health Connect) appears (risk R2).
- Move the consent screen to "In production" (unverified) and confirm the refresh token still works after 8+ days (risk R1).

**Exit check:** a short `spike-report.md` with the data-type availability matrix, filter syntax, token-lifetime result, and a go/no-go decision. Confirm decision D3 in the design (D1 and D2 were resolved on 2026-10-28). Note every manual setup step, because it becomes the self-hosting guide.

### M1: Project Skeleton And First Resource (Local DuckDB)

**Outcome:** `steps` loads incrementally into a local `health.duckdb`.

- `uv init`; add `dlt[duckdb,motherduck]`, `dbt-core==1.12.*`, `dbt-duckdb[md]==1.11.*`, `google-auth`, `google-auth-oauthlib`, `keyring`, plus dev tools `ruff`, `pyright`, `pytest` and `pre-commit` (with `gitleaks`). Add `.gitignore` (`.dlt/secrets.toml`, `*.duckdb`, `.env`), then `git init`.
- Credentials (design section 9): a typed `GcpOAuthCredentials` resolved from env vars; `scripts/run_local.py` (Keychain → env for one run); `make reauth` and `make forget-credentials`. Tests: a missing credential fails before any API call; `invalid_grant` fails without retry; a synthetic token string never appears in captured logs.
- Implement `GoogleHealthAuth` (dlt `AuthConfigBase` that refreshes the access token), the `DataTypeSpec` registry, and a generic resource factory with paginator, backoff and `dlt.sources.incremental` with lookback.
- `pipeline.py --destination duckdb --types steps --since 2026-10-01`.
- Tests: paginator and filter construction against synthetic fixtures, a merge-idempotency test (run twice → same row count), and the cursor/lookback window calculation.

**Exit check:** running twice in a row adds no duplicates. Stopping mid-run and re-running finishes the load. `ruff`, `pyright` and `pytest` pass.

### M2: Core Data Types And Backfill

**Outcome:** every core type from design section 5, with history.

- Add registry entries for the interval, sample, daily and session kinds, plus the `dailyRollUp` path for `total-calories`.
- Windowed backfill (30-day chunks, configurable `initial_value`) with client-side pacing of ≤ 4 requests/second.
- Apply a dlt schema contract (`columns: freeze`) on the fields dbt depends on.

**Exit check:** a 2-year backfill completes locally. Row counts per type are logged. A re-run only fetches the lookback window.

### M3: dbt Models And Tests

**Outcome:** trustworthy daily marts.

- Run `uv run dbt init` with the `duckdb` adapter: a `dev` target (local file) and a `prod` target (`md:health`). Declare the dlt raw tables as `sources` and use SQL models (design section 6).
- Build the `stg_*` models (typing, local time from offsets, dedup on `name`) and the `source_priority.csv` seed.
- Build the marts `dim_date`, `fct_daily_activity`, `fct_daily_recovery`, `fct_workouts`, `fct_body` and `fct_heart_rate_hourly`.
- Tests: keys, value ranges, freshness. Add one synthetic edge-case fixture each for DST change, travel across time zones, overlapping phone and watch steps, and sleep across midnight.

**Exit check:** `dbt build` passes locally. Spot-check 3 days against what the Fitbit or Google Health app shows. Any differences are explained (source priority) or fixed.

### M4: MotherDuck And Daily Schedule

**Outcome:** unattended daily refresh in the cloud warehouse.

- Create a MotherDuck account, the `health` database, a read/write token (pipeline) and a read-scaling token (dashboard).
- Switch the destination with config only. Run the backfill once from your Mac into MotherDuck.
- Create a private GitHub repo. Add `.github/workflows/daily.yml`: cron `0 6 * * *` UTC, `uv sync`, `uv run python pipeline.py`, `dbt build --target prod`, with secrets from a GitHub environment `production` (limited to `main`) mapped to dlt and dbt environment variables. Set `permissions: contents: read`, pin actions by SHA, and use only `schedule` and `workflow_dispatch` triggers. Turn on secret scanning with push protection.
- Add a weekly 30-day re-sync job (deletions and edits). Append rows to the `ops.run_log` table. Turn on email notification for failed workflow runs.

**Exit check:** 3 scheduled runs in a row succeed. Revoking the token produces a failed run with a clear `invalid_grant` message, and the documented re-consent steps restore it.

### M5: Dashboard

**Outcome:** a local Streamlit app answering everyday questions.

- Build the `dashboard/` pages Overview, Activity, Recovery & sleep, Workouts and Body (design section 7), using the read-only token and cached queries.
- Show a freshness badge and per-type latest date. Show missing days as gaps, not zeros.

**Exit check:** the app loads in under 3 seconds with a warm cache, and every chart matches the mart values. A README covers run steps, re-consent and deletion.

## 3. Proposed Repository Layout

This pipeline is a separate sub-project. It does not share storage with the local-only SportBud app.

```text
pipelines/ghealth/
  pyproject.toml            # uv project
  .dlt/config.toml          # non-secret config (types, initial_value, lookback)
  .dlt/secrets.toml         # gitignored; not used for credentials (Keychain / env vars)
  google_health/
    __init__.py             # @dlt.source
    auth.py                 # refresh-token auth for RESTClient
    registry.py             # DataTypeSpec entries
    resources.py            # generic resource factory
  pipeline.py               # CLI entry: --destination, --types, --since, --resync-days
  scripts/consent.py        # local OAuth consent -> Keychain
  scripts/run_local.py      # Keychain -> env for one local run
  Makefile                  # reauth, forget-credentials, run, build
  dbt/                      # dbt project (staging, marts, seeds, tests)
  dashboard/app.py          # Streamlit
  tests/fixtures/           # synthetic API responses only
.github/workflows/ghealth-daily.yml
```

## 4. First Implementation Ticket

**Title:** M0 spike: prove Google Health API access and token lifetime for your own account.

**Done when:** `spike-report.md` lists, for each core data type, whether it returned data, the source devices and the working filter syntax. A refresh token has worked after 8 days or more (or the weekly re-consent fallback is accepted). Decision D3 is recorded. No real health values are committed.

## 5. Later Roadmap

1. More data types (SpO2, respiratory rate, sleep temperature) through registry entries.
2. Derived metrics: training load (ACWR from AZM), and the relation between sleep debt and resting HR.
3. Optional exports of aggregated weekly summaries for SportBud. These would go through SportBud's documented import format, and only by an explicit user action.
4. Other people: publish the repo as a template with a setup guide (own Google Cloud project, OAuth client, MotherDuck account, private repo) and `scripts/setup_check.py`. Each person self-hosts, and the builder never gets access to their data or secrets.
