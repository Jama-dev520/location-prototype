# Location Management Admin Console — Feature Overview

**Audience:** operations, partner-facing and M&MP teams who work with location data
**Document version:** 1.6 (covers console v3.6.0) · **Status:** clickable prototype (not in production)
**Prototype:** https://jama-dev520.github.io/location-prototype/
**One-page version (shareable):** https://jama-dev520.github.io/location-prototype/feature-overview.html
**Languages:** the console itself switches between English and Chinese in the header (v3.5.0); interface only — data stays as delivered  
**Demo video (3:20, English captions):** https://jama-dev520.github.io/location-prototype/demo/location-admin-demo.mp4

---

## 1. What this console is for

It is the internal console for the locations our customers see — pickup points, lockers, package hubs and our own
self-operated stores. It exists so four daily jobs take seconds instead of a ticket:

| # | What someone needs to do | What "done" looks like | Where |
|---|---|---|---|
| **J1** | Find out which locations are live | Filter `Status = active` + `Downstream = Live`, see the count, export exactly that set | OC Location List |
| **J2** | Find out who actually collects parcels (sweeper) | Already on the row, in the detail page, filterable — no spreadsheet | Both lists |
| **J3** | Correct a wrong address | Report it to the owner of the data with a precise reason; it is fixed at the source and arrives with the next sync | Vendor record → `Flag a data issue` |
| **J4** | See what changed, and when | Any field, any location: who, when (to the second), old value → new value | Detail → Change History / Change Log |

Those four jobs are the reason the console has two layers and not one. The next sections explain the shape they forced.

---

## 2. The model in one page

```
PUDO Inc ─┐
LMS ──────┼──► Vendor layer (read-only) ──► intake check ──► duplicate check ──► OC layer ──► Downstream
M&MP ─────┘        raw source data              │                │            enable + flags      customers
                                                │                │
                                          failed → no OC      75%+ → hold
```

### 2.1 Two layers

| Layer | What it holds | Who may change it |
|---|---|---|
| **Vendor Location** | Exactly what the source sent us — every field, including `sweeper` | Nobody in this console. It is a mirror of the source system |
| **OC Location** | The customer-facing record: one per store, assembled from one or more vendor records | The operator, but only two things: `enable` and data-issue flags |

The OC layer is what downstream systems receive. The vendor layer is evidence — kept unchanged so a dispute can always be
settled by looking at what was actually sent.

### 2.2 Three sources, one directory

| Source | What it is | Who maintains the data | Who fixes a problem |
|---|---|---|---|
| **PUDO Inc** | Partner dealer locations — `PUDOpoint` and `Locker` | PUDO Inc sync | PUDO Inc |
| **LMS** | Partner package hubs — `packagehub` only | LMS sync | LMS |
| **M&MP** | Our own **self-operated stores** (first-party) | the store itself | the store owner, internally |

M&MP is a **data source**, not a team or a role. M&MP stores follow exactly the same rules as partner locations: same
directory, same sign-off, same issue ledger. The only difference is ownership — and that first-party data outranks a
partner's when values disagree (see 2.4).

### 2.3 Source fixes the type

A source can only ever produce its own location type, so the UI never offers an impossible combination:

| Source | Allowed type | In the filter |
|---|---|---|
| LMS | `packagehub` | one option, selected automatically |
| PUDO Inc | `PUDOpoint`, `Locker` | both offered, with live counts |
| M&MP | `M&MP` | one option, selected automatically |

Choosing a Source rebuilds the Type filter from this table; each option shows how many rows it would return; clearing the
source clears the type. If a record ever arrives with a type that does not belong to its source, the Type column marks it
**`type ≠ source`** and one filter isolates those rows — it is a signal about the feed, not something we correct by hand.

### 2.4 One store, several vendor records

When a duplicate pair is confirmed as the same physical store, two vendor records feed one OC location (mapping `2:1`).
Each **field** is then resolved independently, by source priority:

**M&MP (self-operated) → LMS → PUDO Inc** — the first source that has a value wins, and the next fills the gaps.

Every field in the detail page shows where its value came from, so "why does this say X?" is answerable without asking
anyone. Type is not merged loosely: it follows the constraint in 2.3.

### 2.5 One visibility rule

> A location is visible to customers **iff** its vendor record is `active` **and** the operator has enabled it.

`status` comes from the source; `is_enable` is our decision. Downstream receives the intersection, and nothing else.

---

## 3. What an operator can actually do

Exactly **two verbs**. Everything else on these screens is read-only, and that is deliberate.

### 3.1 Enable — a sign-off, not a switch

Two different questions, two different owners: **is this value applied?** — it always is, and nothing here decides it;
**is this store visible?** — the operator decides, through this sign-off. Enabling is the only moment a location becomes
customer-visible, so it is treated as a signed release.

Turning a location on is the only moment it becomes customer-visible, so it is treated as a signed release. Five checks
run first; any failure blocks the button and states the reason:

1. An **active** vendor location feeds this store
2. Opening hours are present
3. Intake checks passed (no override)
4. No duplicate pair waiting for a decision
5. No open **blocking** data issue

The dialog then shows the customer-facing values — address, postal code, coordinates, opening hours — each with its
source, plus an acknowledgement the operator must tick. Confirming captures a **signed-off baseline** and writes
`ENABLE_SIGNED_OFF` to the audit trail.

Turning a location **off** is one click. Going offline is always the safe direction, so it needs no ceremony.

### 3.2 Flag a data issue — the replacement for editing

The console never writes a location field. A wrong value is reported to whoever owns it, at four levels of precision:

| Level | Example | Why it matters |
|---|---|---|
| **Field** | `opening_hours` | what is wrong, broadly |
| **Sub-field** | `SATURDAY` / `address_line2` / `latitude` | makes it actionable — a provider cannot fix "hours are wrong" |
| **Type** | `MISSING` `INCORRECT` `INCONSISTENT` `OUT_OF_RANGE` `DUPLICATE` `STALE` `FORMAT` | tells the owner what kind of fix is needed |
| **Severity** | `blocking` (cannot be signed off) / `warning` (live, but wrong for customers) | decides whether it stops a launch |

A free-text reason is mandatory. The flag lives on the source record and moves `open → reported → fixed`; the fix arrives
with the next provider sync. Owners are different: a provider for PUDO Inc / LMS, the store owner for M&MP.

**And the issue closes itself when the fix lands.** The next run compares the delivered value with the value that was flagged:
if it changed *and* is valid, the issue closes (`closed_by = AUTO_SYNC`, run id and verified value kept in the audit trail);
if it changed but is still invalid, the issue stays open and says so; if nothing changed, nothing happens. Only
machine-checkable types close this way — `MISSING`, `FORMAT`, `INCONSISTENT`, `OUT_OF_RANGE`. `INCORRECT`, `STALE` and
`DUPLICATE` always come back to a human, because the feed can deliver a new value but cannot prove it is the right one.

### 3.3 Why there is no edit button

Every safety mechanism this console used to need — field locks, unlock reasons, per-field revert, accept/reject of
provider changes — existed for one purpose: to make manual edits safe. Remove the edit, and those mechanisms have nothing
left to protect. The data also stays trustworthy: what downstream receives can always be traced to one source record.

---

## 4. After go-live

Provider values keep flowing and **always reach downstream** — the console never decides whether a change is applied.
There is no accept, reject or re-sign step anywhere.

What the console keeps is the signed-off **baseline**, purely so it can answer one question: *what moved since we signed
this off?* That shows as **drift** on the OC row, in the detail page and in Data Issues. There is exactly one action:
**flag** a move that is wrong.

---

## 5. Finding things

| Need | How |
|---|---|
| Which locations are live | `Downstream = Live`; the `⏳ Not live yet (N)` preset is the onboarding queue and matches the menu badge |
| Which provider sent this | `Source` filter; `2:1` rows show both |
| Only hubs / only lockers | `Source` then `Type` (2.3) |
| Everything wrong right now | `Data Issues` — one ledger with three sections: open data issues, baseline drift, failed intake |
| Which records look like the same store | `Duplicate Check` — fuzzy score, per-field diff, both sides, `Same store (2:1)` / `Not same (1:1)` |
| What changed last week | `Change Log` — every action, every field, old → new, with second-level timestamps |
| A work list for a provider | **Export open issues (CSV)** on Data Issues — id, both location ids, field · sub-field, type, severity, reason, who raised it; the OC list export carries `Open issues` / `Changed since sign-off` columns |

All timestamps are shown and exported to the second (`YYYY-MM-DD HH:MM:SS`); the header carries a live clock so the
audit trail and the screen agree.

---

## 6. Roles

| Role | Can do |
|---|---|
| **Location Manager** (operations) | enable / disable, sign off, flag issues, decide duplicates, export the issue list |
| **Viewer** | read and export only; every action control is disabled |

M&MP is not a role — self-operated stores are handled by the same people under the same permissions.

---

## 7. Scope of this build

**In the clickable prototype**

- Vendor Location List · OC Location List (with the sign-off queue) · Location detail (5 tabs: Basic Info / Operations /
  Hours & Services / Issues / Change History) · Duplicate Check · Data Issues · Change Log · Sync Status
- 20 vendor records and 16 OC locations of demo data, including deliberately broken records (missing address, `(0,0)`
  coordinates, a 297 km coordinate jump, an undecided duplicate pair)

**Not built yet**

- Overview dashboard (P07) — specified, P2, out of scope for this round
- Batch upload (Excel) — specified in the PRD, not in the prototype
- The demo video at `/demo/location-admin-demo.mp4` (3:20, English captions) is current as of v3.6.0

---

## 8. Version history (short)

| Version | Change |
|---|---|
| v3.4.0 | No approval step: provider data is always applied; drift is information; issues export to CSV |
| v3.3.0 | Issues close themselves when a provider run delivers a valid value |
| v3.2.2 | The self-operated type value is `M&MP` (it used to read “M&MP store”) |
| v3.2.1 | Documentation only — sharper summary of what the console does and does not do |
| **v3.2.0** | Source fixes the type; both lists' Type filter is derived from Source with live counts |
| v3.1.0 | Data issues split into field → sub-field → type → severity |
| v3.0.0 | **Model change:** the console stops editing. Two verbs only — enable and flag. Duplicates get their own page; values merge per field by source priority |
| v2.7.0 | Field locks releasable, field-level revert, sign-off queue on the OC list |
| v2.6.0 | Provider data treated as a proposal: baselines, review queues, sign-off |

---

## 9. Questions this usually raises

**Why not let operations fix the address?** Because then two systems hold different addresses and neither is authoritative.
A flag with a reason reaches the party who can actually fix it, and the fix lands everywhere at once.

**What if a provider is slow?** The issue stays open and keeps blocking new sign-offs (if it is `blocking`); existing live
locations keep serving their current values. Nothing silently changes.

**What about a store that genuinely moved?** The provider updates the record; the next sync brings the new address, it
shows as drift against the baseline, and the operator re-signs. Two clicks, full trace.

**Why LMS first, and M&MP before that?** M&MP is our own data; LMS is the more complete partner feed for hubs. The rule is
per field, so an empty LMS phone never blanks a working PUDO phone.

**Can we see who did what?** Yes — every action above writes an audit row with actor, timestamp (to the second), field,
old value and new value.
