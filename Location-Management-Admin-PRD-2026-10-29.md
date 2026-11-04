# Location Management Admin Console — Product Requirements Document (PRD)

## 1. Document Information

### 1.1 Basic Information

| Field            | Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Document Version | v3.2.1 |
| Author           | Product Team (drafted via requirement clarification session)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Created Date     | 2026-10-29                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Status           | Draft — pending review                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Change Log       | v1.0 Initial draft; v1.1 (2026-10-29) duplicate detection changed from exact address match to fuzzy match-score threshold (75%), added OC location ID filter; v1.2 (2026-10-29) published interactive prototype to public GitHub Pages + converted Mermaid flowcharts to rendered PNG images; **v2.0 (2026-10-30) restructured the console from a single merged location list into a two-layer model — Vendor Location List (provider view) + OC Location List (OC-owned view)**; v2.1.0 (2026-10-30) multi-range opening hours (one day may hold several time ranges, e.g. `Mon – Fri 09:00–12:00, 13:00–18:00`); v2.2.0 (2026-10-30) cross-layer search — vendor list filterable by OC location ID and vice versa; v2.3.0 (2026-10-30) QR-code dropoff flag (LMS only) on both lists, the detail page and CSV export; **v2.4.0 (2026-10-30) detail view returned to a full page with 5 tabs (Basic Info / Operations / Hours & Services / Linked Records / Change History) instead of a side drawer, per-location change history restored, and semantic versioning (MAJOR / MINOR / PATCH) adopted across all deliverables**; **v2.5.0 (2026-10-30) all editing moved into purpose-built modals** — one grouped form for store details (name/address/city/state/postal/phone), a structured 7-day opening-hours editor supporting multiple ranges per day with add/remove/copy, and a small sweeper dialog; OC opening hours are backfilled from the linked vendor location; **v2.6.0 (2026-10-30) provider data is treated as a proposal rather than a fact** — a new **Review Center** (4 tabs: Duplicates / Data Quality / Sync Changes / Enable Sign-off) replaces the single Duplicate Review page; enabled OC locations keep a signed-off baseline that provider sync cannot overwrite; enabling becomes a verification sign-off instead of a one-click switch; operator edits are final (refresh the baseline, lock the field, supersede any pending provider change, and push downstream immediately) — see Change Summary; **v2.6.1 (2026-10-30) consistency pass** — corrected one contradiction in the store-edit modal that still promised "visible after the next sync" (v2.6.0 made operator edits immediate and final), clarified on the OC list that enabling is a sign-off while disabling stays one click, reordered §8 so 8.4 follows 8.6, and re-rendered all flowcharts at 2x for inline reading; **v2.7.0 (2026-11-02) gives the decision back** — a `field_lock` can now be **released** (manually with a mandatory reason, or in one move with «Accept & unlock»), a manual edit can be **reverted field by field** from Change History, and **Enable Sign-off left the Review Center** to become the «Not live yet» queue on the OC list, because enabling is an action on one location rather than a decision about data — see Change Summary| **v3.2.1 (2026-11-04) deliverables pass** — added a plain-English feature overview for readers outside the team (shareable page and Markdown) and a deliverables list in the requirement overview; corrected the 30-second summary, which still described sweeper editing and the pre-v3 duplicate rule. No behaviour change. | **v3.2.0 (2026-11-04) source–type constraint** — a source can only produce its own type (LMS → `packagehub`, PUDO Inc → `PUDOpoint` / `Locker`, M&MP → self-operated store). Both lists cascade: choosing a Source rebuilds the Type filter from that source and auto-selects it when there is only one option, each option showing a live count. A record whose type does not belong to its source is badged in the Type column and isolatable with `Issues → Type ≠ source`. Also corrected two stale lines that still described sweeper editing and a store-detail edit modal. | **v3.1.0 (2026-11-04) issue granularity** — a flag now records `field → sub_field → issue_type → severity` plus the free-text reason, so “opening hours are wrong” becomes “SATURDAY has no range”. Seven types (missing / incorrect / inconsistent / out of range / duplicate / stale / format); only *blocking* issues stop a sign-off. |
| **v3.0.1 (2026-11-04) timestamp precision** — every timestamp in the console now carries seconds (list `Updated`, issue raised-at, audit rows) and the header shows a live clock; no behaviour change.
| **v3.0.0 (2026-11-03) model change** — the OC layer loses the ability to edit: it keeps exactly two verbs, **enable** and **flag a data issue**. `sweeper` now arrives with the provider sync and is read-only; a wrong value is flagged with a reason on the vendor record (no separate page) and fixed at the source. Duplicate checking becomes its own page and is marked **2:1** in the OC list. Downstream values are merged **per field with LMS first**. M&MP works the same data under the same rules. `field_lock`, unlock, field revert and operator edits are **removed**, not deprecated |
| Live Prototype   | <https://jama-dev520.github.io/location-prototype/> (publicly accessible, no login required)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

### 1.2 Requirement Source Traceability

| Field                     | Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Requirement Owner         | Operations / Location Curation team (OC)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Requirement Source        | Internal product request — location data management for multi-provider network                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Source Description        | OC manages location data synced daily from two providers (PUDO Inc and LMS) and needs an internal console to curate which locations are visible to downstream customers                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Original Input Format     | Text requirement + two provider API data samples (PUDO Inc DealerList JSON, LMS location list JSON)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Requirement Maturity      | L2 (semi-formed → clarified)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Clarification Rounds      | 4 rounds (key decisions resolved)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Key Clarification Records | ① Data synced from both providers once per day; ② `status` is provider-sourced business status, `is_enable` is OC's internal switch — a location is visible to downstream customers **only when status = active AND is_enable = true**; ③ Toggling enable requires a dedicated advanced permission; view/export unrestricted for now; ④ Change log must cover both provider status updates and internal enable operations; ⑤ (2026-10-29 follow-up) new location defaults `is_enable = false`; provider "removal" and "deactivation" are distinct; sync changed from full pull to incremental; `location_type` is provider-exclusive; ⑥ (2026-10-29 round 2) sweeper is OC-editable with audit logging; cross-provider address duplicates are flagged for ops confirmation and gate downstream sync via OC location ID; ⑦ (2026-10-29 round 3) duplicate detection is **fuzzy** (weighted match score ≥ 75%); OC location ID added as a list filter; ⑧ **(2026-10-30 round 4 — structural redesign)** the console splits into two menus: **Vendor Location List** (provider view — read-only except sweeper edit; every vendor location generates one OC location ID; the pickup field is renamed to **Sweeper**) and **OC Location List** (OC-owned view — populated by backfilling vendor data; the enable switch lives on the OC location; OC can edit store details such as address and opening hours; when a vendor location's status becomes `inactive`, its linked OC location is automatically disabled and blocked from downstream sync) |
| Requirement Scene Profile | D1: medium clarity · D2: 7+ feature points · D3: B-end internal admin console · D4: structure + operation flow · D5: PC Web · D6: English UI · D7: dual upstream data sources                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Method Stack Applied      | 5W2H + MoSCoW (clarification); User Story + Given-When-Then (PRD); High-fidelity prototype + walkthrough checklist (prototype)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Stakeholders              | OC operations staff, Advanced-permission operators (enable/disable), Engineering, Downstream customer-facing API consumers                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

## 2. Background & Goals

### 2.1 Why this console exists — four operational jobs

Every screen in this document exists to make one of four daily jobs take **seconds instead of a ticket**. They are stated first because the mechanics below (two layers, review center, baselines) are only justified by them.

![Full picture](https://workbuddy-space-static.codebuddy.work/image/M9nBRbD9eQJBNFivHPpu8l.png)

*The whole system on one page: providers → vendor layer → two gates → OC layer → downstream, with the decision queues on the right and the change trail underneath. The J1–J4 tags mark where each daily job happens.*

| # | The job | What "done" looks like | Where it lives |
|---|---|---|---|
| **J1** | **Find out which locations are enabled** | Ask "which stores are live right now?" and get an answer from the UI: filter to `status = active AND is_enable = true`, read the count, export exactly that set. No SQL, no engineering request, no exported-then-filtered spreadsheet | P02 · `Enable` / `Downstream` filters + CSV export |
| **J2** | **Correct who picks up (sweeper)** | Change the sweeper on a location in two clicks — including when the new sweeper belongs to the *other* provider — and have it recorded as `SWEEPER_CHANGE`. This is the only vendor-layer field OC is allowed to touch | P03 → Operations → **Edit Sweeper** (vendor layer) |
| **J3** | **Correct the address** | Fix the address, city, postal code or coordinates a customer is sent to, in one grouped form, and have it take effect immediately — because a wrong address means a customer driving to the wrong place | P03 → **Edit Store Details** (OC layer) |
| **J4** | **See what changed, before → after** | For any location, any field: who changed it, when, and old value → new value. Per location for the incident at hand, across locations when reconciling with a provider | P03 · **Change History** / P05 · **Change Log** |

> **Why these four force the architecture.** They act on *the same store* but belong to *different owners*: J2 is a fact about the provider's record, J3 is a decision OC owns, J1 is a question about the downstream catalog (which is neither — it is `status AND is_enable`), and J4 has to cover all three. That is why the console has two lists instead of one, why the enable switch sits only on the OC layer, and why the change trail is part of the workflow rather than a separate report someone remembers to pull.

Two consequences worth stating up front, because they shape most of the rules that follow:

- **J1 is the highest-frequency job and the cheapest to get wrong.** "Which stores are live?" gets asked before every campaign, every incident, and every provider review. If answering it means exporting 10k rows, the answer is always slightly out of date — so the flag, the filter, the count and the export have to agree with each other at all times.
- **J3 and J4 are a pair.** Being able to fix an address in seconds is only safe if the old value stays visible afterwards. A fast edit with no trail is how the same location gets "fixed" three times in a week.

### 2.2 Business Background

OC's location network is aggregated from two upstream providers:

- **PUDO Inc** — supplies dealer locations (PUDOpoints and Lockers) with dealer numbers, addresses, coordinates, operating hours, supported services (FX/CP/TB), and activation lifecycle dates.
- **LMS** — supplies package hub locations with location codes, cutoff times, services (returns / platform_dropoff / retail), restrictions, and holiday closures.
- **M&MP** — our own **self-operated stores** (US industry term). They are not supplied by a third party: the store maintains its own record, and it follows exactly the same rules as partner locations — same directory, same sign-off, same issue ledger. The only difference is ownership of the data: a data issue on an M&MP store is chased with the store owner internally rather than with a provider.

In v1.x the console exposed a single merged catalog where provider locations and OC-managed attributes were mixed on one row. This conflated two different concerns that OC actually needs to manage separately:

1. **The provider's view** — what each provider actually supplied (their raw location records and business status), where the only OC action is assigning who does the physical pickup (the *sweeper*).
2. **OC's own view** — the store entity OC exposes downstream, whose enable/disable switch and edited store details OC owns and controls.

The v2.0 redesign splits these into two distinct layers and two menus, so that:

- Provider data stays clean and read-only (except the sweeper assignment), and
- OC's downstream-visible catalog is a separate, OC-owned layer that OC can edit and switch on/off independently.

### 2.3 Goals

- **Business Goal**: Establish full, auditable control over the downstream-visible location catalog (visible = `status = active AND is_enable = true` on the OC layer), so that "is this store live?" has one answer that the list, the export and the change log all agree on, and 100% of visibility changes are traceable to an operator or sync event.
- **User Goal**: Operations staff can complete the four jobs in §2.1 without leaving the console: (J1) see which locations are enabled and export that exact set, (J2) set the sweeper on a provider location, (J3) correct a store address, and (J4) read the before → after of any change — each in seconds, with rich filters, not through an engineering request.
- **Product Goal**: Deliver a two-layer location management console: a **Vendor Location List** (provider data) and an **OC Location List** (OC-owned, downstream-facing), with a complete change history per layer.

### 2.4 v3.0.0 amendment — read this before the detailed specs

The console no longer edits location data. Where this document still describes an editing flow, that flow is **retired**; the replacement is the same in every case: **flag it to the provider**.

| Section | Status in v3.0.0 |
|---|---|
| §4 F10 (sweeper edit), F11 (OC store edit) | **Replaced** by F10 (sweeper synced) and F11 (data issue flag) |
| §4 F20 (per-field accept / reject of provider changes) | **Replaced** by F20 (baseline drift → re-sign or flag) |
| §4 F23 / F24 (field lock release, field revert) | **Retired** |
| §7.4 P04 Review Center (three decision tabs) | **Replaced** by two pages: **Duplicate Check** (F26) and **Data Issues** (F11/F19/F20) |
| §8.4 field tiers, `field_lock`, `sync_change_set` decisions | **Retired**; see the data mapping doc for `data_issue` and the LMS merge rule |
| §8.5 / §8.6 field policy matrix and write paths | **Superseded** by the merge rule (F25) and the flag loop |

Two invariants replace them:

1. **The console writes no location field.** The only writes are `is_enable`, `data_issue`, baseline captures and audit rows.
2. **Every field has a source.** A value shown to a customer is always attributable to one provider record, and LMS wins when both have one.

## 3. Target Users & Scenarios

### 3.1 User Roles

| Role                                                                     | Characteristics                                        | Core Needs                                                                          | Permissions                                                                                |
| ------------------------------------------------------------------------ | ------------------------------------------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Viewer** (default, all internal staff)                                 | Ops/support/BD staff who need to look up location info | Search, view details, export, review change history                                 | View both lists / view detail / export / view change log                                   |
| **Location Manager** (advanced permission, granted to a subset of staff) | Trusted operators responsible for location curation    | Set sweeper, edit OC store details, toggle `is_enable` for one or many OC locations | All Viewer permissions + sweeper edit + OC store detail edit + single/batch enable/disable |

> Permission provisioning (granting/revoking the advanced permission) is handled outside this console in v1 (e.g., by IT/admin configuration). No role management UI in this release. — *PM supplemented, confirm if acceptable*

### 3.2 Usage Scenarios

- **Scenario 1 — Set a location's sweeper**: A PUDO Inc location is physically picked up by LMS staff. The Location Manager opens the Vendor Location List, edits the Sweeper field from PUDO Inc to LMS, and the change is logged (SWEEPER_CHANGE). The field "Pickup" is now uniformly named **Sweeper** across the console.
- **Scenario 2 — Backfill & activate an OC location**: The nightly sync creates a new vendor location. Because it is not a suspected duplicate, the system auto-generates a corresponding OC location (backfilling name/address/hours from the vendor record) with `is_enable = false`. A Location Manager reviews it in the OC Location List and enables it, making it visible downstream.
- **Scenario 3 — Edit a store detail**: Ops notices an OC location's opening hours are wrong. The Location Manager edits the hours on the OC Location List; the change is stored as an OC-level override and logged (STORE_DETAIL_EDIT), without touching the underlying vendor record.
- **Scenario 4 — Vendor goes inactive → OC auto-disable**: The nightly sync flips a vendor location's status to `inactive`. The linked OC location is automatically disabled and blocked from downstream sync. If a Manager tries to force-enable it, the system raises an error ("vendor location is inactive"). When the provider later reactivates, the OC location can be enabled again.
- **Scenario 5 — Suspected duplicate does not generate an OC location**: The sync flags a new LMS location whose address fuzzy-matches an existing PUDO Inc location. Because it is a suspected duplicate, the system does **not** generate an OC location ID yet — it enters the Duplicate Review queue. Ops confirms "same store" (both link to one OC location) or "not same" (each gets its own OC location).


- **Scenario 6 — Batch suppression for a region**: OC decides to hide all Locker-type OC locations in Brooklyn. Staff filters the OC Location List, selects all results, clicks "Batch Disable", reviews the confirmation dialog, and confirms — each OC location gets an audit entry.
- **Scenario 7 — Daily reconciliation**: After the nightly sync, staff reviews the sync status page and the global change log to see which vendor locations changed and how OC locations were auto-updated (including auto-disable from vendor deactivation).


## 4. Feature Requirements List

> **Priority definitions**: P0 = must-have (MVP) / P1 = important / P2 = deferrable

| ID | Module | Feature | Description | Priority | Depends On | Page | API Ownership | Notes |
|----|--------|---------|-------------|----------|------------|------|---------------|-------|
| F01 | Vendor Location List | Paginated vendor location list | Display raw provider locations with columns: Location ID, Name, Address, Provider, Type, Sweeper, Status, Created Time, Updated Time (+ City/State, row actions) | P0 | — | P01 | New: `GET /api/vendor-locations` | Server-side pagination |
| F02 | Vendor Location List | Filter & search | Filter by Location ID, Name, Address keyword, Provider, Sweeper, Status, Type, City, State, Created/Updated time range (second-level precision) | P0 | F01 | P01 | New: `GET /api/vendor-locations` (query params) | Filters combine with AND logic |
| F03 | OC Location List | Paginated OC location list | Display OC-owned locations with columns: OC Location ID, Name, Address, Provider, Type, Sweeper, Status, Enable, Created/Updated Time, Linked Vendor Location(s) | P0 | — | P02 | New: `GET /api/oc-locations` | Server-side pagination |
| F04 | OC Location List | Filter & search | Filter by OC Location ID, Name, Address keyword, Provider, Sweeper, Status, Enable, Type, City, State, time range | P0 | F03 | P02 | New: `GET /api/oc-locations` (query params) | Filters combine with AND logic |
| F05 | Export | Export current view | Export filtered results (either list) to CSV, columns identical to the list; export follows active filters | P0 | F02/F04 | P01/P02 | New: `GET /api/vendor-locations/export` + `GET /api/oc-locations/export` | Max 10,000 rows per export |
| F06 | Detail | Location detail view | Full profile as a full-width page with five tabs: **Basic Info / Operations / Hours & Services / Linked Records / Change History**. Vendor detail is read-only except sweeper; OC detail additionally shows editable store details, the linked vendor location(s) and the downstream-visibility block | P0 | — | P03 | New: `GET /api/vendor-locations/{id}` + `GET /api/oc-locations/{id}` | Provider-sourced fields read-only |
| F07 | Change History | Per-location change log | Timeline of every change for one location (vendor or OC): operator, time, type, field, old → new; covers provider sync changes, sweeper changes, OC store-detail edits, enable operations, OC link operations | P0 | F06 | P03 | New: `GET /api/locations/{id}/changes` | Chronological, latest first |
| F08 | Enable Control | Single enable/disable toggle | Location Manager toggles `is_enable` on/off on an **OC location**; requires advanced permission; writes audit entry; blocked (with error) when the linked vendor location is `inactive`. Switching **on** runs the sign-off in F21; switching **off** stays one click | P0 | F03 | P02/P03 | New: `PATCH /api/oc-locations/{id}/enable` | Viewers see the control disabled/hidden |
| F09 | Enable Control | Batch enable/disable (two modes) | **Mode A — Inline selection**: select OC rows → "Batch Enable/Disable" → confirmation → execute. **Mode B — Excel bulk upload**: download template (`ocLocationID`, `is_enable`), upload, validate, preview, confirm; each changed OC location gets its own audit entry | P0 | F04/F08 | P02 | New: `POST /api/oc-locations/batch-enable` + `POST /api/oc-locations/batch-enable/upload` + `GET /api/oc-locations/batch-enable/template` | Mode A max 200/batch; Mode B max 5,000 rows, ≤ 5MB; advanced permission required |
| F10 | Sweeper (synced) | Sweeper arrives with the provider sync | The **Sweeper** field is supplied by the provider and is **read-only** on both layers. A wrong or missing sweeper is reported as a data issue on the vendor record (`DATA_ISSUE_RAISED`) and fixed by the provider; the console never reassigns it | P0 | F01/F11 | P01/P03 | Provider payload field; no write endpoint | Replaces the v2.x "edit sweeper" exception — OC owns no vendor field any more |
| F11 | Data Issue Flag | Report a data problem instead of editing it | A Location Manager flags a source record at **four levels of granularity**: `field` (which column) → `sub_field` (which address line, which weekday, which coordinate) → `issue_type` (**MISSING / INCORRECT / INCONSISTENT / OUT_OF_RANGE / DUPLICATE / STALE / FORMAT**) → `severity` (**blocking** = cannot be signed off, **warning** = live but wrong). A free-text reason is mandatory. Stored on the source record (`data_issue`, open → reported → fixed). **No value is ever written by the console** | P0 | F01 | P01/P03/P04 | New: `POST /api/vendor-locations/{id}/issues`, `PATCH /api/issues/{id}` | The single mechanism that replaces every former edit path (address, hours, name, sweeper). Sub-field is what makes the flag actionable: “hours are wrong” cannot be fixed, “SATURDAY has no range” can |
| F12 | OC Generation | Auto-generate OC location from vendor | On sync, a new vendor location that is **not** a suspected duplicate auto-generates one OC location (backfilling name/address/hours from the vendor record) with `is_enable = false` | P0 | F01 | — | System (sync job) | One vendor location → one OC location ID (principle 1:1); cross-provider duplicate → no OC ID yet |
| F13 | Status Propagation | Vendor inactive → OC auto-disable | When a vendor location's status becomes `inactive`, its linked OC location is auto-disabled and blocked from downstream sync; force-enabling raises an error until the vendor is active again | P0 | F12 | P02/P03 | System (sync job) | Enforced on sync and on the enable endpoint |
| F14 | Duplicate Detection | Cross-provider address duplicate review (fuzzy) | Auto-detect cross-provider vendor locations whose addresses fuzzy-match (weighted match score ≥ 75%); flagged pairs do **not** generate OC locations until resolved; ops confirms "same store" (link to one OC location) or "not same" (each gets its own OC location) | P0 | F01 | P04 | New: `GET /api/duplicates` + `POST /api/duplicates/{id}/resolve` | OC location assignment gates downstream sync |
| F15 | Change History | Global change log | Cross-location searchable log with filters (time range, operator, provider, operation type); supplements per-location view | P1 | F07 | P05 | New: `GET /api/changes` | Ops reconciliation entry point |
| F16 | Data Sync | Daily sync status | Show last sync run per provider (time, result, records pulled/created/updated/deactivated/removed) and failure alerts | P1 | F01 | P06 | New: `GET /api/sync/status` | Sync daily 02:00 PUDO / 02:30 LMS; incremental pull |
| F17 | Overview | Dashboard | KPI cards: total vendor locations, total OC locations, visible OC locations, disabled OC locations, provider-deactivated count | P2 | F01/F03 | P07 | New: `GET /api/stats/overview` | Nice-to-have for v1.1 |
| F18 | QR Dropoff | QR-code dropoff capability flag | Display a **QR Dropoff** marker and field on both lists (and in the detail profile): LMS stores that accept a package drop-off where the customer presents a **QR code instead of a printed label** are flagged; PUDO records show N/A. The OC layer backfills the flag from its linked vendor location. Filterable in both lists and included in CSV export | P0 | F01/F03 | P01/P02/P03 | Sync-sourced field `qr_dropoff` on `vendor_location`; backfilled to `oc_location` | Read-only (provider-sourced); LMS only |
| F19 | Intake Validation | Data-quality gate before an OC location is created | Every synced vendor location runs an intake check (required address fields, postal ↔ city ↔ state consistency, coordinate plausibility, opening-hours structure). `fail` → **no OC location is generated**, the record enters the **Data Quality** queue; `warn` → an OC location is created but **cannot be enabled** until the issue is cleared | P0 | F12 | P04 (Data Quality tab) | New: `GET /api/review/data-quality` + `POST /api/review/data-quality/{id}/override` | «Not generated» now has two reasons: suspected duplicate **or** failed intake |
| F20 | Baseline Drift | Provider changes after sign-off | Values always follow the provider (data belongs to the provider), so a change to a live location takes effect downstream. The console compares current merged values with the signed-off baseline and surfaces **drift** (field, old → new). The operator either **re-signs** (baseline moves to the new values, `RE_SIGNED_OFF`) or **flags** the change as an issue | P0 | F03/F08 | P02/P03/P04 | New: `GET /api/oc-locations/{id}/drift` + `POST /api/oc-locations/{id}/resign` | Replaces per-field accept/reject: there is nothing to accept into, the provider value already applies |
| F21 | Enable Sign-off | Enabling becomes a verification step, not a toggle | Turning `is_enable` from false → true opens a sign-off dialog showing the customer-facing data (address, coordinates, hours, phone), its source (provider + vendor location ID + last sync), and 5 pass/fail checks. Confirm stays disabled until the operator ticks «I verified address, coordinates and opening hours». On confirm: `is_enable = true`, **baseline captured**, `ENABLE_SIGNED_OFF` logged. Disabling needs no sign-off (going offline is always safe) | P0 | F08 | P02/P03 | Extends `PATCH /api/oc-locations/{id}/enable` | Blocked when: vendor inactive · N changes pending review · intake overridden · hours missing. Batch enable runs the same checks and reports skipped rows |
| F22 | Sync Breaker & Rollback | Stop a bad provider run from spreading | A run is downgraded to dry-run (vendor layer written, nothing propagated) when changed records > 30%, address-type changes > 20, or batch `inactive` > 5. A global **SAFE MODE** switch freezes all propagation. Every run stores a before-image of the OC rows it would touch (7 days), so a run can be rolled back as a batch (`SYNC_ROLLBACK`) and a single field can be reverted to its previous version | P1 | F20 | P04/P06 | New: `POST /api/sync/runs/{id}/rollback` + `PATCH /api/sync/safe-mode` | Rollback is the fastest remedy after an upstream mistake — target MTTR one operation |
| F23 | *(retired in v3.0.0)* | Field lock release | Removed with the editing model: with no manual edits there is nothing to lock or unlock. `field_lock` and `FIELD_UNLOCKED` are dropped | — | — | — | — | Kept as a row so readers of older material can see it is gone on purpose |
| F24 | *(retired in v3.0.0)* | Field revert | Removed: reverting a value is an edit, which the OC layer no longer performs. A wrong provider value is flagged, not reverted | — | — | — | — | Same rationale as F23 |
| F25 | Field Merge | Source-priority downstream merge | When one OC location is fed by several source records, each **field** is taken independently: **M&MP (our own store) wins first, then LMS, then PUDO Inc fills the gap**. Every field shows its source in the detail page and the list carries a Source column | P0 | F03/F14 | P02/P03 | Computed view; new: `GET /api/oc-locations/{id}/resolved` | Field-level, not record-level: an empty field from the winning source never blanks the merged value |
| F26 | Duplicate Check | Standalone duplicate page | Duplicate pairs get their own page with fuzzy score, per-field diff and both OC links; the decision is **same store (2:1)** — both vendor records feed one OC location, which is marked `2:1` — or **not the same (1:1)**, which gives the second record its own OC location. A pending pair blocks enablement | P0 | F14 | Duplicate Check | Existing `GET /api/duplicates`, `POST /api/duplicates/{id}/resolve` | Split out of the Review Center so that a query page is not mixed with decision queues |

## 5. User Flows

### 5.1 Vendor Sync & OC Generation Path

![Vendor Sync & OC Generation Path](https://workbuddy-space-static.codebuddy.work/image/tQa1reB86SbeW5E7dHCEvt.png)

> **Business decisions (confirmed 2026-10-30)**:
> - A newly synced vendor location that is not a suspected duplicate auto-generates **one** OC location (backfilled from the vendor record) with `is_enable = false` — it is NOT visible downstream until a Location Manager enables it.
> - A suspected duplicate (fuzzy match ≥ 75% across providers) does **not** generate an OC location — it enters the Duplicate Review queue.
> - Provider "removal" and "deactivation" are distinct semantics (see §8.2).

### 5.2 Status Propagation — Vendor Inactive → OC Auto-Disable

![Status Propagation Path](https://workbuddy-space-static.codebuddy.work/image/F1Z3SApwjlM6stfEn0tCnp.png)

### 5.3 Operator Batch Disable (OC Layer)

![Operator Batch Disable](https://workbuddy-space-static.codebuddy.work/image/bRWhlIpawHygvuqvunNgMg.png)

### 5.4 Sweeper Edit Path

![Sweeper Edit Path](https://workbuddy-space-static.codebuddy.work/image/jKmlOtst4dF3BjBCWQ3NBm.png)

### 5.5 Duplicate Review Path

![Duplicate Review Path](https://workbuddy-space-static.codebuddy.work/image/THq85hNgrIbu1LIbM5Pwrz.png)

### 5.6 Exception Path

![Exception Path](https://workbuddy-space-static.codebuddy.work/image/NkJGL93Qos7exTRSPNYWNK.png)

### 5.7 Baseline drift after sign-off (v3.0.0)

Provider values always apply downstream — the data belongs to the provider, so the console does not freeze it. What the console keeps is the **signed-off baseline**, and it compares the two: any field that moved since sign-off shows up as drift on the OC row, in the detail page and in Data Issues.

The operator has two moves, neither of which edits a value: **re-sign** (accept the new values as the new baseline, `RE_SIGNED_OFF`) or **flag** (the values are wrong → `data_issue`, the provider fixes the source). Replaces the v2.x accept/reject-per-field flow (flow7), which is retired.

### 5.8 Enable sign-off — unchanged in shape (v2.6.0)

![Enable Sign-off](https://workbuddy-space-static.codebuddy.work/image/UggYn65vVXnsaDXfZaUvPM.png)

> Four blockers plus an acknowledgement, and a baseline is captured on confirm. In v3.0.0 the dialog additionally shows **where each value comes from** (LMS or PUDO), and the blocker list includes open data issues.

### 5.9 Data issue loop — flag → chase → fix → next sync (v3.0.0)

```
value is wrong  →  flag it (field + reason)  →  status OPEN
                →  chased with the provider →  status REPORTED
                →  provider corrects the source
                →  next sync brings the new value →  status FIXED
```

> The console never writes the value. This loop replaces the operator-edit flow (flow9), field locks (flow10) and field revert (flow11) — all three existed only to make manual edits safe, and manual edits are gone.

### 5.10 Duplicate decision (F26)

Pairs are decided on their own page: **same store → 2:1** (both vendor records feed one OC location, merged per field with LMS first) or **not the same → 1:1**. A pending pair blocks enablement, and a 2:1 OC location is marked as such in the list.

## 6. Page Map

| Page ID | Page Name | Module | Core Functions | Entry | Exit | Data Passed | Feature IDs | Priority |
|---------|-----------|--------|----------------|-------|------|-------------|-------------|----------|
| P01 | Vendor Location List | Vendor Location List | Filter, list, export, sweeper edit | Sidebar "Vendor Location List" | P03 | Vendor Location ID (to detail) | F01/F02/F05/F10 | P0 |
| P02 | OC Location List | OC Location List | Filter, list, export, single/batch enable, store detail edit, **Sync column** (`⚠ N held`) | Sidebar "OC Location List" (default landing) | P03 / P04 | OC Location ID (to detail) | F03/F04/F05/F08/F09/F11/F20 | P0 |
| P03 | Location Detail | Detail (vendor / OC) | Full profile tabs (Basic Info / Operations / Hours & Services / Linked Records / Change History), per-location change history, contextual actions (sweeper edit for vendor; enable + store edit for OC), **pending-change banner + downstream push status** | P01/P02 row click | P01, P02, P05 | Location ID | F06/F07/F08/F10/F11/F20/F21 | P0 |
| P04 | **Duplicate Check** | Query + decide | One question only: are these two vendor records the same store? Fuzzy score, per-field diff, both OC links, `Same store (2:1)` / `Not same (1:1 each)`; filters by provider, status, minimum score | Sidebar "Duplicate Check" | P02/P03 | Pair ID | F26 | P0 |
| P05 | **Data Issues** | One anomaly ledger | Three sections — provider data issues (open → reported → fixed), drift after sign-off (re-sign or flag), failed intake (chase only). No decision queues | Sidebar "Data Issues" | P01/P02/P03 | Issue / OC ID | F11/F19/F20 | P0 |
| P05 | Change Log (global) | Change History | Cross-location change log with filters | Sidebar "Change Log" | P03 | Optional location ID prefill | F15 | P1 |
| P06 | Sync Status | Data Sync | Last sync run per provider, results, failure alerts | Sidebar "Sync Status" | P05 | — | F16 | P1 |
| P07 | Overview Dashboard | Overview | KPI cards and trends | Sidebar "Overview" | P01 | — | F17 | P2 |

> **Prototype scope:** P01, P02, P03, P04, P05 and P06 are built in the clickable prototype. P07 (Overview Dashboard) is specified but **not built** — it is P2 and out of scope for this round.
> **DOM ids are historical:** the prototype implements these pages as `page-p01`, `page-p02`, `page-p03`, `page-p04`, `page-p06` and `page-pd`. Those ids predate this page numbering and do not map to the P-numbers above — navigate by menu label, not by DOM id.

## 7. Interaction Specifications

### 7.1 P01 — Vendor Location List

- **Entry**: Sidebar "Vendor Location List".
- **Elements**: Filter panel (collapsible), data table (columns: checkbox, Location ID, Name, Address, City/State, Provider, Type, Sweeper, **QR Dropoff**, Status, Created Time, Updated Time, Actions), pagination bar (20/page, options 20/50/100), "Export CSV" button, selected-count indicator.
- **Filter fields**: Location ID, **OC Location ID (reverse lookup)**, Name, Address keyword, City, State/Province, Provider, Sweeper, Status, Type, **QR Dropoff (Supported / Not supported)**, Created from/to, Updated from/to. All AND-combined.
- **Cross-layer search**: Filtering by **OC Location ID** on the vendor list resolves through `vendor_oc_mapping` and returns the vendor location(s) feeding that OC location.
- **QR Dropoff marker**: LMS stores that accept QR-code drop-off (customer presents a QR code instead of a printed label) show a teal **◉ QR dropoff** badge in the QR Dropoff column; stores without the capability show `—`; PUDO records are not applicable and show `—`. The field is provider-sourced and read-only.
- **Sweeper semantics**: The field formerly named "Pickup" is now uniformly named **Sweeper**. It identifies who performs the physical pickup for a location. Since v3.0.0 it arrives with the sync like every other field — **read-only here**; a wrong or missing sweeper is flagged (F11) and the provider corrects the source.
- **Source → Type is a constraint, not a preference (v3.2.0)**: a source can only ever produce its own types — `LMS → packagehub`, `PUDO Inc → PUDOpoint | Locker`, `M&MP → M&MP store`. Choosing a **Source** rebuilds the **Type** dropdown from that map (with a per-type count) and auto-selects the only option when there is one, so an impossible combination cannot be expressed in the UI. Clearing the source clears the type. A record whose `type` does not belong to its `source_provider` is marked **`type ≠ source`** in the Type column and can be isolated with `Issues → Type ≠ source` — it is a data-quality finding to flag, not something the console rewrites.
- **Timestamp precision**: All `Created Time` / `Updated Time` values are displayed, filtered, and exported with **second-level precision** (`YYYY-MM-DD HH:MM:SS`).
- **Interactions**:
  - Filter apply on "Apply"; "Reset" clears all; active filter chips shown above table.
  - Click Location ID or Name → navigate to P03 (vendor detail).
  - Sweeper edit dialog (Manager only) → confirm → executes F10.
  - Export downloads `vendor_locations_YYYYMMDD_HHmmss.csv`.
- **States**: Default / loading / empty / error.
- **Data volume**: hundreds to low thousands; server-side pagination + filtering.
- **Validation rules**: Location ID alphanumeric, ≤ 32 chars; time range end ≥ start, max span 1 year.

### 7.2 P02 — OC Location List

- **Entry**: Sidebar "OC Location List"; default landing page after login.
- **Elements**: Filter panel (collapsible), data table (columns: checkbox, OC Location ID, Name, Address, City/State, Provider, Type, Sweeper, **QR Dropoff**, Status, **Sync**, Enable, Linked Vendor Location(s), Created Time, Updated Time, Actions), pagination bar, "Export CSV", "Batch Enable" / "Batch Disable", "Batch Upload (Excel)", selected-count indicator.
- **Sync column (v2.6.0)**: shows `⚠ N held` when provider changes on this location are waiting for a decision, `—` otherwise. Clicking it opens P04 · Sync Changes. It is a **status indicator**, not an error — the location stays visible with its signed-off values.
- **Filter fields**: OC Location ID, **Vendor Location ID (forward lookup)**, Name, Address keyword, City, State/Province, Provider, Sweeper, Status, Enable, **Downstream (Not live yet / Live)** (v2.7.0), Type, **QR Dropoff (Supported / Not supported)**, **Sync state (changes held / none)** (v2.6.0), Created from/to, Updated from/to. All AND-combined.
- **Sign-off queue (v2.7.0)**: the toolbar carries a one-click preset **⏳ Not live yet (N)** that applies `Downstream = Not live yet`. This *is* the onboarding queue — it replaced the fourth Review Center tab, because enabling is an action on one location, not a decision about a queue of data. The menu item shows the same count. Opening a row's switch starts the sign-off dialog (F21).
- **Cross-layer search**: Filtering by **Vendor Location ID** on the OC list returns the OC location(s) fed by that vendor location (supports the 1:N mapping after a duplicate pair is confirmed as the same store).
- **QR Dropoff marker**: Backfilled from the linked vendor location — the OC location shows a teal **◉ QR dropoff** badge when **any** linked vendor location supports QR-code drop-off. Read-only on this layer.
- **Enable control (sign-off, not a switch)**: The `is_enable` switch lives **only** on the OC layer. **Disabling** takes effect immediately (going offline is always safe). **Enabling** opens the sign-off dialog (F21) — it does not flip directly. Toggling is also gated by the linked vendor location's status — if the vendor location is `inactive`, the dialog opens with the confirm button in a **Blocked** state and the reason shown, so a click never silently does nothing.
- **No store detail editing (v3.0.0)**: the OC layer exposes no edit affordance for name, address, city, state, postal code, phone or opening hours. A wrong value is reported with `Flag a data issue` on the source vendor record (F11); the fix arrives with the next provider sync.
- **Source → Type is a constraint (v3.2.0)**: the Type filter is derived from the selected Source exactly as on P01 (`LMS → packagehub`, `PUDO Inc → PUDOpoint | Locker`, `M&MP → M&MP store`), with live counts; on this list the type is the merged value of the linked source records, so a 2:1 location reports the type of the source that wins the merge. The toolbar preset **Not live yet** clears Source and Type before applying, so the number on screen always matches the menu badge.
- **Two actions only (v3.0.0)**: `enable` (sign-off, F21) and `flag an issue` (F11). Every other control on this list is read-only. Flags are recorded against the vendor record and surface here as `N data` / `N drift` / `2:1` badges.
- **Interactions**:
  - Filter apply/reset same as P01.
  - Click OC Location ID or Name → navigate to P03 (OC detail).
  - Row toggle (switch) → enabling opens the sign-off dialog (F21); disabling applies immediately. Blocked when the vendor location is inactive or a check fails.
  - Checkbox selection → batch action bar; "Batch Disable/Enable" → confirmation modal (count + sample IDs + **rows that will be skipped and why** + warning) → executes F09 Mode A. Batch enable runs the same sign-off checks per row and captures the baseline for every row it enables.
  - "Batch Upload (Excel)" → upload dialog (download template, drag/drop, preview, invalid-row report) → executes F09 Mode B.
  - `Flag a data issue` on a row → executes F11. There is no edit modal (v3.0.0).
  - Export downloads `oc_locations_YYYYMMDD_HHmmss.csv`.
- **States**: Default / loading / empty / error.
- **Validation rules**: Batch upload file `.xlsx`/`.csv`, ≤ 5MB, ≤ 5,000 rows; header exactly `ocLocationID,is_enable`; `is_enable` values `T`/`F`.

### 7.3 P03 — Location Detail

- **Entry**: P01/P02 row click / Actions "View".
- **Vendor detail** (from P01): header (name, Location ID, provider badge, status pill), tabs **Basic Info** / **Hours & Services** / **Change History**; all fields read-only except Sweeper (Manager only).
- **OC detail** (from P02): header (name, OC Location ID, visibility pill, mapping badge 1:1 / 2:1), action buttons (**Enable/Disable**, **Flag a data issue** — no Edit), five tabs **Basic Info** / **Operations** / **Hours & Services** / **Issues** / **Change History**. Every field shows its **source** (LMS / PUDO) and the page explains how the record is assembled from its vendor records.
- **Change History**: reverse-chronological timeline; each entry: time, operator, operation type badge, field, old → new, remark. **Revert (v2.7.0)**: the newest manual edit of each field carries **↩ Revert this field**; older edits of the same field do not, because reverting one would silently overwrite whatever happened afterwards. Reverting restores the logged previous value, writes `FIELD_REVERTED`, refreshes the baseline, keeps the field locked, and pushes downstream when the location is visible.
- **Pending-change banner (v2.6.0)**: when provider changes are held for this location, the top of the page shows «⚠ N provider change(s) held for review — this location keeps its signed-off value (baseline <timestamp>)» with a link into P04 · Sync Changes. The banner is informational; the customer keeps seeing the baseline value.
- **Downstream block (OC only)**: `Enable (is_enable)` · `Visible downstream` (active AND enabled) · `Linked vendors` · `Mapping` (1:1 / 2:1) · `Signed-off baseline` state; when drift exists the block offers **Re-sign with current values** (F20).
- **States**: Loading / loaded / not-found.
- **Validation rules**: None (read-only except contextual actions); a blocked store-detail save (clearing a required address field, or clearing every opening-hours range while enabled) is rejected with an inline explanation and leaves the record untouched.

### 7.4 P04 — Review Center (v2.6.0, three tabs since v2.7.0)

> **One entry point, three decision types.** They share the list component, the field-diff view, batch actions and audit writing — but each has its own state machine, permission and blocking scope. Merging them into a single queue would blur «what am I approving?».
>
> **What left the queue in v2.7.0:** Enable Sign-off. The other three tabs decide *about data* (a pair, a record, a field) and the decision changes what the system believes. Sign-off decides *about a moment* — it is the action of making one location visible, and it belongs next to that location. It now lives on P02 as the **⏳ Not live yet** queue, with the same dialog, the same checks and the same baseline capture. Nothing was weakened; one entry point was removed.

| Tab | Subject of the decision | Blocks | Who handles it | Wrong decision costs |
|---|---|---|---|---|
| **T1 Duplicates** | a **pair** of vendor locations → one OC location (2:1) or two (1:1) | OC location creation | Store operations | Two stores merged into one → customer sent to the wrong address |
| **T2 Data Quality** | a **whole record** that failed intake → fix / ask provider / override | Enablement (no OC record exists yet) | Operations + provider liaison | A poor-data store goes live |
| **T3 Sync Changes** | a **single field** on a live location → accept / reject & lock / accept & unlock | The OC field write | Technical operations (needs the diff) | Correct data already live gets overwritten |

- **Entry**: Sidebar "Review Center". The sidebar badge counts **decisions waiting** (T1 + T2 + T3) and nothing else — the number of locations waiting to go live is a separate count on the OC Location List menu item, so the two never get confused again.
- **T1 Duplicates**: unchanged from v2.0 — pending table (Pair, Location A, Location B, Match score, Address diff, Detected, Status, Action), colour-coded score, per-field diff, «Same store» / «Not same», writes `OC_LINK_CONFIRMED` / `OC_LINK_REJECTED`. Until resolved, no OC location exists.
- **T2 Data Quality**: list of vendor locations that failed intake, each with the failed checks (cross marks), detected time and status. Actions: «Request provider fix» (stays blocked) and «Override & create OC location» (creates the record with `is_enable = false` — it still cannot go live without sign-off). Writes `INTAKE_OVERRIDDEN`.
- **T3 Sync Changes**: grouped by OC location. Header shows name, OC location ID, «N fields held», source vendor location. A run summary line shows run id, provider, breaker state, pulled / created / held / auto-applied. Each field shows `oldValue` struck through → `newValue` in amber, the reason it was held, a **`locked` badge** when the field is in `field_lock`, and **Accept** / **Reject & lock** — plus **Accept & unlock** on locked fields (F23). Per-location «Accept all» / «Reject all & lock». Accept writes the value, refreshes the baseline and logs `SYNC_CHANGE_ACCEPTED`; accept & unlock also removes the lock and logs `FIELD_UNLOCKED`; reject keeps the baseline value, adds the field to `field_lock` and logs `SYNC_CHANGE_REJECTED`. A separate collapsed block lists fields auto-applied in the run (low-risk), for transparency.
- **Where sign-off went (v2.7.0)**: P02 · «⏳ Not live yet (N)» → each row opens the same sign-off dialog as the row switch (F21). Blocker rules are unchanged: vendor inactive · N changes pending review · intake overridden · hours missing. Rows that are blocked open the dialog with the confirm button unavailable and the reason listed, so a click never silently does nothing.
- **States**: pending / empty (per tab). Leaving an item unresolved is always safe — the system defaults to *not generating*, *not enabling*, or *keeping the old value*.

### 7.5 P05 — Change Log (Global) (P1)

- **Entry**: Sidebar "Change Log".
- **Elements**: Filters (time range, operator, provider, operation type, location ID), log table, pagination.
- **Interactions**: Click location → P03; filter apply/reset.

### 7.6 P06 — Sync Status (P1)

- **Entry**: Sidebar "Sync Status".
- **Elements**: One card per provider (PUDO Inc, LMS): last sync time, status, records pulled, created/updated/deactivated counts, error message; "Sync history" table (last 30 runs).
- **Interactions**: Refresh; failed run expands to show error detail.

### 7.7 P07 — Overview Dashboard (P2)

- **Entry**: Sidebar "Overview".
- **Elements**: KPI cards (Total Vendor Locations, Total OC Locations, Visible OC Locations, Disabled OC Locations, Provider-Deactivated), provider split, 30-day enable/disable trend.

## 8. Data Requirements

### 8.1 Core Data Objects

| Object | Key Fields | Source | Notes |
|--------|-----------|--------|-------|
| **Vendor Location** (provider layer) | vendor_location_id (internal), source_provider (PUDO/LMS), provider_ref (dealerNo or loc_id), store_code, name, location_type (PUDOpoint/Locker/packagehub), sweeper (PUDO/LMS — who performs the physical pickup), address_line1/2, city, state, postal_code, country, latitude, longitude, phone, timezone, status (active/inactive/removed), services[], restrictions[], **qr_dropoff (bool, LMS only)**, hours_of_operation{}, cutoff_time, holiday_closures[], created_at, updated_at, activated_at, deactivated_at, last_synced_at | PUDO Inc + LMS daily sync | Provider-sourced; read-only except `sweeper`; `location_type` provider-exclusive; `sweeper` defaults to `source_provider` and is OC-editable |
| **OC Location** (OC layer) | oc_location_id (OC-owned, gates downstream sync), name, address_line1/2, city, state, postal_code, country, latitude, longitude, phone, opening_hours{}, store details (editable), **qr_dropoff (bool, backfilled)**, is_enable (bool), status (derived), **intake_status (pass/warn/fail/overridden)**, **baseline_at (timestamp of the last human-verified version)**, **propagated_at (timestamp of the last downstream push)**, created_at, updated_at, linked_vendor_location_ids[] | Backfilled from Vendor Location; OC-editable store details | The downstream-visible entity; enable switch lives here; store details backfilled from vendor then OC-overridable; 1:N mapping to vendor locations (principle 1:1) |
| **OC Field Baseline** (v2.6.0) | baseline_id, oc_location_id, field_name, baseline_value, version_no (keep last 5), captured_at, captured_by, capture_reason (SIGNOFF / ACCEPTED_CHANGE / OPERATOR_EDIT) | This system | The **reference for good data** — «what a human verified», not «what the last sync wrote». Provider changes are compared against this |
| **Sync Change Set** (v2.6.0) | change_id, run_id, oc_location_id, vendor_location_id, field_name, old_value, new_value, risk_tier (auto/review/block), decision (pending/accepted/rejected/superseded), reason, decided_by, decided_at | Sync job | One row per changed field per run. `pending` = held, nothing written to OC. Superseded when an operator edits the same field |
| **Field Lock** (v2.6.0) | lock_id, oc_location_id, field_name, locked_at, locked_by, reason (OPERATOR_EDIT / REJECTED_CHANGE) | This system | Once locked, a provider change to that field **must** go through review even if the field's tier is `auto` |
| **Intake Issue** (v2.6.0) | issue_id, vendor_location_id, check_code, severity (warn/fail), detected_at, status (open/overridden/resolved) | Intake check on sync | `fail` blocks OC location creation; `warn` allows creation but blocks enablement |
| **Vendor ↔ OC Mapping** | mapping_id, vendor_location_id, oc_location_id, created_at | This system | Links vendor location(s) to the OC location; principle 1:1, but a confirmed duplicate pair links 2 vendor locations → 1 OC location |
| **Change Log Entry** | log_id, location_id, location_layer (VENDOR/OC), occurred_at, operator, operation_type (ENABLE / DISABLE / BATCH_ENABLE / BATCH_DISABLE / ENABLE_SIGNED_OFF / SWEEPER_CHANGE / STORE_DETAIL_EDIT / DOWNSTREAM_PROPAGATED / OC_LINK_CONFIRMED / OC_LINK_REJECTED / OC_GENERATED / INTAKE_OVERRIDDEN / PROVIDER_STATUS_CHANGE / PROVIDER_FIELD_UPDATE / PROVIDER_CREATED / PROVIDER_REMOVED / SYNC_CHANGE_ACCEPTED / SYNC_CHANGE_REJECTED / SYNC_ROLLBACK / SYNC_QUARANTINED), field_name, old_value, new_value, source (MANUAL/SYNC), remark | This system | Append-only, immutable |
| **Duplicate Pair** | pair_id, vendor_location_a, vendor_location_b, matched_on, match_score, diff, status (pending/confirmed/rejected), detected_at, resolved_by, resolved_at | This system | Cross-provider (LMS ↔ PUDO Inc) only |
| **Sync Run** | run_id, provider, started_at, finished_at, result, records_pulled, created_count, updated_count, deactivated_count, error_message | This system | One record per provider per day |
| **User** | user_id, name, email, has_enable_permission (bool) | Corporate SSO | No self-service role management in v1 |


### 8.2 Data Rules

- **Two-layer model (core, v2.0)**: The console has two distinct layers — **Vendor Location** (raw provider data) and **OC Location** (OC-owned, downstream-facing). Provider data is read-only (except sweeper); OC controls visibility and store details on its own layer.
- **Visibility rule (core)**: An OC location is synced downstream **iff `status = active` AND `is_enable = true`**. The `status` here is derived from the linked vendor location(s). When the vendor location is `inactive`, the OC location is auto-disabled and cannot be force-enabled.
- **Field ownership**:
  - **Vendor layer**: `status` and all business attributes owned by providers (read-only, refreshed by daily sync); `sweeper` owned by OC (writable via F10).
  - **OC layer**: `is_enable` owned by OC (writable via F08/F09); store details (address, opening hours, etc.) owned by OC (writable via F11); `oc_location_id` owned by system.
- **Sweeper (renamed from Pickup)**: The field formerly named "Pickup"/`pickup_provider` is now uniformly **Sweeper** (`sweeper`). It identifies who performs the physical pickup. Defaults to `source_provider`, may differ (cross-provider sweep). Writable on the vendor layer via F10; every change writes `SWEEPER_CHANGE`.
- **OC generation (v2.0)**: On sync, each new vendor location that is **not** a suspected duplicate auto-generates one OC location (backfilled from the vendor record) with `is_enable = false`, logged as `OC_GENERATED`. A suspected duplicate (fuzzy match ≥ 75% across providers) does **not** generate an OC location — it enters the Duplicate Review queue.
- **Mapping (1:N)**: Principle is 1 vendor location → 1 OC location (1:1). A confirmed cross-provider duplicate links 2 vendor locations → 1 OC location (2:1). A rejected duplicate assigns independent OC locations (1:1 each).
- **Status propagation (v2.0)**: When a vendor location's status becomes `inactive`, its linked OC location is auto-disabled and blocked from downstream sync. Force-enabling raises an error ("vendor location is inactive"). When the provider reactivates the vendor location (status → `active`), the OC location can be enabled again.
- **Removal vs deactivation**: Provider "removal" (location disappears from provider's list) = permanent delisting, `status = removed`, logged `PROVIDER_REMOVED`. Provider "deactivation" (`status = inactive`) = temporary closure, location remains, logged `PROVIDER_STATUS_CHANGE`.
- **QR-code dropoff (v2.1)**: Some LMS stores accept a drop-off where the customer presents a **QR code instead of a printed label** (staff scan the QR to bind the parcel). This capability is surfaced as a read-only `qr_dropoff` boolean: provider-sourced on the vendor layer, backfilled to the OC layer (`true` if **any** linked vendor location supports it). It is displayed as a marker/field on both lists and in the detail profile, is filterable, and is included in CSV export. PUDO does not expose the capability and is shown as N/A.
- **Duplicate detection (fuzzy)**: Cross-provider duplicate = two vendor locations from different providers whose addresses fuzzy-match above a threshold (weighted match score: address_line1 50% + postal_code 25% + city 15% + state 10%, token Jaccard; threshold 75%).
- **Sync does not reset OC fields**: Provider sync never overwrites OC-owned fields (`is_enable`, OC store details, `oc_location_id`). It only updates vendor-owned fields and propagates status.
- **Provider data is a proposal, not a fact (v2.6.0)**: For an OC location that is **not yet visible** (`is_enable = false`), provider changes are written straight through — nobody can see it, so review adds cost without adding safety. For a **visible** OC location, each changed field is judged by §8.6: low-risk tiers are applied automatically, everything else is **held** and the OC keeps its signed-off baseline value. Leaving a held change unresolved is always safe.
- **Baseline (v2.6.0)**: `baseline_at` marks the version a human verified. It is captured on enable sign-off, on accepting a provider change, and on every operator edit. The baseline — not «the value after the last sync» — is the reference for «good data»; this is what makes protection possible at all.
- **Two-run confirmation (v2.6.0)**: a change is auto-applied only if the same field carries the same new value in **two consecutive successful runs**. Upstream mistakes are frequently batch scripts that correct themselves the next day; this costs nothing and filters most transient noise.
- **Drift thresholds (v2.6.0)**: coordinates moving > 1 km, a postal/city change, a > 20-point drop in address similarity, or opening hours changing > 2 h on a single day (or becoming closed all week) force `review` regardless of tier.
- **Field lock upgrades the tier (v2.6.0)**: once a field is in `field_lock` (because an operator edited it, or a change was rejected), any later provider change to that field goes to review even if the field's base tier is `auto`. The lock is not a one-time mute — it is a permanent "ask me".
- **Operator edits are final (v2.6.0)**: a manual edit is a decision, not a proposal. Saving one (a) refreshes the baseline, (b) adds the edited fields to `field_lock`, (c) marks any pending change on those fields `superseded` with `SYNC_CHANGE_REJECTED`, and (d) if the location is visible, pushes the new value downstream immediately (`propagated_at`, `DOWNSTREAM_PROPAGATED`, downstream cache invalidated). Operator edits **never** enter the review queue — asking an operator to approve their own edit is a step without a decision. Visibility itself (`is_enable`) is unchanged by an edit, so no re-sign-off is needed.
- **Edit guardrails while visible (v2.6.0)**: clearing `address_line1` / `city` / `postal_code`, or clearing every opening-hours range, is rejected while the location is enabled (message: disable it first if it is closing). Validation runs **before** the record is mutated, so a rejected save leaves the data untouched. A customer seeing an empty address is worse than seeing a wrong one.
- **Enable is a sign-off, disable is a click (v2.6.0)**: `is_enable` false → true requires the sign-off dialog with 5 checks and a mandatory acknowledgement; true → false takes effect immediately, because going offline is always the safe direction. Batch enable runs the same checks per row, skips failures with a reason, and captures the baseline for every row it enables.
- **Breaker & shadow mode (v2.6.0)**: a run where changed records > 30%, address-type changes > 20, or batch `inactive` > 5 is downgraded to dry-run (vendor layer written, nothing propagated) and alerts. A newly onboarded provider runs in shadow mode for its first N runs (recorded, not propagated). `status` is the one exception to batching: `active → inactive` takes effect immediately (better offline than wrong) — which is precisely why batch `inactive` trips the breaker.
- **Rollback (v2.6.0)**: each run stores a before-image of the OC rows it would touch (7 days). A run can be rolled back as a batch (`SYNC_ROLLBACK`; fastest remedy after an upstream mistake) and a single field can be reverted to its previous version.
- **Change log completeness**: Every write produces exactly one log entry. Batch operations produce one entry per changed location; no-op rows skipped.
- **Batch upload rules**: Template has exactly 2 columns — `ocLocationID`, `is_enable`. `T` = enable, `F` = disable. Duplicate IDs: last wins. Unknown IDs rejected. Synchronous ≤ 5,000 rows with result summary.
- **Audit immutability**: Change log entries never edited or deleted.
- **ID stability**: Internal `vendor_location_id` and `oc_location_id` are permanent.

### 8.3 External System Integration

> **⚠️ Sync architecture is INCREMENTAL (confirmed 2026-10-29).** See §8.5.

| External System | Interface Purpose | Data Flow | Sync/Async | Integration Status | Notes |
|----------------|-------------------|-----------|------------|-------------------|-------|
| PUDO Inc Dealer API | Pull INCREMENTAL dealer changes (daily) | In: auth + `updated_since` cursor → Out: changed DealerList records | Async scheduled job (daily 02:00) | Existing API; incremental endpoint to be confirmed | Delta keyed on `LastUpdatedDate`; fallback full pull |
| LMS Location API | Pull INCREMENTAL location changes (daily) | In: auth + cursor + `updated_since` → Out: changed location records | Async scheduled job (daily 02:30) | Existing API; incremental endpoint to be confirmed | Iterate `has_more` for changed-record window |
| Customer-facing Location Query API (downstream) | Serves visible OC locations to downstream customers | Reads OC Location table with `status=active AND is_enable=true` filter | Sync (per request) | Existing | Enable/disable and status propagation take effect on next query |
| Corporate SSO | Staff authentication | In: credentials → Out: user identity + email | Sync | Existing | Groups → advanced permission mapping TBD |

### 8.4 Non-Functional Requirements

| Dimension | Requirement | Target | Priority |
|-----------|------------|--------|----------|
| Performance | List query p95 latency | ≤ 2s with filters, ≤ 5s export (10k rows) | P0 |
| Availability | Internal console SLA | 99.5% business hours; sync job must not block console | P0 |
| Security | Access control | SSO login mandatory; enable/disable + store edit gated by advanced permission; audit log immutable | P0 |
| Compatibility | Browser | Chrome / Edge latest two major versions, 1440px+ desktop | P0 |
| Scalability | Data growth | Support 100k vendor locations, 100k OC locations, 1M change log entries | P1 |
| Compliance | Data handling | Location data is business (non-PII); operator identity retained for audit | P1 |

---

### 8.5 Sync Failure Fallback & Incremental Strategy

**Incremental pull scope (confirmed 2026-10-29):**

- Each daily run pulls only records changed since the last successful sync (keyed on `updated_since` cursor).
- Scale bound: incremental window is a small fraction of the catalog.
- If a provider only supports full pull, fall back to full pull + local diff for that provider only.

**Sync failure fallback (confirmed 2026-10-29):**

- On failure, the local catalog is NOT cleared or partially updated — the last successful snapshot is preserved.
- Failed run marked `FAILED` on P06; `last_synced_at` not advanced.
- One automatic retry; then wait for next cycle.
- Incremental cursor advanced only on fully successful run.

### 8.6 Field-Level Sync Policy Matrix (v2.6.0)

> Only applies to OC locations that are **visible** (`status = active AND is_enable = true`). Before visibility, everything is written straight through.

| Field | Base tier | Rule | Why |
|---|---|---|---|
| `qr_dropoff`, `services`, `restrictions`, `cutoff_time` | **auto** | Apply, log `PROVIDER_FIELD_UPDATE` | Narrow blast radius — a wrong value does not send a customer to the wrong door |
| `phone`, `timezone` | **auto** | Apply, but block if the format check fails | Format is machine-checkable |
| `opening_hours` | **conditional auto** | Auto if the structure is valid **and** no day moves more than 2 h; otherwise `review` | Directly answers "can I go now?" — the top complaint source |
| `name` | **review** | Hold | Customer-facing display and search |
| `address_line1/2`, `city`, `state`, `postal_code` | **review** | Always hold | A wrong address sends a customer to the wrong store — the most expensive error |
| `latitude`, `longitude` | **review + hard check** | Hold; quarantine if drift > 1 km, outside the country, or in water | Bad coordinates navigate a customer into a river |
| `status` | **immediate (one-way)** | `active → inactive` applies at once; reactivation may also apply at once | Better offline than wrong — but batch `inactive` trips the breaker (§8.2) |
| **any field already in `field_lock`** | **review (forced)** | Hold regardless of base tier | The lock is a permanent "ask me", not a one-time mute |

**Hard checks (no judgement, straight to quarantine):** address becoming empty · postal code inconsistent with city/state · coordinate out of range · opening-hours structure invalid or `end < start`. A quarantined record writes nothing to the OC layer.

## 9. Non-Goals

- ❌ No editing of provider-owned business data on the **vendor layer** (address, hours, services) — vendor layer is read-only except sweeper.
- ❌ No location creation/deletion by internal users on the vendor layer — vendor catalog is provider-driven (enter/exit only via sync).
- ❌ OC store-detail editing is limited to OC-owned fields (address, opening hours, and flagged fields) — it does **not** rewrite provider source records.
- ❌ No end-customer-facing UI — downstream customers consume locations through the existing query API.
- ❌ No real-time push/webhook to downstream customers.
- ❌ No role management UI in v1.
- ❌ No non-English localization in v1.
- ❌ No provider management UI in v1.

## 10. Acceptance Criteria

> Given-When-Then format; all criteria marked "suggest PM review before finalizing".

### 10.1 Functional Acceptance Mapping

| ID | Feature | Happy Path | Boundary | Exception Data | Verification | Priority |
|----|---------|-----------|----------|----------------|--------------|----------|
| F01/F03 | Vendor & OC lists | ①Open P01 → both-provider vendor rows shown; ②Open P02 → OC rows shown with all columns | 0 rows → empty state; 100/page renders cleanly | Missing address2/phone renders "—" | UI walkthrough + API comparison | P0 |
| F02/F04 | Filter & search | ①Filter Provider=LMS, Status=active → only matching rows; ②chips visible; ③Reset restores | Combined filters AND-logic; time range crossing year | Invalid Location ID rejected inline | UI walkthrough | P0 |
| F05 | Export | ①Apply filters → ②Export → CSV matches filtered set | No filter = full catalog (≤10k); 10,001 → prompt to narrow | Null fields export as empty cells | File open + row count | P0 |
| F06 | Detail | ①Vendor detail read-only (except sweeper); ②OC detail shows editable fields + linked vendor | Empty hours/restrictions → empty-state text | Non-existent ID → not-found + back | UI walkthrough | P0 |
| F07 | Change log | ①Entries reverse-chronological with operator/time/type/field/old→new | Zero changes → creation entry only; 20+ paginate | Sync entries show "System · {Provider} Sync" | Seeded records | P0 |
| F08 | Single enable | ①Manager toggles OC location → confirm → `is_enable` flips + audit entry; ②Vendor inactive → toggle blocked with error | Toggle on already-enabled = no-op toast | API failure → error toast, state unchanged | Permission A/B + audit check | P0 |
| F09 | Batch enable/disable | **Mode A**: select 3 OC rows → disable → modal → confirm → 3 flip + 3 entries. **Mode B**: upload 100-row file → preview → confirm → applied + entries | 200 max (Mode A); 5,000 rows (Mode B); duplicate IDs last-wins; no-op rows skipped | Unknown ID / invalid T/F / wrong header / oversized → row-level error report | Seeded batch test | P0 |
| F10 | Sweeper edit | ①Manager edits Sweeper on vendor location → confirm → saved + `SWEEPER_CHANGE` entry | Viewer sees read-only; selecting current value = no-op | API failure → value unchanged | Permission A/B + audit | P0 |
| F11 | OC store edit | ①Manager edits OC address/hours → saved + `STORE_DETAIL_EDIT` entry; vendor record unchanged | Viewer sees read-only | API failure → value unchanged | Permission A/B + audit | P0 |
| F12 | OC generation | ①New vendor location (non-duplicate) → one OC location backfilled with `is_enable=false` + `OC_GENERATED` entry | Suspected duplicate → no OC location, enters review queue | — | Trigger test sync | P0 |
| F13 | Status propagation | ①Vendor → `inactive` → OC auto-disabled + blocked; ②force-enable raises error; ③vendor → `active` → OC can enable again | — | Provider reactivation mid-cycle | Trigger test sync | P0 |
| F14 | Duplicate review | ①Pair at ≥75% → shown with score + diff; ②"Same store" → link to one OC location; ③"Not same" → independent OC locations | Score below threshold not surfaced; identical address = 100% | Unit-suffix/postal-extension differences still match | Seeded pair + audit | P0 |
| F19 | Intake validation | ①Vendor location with an empty `address_line1` → **no OC location generated**, appears in Data Quality with the failed check; ②override → OC created with `is_enable=false` and `INTAKE_OVERRIDDEN` logged | `warn`-level issues allow creation but the sign-off stays blocked | Coordinates (0,0) flagged as placeholder | Seeded bad records + audit | P0 |
| F20 | Sync change review | ①Enabled OC location + provider changes `latitude` by 297 km → value **not** written, `⚠ N held` on the row, banner in detail, entry in Sync Changes; ②Accept → only that field changes, baseline refreshed, `SYNC_CHANGE_ACCEPTED`; ③Reject → baseline value kept, field locked, `SYNC_CHANGE_REJECTED` | Unresolved change → data unchanged after any number of days; accepting one field does not accept its neighbours | Auto-tier field (e.g. `phone`) applied without review; a locked field forced to review even though its tier is `auto` | Seeded run + audit + field-level assertion | P0 |
| F21 | Enable sign-off | ①Toggle enable on a ready location → dialog lists address/coordinates/hours + source + 5 checks; confirm disabled until the box is ticked; ②confirm → visible downstream + baseline captured + `ENABLE_SIGNED_OFF`; ③disable → immediate, no dialog | Blocked rows: vendor inactive / N pending / intake overridden / hours missing — dialog opens but confirm is unavailable with the reason shown; batch enable skips them and reports the count | Enabling an already-enabled location = no-op toast | Seed 4 blocked + 2 ready locations | P0 |
| F22 | Breaker & rollback | ①Run with > 30% changed records → dry-run, nothing propagated, alert shown; ②`SAFE MODE` on → all propagation frozen; ③rollback a run → affected OC rows return to the before-image | 7-day before-image window; rollback of an already-rolled-back run rejected | Provider `inactive` batch of 6 in one run trips the breaker | Simulated run + before/after comparison | P1 |
| F20/F21 | Operator edit semantics | ①Edit `phone` on a visible OC location → saved, baseline refreshed, `phone` locked, `DOWNSTREAM_PROPAGATED` written, toast says "pushed downstream now"; ②edit a field that has a pending provider change → that change becomes `superseded`, disappears from Sync Changes; ③edit on a non-visible location → not pushed, but baseline and lock still recorded | Clearing `address_line1` / `city` / `postal_code` while visible → rejected, record untouched (validation runs before the write) | Clearing every opening-hours range while visible → rejected with "disable it first" | Field-level assertion + audit order check | P0 |

| F23 | Field lock release | ①Edit `city` by hand → `city` appears under Locked fields; ②click unlock → **confirm is refused with no reason** → type a reason → confirmed, `FIELD_UNLOCKED` logged, chip gone; ③alternative path: on a held change for a locked field use «Accept & unlock» → value written, lock released, both `SYNC_CHANGE_ACCEPTED` and `FIELD_UNLOCKED` logged | Unlock without a reason is rejected; a Viewer cannot see the unlock affordance; unlocking a field with no pending change leaves the data untouched | Unlocking does not re-open or resurrect a change that was already rejected | Permission A/B + audit order check | P0 |
| F24 | Field revert | ①Edit `name` by hand → wrong value pushed downstream; ②Change History → «↩ Revert this field» → previous value restored, `FIELD_REVERTED` written, `DOWNSTREAM_PROPAGATED` follows, lock stays | Only the newest edit of a field offers Revert; older edits show none. A field edited twice shows exactly one revert link | Reverting an edit on a location that is not visible → stored and logged, **not** pushed | Field-level assertion + audit order check | P0 |

### 10.2 End-to-End Flow Acceptance

- [ ] Core path walkable: filter → select OC rows → batch disable → confirm → audit entries → location hidden downstream.
- [ ] Sync path walkable: nightly sync → vendor status change → OC auto-disable → log entry → staff reviews.
- [ ] OC generation path: new vendor location → OC backfilled (disabled) → Manager enables → visible downstream.
- [ ] **Protection path (v2.6.0): provider changes a live location → value held, customer keeps the verified value → operator accepts or rejects → only the accepted field changes and is logged.**
- [ ] **Intake path (v2.6.0): a vendor location with an incomplete address never becomes a live OC location — it lands in Data Quality instead.**
- [ ] **Sign-off path (v2.6.0): enabling requires the 5 checks and a ticked acknowledgement; batch enable cannot bypass them.**
- [ ] **Operator edit path (v2.6.0): a manual edit is pushed downstream immediately, refreshes the baseline, locks the field and suppresses any pending provider change on it.**
- [ ] **Fail-safe default (v2.6.0): if nobody touches a queue, the system never moves toward the provider's value — it stays at "no OC location", "not enabled", or "baseline value".**
- [ ] **Lock release path (v2.7.0): a hand-corrected field can be released — with a reason, or by accepting the provider value and unlocking in the same action.**
- [ ] **Revert path (v2.7.0): a wrong manual edit can be undone field by field from Change History, and the undo itself is audited and pushed downstream.**
- [ ] **One sign-off entry (v2.7.0): "Not live yet" on the OC list is the only place a location is enabled; the Review Center badge counts decisions only.**
- [ ] All P0 features clickable in the prototype.

## 11. Success Metrics

### 11.1 Metric Definitions

| Level | Metric | Definition | Baseline | Target | Cadence | Source |
|-------|--------|-----------|----------|--------|---------|--------|
| 🎯 North Star | Downstream catalog accuracy | % of OC locations whose effective visibility matches OC's intent | — | ≥ 99.5% | Monthly | Change log + spot audit |
| 📊 Core | Audit coverage of visibility changes | % of visibility changes with complete record | 0% | 100% | Monthly | Change log |
| 📊 Core | Daily sync success rate | % of days both provider syncs succeed | — | ≥ 99% | Monthly | Sync run records |
| 📊 Core | Time-to-locate | Median time to find a location via filters | ~minutes | < 30s | Quarterly | Usability sampling |
| 📊 Core | Batch operation adoption | % of bulk changes via batch vs one-by-one | — | ≥ 80% | Monthly | Change log |
| 📈 Auxiliary | Export usage | Exports per week | — | Track only | Weekly | Telemetry |
| 📈 Auxiliary | List query p95 latency | 95th percentile list API latency | — | ≤ 2s | Monthly | APM |

### 11.2 Measurement Plan

- T+7: verify audit coverage = 100% of visibility changes.
- T+30: first monthly reconciliation; review sync success rate.
- T+90: evaluate time-to-locate and batch adoption; decide v1.1 scope.

## 12. Risks & Dependencies

| Risk / Dependency | Type | Impact | Mitigation |
|-------------------|------|--------|------------|
| PUDO Inc / LMS API schema drift | Risk | Sync breaks, stale catalog | Contract tests; sync failure alerts (F16) |
| Provider incremental delta endpoint unavailable | Dependency | Fall back to full pull for that provider | Confirm `updated_since` capability; fallback flagged |
| Sync failure leaves stale catalog | Risk | Staff/customers act on outdated data | Last-success snapshot preserved; freshness indicator |
| SSO group → advanced permission mapping undefined | Dependency | Enable/edit control cannot ship | Confirm with IT before F08/F09/F10/F11 |
| Batch misuse (mass accidental disable) | Risk | Downstream catalog suddenly shrinks | Confirmation dialog; batch cap 200/5,000 |
| Malformed bulk upload file | Risk | Batch partially applied | Strict validation; row-level error report; dry-run preview |
| Data volume exceeds 10k export cap | Risk | Export blocked | Prompt to narrow filters |
| Timezone handling | Risk | Confusing displayed times | Normalize to console timezone (UTC) with labels |
| OC/vendor drift (OC overrides diverge from provider) | Risk | OC store details stale vs provider | Re-backfill on major provider field change (flagged for ops review) |
| **Provider writes a batch of wrong data** (v2.6.0) | Risk | Wrong addresses pushed to every customer in one night | Held baseline for live locations + two-run confirmation + drift thresholds + breaker (§8.2/§8.6) |
| **Operator edits something wrong themselves** (v2.6.0) | Risk | A manual edit is trusted by design and goes downstream immediately | Field-level baseline history (last 5 versions) + per-location Change History + rollback; option to add "revert to previous value" in the detail page |
| **Review backlog during onboarding** (v2.6.0) | Risk | A few hundred Data Quality items arrive at once; staff start bulk-approving and the gate becomes decorative | Plan onboarding capacity up front; show per-provider rejection rate so the conversation with the provider is data-backed |

## 13. Appendix

### 13.1 Glossary

| Term | Definition |
|------|-----------|
| OC | Operations / Location Curation team — internal owner of this console |
| PUDO Inc | Upstream provider #1 — supplies dealer locations (PUDOpoints / Lockers) |
| LMS | Upstream provider #2 — supplies package hub locations |
| Vendor Location | The raw provider-sourced location record (PUDO Inc / LMS); read-only except Sweeper |
| OC Location | The OC-owned, downstream-facing store entity; backfilled from vendor, OC-editable, carries the enable switch |
| Sweeper | Who physically picks up packages from a location (renamed from "Pickup"/`pickup_provider`); defaults to source_provider, OC-editable |
| PUDOpoint / Locker | PUDO Inc location types |
| packagehub | LMS location type |
| status | Provider-sourced business status (active / inactive / removed) |
| is_enable | OC-controlled switch on the OC location; determines downstream visibility |
| removed | Terminal provider state — permanently delisted |
| inactive | Temporary provider state — store closed / on break |
| OC location ID | OC-owned identifier on the OC layer; gates downstream sync |
| Duplicate pair | Two cross-provider vendor locations whose normalized addresses fuzzy-match above threshold |
| Match score | Weighted 0–100% address similarity (addr1 50% + postal 25% + city 15% + state 10%); ≥ 75% flags duplicate |
| Visible to customers | OC location with `status = active` AND `is_enable = true` |
| Incremental sync | Daily pull of only changed records since last success |
| FX / CP / TB | PUDO Inc service codes |
| Cutoff time | LMS — latest drop-off time for same-day processing |
| **Baseline** (v2.6.0) | The version a human verified — captured at sign-off, at accepting a change, or on an operator edit. The reference for "good data", not "whatever the last sync wrote" |
| **Held** (v2.6.0) | A provider change that was recorded on the vendor layer but deliberately **not** written to the OC layer, pending a decision |
| **Field lock** *(retired v3.0.0)* | Removed with manual editing — there is nothing to lock once the console stops writing values |
| **Sign-off** (v2.6.0) | The verification step that turns `is_enable` from false to true; captures the baseline |
| **Intake** (v2.6.0) | The quality check a new vendor location must pass before an OC location is created for it |
| **Breaker** (v2.6.0) | Automatic dry-run downgrade when a single sync run looks abnormal (too many changes at once) |
| **Duplicate Check** (v3.0.0) | Standalone page answering one question — same store (2:1) or not (1:1). Replaces the Duplicates tab |
| **Data Issues** (v3.0.0) | One ledger: provider issues, baseline drift, failed intake. Replaces the Review Center |
| **data_issue** (v3.0.0) | A flag on a source field with a mandatory reason. Never changes the value. v3.1.0 adds `sub_field`, `issue_type` and `severity`; only **blocking** issues stop a sign-off |
| **issue type** (v3.1.0) | One of seven: missing / incorrect / inconsistent / out of range / duplicate / stale / format. Each has a default severity and a one-line definition |
| **Source-priority merge** (v3.0.0) | Per-field rule: M&MP (self-operated) → LMS → PUDO Inc; first source with a value wins |
| **Source–type constraint** (v3.2.0) | One source can only produce certain location types. Enforced in the Type filter (derived from Source) and checked on every record as `type ≠ source` |
| **M&MP** (v3.0.0) | Our own self-operated store (US term). A location source like PUDO Inc and LMS, but first-party: the store maintains its own record and data issues are chased internally |
| *(see above — field lock is retired in v3.0.0)* |
| **Revert** *(retired v3.0.0)* | Removed: reverting is an edit. A wrong provider value is flagged, not reverted |

### 13.2 References

- PUDO Inc DealerList API sample (provided 2026-10-29)
- LMS Location List API sample (provided 2026-10-29)
- **Feature overview (English, written for readers outside the team)**: `https://jama-dev520.github.io/location-prototype/feature-overview.html` — one page, no internal feature IDs; the Markdown source sits next to it as `Location-Management-Admin-Feature-Overview-EN.md`

### 13.3 Prototype Link

- `Location-Management-Admin-Prototype-2026-10-29.html` — v2.6.1, hi-fi single-file prototype
- Public: https://jama-dev520.github.io/location-prototype/
- **Demo video**: the recording at `https://jama-dev520.github.io/location-prototype/demo/location-admin-demo.mp4` was captured at **v2.7.0**, before the v3.0.0 model change — it still shows store editing, which no longer exists. **Re-shoot pending**; do not use it for v3 acceptance. The clickable prototype is the source of truth.
- Flowcharts (2x PNG, safe to embed in Feishu by absolute URL): `https://jama-dev520.github.io/location-prototype/diagrams/flow{1..9}-*.png`
- Demo seed data deliberately includes: a live location whose provider moved the coordinates 297 km (`OC-00102`) · a held postal-code change (`OC-00103`) · a held name change (`OC-00106`) · a held opening-hours change (`OC-00109`) · two intake failures with no OC record (`LOC-00187` / `LOC-00188`) · one blocked sign-off (`OC-00002`, vendor inactive) and two ready ones

### 13.4 Test Requirements

| Dimension | Requirement | Notes |
|-----------|------------|-------|
| External system test environments | PUDO Inc / LMS sandbox endpoints TBD | If unavailable, mock provider APIs |
| Test data source | Seed from samples + ~500 synthetic locations across both providers | Include edge cases: empty hours, missing address2, inactive status |
| Test accounts | 1 Viewer + 1 Location Manager | Per §3.1 |
| Mock requirements | Mock nightly sync trigger; mock partial batch failure; mock vendor→inactive transition; **mock a provider run that moves coordinates 297 km, a run with > 30% changed records (breaker), and an intake failure (empty address1 / coordinates (0,0))** | Verify F09 partial-failure, F13 auto-disable, F19 intake, F20 held change, F22 breaker |
| **Plain-language acceptance** | Plain-language acceptance checklist (Chinese, for business and test readers) — an 8-item quick check, per-page checklists, 6 end-to-end stories, and a "must never happen" list | Written for operations staff and PM sign-off, not engineers |
