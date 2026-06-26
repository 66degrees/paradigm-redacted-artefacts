# BP-001 — Item Master Data Management

**Status:** Draft — pending SME review
**Owners:** Item Master team, Merchandising IT
**Anchor programs:** `XXCAD63`, `MCCAD65J`, `XXDL656`, `MCDL656J`, plus per-division `<DIV>.MSTR.SIM` flows
**Related SRS:** FR-ITEM-001, FR-ITEM-002, FR-ITEM-003, FR-COST-001
**Related HLD:** §5.1 (DB2 schema), §5.2 (divisional dataset families)

---

## 1. Overview

This business process owns the **item master** of the merchandising platform across 32 divisions. It maintains:

- The **divisional item master** datasets (`<DIV>.MSTR.SIM`) per division, with the `ACME.*` corporate-level item entities reconciled to them.
- The **item–vendor** relationship (`ACME.ITEM_VNDR_DE6V`).
- The **UPC catalogue** (`ACME.ITEM_UPC_DE6Y`).
- The **division ↔ item ↔ pack** hierarchy (`ACME.DIV_ITEM_PACK_DE1I`).
- The corporate item-attribute store (`ACME.UIN_ITEM_DE6C`, `ACME.ITEM_MASTER_IM3I`).

The process is triggered by item-master pushes from upstream merchandising systems, by buyer-driven changes via CICS, and by the costing pipeline (which validates and enriches item costs).

---

## 2. Programs Involved

| Type | Name | Purpose |
|---|---|---|
| JCL job | `MCCAD65J` | Orchestrates processing of item cost data; executes `XXCAD63`. |
| COBOL | `XXCAD63` | Validates and processes item cost data; basis-code mapping; discrepancy reporting. |
| COBOL | `XXCAD65` | Companion to `XXCAD63`; cursor-driven matching of `XXCAD` key vs. `XXDE9E` key with discrepancy handling. |
| JCL job | `MCDL656J` | Orchestrates deal data enrichment for historical analysis; per-division extract production via `iefbr02` / `iefbr03`; executes `XXDL656`. |
| COBOL | `XXDL656` | Enriches deal data with item-specific information. |
| Common | `D0325` (data dictionary), `D0703` (print copy), `D0007` (extended errors) | Cross-cutting utilities |
| `[CODMOD]` | (additional item-touching programs from the 312-program inventory) | TBD |

---

## 3. Business Rules

### 3.1 From `XXCAD63` — item cost validation

| ID | Rule | Source |
|---|---|---|
| BR-001-01 | Vendor is *active* iff `VNKY-RECORD-STATUS-CODE = 'A'`. | `XXCAD63` |
| BR-001-02 | Vendor cost control is enabled iff `VNRT-COST-CONTROL-FLAG = 'Y'`. | `XXCAD63` |
| BR-001-03 | A cost record is "found" iff `WS-CUR-COST-SW = 'Y'` OR `WS-FUT-COST-SW = 'Y'`. | `XXCAD63` |
| BR-001-04 | Basis-code mapping: `CDIC-LIST-AMT-TYPE = 'F'` → cost basis `'ACME'`. | `XXCAD63` |
| BR-001-05 | Basis-code mapping: `CDIC-LIST-AMT-TYPE = '1'` → cost basis `'CW'`. | `XXCAD63` |
| BR-001-06 | Basis-code mapping: any other value → cost basis `'SC'`. | `XXCAD63` |
| BR-001-07 | Effective-date discrepancy between current and future cost records → emit a discrepancy report entry (do not abort). | `XXCAD63` |
| BR-001-08 | Cost-amount discrepancy between current and future records → emit a discrepancy report entry. | `XXCAD63` |
| BR-001-09 | Exception items: `WS-EXC-ITEM-SW = 'Y'` AND `WS-AP1R-SW = 'Y'` → log `"ITEM EXCEPTION - NO UPDATES"` and continue. | `XXCAD63` |
| BR-001-10 | File status `'23'` or `'10'` → abend with `RETURN-CODE = 16`. | `XXCAD63` |

### 3.2 From `XXCAD65` — cursor-matching

| ID | Rule | Source |
|---|---|---|
| BR-001-11 | If `XXCAD` key == `XXDE9E` key and data matches: advance both readers. | `XXCAD65` |
| BR-001-12 | If keys match but data differs: invoke `2100-CHECK-DATA-PARA` (discrepancy handler), then advance both readers. | `XXCAD65` |
| BR-001-13 | If `XXCAD` key > `XXDE9E` key: advance `XXDE9E` reader (and corresponding orphan handling). | `XXCAD65` |
| BR-001-14 | (Symmetric for `XXCAD` key < `XXDE9E` key.) | `XXCAD65` |

### 3.3 Divisional consistency rules (implicit, to be confirmed by SME)

| ID | Rule | Status |
|---|---|---|
| BR-001-20 | Every `<DIV>.MSTR.SIM` record MUST resolve to a corresponding entry in `ACME.DIVMSTRDI1D`. | `[SME]` confirm |
| BR-001-21 | Item changes MUST update both the divisional `<DIV>.MSTR.SIM` and the corporate `ACME.ITEM_MASTER_IM3I` within the same orchestrating job. | `[SME]` confirm |
| BR-001-22 | UPC writes (`ACME.ITEM_UPC_DE6Y`) MUST resolve their parent item via `ACME.UIN_ITEM_DE6C`. | `[SME]` confirm |

---

## 4. Data Structures

### 4.1 DB2 entities

| Entity | Role | Programs referencing |
|---|---|---:|
| `ACME.DIVMSTRDI1D` | Division master (corporate ↔ division mapping) | 124 |
| `ACME.UIN_ITEM_DE6C` | UIN-keyed item attributes | 64 |
| `ACME.DIV_ITEM_PACK_DE1I` | Division item pack | 62 |
| `ACME.ITEM_VNDR_DE6V` | Item–vendor relationship | 35 |
| `ACME.ITEM_UPC_DE6Y` | Item ↔ UPC | 30 |
| `ACME.ITEM_MASTER_IM3I` | Corporate item master attributes | (suppression cluster) |

### 4.2 Divisional dataset families

`<DIV>.MSTR.SIM` for each of: `MZ, SW, SZ, SE, NC, ME, NE, PA, NW, SO, MS, MK, MP, MI, MO, HP, MW, MD, MY, MN, WJ, FE, MG, NT, GM, GA, GF, WK, C1, C2, C3, ACME`.

`[SME]` provide the full record layout for `<DIV>.MSTR.SIM` and confirm whether divisions differ in layout or all share a common record schema.

### 4.3 Working-storage anchors used by rules

`VNKY-RECORD-STATUS-CODE`, `VNRT-COST-CONTROL-FLAG`, `CDIC-LIST-AMT-TYPE`, `WS-CUR-COST-SW`, `WS-FUT-COST-SW`, `WS-CUR-EFF-DT-SW`, `WS-FUT-EFF-DT-SW`, `WS-EXC-ITEM-SW`, `WS-AP1R-SW`.

---

## 5. Sequence Diagrams

### 5.1 `MCCAD65J` — item cost validation pipeline

```
Operator        Scheduler           MCCAD65J (JCL)            XXCAD63             DB2 / files
   │               │                     │                       │                     │
   │── submit ────►│                     │                       │                     │
   │               │── start ───────────►│                       │                     │
   │               │                     │── EXEC XXCAD63 ──────►│                     │
   │               │                     │                       │── set DB2 package ─►│
   │               │                     │                       │── fetch vendor ◄────│
   │               │                     │                       │── if !A → skip      │
   │               │                     │                       │── fetch cost cur ◄──│
   │               │                     │                       │── fetch cost fut ◄──│
   │               │                     │                       │── BR-001-04..06     │
   │               │                     │                       │── BR-001-07..08 → report
   │               │                     │                       │── BR-001-09 → log
   │               │                     │                       │── on f-st 23/10 → RC=16 ──► ABEND
   │               │                     │                       │                     │
   │               │                     │◄── RC=00 ─────────────│                     │
   │               │◄── job end ─────────│                       │                     │
```

### 5.2 `MCDL656J` — per-division enrichment fan-out

```
MCDL656J (JCL) ── invokes XXDL656 per division (MZ, SW, SZ, … via iefbr02/03)
                  │
                  ├── reads ACME.DIV* + per-division <DIV>.MSTR.SIM
                  ├── enriches deal data
                  └── writes <DIV>.PERM.<DIV>DL656{1,2,3} extracts
```

---

## 6. Test Cases

| ID | Scenario | Inputs | Expected | Maps to |
|---|---|---|---|---|
| TC-001-01 | Inactive vendor is skipped | `VNKY-RECORD-STATUS-CODE = 'X'` | No cost-record evaluation; no discrepancy entry | BR-001-01 |
| TC-001-02 | Cost-control off | `VNRT-COST-CONTROL-FLAG = 'N'` | No cost evaluation | BR-001-02 |
| TC-001-03 | Basis code 'F' | `CDIC-LIST-AMT-TYPE = 'F'` | Output basis = `'ACME'` | BR-001-04 |
| TC-001-04 | Basis code '1' | `CDIC-LIST-AMT-TYPE = '1'` | Output basis = `'CW'` | BR-001-05 |
| TC-001-05 | Basis code 'Z' (other) | `CDIC-LIST-AMT-TYPE = 'Z'` | Output basis = `'SC'` | BR-001-06 |
| TC-001-06 | Effective-date discrepancy | Current vs future eff dates differ | Discrepancy report entry written; job continues | BR-001-07 |
| TC-001-07 | Cost-amount discrepancy | Current vs future cost amounts differ | Discrepancy report entry; job continues | BR-001-08 |
| TC-001-08 | Exception item, both switches Y | `WS-EXC-ITEM-SW='Y'`, `WS-AP1R-SW='Y'` | Log `"ITEM EXCEPTION - NO UPDATES"`; no abend | BR-001-09 |
| TC-001-09 | File status 23 | Cursor read returns 23 | Program ends with `RETURN-CODE = 16` | BR-001-10 |
| TC-001-10 | File status 10 | EOF unexpected | Program ends with `RETURN-CODE = 16` | BR-001-10 |
| TC-001-11 | Per-division item fan-out | Items across all 32 divisions | Each division processed; reconciled to `ACME.DIVMSTRDI1D` | BR-001-20 |
| TC-001-12 | Item ↔ UPC resolution | UPC scan input | Resolves to expected item via `ACME.UIN_ITEM_DE6C` → `ACME.ITEM_UPC_DE6Y` | BR-001-22 |

---

## 7. Open Questions

- `[SME]` `<DIV>.MSTR.SIM` record layout — same across divisions?
- `[SME]` What populates `WS-AP1R-SW`? It gates the exception-item soft-fail behaviour (BR-001-09).
- `[SME]` Are there reasons a discrepancy entry should escalate to a hard fail vs. soft (BR-001-07/08)?
- `[CODMOD]` Full enumeration of every program that writes to `ACME.ITEM_*` tables (not just the anchor set).
- `[RAG]` Full call chain of `XXCAD65` — used directly or only via `MCCAD65J`?

---

## 8. Modernization Notes

- **Blast radius (graph):** `ACME.DIVMSTRDI1D` (124 programs), `ACME.UIN_ITEM_DE6C` (64), `ACME.DIV_ITEM_PACK_DE1I` (62), `ACME.ITEM_VNDR_DE6V` (35), `ACME.ITEM_UPC_DE6Y` (30). Any DAL change to these entities touches a large fraction of the platform — introduce a facade early.
- **Preservation constraints:**
  - The basis-code mapping (BR-001-04..06) is canonical and trivially testable; encode as a pure function early.
  - File-status `'23'`/`'10'` → `RETURN-CODE = 16` (BR-001-10) is the platform abend convention; preserve operationally.
- **Suggested cutover unit:** This BP can cut over **per division**, because `<DIV>.MSTR.SIM` is per-division. Start with a small division (e.g., `WJ` or `FE`) to validate before scaling.
