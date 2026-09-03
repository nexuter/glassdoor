# Glassdoor Panel (Coresignal) — Data Review & Build Plan

_Prepared 2026-09-03. Source: Coresignal Glassdoor Companies + Reviews, 47 monthly
vintages, 2022-06 through 2026-04._

> **Licensing constraint.** Every vintage ships a `DISCLAIMER.txt` stating the data is
> subject to copyright and trade-secret rights, and that the database structure may not be
> disclosed publicly nor the data re-distributed. The web interface must therefore be
> authenticated and access-logged. This is not optional polish — it shapes the deployment.

---

## 1. What is actually on disk

```
glassdoor/
  Glassdoor Data Dictionaries.pdf     # Coresignal docs: firmographic (7pp) + reviews (4pp)
  companies/YYYYMM/<3-hex>/<md5>.json.gz
  reviews/YYYYMM/<3-hex>/<md5>.json.gz
  reviews/companies/202601/           # STRAY: duplicate extraction of companies/202601
  *.zip                               # source archives, ~288 GB, deletable after verification
```

Each `YYYYMM` directory also carries four sidecar files that are worth more than they look:

| File | What it gives you |
|---|---|
| `ids_*.txt` | Complete list of record ids in that vintage — an exact, free row count and a load-completeness check |
| `glassdoor_*_latest.json` | JSON Schema for the record. **Stale** — the reviews one omits `language`, `author_status`, `advice_to_management`. Trust the data, not this file. |
| `CHANGELOG.*.yaml` | Coresignal's own field-change log (stops at 2022-03, so it does not cover the drift observed below) |
| `DISCLAIMER.txt` | Licence terms (see above) |

### File format

Every `.json.gz` holds **exactly 10,000 records**, pretty-printed as a JSON array with
**one complete record per line** (leading tab, trailing comma). The loader can therefore
stream line-by-line — no incremental JSON parser needed, and shards are trivially
parallelisable. Measured single-threaded throughput on this machine: **~43,500 records/s**
(gunzip + regex id extract + hash). A full 47-vintage pass is ~9 h serial, ~1.5 h at 8
workers.

Directory buckets are the first 3 hex chars of the **filename**, and the filename is
`md5(first record's doc.id)`. It is *not* a content-hash bucket — chunk boundaries and
filenames shift between vintages, so you cannot diff two months by filename.

### Volume

| | 2022-06 | 2026-04 | Vintages | Archive size |
|---|---|---|---|---|
| Companies | 829,559 | 1,852,962 | 47 | 10.8 GB |
| Reviews | 17,096,860 | 42,282,291 | 47 | 277 GB |

Growth is stepwise, not organic. Companies jump +134 k in 2023-07, +231 k in 2023-11,
+130 k in 2025-03 and +383 k in 2025-10; reviews jump +5.4 M in 2023-06 and +3.2 M in
2026-03. These are collection events, not firm entry or a surge in posting — the panel
needs an explicit `first_vintage` column and the codebook has to name these dates.

---

## 2. The finding that drives the whole design

**Each monthly directory is a full, cumulative re-dump — not a delta.** Measured on
companies and reviews, 2022-06 → 2022-07 (full vintages, `doc` payload hashed):

```
companies 202206 → 202207          reviews 202206 → 202207
  202206   829,559                   202206  17,096,860
  202207   829,589                   202207  17,228,547
  common   829,559                   common  17,096,860
  dropped        0                   dropped          0
  added         30                   added      131,687
  doc identical  826,000 (99.57%)    doc identical 17,076,493 (99.88%)
  doc changed      3,559 ( 0.43%)    doc changed       20,367 ( 0.12%)
```

Nothing ever disappears in either feed, and the payloads are almost static: companies
change at **0.43%** per month, reviews at **0.12%**. (Measured on `doc` only. Hashing the
whole record instead puts reviews at 11.7%, but that is `_meta.updated_at_timestamp` moving
on re-scrape, not content — a trap worth avoiding in the change-detection logic.)

Stored literally, 47 snapshots would be ~56 M company-rows and **~1.4 billion
review-rows**, of which more than 99% are exact duplicates of a row already present.

So: **load once, version the changes.** A type-2 slowly-changing-dimension keyed on a row
hash collapses companies to roughly 1.9 M base rows plus a few million change rows, and
reviews to the ~42 M distinct reviews that actually exist. Users can still request "the
panel as of 2024-03" — that is a range predicate, not a stored copy.

---

## 3. Schema drift — eight versions of each object

`_meta.version_id` identifies the extraction schema. Old records are **never
back-filled**: a single 2026 vintage still contains records emitted under the 2021 schema,
so *within one snapshot* you will find both `recommend.rating_ceo` and
`recommend.ratingCeo`. The normaliser must be version-aware, not vintage-aware.

### Companies — 8 versions

`ec3bbb21` / `52b230da` / `b7958aa1` → `3c6cd3e6` (2023-05) → `1e079f04` (2023-07) →
`2aa31948` (2023-11) → `b4615eeb` (2024-05) → `28ff7dfe` (2024-12, current)

| Vintage | Change |
|---|---|
| 2023-05 | `rating.star_distribution` / `percentage_distribution` promoted out of `rating.overall` (both nestings coexist for ~2 years) |
| 2023-09 | `_meta.is_deleted` added |
| 2024-05 | `doc.main_source_id` added (branch → parent profile pointer) |
| 2024-05 … 2025-02 | `contact_info` gains then loses instagram/linkedin/youtube; whole object disappears in `28ff7dfe` |
| 2024-12 | `doc.affiliated_companies[]` added |

### Reviews — 8 versions

`0399c54a` / `a7fb44ed` → `4b8bd062` (2023-06) → `784e5060` (2023-08) → `43b86bce`
(2023-09) → `30490a9e` (2023-10) → `454126bb` (2024-02) → `d5d447b0` (2024-09, current)

| Vintage | Change |
|---|---|
| 2023-06 | `recommend.ratingCeo` → `rating_ceo` (camel → snake; both forms persist) |
| 2023-08 | `author_status`, `is_current`, `author_employment_length` added |
| 2023-09 | `language`, `_meta.is_deleted` added |
| 2023-10 | `work_text` retired (still present on un-refreshed old records) |
| 2024-02 | `advice_to_management` added |

**Consequence for panel work:** any variable added after 2023 is missing-by-construction
for older vintages *and* for stale records inside new vintages. The codebook must state a
`first_available` vintage per variable, and the exporter should emit a coverage note.

---

## 4. Data-quality issues found (all confirmed against the data)

| # | Issue | Scale | Fix in ETL |
|---|---|---|---|
| 1 | **Review natural key is `doc.source_id`, not `doc.id`.** Legacy version `0399c54a` emits `id = glassdoor_<source_id>`; all others emit `glassdoor_review_<source_id>`. The same review therefore has two different `id` values across vintages. | 0.3–3% of rows per vintage | Key on `source_id`; store `id` as an attribute |
| 2 | The numeric `RVW…` component of `source_id` is **not unique** — 7,705 collisions in a 340 k sample (2.4%). `source_id` in full *is* unique. | 2.4% | Never key on the RVW integer alone |
| 3 | `doc.company_id` is sometimes `"Company Name:E123456"` instead of `"123456"` | 763 / 660,000 (0.12%) | Regex `:E(\d+)$`; keep raw value in `company_id_raw` |
| 4 | ~0.2% of numeric `company_id`s have **no matching company record** in the same vintage | 52 / 26,563 distinct | Load as orphans with a flag; do not silently drop |
| 5 | `recommend` is sometimes a **string**, e.g. `"Recommend, CEO Approval, Business Outlook"` (and variants with doubled spaces), instead of an object | ~0.66% | Parse to flags, retain raw |
| 6 | `recommend` is `null` outright | 26% | Genuine missing |
| 7 | `is_current` mixes booleans and the **strings** `"True"` / `"False"` | ~0.3% | Cast |
| 8 | `star_rating.overall` mixes int (`4`) and float (`5.0`) | widespread | Cast to smallint |
| 9 | **Sentinel values, not nulls.** Company ratings use `-0.1` (no rating — 302,871 companies in 2026-04), `-1.0`, `-10`. Review sub-ratings use `0` for "not rated" (146,443 of 660,000 sampled `culture_values`). | very large | Map to NULL; keep a `*_raw` copy if the distinction matters |
| 10 | **`location` semantics changed.** 2022 vintages use `"Plano, TX"` (US state abbrev.); by late 2023 the same field is `"City, Country"`. | all US rows | Parse to `city` + `region` + `country`, normalising state → United States |
| 11 | Free-text fields contain control characters (Coresignal documents this) and large payloads — `description` max 6,294 chars, `cons` max 18,230, `advice_to_management` max 10,010 | — | Strip C0 controls; store as `text` |
| 12 | Sparsity: `industry` null for 66% of companies, `founded` 87%, `ceo` 81%, `mission` 99%, `affiliated_companies` 99.5% | — | Document coverage in the codebook |
| 13 | `reviews/companies/202601/` is a duplicate extraction of `companies/202601/` (185 files each) | 296 MB | Delete after byte-verification |
| 14 | **Some consecutive vintages are identical re-deliveries.** Companies 2022-12 … 2023-03 all report 829,685 records with identical version mixes; 2023-08 … 2023-10 all report 965,329; reviews 2023-07 and 2023-08 both report 27,019,194. | ≥6 vintage pairs | Detect via manifest hash, mark as repeat, exclude from change series so the panel does not read them as "no change" |
| 15 | `main_source_id` differs from `source_id` for 4,998 companies (branch profiles); only 9,314 companies carry `affiliated_companies` | 0.3% / 0.5% | Expose both as joinable relations, not buried JSON |

---

## 5. Proposed database

### Engine

**PostgreSQL 17 as the system of record, with a Parquet mirror for bulk export.**
Postgres gives concurrent web users, authentication, row-level access control, an audit
trail, and good-enough full-text search (GIN over `tsvector`) for review text. DuckDB
reading a partitioned Parquet copy of the same tables is the fast path for large CSV
exports and for anyone who wants to analyse off-line. Estimated footprint: ~100 GB
Postgres (reviews ~60–80 GB including text and indexes), ~35 GB Parquet + zstd.

_(ClickHouse would be faster for full-corpus text scans but adds ops burden a small
research group probably should not take on. Revisit only if text search proves too slow.)_

### Tables

```
vintage(vintage_id, yyyymm, object, n_records_manifest, n_records_loaded, loaded_at)

company(company_sk, source_id UNIQUE, main_source_id, first_vintage, last_vintage,
        is_deleted, url, glassdoor_slug)
company_version(company_sk, valid_from_vintage, valid_to_vintage, row_hash,
        name, description, mission, website, type, founded, industry,
        employee_count_band, revenue_band,
        city, region, country,                       -- parsed & harmonised
        ceo, ceo_approval_pct,
        job_count, salary_count, benefit_count, review_count, interview_count,
        rating_overall, rating_culture_values, rating_career_opportunities,
        rating_compensation_benefits, rating_senior_management,
        rating_work_life_balance, rating_diversity_inclusion,
        pct_biz_outlook, pct_ceo_approval, pct_recommend,
        stars_1..stars_5, pct_stars_1..pct_stars_5,
        schema_version_id,
        PRIMARY KEY (company_sk, valid_from_vintage))
company_affiliation(company_sk, affiliate_url, affiliate_name, is_parent,
        first_vintage, last_vintage)

review(review_sk, source_id UNIQUE, company_sk, company_id_raw,
        review_date, review_year, review_month, language,
        author_location, author_title, author_status, is_current,
        author_employment_length,
        star_overall, star_culture_values, star_career_opportunities,
        star_compensation_benefits, star_senior_management,
        star_work_life_balance, star_diversity_inclusion,
        rec_ceo, rec_business_outlook, rec_recommend_to_friend, recommend_raw,
        first_vintage, last_vintage, is_deleted, schema_version_id)
        PARTITION BY RANGE (review_year)
review_text(review_sk, summary, pros, cons, advice_to_management, work_text,
        search_tsv)                                   -- GIN index
review_version(review_sk, valid_from_vintage, valid_to_vintage, row_hash, ...)
        -- populated only for reviews whose content actually changed

-- convenience, materialised
company_month(company_sk, yyyymm, <company_version fields>)   -- the flat panel
company_review_month(company_sk, yyyymm, n_reviews, avg_overall, ...)
```

Reference tables for `industry`, `company_type`, `employee_count_band`, `revenue_band`
and `country`, so filters are dropdowns rather than free text.

### Indexes

- `review (company_sk, review_date)` — the dominant access path
- `review (review_date)` per partition
- `review (star_overall)`, `(language)`, `(author_status)` — filter selectivity
- GIN on `review_text.search_tsv`
- `company_version (valid_from_vintage, valid_to_vintage)`, `company (source_id)`,
  trigram index on `company.name` for autocomplete

---

## 6. Web interface (the WRDS part)

The point is that a researcher with no SQL can build a sample, pick variables, and get a
CSV. Five screens:

1. **Company finder** — name autocomplete plus filters on industry, country, size band,
   revenue band, founding year, rating range. Paste or upload a list of Glassdoor company
   ids to define a sample directly.
2. **Review query builder** — sample (from screen 1 or an uploaded id list) × date range ×
   filters (overall and sub-ratings, recommend / CEO / outlook, author status, current vs
   former, tenure, language, has-text) × keyword search in pros / cons / summary / advice.
   Variables chosen from a grouped checkbox list. Live row-count estimate before committing.
3. **Company panel builder** — the feature raw Glassdoor cannot give you: pick companies, a
   date range, a frequency and variables; get a balanced monthly panel assembled from the
   version history, with `first_available` coverage warnings attached.
4. **Download centre** — every non-trivial request runs as a background job, exactly the
   WRDS model. CSV / TSV / Parquet / Stata, gzipped, expiring links, and the exact query
   parameters stored alongside the output so results are reproducible and citable.
5. **Codebook** — generated from the schema plus the profiling statistics in §4, so every
   variable page shows type, coverage, sentinel handling, and first available vintage.

Plus: a read-only SQL console for power users (separate role, statement timeout, row cap);
mandatory login with terms acceptance; a download audit log.

Suggested stack — FastAPI + SQLAlchemy, a JSON→SQL query-builder layer, `arq` or Celery
workers for exports, and a server-rendered HTMX front end (fast to build, no SPA to
maintain) unless a React front end is specifically wanted.

---

## 7. Phased plan

| Phase | Work | Est. |
|---|---|---|
| 0 | Decisions (§8), provision Postgres and object store, repo hygiene | 1–2 d |
| 1 | Streaming reader + version-aware normaliser for all 16 schema versions; load one vintage end-to-end; reconcile against `ids_*.txt` | 1 w |
| 2 | Full historical load with dedup / SCD-2, parallel by vintage; orphan and anomaly capture | 1–2 w (≈1.5 h compute per full pass, several passes expected) |
| 3 | Indexes, partitions, materialised panel + aggregates, QA report, generated codebook | 1 w |
| 4 | Query API: sample definition, variable selection, row estimation, export job queue | 1–2 w |
| 5 | Web UI: the five screens above | 2–3 w |
| 6 | Auth, terms acceptance, audit logging, download expiry, deployment | 1 w |
| 7 | Monthly refresh automation: ingest new vintage, close/open SCD rows, refresh matviews, diff report | 3–5 d |

Roughly 8–11 weeks for one engineer to a usable internal service.

**Validation gates.** Phases 1 and 2 are not done until, for every vintage: rows loaded ==
`wc -l ids_*.txt`; every `source_id` in the manifest resolves; no review lands in a date
partition outside 2008-01 … current; sentinel counts match the profiling baseline.

---

## 8. Decisions needed before Phase 1

1. ~~**Reviews: full history or current-state only?**~~ **Answered — keep the history.**
   Review `doc` churn is 0.12% per month (20,367 rows between 2022-06 and 2022-07), so a
   `review_version` table costs roughly 1 M rows across all 47 vintages. Build it.
2. **Postgres vs DuckDB-only.** How many concurrent users? If it is really one to three
   people doing research, DuckDB + Parquet + a thin web front end halves the build.
3. **Where does it run?** Local server on this D: volume (5.5 TB free), or institutional /
   cloud hosting? This determines the auth story.
4. **Who may log in?** Given the Coresignal terms — named accounts inside the licensed
   institution only, presumably. Confirm.
5. **Export ceilings.** Row cap per download, retention period, and whether raw review text
   may be exported in bulk or only aggregates.
6. **Do the `.zip` archives get deleted** after byte-verification of the extracted trees?
   That reclaims ~288 GB.

---

## 9. Housekeeping — done 2026-09-03

- **Removed `reviews/companies/202601/`.** Verified against `companies/202601/` before deleting:
  189 files each, 309,891,751 bytes each, matching per-file content hashes. (The archive it came
  from, `reviews/companies.zip`, 267 MB, is still on disk and covered by the `.zip` decision in
  §8 Q6.)
- **Added `.gitignore`** for `companies/`, `reviews/`, `*.zip`, `*.pdf` — plus `*.download`,
  `scratch/` and the usual Python noise. `git status` no longer lists ~290 GB of untracked data.
- **Version-controlled the sidecars under `provenance/`** — 47 vintages × 2 objects × 3 files
  (schema, changelog, disclaimer) = 282 files, 552 KB. See `provenance/README.md`.
- **Correction to what this plan originally claimed.** The `ids_*.txt` manifests are *not* small:
  1.46 GB across the company vintages and **61.4 GB** across the review vintages, 63 GB in total.
  They cannot be committed. `provenance/manifest-digests.csv` records each manifest's filename,
  record count, byte size and SHA-256 instead — 94 rows — which preserves the verification
  function while the files stay on disk beside the data.
- **Published the plan as HTML** in `docs/blueprint.html` and `docs/blueprint.ko.html` (Korean).

## 10. Still open

- Delete the `.zip` archives after byte-verifying the extracted trees (§8 Q6) — reclaims ~288 GB.
- The five remaining decisions in §8.
