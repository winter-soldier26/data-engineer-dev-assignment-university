# Data Product Contract — University Chapters (Gold)

## 1. Name & Owner

| Field | Value |
|---|---|
| **Product name** | `university_chapters` (Gold, v1) |
| **Technical owner** | Rajat Sarkar — sarkarrajat8@gmail.com |
| **Consumer use cases** | Analytics and reporting on active university chapters across CA, OR, and WA (e.g. regional coverage dashboards, chapter directory lookups) |

## 2. Interface

**Storage location:** `gold/university_chapters/v1/` (Delta table)

**Grain:** One row per `chapter_id` (stable business key). No duplicates.

| Column | Type | Description |
|---|---|---|
| `chapter_id` | string | Stable business key from source (e.g. `CA-0355`). Primary identifier — never null. |
| `chapter_name` | string | Display name of the university chapter. |
| `city` | string | City where the chapter is located. May be blank/unknown (see DQ-W1). |
| `state` | string | USPS 2-letter state code. Scoped to `CA`, `OR`, `WA` only. |
| `longitude` | double | WGS84 longitude. Validated range: [-180, 180]. |
| `latitude` | double | WGS84 latitude. Validated range: [-90, 90]. |
| `dq_status` | string | `'OK'` or `'WARNING'`. Never contains quarantined rows (see §4). |
| `dq_warnings` | string | Empty string when `dq_status = 'OK'`. Contains reason code(s) when `dq_status = 'WARNING'` (currently only `MISSING_OR_UNKNOWN_CITY`). |

**Excluded from this contract (technical/internal, not published to consumers):**
- `source_object_id` (from source `OBJECTID`)
- `ingest_run_id`
- Any other source-system fields not explicitly mapped above

## 3. Freshness

**Intended SLA:** Refreshed daily by 06:00 UTC.

**Current state:** Pipeline is run manually (no scheduler configured — out of scope for this exercise). Each run is a full extract from the source API; there is no incremental/CDC ingestion at this time.

## 4. Quality

### Row-level rules

| Rule ID | Severity | Condition | Behavior |
|---|---|---|---|
| **DQ-Q1** | Quarantine (hard fail) | `longitude`/`latitude` missing, null, non-numeric, or out of valid range (`lon ∉ [-180,180]` or `lat ∉ [-90,90]`) | Row is excluded from Silver/Gold entirely. Written to `quarantine/university_chapters/<run_id>/` with `reason_code = INVALID_COORDINATES`, `ingest_run_id`, and the raw row payload for debugging. |
| **DQ-W1** | Warning (soft) | `city` is null, blank, or the literal `UNKNOWN` (case-insensitive) | Row still publishes to Gold. `dq_status = 'WARNING'`, `dq_warnings = MISSING_OR_UNKNOWN_CITY`. |
| *(clean rows)* | — | Neither condition above applies | `dq_status = 'OK'`, `dq_warnings = ''` (empty). |

**Guarantee:** Quarantined rows never appear in Gold. Verified by automated test (`04_tests.py`).

### Batch-level rules

- **Empty batch:** If the entire batch (post-DQ) is empty, the pipeline **fails loudly** (raises an error) rather than publishing an empty Gold table as a successful run.
- **CA row count:** CA is expected to always have active chapters. If CA drops to zero, the pipeline **fails loudly** — this is treated as an anomaly, not a valid outcome.
- **OR/WA row count:** OR and/or WA may legitimately be zero. This is a known data fact (not all three states currently have active chapters) and does **not** trigger a failure.

### Run-level logging

Every run logs: `rows_in`, `rows_quarantined`, `rows_warned`, `rows_ok`.

## 5. Versioning

- The Gold table is published under a versioned path: `gold/university_chapters/v1/`.
- **Non-breaking changes** (e.g. adding a new optional column) may be applied in place within `v1`.
- **Breaking changes** (e.g. renaming/removing a column, changing a column's type or semantics) require publishing under a new path, `gold/university_chapters/v2/`, so existing consumers on `v1` are not disrupted. Consumers migrate on their own schedule.

## 6. Classification

- **Data sensitivity:** Public source data (public ArcGIS FeatureServer). No PII is expected or published — the schema contains only chapter/organizational and geographic information, not personal data about individuals.
- **Access:** No special access controls required beyond standard workspace permissions, given the public nature of the source data.
