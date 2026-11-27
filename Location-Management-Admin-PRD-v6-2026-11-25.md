# Location Management Admin Console — Product Requirements Document

**Version 8.3.0 ｜ 2026-11-27 ｜ Status: draft for review**

Companion files: clickable prototype (`index.html`), Duplicate check v3.0 (rule document), field mapping (v5.1.0, pending update to this model).

---

## 1. What changed, and why

Version 5.x split the work across three pages: a vendor list, an OC list, and a separate duplicate-check page. In practice the operator did not work that way. Every decision started from a provider record and ended on the same record, and the split forced three navigations to answer one question.

Version 6.0.0 collapses the console into **one list**. Sync state, the whitelist switch, the duplicate decision and the OC location id all live on the row they belong to.

| | v5.1.0 | v6.0.0 |
|---|---|---|
| Pages | Vendor list, OC list, Duplicate Check | **Vendor Location List only** |
| Store identity | a second id (`OC-…`) held on a separate record | **the vendor location id itself**; duplicates linked by a marker |
| Change awareness | implied by drift badges on the OC page | explicit **New / Changed** marker against the last snapshot |
| Pushing to eBay | values flowed downstream on their own | **going live is the publish**; no separate apply step |
| Duplicate decision | a dedicated page with a queue | select two rows, **Mark as duplicate** |
| Whitelist | per-store sign-off on the OC page | **Live switch on the row**, plus batch add/remove |
| Filters | three text conditions plus facets | adds **Zipcode**, **Live**, **Change**, **Duplicates** |

The rule document (Duplicate check v3.0) still governs how similarity is computed. This PRD governs what the operator sees and does. Where the two disagree about identity, this document wins: the prototype is the contract for this release.

---

## 2. Scope

### 2.1 In scope

- One list of every vendor location the providers send, with its sync state, live state and OC location id.
- Change detection against the previous sync snapshot, at field level.
- Applying new or changed records to eBay, one at a time or in bulk.
- Putting records on the whitelist (going live) and taking them off, one at a time or in bulk.
- Marking two records as one store, which produces the OC location id both of them carry.
- Filtering, sorting and exporting the list.
- A record profile that shows what changed and the history of it.

### 2.2 Out of scope for this release

- Editing provider values. The console does not correct addresses; it flags them, and the provider fixes the source.
- Automatic duplicate detection. Similarity may be shown as a hint, but a person makes the call.
- Automatic go-live. A record never enters the whitelist by itself.
- Multi-store batching by region or campaign. Bulk operations operate on selected rows only.
- Permissions beyond the two existing roles (Operations / Viewer).

---

## 3. The model in one page

```
Provider sync (23:30, one batch)
        |
        v
+------------------------------------------------+
|  vendor_location                               |
|  id            <- the only identity            |
|  name, address, city, state, zip               |
|  source, sweeper, qr_dropoff, status           |
|  live, blocked                                 |
|  duplicate_of  (points at the record it dups)  |
|  sync_state (new | changed | synced)           |
|  sync_fields, last_synced_at                   |
+------------------------------------------------+
```

Three facts drive every behaviour in this document.

1. **The vendor location id is the identity.** There is no second key. A record is a store; when two records turn out to be the same store, one carries `duplicate_of` pointing at the other, and both keep their ids.
2. **A store has one live record.** Of the two records in a duplicate pair, exactly one may be live at a time; putting one live takes the other off.
3. **A changed record is not live.** When the provider moves a value, the record leaves the whitelist until somebody puts it back. Nothing stale reaches a customer.

---

## 4. Data model

### 4.1 vendor_location

| Field | Type | Notes |
|---|---|---|
| `id` | varchar | `LOC-xxxxx`, assigned at intake, never rewritten. The downstream key |
| `name`, `address_line1`, `address_line2`, `city`, `state`, `postal_code` | varchar | as delivered; never edited here |
| `source` | enum | `PUDOpoint (PUDO)` / `packagehub (LMS)` / `M&MP` |
| `sweeper` | enum | who actually collects; `PUDO Inc` / `LMS` |
| `status` | enum | `active` / `inactive` / `removed`, from the provider |
| `qr_dropoff` | boolean | provider capability: the counter scans a customer code |
| `live` | boolean | on the whitelist; being live is how a value reaches eBay |
| `blocked` | boolean | read-only here; set by the platform |
| `list_removed_reason` | enum NULL | `UNSIGNED` / `PENDING_CHANGE` / `DUPLICATE_SWITCHED` / `PROVIDER_DISABLED` / `RECORD_VOIDED` |
| `duplicate_of` | varchar NULL | the record this one duplicates; `NULL` for the primary |
| `sync_state` | enum | `new` / `changed` / `synced` |
| `sync_fields` | json NULL | which fields moved, for the Changed marker |
| `last_synced_at` | timestamp | when eBay last accepted this record |

### 4.2 What is *not* in the model

**There is no `oc_location` table.** Version 8.0.0 removed it, and with it the `OC-{ST2}-{City3}-{Zip5}-{seq6}` id. The table existed to give a store one identity while several source records fed it; once a record *is* the store, that identity was a second name for the same row, and every rule about it (minting, never rewriting, retiring on merge) was bookkeeping for a duplicate key.

Consequences worth naming: no mint sequence, no "OC Location ID" filter, no store profile separate from the record, and downstream joins on `vendor_location.id` directly.

### 4.3 Invariants

| # | Invariant | Enforced by |
|---|---|---|
| I1 | A duplicate keeps its own id and never adopts its primary's | the decision writes `duplicate_of` only |
| I2 | At most one record of a duplicate pair is live | going live unlists its counterpart |
| I3 | `blocked` is never written by this console | no write path exists |
| I4 | A record with `sync_state != synced` is not live | changed records are taken off the list on ingest |
| I5 | An id is never rewritten once issued | minting is append-only |
| I6 | A record whose provider status is not `active` is never live | the switch is disabled, the API refuses, batch add skips |

---

## 5. Sync and change detection

### 5.1 Snapshot

Every run stores the full provider payload before it is applied. The next run diffs against that snapshot, not against the last value we happened to keep on the row.

```
run N:   snapshot_N = provider payload
         diff(N, N-1) -> per record: new | changed(fields) | unchanged
run N+1: snapshot_{N+1} = provider payload
         diff(N+1, N) -> ...
```

### 5.2 Classification

| Condition | `sync_state` | On the list |
|---|---|---|
| The record did not exist in the previous snapshot | `new` | `New` badge under the id |
| The record exists and at least one compared field differs | `changed` | `Changed` badge; the differences are listed in the profile |
| The record exists and nothing compared differs | `synced` | no badge |

Compared fields: `name`, `address_line1`, `address_line2`, `city`, `state`, `postal_code`, `phone`, `opening_hours`, `services`, `sweeper`, `status`. Coordinates are stored and shown but do not produce a change signal of their own.

### 5.3 The consequence: a changed record leaves the list

When a record is classified `changed` or `new` and it is currently supplying a store, the store is taken down as part of ingest:

```
vendor_location.live = false
log PENDING_CHANGE_OFFLINE(vendor_id)
log WHITELIST_REMOVED(vendor_id, reason = PENDING_CHANGE)
```

The id is not touched. The store is simply not live until a person acts. This is the one place where the console refuses to be helpful: a customer must never receive a value the operator has not seen.

---

## 6. The list

### 6.1 Columns

| # | Column | Content |
|---|---|---|
| 1 | Select | checkbox for batch actions |
| 2 | **Vendor Location ID** | id, clickable to the profile; `New` or `Changed` badge underneath when applicable |
| 3 | Name | provider name |
| 4 | Address | line 1, line 2 in a lighter tone |
| 5 | City / Province | `Brooklyn, NY` |
| 6 | Zip | first five digits |
| 7 | Source | `PUDOpoint (PUDO)` etc. |
| 8 | Sweeper | badge, or `missing` |
| 9 | **QR** | `QR` badge when the source supports QR drop-off, `—` when it does not |
| 10 | Status | provider status badge |
| 11 | Live | switch on every row; `Blocked` badge when the platform blocks the store |
| 12 | Duplicate | `duplicate of LOC-xxxxx` on the duplicate, `—` on a primary |
| 13 | Issues | count of open issues, clickable |
| 14 | Updated | provider timestamp |

QR drop-off is a **provider capability**, not a note: the customer receives a code and the counter scans it. It sits beside Sweeper because both are facts about how the store operates, and it is read from the feed — the console never guesses it, and a dash means the source does not offer it.

### 6.2 Badges, not a column

There is no Sync column. A record that matches eBay carries no badge at all; only the ones that moved are marked. A list where every row shouts is a list nobody scans.

### 6.3 Ordering

Rows are grouped by pair: a store and its duplicate sit on adjacent lines, primary first. Within a group the order is stable. Outside a group, the provider order is preserved.

When a Zipcode filter is active and more than one row matches, the rows are ranked by **the duplicate check's own score against one anchor**, before grouping. The anchor is the **oldest record in the result set** — earliest `created`, tie-broken by id. The score is address 40 · zip 25 · state 15 · city 10 · name 10, computed on adjacent character pairs exactly as the pair modal computes it, and printed on every other row; the anchor carries an `anchor` chip in place of a number.

Using the duplicate score rather than a bespoke one is the point: *"why is this row first?"* has to have the same answer on the list and in the pair dialog. Inside one zip the 25 zip points are constant, so the order is carried by the street, the name and the occasional city mismatch — which is the evidence a person reads anyway. Name similarity breaks ties, then the id, so the order is total and stable.

The anchor is deliberately not "the first row". A base that depends on the current sort re-ranks the whole list whenever another filter changes, which reads as if the data moved. The oldest record is fixed, and it is also the row every later record could be a duplicate of. Grouping happens after ranking, so a **marked** duplicate still sits next to its primary.

### 6.4 Empty state

A filter combination with no matches says so and offers a Reset. It does not silently show the full list.

---

## 7. Live and the whitelist

### 7.1 The switch

| Row state | Rendered |
|---|---|
| `status` is `inactive` or `removed` | disabled `Not live` switch: the store is not operating, so it cannot be released to eBay |
| `blocked` | red `Blocked` badge, not clickable |
| `live = true` | green capsule `Live` |
| `live = false` | grey capsule `Not live` |

Clicking a capsule toggles the store. The operation requires the Operations role.

There is nothing to mint first: a record always has its id, so the switch is a single write on the row.

### 7.2 One source per store

The two records of a duplicate pair can never both be live. They hold different ids now, so the rule cannot come from the id — it is enforced on the pair. Turning one on turns the other off in the same click:

```
live(record A) = true
if the pair's other record is live:
    live(record B) = false, list_removed_reason(B) = DUPLICATE_SWITCHED
    log DUPLICATE_UNLISTED(B.id, reason = DUPLICATE_SWITCHED)
    log WHITELIST_REMOVED(B, reason = DUPLICATE_SWITCHED)
```

`LIVE_SOURCE_CHANGED` is gone with the concept of a store that can switch supplier: a record either supplies the store or does not.

Turning off the record that is currently live takes the **store** down.

### 7.3 Going live applies the values

A record whose `sync_state` is not `synced` cannot be live (I4). Going live therefore does two things at once: it writes the current values to eBay and flips the switch.

```
if sync_state != synced:
    sync_state = synced
    log CHANGE_APPLIED(vendor_id, previous_state, synced)
live = true
```

This is deliberate. There is no separate "confirm" step; confirming without publishing would leave a record marked clean while eBay still holds something else.

### 7.4 Batch

Selecting rows reveals a batch bar: `Add to whitelist & apply`, `Remove from whitelist`, `Mark as duplicate`, `Clear selection`.

Batch add follows the same rules per row: an id is minted when missing, the switch flips, the values are applied, and a row that cannot go live is skipped and counted. The result toast reports processed and skipped counts.

---

## 8. Duplicate handling

### 8.1 The decision

A store may arrive as **two records or three or four**. The decision is the same shape at every size: pick **one record to keep**, and every other record in the group points at it.

| Group size | Where it starts | The dialog |
|---|---|---|
| 2 | two selected rows, or a pair row on the suspected board | both records side by side (id, name, address, city, zip) + the score as **reference only** |
| 3+ | the cluster card on the suspected board, or 3 rows selected in the table | a **picker**: one row per record (id, name, address, live state), radio to keep one |

- **Keep as the store** is the only control — a radio, defaulting to the record that is live, else the one with the earliest `created`, else the lowest id.
- There is no id to choose, type or merge: `duplicate_of` **is** the decision, and every record keeps its own id.

### 8.2 Outcomes

| Situation | Result |
|---|---|
| Group of 2 | the other record gets `duplicate_of = kept.id` |
| Group of 3+ | **every** other record gets `duplicate_of = kept.id` — a star, never a chain |
| A dropped record is live | it is unlisted, because only one record of a store may supply eBay |
| The kept record was itself a duplicate | that old link is cleared first, so the keeper becomes the root of its group |
| Re-marking an already-marked group | the direction is refreshed; no id ever changes |

`duplicate_of` always points at the **keeper**, never at another duplicate. That is a rule, not a preference: "who is the store" has to be one hop away, because ordering, the `Duplicate` column, live exclusivity and the export all read it. A chain would make each of them walk.

After confirmation each dropped record is written to `duplicate_pair` with status `merged`. **No id is rewritten, merged or retired** — the operation writes a relationship, not an identity.

### 8.3 What a duplicate is not

A duplicate is a **record**, not a store. It keeps its own values, its own issues and its own history. It can be the live source if the operator prefers it (section 7.2) — going live on any member takes every other member off, not just the one the dialog named. It cannot exist without a primary, and it can never be a keeper.

### 8.4 Filtering duplicates

| Value | Returns |
|---|---|
| `Duplicated records` | both sides of every group, keeper first, members adjacent |
| `Suspected duplicates` | see 10.1 — a board of pairs and groups that are not marked yet |
| `No duplicate` | the rest |

The filter reads the relationship, not the id.

---

## 9. How a value reaches eBay

There is one path: **going live**. A record's values are written to eBay when the record is put on the whitelist, and at no other time.

That is the whole publish model. It also answers the question the old screen asked with a button: if a store is not live, eBay has no reason to hold its values; if it is live and the provider moved a field, it was taken off the list on ingest, and putting it back publishes the new value.

Two ways to go live, same effect per row:

| Action | Writes values to eBay | Flips live |
|---|---|---|
| Clicking `Live` on a row | yes, for that record | yes |
| `Add to whitelist & apply` (batch, after filtering) | yes, for each selected record | yes |

The described flow is therefore: filter `Change = New` or `Changed`, select the rows, `Add to whitelist & apply`. Nothing else is needed, and there is nothing to reconcile afterwards.

An earlier draft carried `Apply to eBay` and, before that, `Push differences`. Both were second paths to the same effect, and both allowed a record to be marked clean while eBay still held the previous value. A single path cannot drift from itself.

---

## 10. Filters

All filters are multi-select with type-to-find. Values within one filter are OR; filters combine with AND. Selected values are shown as chips; Reset clears every control, including the ones added in this release.

| Filter | Values | Notes |
|---|---|---|
| **Vendor Location ID** | up to 20 conditions, space or comma separated | substring; any match |
| Name | up to 20 conditions | a condition is a phrase, all words required |
| Province | facet | counts reflect the other filters |
| City | facet | cascades from Province |
| Zipcode | up to 20 conditions | first five digits; a multi-row result is ranked by the duplicate score against the oldest record, and that score is printed on the row |
| Source | facet | three values |
| Status | facet | provider status |
| Live | facet | `Live` / `Not live` / `Blocked by system` |
| Change | facet | `New` / `Changed` / `No change` |
| Duplicates | facet | `Duplicated records` / `Suspected duplicates` / `No duplicate`; the option carries the pair count |
| Sweeper | facet | `PUDO Inc` / `LMS` |
| Issues | facet | has open issue / none |

### 10.1 Suspected duplicates (8.2.0)

A **marked** duplicate is a fact about a record — the row says `duplicate of` and the table is the right place for it. A **suspected** one is a fact about a *pair*, and one record can sit in several. Listing those as rows is what turns a nine-record zip into eleven rows that look like eleven stores, so `Duplicates = Suspected duplicates` opens a **board** instead of filtering the table.

The board is always scoped, in three steps:

| Step | Shown | Why |
|---|---|---|
| No city chosen | one card per city: `Brooklyn · 2 zips · 12 pairs` | Nothing is decided at city level, but it is the smallest unit an operator owns — and it keeps the next screen short |
| City chosen | one tab per zip that **has** a pair, with the count; no tab for an empty zip | The zip is the neighbourhood a person can actually read; a tab that leads nowhere is a dead end |
| Zip chosen | one card per group of three or more, then the plain pairs, HIGH first; MEDIUM collapsed behind one summary line | A pair is the decision unit, so it gets a row — and when three records are all HIGH against each other, three rows would pretend to be three decisions instead of one |

Rules:

- The pair list is computed on **the same filters as the table** (source, status, live, change, issues). The city list and the zip tabs are derived from that result, so a tab can never open onto nothing.
- Pairs where either record is already marked are excluded — a confirmed decision leaves the board and lives in the `Duplicate` column.
- The city facet narrows itself to cities that have suspected pairs while this filter is on; it is therefore impossible to select a city that renders an empty board.
- The score is the duplicate check's score, the same number the pair dialog prints.
- A **cluster card** holds three or more records connected by HIGH pairs. It names the members, the number of pairs inside it and the strongest score, and offers one action — `These N are one store` — which opens the picker in 8.1. MEDIUM links never create a cluster: within one zip, zip + state + city already pay 50 points, so a MEDIUM edge between neighbours would chain a whole neighbourhood into one blob.
- Clearing the filter brings the table back. The board never coexists with the table: they answer different questions.

---

## 11. Record profile

Clicking a Location ID opens the profile. Five tabs: Basic Info, Operations, Hours and Services, Issues, Change History.

### 11.1 Field-level change

In Basic Info, a field that moved since the last snapshot carries a `changed` badge and the previous value:

```
Address        2918 Avenue I   [changed]
               was 2908 Avenue I
Postal code    11210           [changed]
               was 11210-2214
```

Only fields that actually changed are marked. A profile with nothing changed looks like a normal profile.

### 11.2 Change history

The history tab lists every entry for the record, newest first, including one row per changed field from the sync run:

| Time | Type | Field | Old | New |
|---|---|---|---|---|
| 2026-11-25 02:14 | PROVIDER_SYNC | addr1 | 2908 Avenue I | 2918 Avenue I |
| 2026-11-25 02:14 | PROVIDER_SYNC | postal | 11210-2214 | 11210 |

Operator actions appear in the same timeline: whitelist changes, applied values, duplicate decisions.

---

## 12. State machine

Two independent dimensions, which is why a single status column was a mistake.

**Sync state**

```
new ──────────(go live)─────────> synced
 │                                   │
 │                          (provider moves a field)
 │                                   v
 └────────(provider moves)──────> changed ──(go live)──> synced
```

**Live state**

```
not live ──(switch on, sync_state = synced)──> live
    ^                                            │
    │                              (provider moves a field)
    └──────────── live = false ──────────────────┘
```

`blocked` sits outside both: it is a platform state, readable and not writable here. A blocked store cannot be switched on, and the attempt is refused rather than silently failing.

---

## 13. API surface (draft)

| Method | Path | Purpose |
|---|---|---|
| GET | `/vendor-locations` | list with filters, paging, sort |
| GET | `/vendor-locations/{id}` | profile, including `sync_fields` and previous values |
| POST | `/vendor-locations/{id}/live` | `{ live: true }` — applies the current values, flips the switch |
| POST | `/vendor-locations/live` | bulk, `{ ids, live }` |
| POST | `/duplicates` | `{ a, b, keep }` — write `duplicate_of`, unlist the dropped record if it was live |
| GET | `/duplicates/{id}` | pair detail |
| GET | `/change-log?loc_id=` | history for one record |

Errors worth designing for: switching a blocked store, going live on an `inactive` record, and marking a pair that would leave two live records for one store.

---

## 14. Acceptance criteria

1. The sidebar contains exactly one business page. No store list, no duplicate page, and no OC id anywhere in the interface.
2. A record that is new or changed shows a badge under its id. A synced record shows nothing in that position.
3. A changed record is not live after ingest, and the whitelist shows it as removed with reason `PENDING_CHANGE`.
4. Clicking the badge-bearing id opens a profile where the moved fields are marked `changed` and show the previous value.
5. The Change History tab lists one `PROVIDER_SYNC` row per moved field, newest first.
6. Clicking `Live` on a record with `sync_state = changed` publishes the current values and clears the badge in one action.
7. Two records of a pair cannot both be live. Turning one on unlists the other and logs `DUPLICATE_UNLISTED`.
8. A blocked store renders a badge, not a switch.
9. There is no `Apply to eBay` button: filtering `Change = New`, selecting the rows and pressing `Add to whitelist & apply` writes the values, clears the badges and puts the stores live in one action.
10. Selecting two records and confirming `Mark as duplicate` writes `duplicate_of` on the dropped one; neither id changes, and nothing is minted.
11. The duplicate row shows `duplicate of LOC-xxxxx`; the primary shows `—`. Both rows still show their own vendor location id.
12. The `Duplicates` filter returns the pair together, adjacent, primary first.
13. A Zipcode lookup returns every record of that zip, ranked by the duplicate-check score against the oldest record in the result; the score is printed on each other row and the anchor is labelled. The same number appears in `Mark as duplicate` for that pair. Narrowing the same zip with another filter changes neither the anchor nor the relative order.
14. Reset clears the text fields, all nine facets and the zip condition; the row count returns to the full list.
15. Either record of a pair can be made live; doing so unlists the other one and logs `DUPLICATE_UNLISTED`.
16. Batch add skips rows that cannot go live and reports how many were skipped.
17. Export writes one row per vendor record with its id, live state and duplicate marker.
18. Every row carries a live switch, including records with no id; clicking one mints the id and puts the store live in a single action.
20. A record whose provider status is `inactive` (or `removed`) renders a disabled switch, cannot be switched live by any path, and mints no id.
21. The QR column sits beside Sweeper and shows `QR` only where the source offers drop-off; elsewhere a dash.
19. The id column and the id filter are both labelled `Vendor Location ID`; no bare "Location ID" survives in the interface.
20. No `OC-…` id appears in the list, the filters, the profile, the export or the API. Release history is the only place the old id is still named.
21. `Duplicates = Suspected duplicates` hides the record table and shows the pair board. With no city selected it lists cities that have pairs; with a city it lists one tab per zip that has a pair, and no tab otherwise.
22. The board lists unmarked pairs only, HIGH before MEDIUM, each with both ids, the score and one `Mark as duplicate`; confirming a pair removes it from the board and writes `duplicate_of`.
23. Three or more records connected by HIGH pairs appear as **one cluster card** with N members and one action; the action opens a picker listing every member, and confirming writes `duplicate_of = keeper` on all the others — never on another duplicate.
24. Going live on any member of a group unlists every other member, and each unlist is logged with its own id.

---

## 15. Open questions

| # | Question | Owner |
|---|---|---|
| 1 | When a changed record is taken off the list, should the store go down immediately or at the end of the sync window? | Operations |
| 2 | Should going live on a changed record require an explicit confirmation of the diff, rather than doing both in one click? | Product |
| 3 | When a store's two records sit in different zips, each id follows its own record's zip — is that the intent, or should the pair share one? | Product / Data |
| 4 | How long should a retired id keep resolving, for downstream orders that still reference it? | Downstream contract |
| 5 | Should the duplicate marker be reversible without a merge, i.e. can a pair be un-marked? | Operations |
| 6 | Do we need a bulk duplicate action (many pairs at once), or is two-at-a-time enough? | Operations |

---

## 16. Migration from 5.x

| Step | Action |
|---|---|
| 1 | Copy `live` and `blocked` from `oc_location` onto the record that supplied it (`live_source`), then drop the table |
| 2 | Backfill `duplicate_of` from the old shared-id groups: every member except the live one points at the record that kept the group |
| 3 | Leave `vendor_location.id` untouched — it is now the only key, and downstream joins on it |
| 4 | Tell downstream once: `MC_LEGACY_OC_ID(vendor_id, legacy_oc_id)` for every record that used to answer to an `OC-…` id, so they can re-key without guessing |
| 5 | Retire the `/oc-locations` endpoints after the last consumer confirms |

---

*End of document. Version 8.3.0, 2026-11-27.*
