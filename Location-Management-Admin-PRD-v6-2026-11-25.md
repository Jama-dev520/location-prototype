# Location Management Admin Console — Product Requirements Document

**Version 6.0.1 ｜ 2026-11-25 ｜ Status: draft for review**

Companion files: clickable prototype (`index.html`), Duplicate check v3.0 (rule document), field mapping (v5.1.0, pending update to this model).

---

## 1. What changed, and why

Version 5.x split the work across three pages: a vendor list, an OC list, and a separate duplicate-check page. In practice the operator did not work that way. Every decision started from a provider record and ended on the same record, and the split forced three navigations to answer one question.

Version 6.0.0 collapses the console into **one list**. Sync state, the whitelist switch, the duplicate decision and the OC location id all live on the row they belong to.

| | v5.1.0 | v6.0.0 |
|---|---|---|
| Pages | Vendor list, OC list, Duplicate Check | **Vendor Location List only** |
| OC identity | separate entity holding a list of vendor records | a value on the vendor record; duplicate records share the same value |
| Change awareness | implied by drift badges on the OC page | explicit **New / Changed** marker against the last snapshot |
| Pushing to eBay | values flowed downstream on their own | **Apply to eBay**, plus going live, which also applies |
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
+-------------------------------------------+
|  vendor_location                          |
|  id, name, address, city, state, zip,     |
|  source, sweeper, status                  |
|  oc_location_id  ----------------------+  |
|  duplicate_of    --------------------+ |  |
|  sync_state (new | changed | synced) | |  |
+--------------------------------------|-|-+
                                       | |
                        +--------------+ |
                        v                v
                 +-------------------------------+
                 |  oc_location                  |
                 |  id, name, address            |
                 |  live (boolean)               |
                 |  live_source (one vendor id)  |
                 |  blocked (database only)      |
                 +-------------------------------+
```

Three facts drive every behaviour in this document.

1. **The OC location id is a value on the vendor record, not a container.** One record can hold it, or several can hold the same one when they are the same store. There is no separate "OC record with a list of members" to keep in sync.
2. **A store has one live source.** Of the records that share an id, exactly one may be live at a time. That is enforced by a single pointer, not by the interface.
3. **A changed record is not live.** When the provider moves a value, the record leaves the whitelist until somebody puts it back. Nothing stale reaches a customer.

---

## 4. Data model

### 4.1 vendor_location

| Field | Type | Notes |
|---|---|---|
| `id` | varchar | `LOC-xxxxx` |
| `name`, `address_line1`, `address_line2`, `city`, `state`, `postal_code` | varchar | as delivered; never edited here |
| `source` | enum | `PUDOpoint (PUDO)` / `packagehub (LMS)` / `M&MP` |
| `sweeper` | enum | who actually collects; `PUDO Inc` / `LMS` |
| `status` | enum | `active` / `inactive` / `removed`, from the provider |
| `oc_location_id` | varchar NULL | shared by duplicate records |
| `duplicate_of` | varchar NULL | the record this one duplicates; `NULL` for the primary |
| `sync_state` | enum | `new` / `changed` / `synced` |
| `sync_fields` | json NULL | which fields moved, for the Changed marker |
| `last_synced_at` | timestamp | when eBay last accepted this record |

### 4.2 oc_location

| Field | Type | Notes |
|---|---|---|
| `id` | varchar(22) | `OC-{ST2}-{City3}-{Zip5}-{seq6}` |
| `live` | boolean | on the whitelist |
| `live_source` | varchar NULL | the vendor record currently supplying the store |
| `blocked` | boolean | read-only in this console; set by the platform |
| `name`, `address`, `city`, `state`, `postal_code` | varchar | snapshot at issuance; never used for filtering |

### 4.3 Invariants

| # | Invariant | Enforced by |
|---|---|---|
| I1 | A record with `duplicate_of` set never holds a different id from its primary | duplicate decision |
| I2 | At most one record per `oc_location_id` is live | `live_source` is a single value |
| I3 | `blocked` is never written by this console | no write path exists |
| I4 | A record with `sync_state != synced` is not live | changed records are taken off the list on ingest |
| I5 | An id is never rewritten once issued | minting is append-only |

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
oc_location.live = false
oc_location.live_source = null
log PENDING_CHANGE_OFFLINE(oc_id, vendor_id)
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
| 9 | Status | provider status badge |
| 10 | Live | switch on every row; `Blocked` badge when the platform blocks the store |
| 11 | OC Location | the id, plus `duplicate of LOC-xxxxx` on the duplicate |
| 12 | Issues | count of open issues, clickable |
| 13 | Updated | provider timestamp |

### 6.2 Badges, not a column

There is no Sync column. A record that matches eBay carries no badge at all; only the ones that moved are marked. A list where every row shouts is a list nobody scans.

### 6.3 Ordering

Rows are grouped by `oc_location_id`. A store and its duplicate sit on adjacent lines, primary first. Within a group the order is stable. Outside a group, the provider order is preserved.

When a Zipcode filter is active and more than one row matches, the result is sorted by similarity to the first match, descending, before grouping. Looking up a zip is an act of hunting for a duplicate; the most similar records belong at the top.

### 6.4 Empty state

A filter combination with no matches says so and offers a Reset. It does not silently show the full list.

---

## 7. Live and the whitelist

### 7.1 The switch

| Row state | Rendered |
|---|---|
| No `oc_location_id` | grey `Not live` switch, tooltip: going live mints its OC location id |
| `blocked` | red `Blocked` badge, not clickable |
| live, this record is the source | green capsule `Live` |
| not live, or another record is the source | grey capsule `Not live` |

Clicking a capsule toggles the store. The operation requires the Operations role.

A record with no id is not special-cased any more: its switch mints the id and flips the store in the same click, so nobody has to whitelist twice.

```
if oc_location_id is null:
    oc_location_id = mint(state, city, zip5)
    log OC_ID_MINTED(auto, no_candidate)
```

### 7.2 One source per store

Two records that share an id cannot both be live. Turning one on turns the other off, in the same transaction:

```
live = true, live_source = this record
if previous source existed and differs:
    log LIVE_SOURCE_CHANGED(oc_id, previous, current)
    log WHITELIST_REMOVED(previous, reason = SOURCE_SWITCHED)
    log WHITELIST_ADDED(current, oc_id)
```

Turning off the record that is currently live takes the **store** down, not just the record.

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

Selecting rows reveals a batch bar: `Add to whitelist`, `Remove from whitelist`, `Mark as duplicate`, `Clear selection`.

Batch add follows the same rules per row: an id is minted when missing, the switch flips, the values are applied, and a row that cannot go live is skipped and counted. The result toast reports processed and skipped counts.

---

## 8. Duplicate handling

### 8.1 The decision

Select exactly two rows, then `Mark as duplicate`. The dialog shows both records side by side (id, name, address, city, zip) and the similarity score as **reference only**. Two controls matter:

- **Keep as the store** — a radio choice of A or B, defaulting to whichever record already holds an id.
- **OC Location ID** — prefilled from the kept record's state, city and zip; editable, and explicitly **not required to match either address**. A store merged across two zips needs an id that describes neither.

### 8.2 Outcomes

| Situation | Result |
|---|---|
| Neither record has an id | mint one; both records carry it |
| The kept record has an id | the other record adopts it |
| Both have ids and they differ | keep the one on the kept record, retire the other (`discarded_reason = merged`), both records end on the survivor |
| Both already share an id | nothing to mint; the dialog says so and the action only re-marks the pair |

After confirmation the dropped record gets `duplicate_of = kept.id`, the store's `live_source` is set to the kept record, and the pair is written to `duplicate_pair` with status `merged`.

### 8.3 What a duplicate is not

A duplicate is a **record**, not a store. It keeps its own values, its own issues and its own history. It can be the live source if the operator prefers it (section 7.2). It cannot exist without a primary.

### 8.4 Filtering duplicates

`Duplicates` filter: `Duplicated records` returns every record whose id carries more than one record, primary and duplicate together, adjacent. `No duplicate` returns the rest.

---

## 9. Apply to eBay

`Apply to eBay · n` pushes every record whose `sync_state` is `new` or `changed` and marks them `synced`. The counter is the number of pending records; the button is disabled at zero.

What it does not do: it does not put anything live. A record can be fully in sync and still not be on the whitelist.

| Action | Writes values to eBay | Flips live |
|---|---|---|
| `Apply to eBay` | yes, for every pending record | no |
| Clicking `Live` on a row | yes, for that record | yes |
| `Add to whitelist` (batch) | yes, for each selected record | yes |

There is no third push action. An earlier draft had both `Apply to eBay` and `Push differences`; they were the same operation with different filters, and the narrower one was a subset of the wider one.

---

## 10. Filters

All filters are multi-select with type-to-find. Values within one filter are OR; filters combine with AND. Selected values are shown as chips; Reset clears every control, including the ones added in this release.

| Filter | Values | Notes |
|---|---|---|
| **Vendor Location ID** | up to 20 conditions, space or comma separated | substring; any match |
| OC Location ID | up to 20 conditions | substring; matches the id on the record |
| Name | up to 20 conditions | a condition is a phrase, all words required |
| Province | facet | counts reflect the other filters |
| City | facet | cascades from Province |
| Zipcode | up to 20 conditions | first five digits; activates similarity ordering |
| Source | facet | three values |
| Status | facet | provider status |
| Live | facet | `Live` / `Not live` / `Blocked by system` |
| Change | facet | `New` / `Changed` / `No change` |
| Duplicates | facet | `Duplicated records` / `No duplicate` |
| Sweeper | facet | `PUDO Inc` / `LMS` |
| Issues | facet | has open issue / none |

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
new ──────(apply / go live)──────> synced
 │                                   │
 │                          (provider moves a field)
 │                                   v
 └────────(provider moves)──────> changed ──(apply / go live)──> synced
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
| POST | `/vendor-locations/{id}/apply` | push current values to eBay, mark synced |
| POST | `/vendor-locations/apply` | bulk apply by id list |
| POST | `/vendor-locations/{id}/live` | `{ live: true }` — mints an id if missing, applies, flips |
| POST | `/vendor-locations/live` | bulk, `{ ids, live }` |
| POST | `/duplicates` | `{ a, b, keep, oc_location_id? }` — mark, mint or reuse, return the id |
| GET | `/duplicates/{id}` | pair detail |
| GET | `/oc-locations/{id}` | store detail, live source, members |
| GET | `/change-log?loc_id=` | history for one record |

Errors worth designing for: minting a record that already has an id, switching a blocked store, going live with `sync_state != synced`, and marking a pair whose two records already sit on different ids (this one needs a merge, not a link).

---

## 14. Acceptance criteria

1. The sidebar contains exactly one business page. The OC list and the duplicate page are not reachable.
2. A record that is new or changed shows a badge under its id. A synced record shows nothing in that position.
3. A changed record is not live after ingest, and the whitelist shows it as removed with reason `PENDING_CHANGE`.
4. Clicking the badge-bearing id opens a profile where the moved fields are marked `changed` and show the previous value.
5. The Change History tab lists one `PROVIDER_SYNC` row per moved field, newest first.
6. Clicking `Live` on a record with `sync_state = changed` publishes the current values and clears the badge in one action.
7. Two records that share an id cannot both be live. Turning one on turns the other off and logs `LIVE_SOURCE_CHANGED`.
8. A blocked store renders a badge, not a switch.
9. `Apply to eBay` shows the pending count, is disabled at zero, and clears every badge when pressed.
10. Selecting two free records and confirming `Mark as duplicate` produces one id, both records carry it, and the dropped one carries `duplicate_of`.
11. Marking a pair whose records already hold different ids retires one id and leaves both records on the survivor.
12. The `Duplicates` filter returns the pair together, adjacent, primary first.
13. A Zipcode lookup returns every record in that zip, most similar first.
14. Reset clears the text fields, all nine facets and the zip condition; the row count returns to the full list.
15. A duplicate row can be made live on its own, and doing so takes the previous source off the list.
16. Batch add skips rows that cannot go live and reports how many were skipped.
17. Export writes one row per vendor record with its id, live state and duplicate marker.
18. Every row carries a live switch, including records with no id; clicking one mints the id and puts the store live in a single action.
19. The id column and the id filter are both labelled `Vendor Location ID`; no bare "Location ID" survives in the interface.

---

## 15. Open questions

| # | Question | Owner |
|---|---|---|
| 1 | When a changed record is taken off the list, should the store go down immediately or at the end of the sync window? | Operations |
| 2 | Should going live on a changed record require an explicit confirmation of the diff, rather than doing both in one click? | Product |
| 3 | For a store merged across two zips, which zip belongs in the id when the operator does not choose one? | Product / Data |
| 4 | How long should a retired id keep resolving, for downstream orders that still reference it? | Downstream contract |
| 5 | Should the duplicate marker be reversible without a merge, i.e. can a pair be un-marked? | Operations |
| 6 | Do we need a bulk duplicate action (many pairs at once), or is two-at-a-time enough? | Operations |

---

## 16. Migration from 5.x

| Step | Action |
|---|---|
| 1 | Backfill `vendor_location.oc_location_id` from the existing mapping table; every member of an OC receives the same value |
| 2 | Backfill `duplicate_of` for records that share an id: the non-live member points at the live one |
| 3 | Keep `oc_location` as the store record (`live`, `live_source`, `blocked`); drop the member list, which is now derived |
| 4 | Leave existing ids untouched. A record's id does not change because the console changed |
| 5 | Retire the OC list and duplicate page from navigation, keep their APIs until downstream confirms no consumer |

---

*End of document. Version 6.0.1, 2026-11-25.*
