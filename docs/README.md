# docs/

Standalone HTML renderings of [`../PLAN.md`](../PLAN.md). Open either file directly in a
browser — no build step, no server, no local assets. Fonts come from Google Fonts; with no
network they fall back to system faces and the pages still read correctly.

| File | |
|---|---|
| [`blueprint.html`](blueprint.html) | English |
| [`blueprint.ko.html`](blueprint.ko.html) | 한국어 |

The two link to each other from the masthead. Both carry the same figures, the same schema-drift
strip and the same fifteen data-quality findings; the Korean version is a translation, not a
summary. Field names, `version_id` hashes, table and column names stay in English in both, because
they are identifiers in the data.

## Keeping them in sync

`PLAN.md` is the source of record for the *content*. These pages are hand-maintained alongside it —
if a figure changes in one, change it in all three.

The published Artifact copies are the same documents minus the `<!doctype>` / `<html>` / `<head>`
wrapper, which the Artifact host supplies itself.
