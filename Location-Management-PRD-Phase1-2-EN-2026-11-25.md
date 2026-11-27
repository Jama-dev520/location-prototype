# Location Management Admin Console — PRD, Phase 1 and Phase 2

**Version 1.1 ｜ 2026-11-25 ｜ Status: draft for review**

Companion files: clickable prototype (`index.html`, v8.1.0) · Duplicate check v3.0 (similarity rules) · **swimlane** `diagrams/location-swimlane-phase1-2.html` — the two phases side by side, same four lanes (Provider / Sync / Console / eBay), with the one step that decides customer visibility marked.

This document describes the same product twice. **Phase 1** is the console as it stood **before 2026-11-25** (the v5.x build; the published snapshot is v5.0.0). **Phase 2** is **8.0.0**, the single-list model that replaces it. The two phases share the data and the eBay contract; they differ in how many pages an operator touches, and in whether a second identity key exists. This document keeps that difference visible instead of rewriting history.

---

## 1. One product, two phases

| | Phase 1 (before 2026-11-25) | Phase 2 (8.0.0) |
|---|---|---|
| Pages | Vendor list + OC list + Duplicate Check | **one list** |
| Identity | two keys: `LOC-…` and `OC-…` | **one key: the vendor location id** |
| Duplicate decision | a queue on its own page | select two rows, mark one |
| Live control | per store, on the OC list | per record, on the row |
| Sync awareness | implied by drift on the OC page | explicit `New` / `Changed` badge |
| Who eBay sees | a store with a live source | a record with `live = true` |

Shared by both phases:

- Provider values are read-only. The console flags, it never corrects.
- A person decides anything a merge could not undo.
- Nothing reaches a customer without someone acting on it.

---

## 2. Phase 1 — before 2026-11-25

### 2.1 Scope

Three working pages:

| Page | Purpose |
|---|---|
| **Vendor Location List** (P1) | what the providers sent, read-only, one row per record |
| **OC Location List** (P2) | one row per store: live / not live, enable, batch enable and disable |
| **Duplicate Check** (P2) | the queue of suspected pairs and the decision on each |

The vendor list is the reference table. It answers "what do we have", and nothing on it changes state: no switch, no batch action, no decision. Those live on the other two pages, and the OC id is the thread that connects them.

### 2.2 The vendor list — columns (11)

| # | Column | Content |
|---|---|---|
| 1 | Location ID | `LOC-xxxxx`, clickable to the profile |
| 2 | Name | provider name |
| 3 | Address | line 1; line 2 in a lighter tone |
| 4 | City / Province | `Brooklyn, NY` |
| 5 | Source | `PUDOpoint (PUDO)` / `packagehub (LMS)` / `M&MP` |
| 6 | QR | `QR` badge where the source supports drop-off, `—` otherwise |
| 7 | Sweeper | `PUDO Inc` / `LMS`, or `missing` |
| 8 | Status | provider status: `active` / `inactive` / `removed` |
| 9 | Issues | count of open issues, clickable |
| 10 | OC Location | the `OC-…` id, plus a state badge: `live`, `standby`, `not live`, `blocked`; `none` when the record has never been assigned |
| 11 | Updated | provider timestamp |

There is no checkbox, no `Zip` column, no `Live` switch and no `New` / `Changed` badge. Status is described where it lives: the `OC Location` cell carries `live` / `standby` / `not live` / `blocked` because the store, not the record, was the thing that could be live.

### 2.3 The vendor list — filters (9)

| Filter | Values | Behaviour |
|---|---|---|
| Location ID | up to 20 conditions, space or comma separated | substring, any match |
| OC Location ID | up to 20 conditions | substring against the store id |
| Name | up to 20 conditions | a condition is a phrase, all words required |
| Province | facet | counts reflect the other filters |
| City | facet | cascades from Province |
| Source | facet | three values |
| Status | facet | provider status |
| Sweeper | facet | `PUDO Inc` / `LMS` |
| Issues | facet | has open issue / none |

There is no `Zipcode` filter, no `Live` filter, no `Change` filter and no `Duplicates` filter. A zip lookup, a "show me what moved" view and a "show me the pairs" view were not expressible on this page; each of them had a different home (`Zipcode` did not exist yet, changes surfaced as drift on the store page, and pairs belonged to the Duplicate Check queue).

The page has `Reset` and `Apply Filters` as explicit buttons — filters are not applied on every keystroke.

### 2.4 What the operator can do on this page

| Action | Where | Effect |
|---|---|---|
| Browse and filter | the table | read-only |
| Open a record | click the id | profile with five tabs: Basic Info, Operations, Hours & Services, Issues, Change History |
| Flag a data issue | profile | records the issue; no value changes |
| Export | `Export CSV` | one row per record as filtered |
| Pull provider data | `Refresh from providers` | loads the next snapshot; values are replaced, decisions are not touched |

What is **not** on this page: going live, batch enable, and the duplicate decision.

| Action | Phase-1 home | What it did |
|---|---|---|
| Go live / take down | OC Location List | per store, with a sign-off that names which record feeds it |
| Batch enable / disable | OC Location List | `Batch Enable` / `Batch Disable` over selected stores |
| Decide a pair | Duplicate Check | same station / two tenants / address to fix |
| See what drifted | OC Location List | drift rows comparing the store against its source |

### 2.5 How identity worked in Phase 1

- A store had one `OC-…` id. Two records that were the same physical store **shared that id** (a 2:1 store): one fed the store and was `live`, the other was `standby` — listed, readable, and never live on its own.
- A pair decided on the Duplicate Check page merged the two records onto one store id.
- The `OC Location` cell was the only place the list spoke about live state, and it spoke in badges, not a switch.

### 2.6 Acceptance — Phase 1

1. The sidebar has three working pages: Vendor Location List, OC Location List, Duplicate Check.
2. The vendor list has eleven columns and no checkbox, `Zip` or `Live` column.
3. The vendor list has nine filters; there is no `Zipcode`, `Live`, `Change` or `Duplicates` filter.
4. Filters apply on `Apply Filters`, not while typing; `Reset` clears them.
5. The `OC Location` cell shows the store id with `live` / `standby` / `not live` / `blocked`, or `none`.
6. Nothing on the vendor list changes state: no switch, no batch action.
7. Going live happens on the OC list, through a sign-off that names the feeding record.
8. Batch enable and batch disable work over selected stores on the OC list.
9. A suspected pair is decided on the Duplicate Check page; the decision merges the records onto one store id.
10. `Export CSV` writes the filtered rows; `Refresh from providers` replaces values and leaves decisions alone.
11. The profile, the five tabs and the issue ledger are the same as in Phase 2.

---

## 3. Phase 2 — 8.0.0

### 3.1 What changed and why

Phase 1's shape came from the two-layer model: a record is what a provider sent, a store is what a customer sees, and an OC id joined them. In practice the operator never worked that way — every decision started on a record and ended on the same record, and the split cost three navigations per question.

8.0.0 collapses the console into one list and, in the same move, removes the second identity key. The order matters: once the list carries the live switch and the duplicate marker, the store stops being an object and the OC id stops being an identity.

### 3.2 Columns (13)

| # | Column | Content |
|---|---|---|
| 1 | Vendor Location ID | `LOC-xxxxx`, clickable; `New` / `Changed` badge underneath when applicable |
| 2 | Name | provider name |
| 3 | Address | line 1; line 2 in a lighter tone |
| 4 | City / Province | `Brooklyn, NY` |
| 5 | Zip | first five digits |
| 6 | Source | three values, one field |
| 7 | Sweeper | badge, or `missing` |
| 8 | QR | `QR` badge or `—` |
| 9 | Status | provider status |
| 10 | Enable | switch reading `Enable` / `Disable`; `Blocked` badge when the platform blocks the store |
| 11 | Duplicate | `duplicate of LOC-xxxxx` on the duplicate, `—` on the primary |
| 12 | Issues | open issue count |
| 13 | Updated | provider timestamp |

A leading checkbox column carries the batch selection; it is not counted as a data column.

### 3.3 Filters (12)

Multi-select facets, OR inside a filter, AND between filters; `Reset` clears everything including the newer controls.

| Filter | Values |
|---|---|
| Vendor Location ID | up to 20 conditions, substring |
| Name | up to 20 conditions, phrase |
| Province | facet, cascades |
| City | facet |
| Zipcode | up to 20 conditions; a multi-row hit is ranked by the duplicate score against the oldest record, that score printed on the row |
| Source | facet |
| Status | facet |
| Enable | `Enable` / `Disable` / `Blocked by system` |
| Change | `New` / `Changed` / `No change` |
| Duplicates | `Duplicated records` / `Suspected duplicates` / `No duplicate`; suspect opens a pair board, scoped city → zip |
| Sweeper | facet |
| Issues | facet |

`OC Location ID` is gone: the id no longer exists, and `Vendor Location ID` answers the same question.

**How a zip lookup orders its rows (8.1.0).** Looking up a zip is how an operator hunts for a duplicate, so a multi-row hit is ranked by **the duplicate check's own score** against one anchor — the **oldest record in the result** (earliest `created`, tie-broken by id). Address 40 · zip 25 · state 15 · city 10 · name 10: the same number the pair dialog prints, so the list and the dialog never disagree. The anchor is not "the first row" on purpose: a base that follows the current sort re-ranks the list whenever another filter changes, which reads as if the data moved. Name breaks ties; the zip points are constant inside one lookup. The `addr_fingerprint` bucket key is gone — with the scan covering the whole table it had nothing left to do.

### 3.4 What the operator can do

| Action | Where | Effect |
|---|---|---|
| Filter for what moved | `Change = New` / `Changed` | the pending work, as a view |
| Go live | the `Enable` switch | writes the current values to eBay, flips the switch |
| Batch go live / take down | select rows → `Add to whitelist & apply` / `Remove from whitelist` | per row, with processed and skipped counts |
| Mark records as one store | select two or more → `Mark as duplicate` | writes `duplicate_of`; with a group, one keeper is picked and the rest point at it; nothing is minted or merged |
| Open a record | click the id | profile, five tabs |
| Flag a data issue | profile | unchanged |
| Export | `Export CSV` | id, live, duplicate marker, QR, issue count |

There is no separate publish action: a value reaches eBay because a record goes live, and a changed record was taken off the list at ingest, so putting it back publishes the new value.

### 3.5 Rules the list enforces

| # | Rule | Where it is enforced |
|---|---|---|
| P2-1 | A record whose provider status is not `active` cannot go live | switch disabled, API refuses, batch skips |
| P2-2 | A changed or new record is not live until someone switches it back on | taken down at ingest, `PENDING_CHANGE` |
| P2-3 | Two records of a pair cannot both be live | going live unlists its counterpart |
| P2-4 | `blocked` is read-only here | no write path exists |
| P2-5 | An id is never rewritten; a duplicate never adopts its pair's id | the decision writes `duplicate_of` only |

### 3.6 Acceptance — Phase 2

1. The sidebar has one business page; the OC list and the duplicate queue are gone.
2. No `OC-…` id appears in the list, the filters, the profile, the export or the API. Release history is the only place it is still named.
3. The list has thirteen data columns; the eleventh reads `Duplicate`.
4. The filter bar has twelve filters and no `OC Location ID`.
5. `Enable` is the only thing that decides eBay visibility.
6. Two records of a pair cannot both be live; turning one on unlists the other.
7. `Mark as duplicate` mints nothing and changes no id.
8. An inactive record renders a disabled switch; a blocked store renders a badge.
9. Filtering `Change = New`, selecting the rows and pressing `Add to whitelist & apply` writes, clears the badge and goes live in one action.
10. Reset clears every control and restores the full list.
11. Export carries `Vendor Location ID`, `Enable`, `Duplicate of` and QR, and no OC column.

---

## 4. Delta at a glance

| Area | Phase 1 | Phase 2 |
|---|---|---|
| Working pages | 3 | 1 |
| Columns on the vendor list | 11 | 13 (+ a checkbox column) |
| Filters on the vendor list | 9 | 12 |
| Filter apply | explicit button | on change |
| Identity | `LOC-…` + `OC-…` | `LOC-…` |
| Live control | OC list, per store, sign-off | row switch, per record |
| Batch enable | OC list | row selection |
| Duplicate decision | own page, queue | two rows, `Mark as duplicate` |
| Sync awareness | drift on the store page | `New` / `Changed` badge |
| Zip lookup | not available | filter, similarity-sorted |
| Export | filtered rows, no live state | adds live, duplicate marker, QR |

What does **not** change: the similarity rules, the read-only stance on provider values, the profile and its five tabs, the issue ledger, and the rule that nothing reaches a customer without someone acting.

---

## 5. What Phase 1 leaves behind

- the OC list page, its batch enable/disable and its sign-off dialog;
- the Duplicate Check page and its queue;
- the `oc_location` table, the `OC-…` id and the mint sequence;
- the `standby` state (a record that was listed but not feeding a store);
- drift as a concept: with one record per store there is nothing to compare, and a provider change is the `Changed` badge.

---

## 6. Rollout

| Step | Action | Phase |
|---|---|---|
| 1 | Ship the single list with `Live` and `Mark as duplicate`; keep the OC pages read-only for one cycle | 1 → 2 |
| 2 | Copy `live` and `blocked` from `oc_location` onto the record that supplied it (`live_source`), then drop the table | 2 |
| 3 | Tell downstream once: `MC_LEGACY_OC_ID(vendor_id, legacy_oc_id)` | 2 |
| 4 | Retire the `/oc-locations` endpoints after the last consumer confirms | 2 |

Record ids never change, so no bookmark and no downstream key breaks on the day of the switch.

---

## 7. Open questions

| # | Question | Owner |
|---|---|---|
| 1 | Should the two records of a pair be able to go live independently, or is "one live per store" the rule? Phase 2 keeps the latter. | Operations |
| 2 | When a store's two records sit in different zips, the pair spans two zips — acceptable downstream? | Product / Data |
| 3 | Does the eBay integration already key on `vendor_location.id`, or does it need the legacy map indefinitely? | Downstream |
| 4 | Is a bulk duplicate action (many pairs) needed, or is two-at-a-time enough? | Operations |

---

*End of document. Version 1.1, 2026-11-25.*
