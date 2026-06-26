# BP-003 — Vendor Data Management

**Status:** Draft — pending SME review
**Owners:** Vendor Master team, Merchandising IT
**Anchor programs:** `MCCST19`, `MCBSM52J`, `AUTHORKAY`, `AUTHORMCLANE`
**Related SRS:** FR-VEND-001 … FR-VEND-004
**Related HLD:** §5.1 (vendor entities), §5.2 (`<DIV>.MSTR.VND`), §3.1 (`AUTHOR*` family)

---

## 1. Overview

This BP owns vendor master data — both the **corporate** view (`ACME.VNDR_MSTR_VN1A`, `CRP_VNDR_XREF_VN1X`) and the **divisional** view (`<DIV>.MSTR.VND` for each of the 32 divisions). It includes:

- Vendor lookups by short name (`MCCST19`).
- Vendor synchronisation pipelines (`MCBSM52J`) that reconcile inbound files (`NEWVNDRS`, `CORPVNDR`) into the masters.
- **Per-vendor specialised handlers** (`AUTHORKAY`, `AUTHORMCLANE`) that surface in the corpus as named, vendor-specific programs.

The BP also serves as the primary lookup source for vendor attributes consumed by BP-001 (item-vendor relationship) and BP-004 (costing / cost-control flag).

---

## 2. Programs Involved

| Type | Name | Purpose |
|---|---|---|
| COBOL | `MCCST19` | Vendor List Retrieval by Short Name. |
| JCL job | `MCBSM52J` | Synchronises and reports vendor data. |
| COBOL | `AUTHORKAY` | Retrieves vendor item cost and deal information (Kay-specific). |
| COBOL | `AUTHORMCLANE` | Generates reports for missing cost records (Acme-specific). |
| Sequential | `NEWVNDRS`, `CORPVNDR` | Inbound vendor data files. |
| `[CODMOD]` | All other programs writing to `ACME.VNDR_MSTR_VN1A` (46 total) | Full enumeration TBD. |

---

## 3. Business Rules

### 3.1 Vendor lookup (`MCCST19`)

| ID | Rule | Source |
|---|---|---|
| BR-003-01 | Lookup keys on **vendor short name** (case sensitivity per legacy COBOL string semantics). | `MCCST19` |
| BR-003-02 | Returned set joins `ACME.VNDR_MSTR_VN1A` with `CRP_VNDR_XREF_VN1X` to provide the corporate-vendor view. | `MCCST19` |
| BR-003-03 | `[SME]` Confirm whether `MCCST19` filters out inactive vendors (`VNKY-RECORD-STATUS-CODE ≠ 'A'`). |  |

### 3.2 Vendor sync (`MCBSM52J`)

| ID | Rule | Source |
|---|---|---|
| BR-003-10 | Inbound `NEWVNDRS` records are merged into `ACME.VNDR_MSTR_VN1A` and the per-division `<DIV>.MSTR.VND`. | `[CODMOD]` |
| BR-003-11 | Inbound `CORPVNDR` records update `CRP_VNDR_XREF_VN1X`. | `[CODMOD]` |
| BR-003-12 | Post-job state must be consistent across the corporate and divisional vendor masters. | NFR-DATA-001 |

### 3.3 Cross-cutting vendor rules

| ID | Rule | Source |
|---|---|---|
| BR-003-20 | Vendor active for **cost evaluation** iff `VNKY-RECORD-STATUS-CODE = 'A'` (consumed by BP-001 BR-001-01). | `XXCAD63` |
| BR-003-21 | Vendor cost control flag (`VNRT-COST-CONTROL-FLAG = 'Y'`) gates whether costs may be set/changed for items keyed to this vendor (consumed by BP-001 BR-001-02 and BP-004). | `XXCAD63` |

### 3.4 Per-vendor specialised handlers

| ID | Rule | Source |
|---|---|---|
| BR-003-30 | `AUTHORKAY` retrieves vendor item cost and deal information for vendor "Kay" (specialised path). | `AUTHORKAY` |
| BR-003-31 | `AUTHORMCLANE` generates a report of missing cost records for vendor "Acme". | `AUTHORMCLANE` |
| BR-003-32 | `[SME]` Determine whether the pattern of vendor-named handlers is enumerable (find all `AUTHOR*` programs) and which business processes invoke them. |  |

---

## 4. Data Structures

### 4.1 DB2 entities

| Entity | Role | Programs |
|---|---|---:|
| `ACME.VNDR_MSTR_VN1A` | Corporate vendor master | 46 |
| `CRP_VNDR_XREF_VN1X` | Corporate-vendor cross-reference | (corpus) |

### 4.2 Divisional dataset families

`<DIV>.MSTR.VND` for each of the 32 divisions.

### 4.3 Inbound sequential files

| File | Direction | Producer | Consumer |
|---|---|---|---|
| `NEWVNDRS` | Inbound | External vendor onboarding | `MCBSM52J` |
| `CORPVNDR` | Inbound | Corporate-level vendor source | `MCBSM52J` |

---

## 5. Sequence Diagrams

### 5.1 Vendor lookup by short name (`MCCST19`)

```
Caller        MCCST19            DB2
  │ short-name ──►│
  │               │── SELECT FROM ACME.VNDR_MSTR_VN1A JOIN CRP_VNDR_XREF_VN1X
  │               │   WHERE short_name = :param
  │               │◄──────── rows ──────────
  │ ◄── vendor list│
```

### 5.2 Vendor synchronisation (`MCBSM52J`)

```
Operator ──► Scheduler ──► MCBSM52J
                              │
                              ├── read NEWVNDRS  ──► merge into ACME.VNDR_MSTR_VN1A
                              ├── read CORPVNDR ──► update CRP_VNDR_XREF_VN1X
                              ├── per division   ──► reconcile <DIV>.MSTR.VND
                              ├── reporting step ──► sync report
                              └── on DB2 / file error → abend (RC=16 per platform convention)
```

### 5.3 Per-vendor specialised handler (`AUTHORKAY`)

```
Caller (deal job) ──► AUTHORKAY
                        │
                        ├── lookup vendor "Kay" specifics
                        ├── pull cost + deal info from ACME.* tables
                        └── return enriched record
```

---

## 6. Test Cases

| ID | Scenario | Expected | Maps to |
|---|---|---|---|
| TC-003-01 | Lookup by short name returns rows | Joined vendor + xref rows | BR-003-01/02 |
| TC-003-02 | Lookup short name with leading spaces | `[SME]` define expected behaviour | BR-003-01 |
| TC-003-03 | Sync inserts new vendor | New row in `ACME.VNDR_MSTR_VN1A` and matching `<DIV>.MSTR.VND` entries | BR-003-10/12 |
| TC-003-04 | Sync updates corp cross-reference | `CRP_VNDR_XREF_VN1X` reflects `CORPVNDR` payload | BR-003-11 |
| TC-003-05 | Vendor flagged inactive | Downstream cost evaluation skips it | BR-003-20 |
| TC-003-06 | Cost-control flag off | Downstream cost evaluation skips it | BR-003-21 |
| TC-003-07 | Kay-specific deal | `AUTHORKAY` invoked, returns enriched record | BR-003-30 |
| TC-003-08 | Acme missing-cost report | `AUTHORMCLANE` produces expected report layout | BR-003-31 |

---

## 7. Open Questions

- `[CODMOD]` Enumerate all 46 programs referencing `ACME.VNDR_MSTR_VN1A`; classify by read-only vs read/write.
- `[CODMOD]` Enumerate all `AUTHOR*` programs and the deal/cost jobs that invoke them.
- `[SME]` Define merge precedence rules for `NEWVNDRS` vs existing `ACME.VNDR_MSTR_VN1A` (most recent wins? source-system precedence?).
- `[SME]` Confirm whether the per-division `<DIV>.MSTR.VND` is the source of truth for the division or a cached projection of the corporate master.
- `[RAG]` Pull and document `MCBSM52J` step list (sort / utility steps interleaved with COBOL).

---

## 8. Modernization Notes

- **Blast radius:** Mid-range — `ACME.VNDR_MSTR_VN1A` (46 programs) is heavily read but writes appear concentrated in `MCBSM52J` + a small set of `MCBSM*` programs.
- **Preservation:**
  - The two vendor-state gates (`VNKY-RECORD-STATUS-CODE = 'A'`, `VNRT-COST-CONTROL-FLAG = 'Y'`) are referenced from costing and item-master BPs — preserve their semantics exactly.
  - `AUTHOR*` per-vendor handlers must remain pluggable; the modernized vendor module should expose a strategy hook keyed by vendor identity.
- **Suggested cutover unit:** Can cut over corporate vendor master independently of divisional projections, provided the consumer BPs (BP-001, BP-002, BP-004) are updated to read from the modernized vendor service via a facade.
