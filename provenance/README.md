# Provenance record

The bulk data (`companies/`, `reviews/`, `*.zip`) is gitignored — it is ~288 GB and, per
`DISCLAIMER.txt`, not redistributable. This directory keeps the parts of each vintage that
document *what the data claimed to be*, so every assertion the ETL makes can be traced back
to a shipped artefact.

## Contents

```
provenance/
  companies/YYYYMM/CHANGELOG.companies.yaml
  companies/YYYYMM/DISCLAIMER.txt
  companies/YYYYMM/glassdoor_companies_latest.json     # vendor JSON Schema
  reviews/YYYYMM/…                                     # same three files
  manifest-digests.csv
```

47 vintages × 2 objects × 3 files = 282 files, 552 KB total.

## Why the `ids_*.txt` manifests are not here

Each vintage also ships an `ids_*.txt` listing every record id. Those are the load-
completeness check for the ETL, but they are **not small**:

| | Per vintage | All 47 vintages |
|---|---|---|
| Companies | ~48 MB | 1.46 GB |
| Reviews | 0.5–1.9 GB | 61.4 GB |

Committing 63 GB of line-delimited ids would defeat the purpose of the `.gitignore`. The
manifests stay on disk beside the data; `manifest-digests.csv` records the filename, byte
size and SHA-256 of each one, which is what makes them verifiable later. Row counts per
vintage are in [`../PLAN.md`](../PLAN.md) §1.

## Caveats on the vendor files

- `glassdoor_*_latest.json` is **stale**. The reviews schema omits `language`,
  `author_status`, `is_current`, `author_employment_length` and `advice_to_management`,
  all of which are present in the data. Treat it as provenance, not as a contract.
- `CHANGELOG.*.yaml` stops at 2022-03 and therefore documents none of the sixteen schema
  versions observed across these vintages. See [`../PLAN.md`](../PLAN.md) §3.
- `DISCLAIMER.txt` is identical across vintages but kept per-vintage so the licence terms
  in force at each delivery are recorded.
