# PRD: Google Health Data Pipeline

## 1. Introduction

Build a self-hosted pipeline that extracts a user's Google Health data, loads it into that user's MotherDuck warehouse, transforms it with dbt, and presents descriptive reports in a local Streamlit dashboard. This gives the user a maintainable way to analyze personal health trends without granting the builder access to health data or credentials.

The system is developed locally with DuckDB and delivered in milestones. An API and policy feasibility spike (M0) gates implementation. The first implementation slice proves one data type end to end through local DuckDB, MotherDuck, dbt, and a minimal report before expanding to the supported data types.

## 2. Goals

- Prove Google Health API access, useful data availability, OAuth behavior, and applicable security-assessment requirements before cloud deployment.
- Extract supported Google Health data daily with dlt and load it into the user's own MotherDuck database.
- Build tested staging models and analytical marts with dbt.
- Provide local, descriptive Streamlit reports that query marts with a read-only MotherDuck token.
- Preserve source provenance and prevent overlapping device data from being double-counted.
- Keep health data and all credentials within accounts managed by the user; never commit real health values or expose secrets in logs.
- Support local DuckDB development and recovery by rebuilding warehouse data from Google Health with a configurable backfill window.

## 3. User Stories

### US-001: Verify Google Health feasibility

**Description:** As the owner of the pipeline, I want to verify API access and token behavior before implementation so that the design is based on data my account can actually provide.

**Acceptance Criteria:**

- [ ] A local consent flow requests only the four planned read-only scope bundles and stores the refresh token directly in macOS Keychain.
- [ ] The spike tests the proposed Desktop OAuth client and `InstalledAppFlow` against Google Health.
- [ ] The spike calls the identity endpoint and queries each candidate core data type over the previous seven days.
- [ ] A report records data availability, source devices/platforms, filter syntax, and whether Android Health Connect-originated data appears.
- [ ] The consent screen is moved to In Production and the refresh token is verified after at least eight days.
- [ ] Google guidance on whether this personal MotherDuck-backed use requires a security assessment is recorded; no cloud health-data load occurs before the requirement is resolved.
- [ ] Only synthetic response fixtures are committed; the report contains no real health values or credentials.
- [ ] A go/no-go decision and any manual setup steps are documented.

### US-002: Run and protect a local pipeline

**Description:** As the owner, I want to run the pipeline locally with credentials from Keychain and data in DuckDB so that I can develop without cloud infrastructure or credentials written to disk.

**Acceptance Criteria:**

- [ ] Local credentials are exposed only to the child process environment for the duration of a run.
- [ ] Missing credentials fail before any API request; `invalid_grant` fails without retry and gives a re-consent instruction.
- [ ] Access tokens remain in memory and tokens, authorization headers, and API response bodies never appear in logs.
- [ ] Local pipeline output is stored in an ignored DuckDB file; `.env` and dlt secret files are ignored.
- [ ] Running the initial `steps` resource twice produces no duplicate records.
- [ ] Ruff, Pyright, and pytest pass.

### US-003: Extract one data type incrementally

**Description:** As the owner, I want the first supported data type to load incrementally so that the pipeline can be validated end to end before adding more data.

**Acceptance Criteria:**

- [ ] A declarative registry defines each type's kind, method, scope, cursor, lookback, page size, and primary key.
- [ ] The generic resource supports API pagination, verified filters, retry/backoff for transient `429`/`5xx` responses, and a maximum client request rate of four requests per second.
- [ ] The first resource loads `steps` into local DuckDB and supports a configurable `--since` value.
- [ ] A rerun merges lookback data idempotently; an interrupted run can resume without duplicating records.
- [ ] Synthetic tests cover filter construction, pagination, cursor/lookback calculation, and merge idempotency.

### US-004: Backfill supported types

**Description:** As the owner, I want supported data types and historical data loaded so that reports can show trends beyond the latest day.

**Acceptance Criteria:**

- [ ] Only data types confirmed useful by M0 are enabled in the initial registry.
- [ ] Supported types include the confirmed interval, sample, daily, session, and rollup-only API methods.
- [ ] The default backfill is two years, is configurable, and proceeds in restartable 30-day windows.
- [ ] Repeated daily loads re-read each type's configured lookback window to account for delayed syncs and edits.
- [ ] A dlt schema contract freezes columns used by dbt; row counts are logged without health values.
- [ ] A synthetic test verifies that interrupted backfill resumes at the correct window.

### US-005: Transform raw records into trusted marts

**Description:** As a report user, I want typed, deduplicated, locally aligned marts so that daily charts are comparable and do not double-count devices.

**Acceptance Criteria:**

- [ ] dbt declares dlt raw tables as sources and builds SQL staging models and marts against local DuckDB.
- [ ] Staging models derive local timestamps and dates from each record's own UTC offset, not the runner's timezone.
- [ ] Sleep sessions are assigned to the local date on which they end; interval records crossing midnight are assigned to their local end date.
- [ ] Per-type source priority is explicit; overlapping sources are not summed, and the selected source is retained in model lineage/columns.
- [ ] Missing measurements remain `NULL`; models do not turn missing days into zero.
- [ ] Marts include daily activity, daily recovery, workouts, body measurements when available, hourly heart rate, and a date dimension.
- [ ] Tests cover keys, accepted value ranges, freshness, daylight-saving changes, travel across time zones, overlapping phone/watch steps, and sleep across midnight.
- [ ] `dbt build` and tests pass locally, and three days of results are checked against the Google Health or device app with discrepancies explained.

### US-006: Prove the first MotherDuck report slice

**Description:** As the owner, I want one data type loaded and modeled in MotherDuck and displayed in a minimal report so that the full deployment path is proven early.

**Acceptance Criteria:**

- [ ] The pipeline can switch between local DuckDB and the user's MotherDuck database by configuration.
- [ ] The first end-to-end slice loads `steps`, builds its dbt mart, and queries it from Streamlit.
- [ ] Pipeline/dbt uses a MotherDuck read/write token; Streamlit uses only a separate read-scaling token.
- [ ] The Google security-assessment requirement is resolved and any required approval is obtained before health data is uploaded to MotherDuck.
- [ ] No credential or health value is added to the repository, CI artifact, or log.

### US-007: Schedule secure daily and weekly runs

**Description:** As the owner, I want unattended refreshes with clear failure behavior so that the warehouse stays current without manual daily operation.

**Acceptance Criteria:**

- [ ] A private GitHub repository runs the daily job at 06:00 UTC and supports manual dispatch; there is no pull-request trigger.
- [ ] The workflow uses a production GitHub environment limited to `main`, least-privilege permissions, SHA-pinned third-party actions, and secrets only in the steps that need them.
- [ ] Runs targeting the same warehouse are serialized and an active run is not cancelled by a later dispatch.
- [ ] One designated daily run per week replaces each type's most recent 30-day window, retaining older records unchanged.
- [ ] A window is replaced only after all pages for that type have been fetched successfully; incomplete extraction preserves the existing window.
- [ ] A run is marked successful and advances the freshness timestamp only after extraction, load, dbt models, and dbt tests pass.
- [ ] Operational logs contain run status, types, row counts, and duration only; no health values, response bodies, or credentials.
- [ ] Failed workflows generate an alert with actionable re-consent guidance for `invalid_grant`.
- [ ] Three consecutive scheduled runs succeed, and revoking a token produces a clear failure that can be recovered through the documented re-consent procedure.

### US-008: View descriptive health reports

**Description:** As the owner, I want local reports over the analytical marts so that I can review personal trends without exposing the dashboard publicly.

**Acceptance Criteria:**

- [ ] The local Streamlit app queries MotherDuck marts with the read-only token and never writes to the warehouse.
- [ ] Reports include Overview, Activity, Recovery & Sleep, Workouts, and Body views only where M0 confirms relevant data is available.
- [ ] Views show measured values and trends only; they do not provide diagnosis or medical/coaching recommendations.
- [ ] Missing days appear as gaps, and a freshness indicator shows the last successful load and latest date per type.
- [ ] Query results are cached for one hour; a warm-cache app load completes in under three seconds.
- [ ] Charts match mart values and a README documents local startup and refresh behavior.
- [ ] Verify in browser using dev-browser skill.

### US-009: Rebuild and delete user data safely

**Description:** As the owner, I want clear recovery and offboarding procedures so that data remains under my control.

**Acceptance Criteria:**

- [ ] Google Health is documented as the source of truth; MotherDuck is rebuilt from the API rather than copied to a separate health-data backup.
- [ ] The default rebuild covers two years; the operator can configure a longer backfill where Google Health history permits.
- [ ] Offboarding instructions revoke Google app access, remove local Keychain entries and GitHub secrets, revoke MotherDuck tokens, and delete the MotherDuck database.
- [ ] The dashboard and documentation state the limitation that edits/deletions older than the 30-day re-sync window may remain until a broader rebuild.

## 4. Functional Requirements

- **FR-1:** The system must extract Google Health API v4 data with dlt; Google Fit REST and direct Health Connect access are not supported sources.
- **FR-2:** Each deployment must use accounts controlled by its owner: Google Cloud/OAuth, MotherDuck, and a private GitHub repository. The builder must not receive credentials, data, logs, or crash reports.
- **FR-3:** The consent flow must request only `activity_and_fitness.readonly`, `health_metrics_and_measurements.readonly`, `sleep.readonly`, and `profile.readonly`; exclude IRN, ECG, location, nutrition, and reproductive-health data.
- **FR-4:** The extraction registry must include only types confirmed useful in M0 and must preserve `dataSource` provenance wherever the API method supports it.
- **FR-5:** Resources must paginate, apply the M0-verified filters, retry transient `429`/`5xx` responses with exponential backoff and jitter, and pace requests at no more than four per second.
- **FR-6:** Incremental loads must use a type-appropriate cursor and configurable lookback. The initial backfill must default to two years and process restartable 30-day windows.
- **FR-7:** The destination must be selectable between local DuckDB and the user's MotherDuck database without changing model SQL.
- **FR-8:** dbt must build typed staging views and full-rebuild marts, test key uniqueness/nullability, accepted ranges, and data freshness, and run only after dlt has loaded raw sources.
- **FR-9:** Source-priority rules must prevent overlapping device measurements from being added together. Missing data must remain `NULL`.
- **FR-10:** The weekly correction run must replace a type's last 30 days only after complete extraction; on failure, previously loaded rows for that window must remain intact.
- **FR-11:** The daily GitHub Actions workflow must run at 06:00 UTC, support manual dispatch, serialize writes to the warehouse, and trigger only on schedule or manual dispatch.
- **FR-12:** Streamlit must be local-only for v1, query marts with a MotherDuck read-only token, cache queries for one hour, and expose only data-supported descriptive views.
- **FR-13:** Operational logs may include run identifiers, types, row counts, duration, and status; they must never include health values, API payloads, access tokens, refresh tokens, client secrets, or authorization headers.
- **FR-14:** MotherDuck cloud loading must not begin until the Google policy/security-assessment requirement is resolved and any required approval is obtained.
- **FR-15:** Recovery must use Google Health as the source of truth and support a configurable API backfill; v1 does not maintain a second warehouse backup.

## 5. Non-Goals

- A shared hosted service, multi-user backend, or builder access to user data.
- Public dashboard hosting or Streamlit Community Cloud deployment.
- SportBud integration in v1; any later export must be aggregated, explicit, and separately designed.
- Medical diagnosis, treatment, or coaching recommendations.
- Real-time streaming, webhooks, Airflow/Dagster, Kafka, Docker/Kubernetes, or a custom web backend.
- Data sources outside Google Health API v4, including Google Fit REST and direct Health Connect access.
- Automatic recovery of source edits/deletions older than the configured re-sync window without a broader backfill.

## 6. Design Considerations

- Keep the dashboard local and work-focused; missing data should be visually represented as gaps, not zeros.
- Expose a freshness badge and per-type latest date so delayed wearable sync is visible.
- Retain source platform/device metadata in staging and make source priority configurable through a dbt seed.
- Organize implementation as milestones: feasibility spike; local first resource; core types/backfill; dbt marts; MotherDuck scheduling; dashboard. The single-type vertical slice through MotherDuck, dbt, and a minimal Streamlit report must be proven before broad expansion.

## 7. Technical Considerations

- Python tooling: `uv`, dlt, `google-auth`, `google-auth-oauthlib`, `keyring`, Ruff, Pyright, pytest, and pre-commit with gitleaks.
- Transformations: dbt-core 1.12.x and dbt-duckdb 1.11.x, SQL models, local DuckDB development, MotherDuck `md:` production profile.
- Dashboard: Streamlit and Plotly. It must use only a MotherDuck read-scaling token.
- Credentials: macOS Keychain locally and GitHub production environment secrets in CI; access tokens remain in process memory.
- M0 must verify API methods, pagination and filter syntax for each data kind, actual account data availability, Desktop OAuth compatibility, token lifetime, and the security-assessment policy before the relevant implementation/deployment gates.
- Tests and fixtures must use synthetic health records only.

## 8. Success Metrics

- M0 produces a data-type availability matrix, verified filter syntax, OAuth/token-lifetime evidence, policy/security-assessment findings, manual setup steps, and a go/no-go decision.
- The first `steps` load can be rerun without duplicates and can recover from interruption.
- The first end-to-end `steps` slice runs from Google Health through MotherDuck and dbt to a local Streamlit chart.
- Local dbt build/tests pass, including the specified synthetic edge cases.
- Three consecutive scheduled daily runs complete successfully after deployment.
- A warm-cache dashboard load completes in under three seconds, and all charts match their mart values.
- No real health values or credentials appear in source control, fixtures, CI logs, artifacts, or support material.

## 9. Open Questions

- Which core data types, source devices, and filter syntax does the user's Google Health account actually support? M0 must determine this.
- Does the proposed single-user MotherDuck-backed use require a Google security assessment, and what approvals or constraints apply? Resolve before cloud loading.
- Does Google Health permit the proposed Desktop OAuth client and loopback `InstalledAppFlow` for this API? M0 must verify.
- How much history is available for each confirmed type? The default backfill is two years, but longer recovery depends on API availability and configuration.
