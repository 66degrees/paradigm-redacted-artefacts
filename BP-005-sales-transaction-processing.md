# BP-005 — Sales Transaction Processing

**Status:** Draft — partial corpus evidence; requires SME follow-up
**Owners:** Billing / AP team, Merchandising IT
**Anchor programs:** `XXDM713` (Deal Sales Update) and the upstream invoice (`ACME.INVC_*`) and sales-consolidation (`ACME.PERM.BSM30S1.DWSLS`) producers
**Related SRS:** FR-SALES-001 (and FR-DEAL-006)

---

## 1. Overview

The Sales Transaction Processing BP feeds **invoice and sales activity** into the deal lifecycle so that accruals, billing, and reporting reflect what actually moved through stores. Round-2 retrieval established that this BP has **two distinct data planes**, not one:

- **The invoice plane (`ACME.INVC_*` family)** — `XXDM713` (Deal Sales Update) reads invoice headers, details (common + item), and timestamps from this family and updates deal-side sales totals.
- **The sales-consolidation plane (`ACME.PERM.BSM30S1.DWSLS`)** — described in the corpus as the central dataset that "plays a critical role in processing and consolidation of sales transaction data." Its producers and consumers within this BP need full enumeration (`[CODMOD]`).

> **Revision:** The first draft of this spec assumed `XXDM713` consumed `ACME.PERM.BSM30S1.DWSLS` directly. The dependency graph shows `XXDM713` reads the `ACME.INVC_*` family (Bill Detail / `BD*` table family) instead. The corpus has no narrative description for `XXDM713` itself (file is empty / boilerplate-only) — its behaviour is inferred from its data references and from its role declared in `index.md`. Mark this section `[CORPUS-EMPTY-FOR-XXDM713]` until the source is enriched or an SME walkthrough is recorded.

---

## 2. Programs Involved

| Type | Name | Purpose |
|---|---|---|
| COBOL | `XXDM713` | **Deal Sales Update** — reads `ACME.INVC_*` invoice family + customer / division / tax masters; updates per-deal sales totals. Corpus narrative is empty; behaviour inferred from graph. |
| DB2 cluster | `ACME.INVC_HDR_BD1H`, `ACME.INVC_DTL_COMN_BD1D`, `ACME.INVC_DTL_ITEM_BD2D`, `ACME.INVC_TS_BD2T` | **Invoice plane** — header, common detail, item detail, timestamp. The `BD*` suffix indicates Bill Detail. |
| DB2 read | `ACME.CUST_XREF_CU1X`, `ACME.DIVMSTRDI1D`, `ACME.US_TAX_CU4U`, `DATECNTLCF1D` | Reference data used during sales-to-deal application by `XXDM713`. |
| Dataset | `ACME.PERM.BSM30S1.DWSLS` | **Sales-consolidation plane** — central dataset for sales consolidation. Producers and consumers within this BP need full enumeration (`[CODMOD]`). |
| `[CODMOD]` | (additional sales-consolidation programs) | Enumerate every program reading/writing `ACME.PERM.BSM30S1.DWSLS` from the graph. |
| Downstream | Billing / AP feeds | Consumers of the consolidated dataset and of the deal-sales totals updated by `XXDM713`. |

---

## 3. Business Rules

| ID | Rule | Source / Status |
|---|---|---|
| BR-005-01 | Sales transactions are aggregated into `ACME.PERM.BSM30S1.DWSLS`. | `acme.perm.bsm30s1.dwsls.md` |
| BR-005-02 | `XXDM713` applies invoice-derived sales activity onto deal records by joining `ACME.INVC_HDR_BD1H` + `ACME.INVC_DTL_COMN_BD1D` + `ACME.INVC_DTL_ITEM_BD2D` with `ACME.CUST_XREF_CU1X` and `ACME.DIVMSTRDI1D`. | graph (`XXDM713` callees) |
| BR-005-03 | Suppressed deals (BP-002 BR-002-31) MUST be excluded from deal-sales updates. | `[SME]` confirm |
| BR-005-04 | Sales-window timing is driven by `ACME.INVC_TS_BD2T` (invoice timestamp) and `DATECNTLCF1D` (date control); the operational rule for which invoices apply to which deal period needs SME confirmation. | derived |
| BR-005-05 | US tax handling uses `ACME.US_TAX_CU4U` — confirm whether tax amount is included in or excluded from the per-deal sales totals. | `[SME]` confirm |
| BR-005-06 | Reprocessing the same input window MUST be idempotent — no double-application of sales onto deals. | `[SME]` confirm |
| BR-005-07 | `[CODMOD]` Enumerate every producer that writes to `ACME.PERM.BSM30S1.DWSLS` (POS, e-commerce, replenishment) and every consumer beyond `XXDM713` / `MCDLS50J`. |  |
| BR-005-08 | `[CORPUS-EMPTY]` `xxdm713.md` carries no business-level description. The rules above are the **inferred** behavioural contract; they are not verbatim from a corpus document. |  |
| BR-005-09 | **Confirmed DWSLS consumer:** `MCDLS50J` reads `ACME.PERM.BSM30S1.DWSLS` via its `SORTSUM` step (a sort-summary aggregation). The downstream `XXDLS50` program uses the aggregated output to identify **profitable deals** using **future expiration dates, unit deal amounts, and specific deal types**. | round 4 (`mcdls50j.md`) |
| BR-005-10 | `MCDLS50J` also reads divisional `&DI2..MSTR.SIM` indexed files for `BASE-FORECAST` data, uses `DS.PERM.SQLBATCH(SQLINFO)` for DB2 batch, and uses `DS.PERM.SORTPARM(DLS50SRT)` for its sort step. | round 4 (`mcdls50j.md`) |
| BR-005-11 | **DWSLS producer chain confirmed (round 5–6, CAST):** `MCBSM50J` (JCL job, CC=274) orchestrates **`XXBSM30P`→`XXBSM30`** (writes `ACME.PERM.BSM30S1.DWITM`) and **`XXBSM31P`→`XXBSM31`** (touches `ACME.PERM.BSM31S1.DWSLS.SRT`) before **`XXBSM32P`→`XXBSM32`** → paragraph `5000-WRITE-DATA` writes `ACME.PERM.BSM30S1.DWSLS`. `MCBSM50J` also contains paragraphs `2100-ITM-DWSLS-CHECK` and `2200-DWSLS-FILE-READ` that read/compare DWSLS. | round 5–6 (CAST) |

---

## 4. Data Structures

### 4.1 Invoice plane (`ACME.INVC_*` — `BD*` Bill-Detail family)

| Entity | Role |
|---|---|
| `ACME.INVC_HDR_BD1H` | Invoice header. |
| `ACME.INVC_DTL_COMN_BD1D` | Invoice detail — common attributes. |
| `ACME.INVC_DTL_ITEM_BD2D` | Invoice detail — item attributes. |
| `ACME.INVC_TS_BD2T` | Invoice timestamp. |
| `DATECNTLCF1D` | Date control. |
| `ACME.US_TAX_CU4U` | US tax. |

### 4.2 Sales-consolidation plane

| Entity | Role |
|---|---|
| `ACME.PERM.BSM30S1.DWSLS` | Central sales-consolidation dataset. |
| `[CODMOD]` upstream feeds | Per-source raw sales files. |

### 4.3 Working-storage anchors

`[CODMOD]` — to be extracted from `XXDM713` source (corpus is empty for this program).

---

## 5. Sequence Diagrams

### 5.1 Invoice-driven deal-sales update (`XXDM713`)

```
[Billing / invoice producers] ──► ACME.INVC_HDR_BD1H ─┐
                                  ACME.INVC_DTL_COMN_BD1D
                                  ACME.INVC_DTL_ITEM_BD2D
                                  ACME.INVC_TS_BD2T
                                                     │
                                                     ▼
                                                  XXDM713 ── join with ── ACME.CUST_XREF_CU1X
                                                     │                    ACME.DIVMSTRDI1D
                                                     │                    ACME.US_TAX_CU4U
                                                     │                    DATECNTLCF1D
                                                     ▼
                                              Per-deal sales totals updated
                                                     │
                                                     ▼
                                       BP-002 accrual / billing pipelines
```

### 5.2 Sales-consolidation plane — producer + consumer chain (round 5 closed)

```
Upstream sales sources ──► [SME] external POS / e-commerce feeds
                                         │
                                         ▼
                                MCBSM50J (JCL)
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
                    XXBSM30P/30    XXBSM31P/31    XXBSM32P/32
                    (DWITM write)  (DWSLS.SRT)    (DWSLS write)
                          │              │              │
                          └──────────────┴──────────────┘
                                         │
                                         ▼
                                XXBSM32P (JCL procedure)
                                         │
                                         ▼
                                XXBSM32 (Cobol Batch Program)
                                         │
                                         ▼
                                paragraph 5000-WRITE-DATA
                                         │
                                         ▼  WRITE
                                ACME.PERM.BSM30S1.DWSLS
                                         │
                                         ▼  SORTSUM step
                                MCDLS50J (JCL)
                                         │
                                         ▼
                                XXDLS50 (deal profitability analysis)
                                         │
                                         ▼
                          Downstream Billing / BI consumers
```

The producer-consumer chain is fully closed for `ACME.PERM.BSM30S1.DWSLS` as of round 5. The only remaining gap is enumerating the **upstream sources** that feed into `XXBSM32` — `[CODMOD]` from CAST `Objects↔DataSources` filtered on `ObjectName='XXBSM32'`.

---

## 6. Test Cases

| ID | Scenario | Expected | Maps to |
|---|---|---|---|
| TC-005-01 | Consolidation completeness | Every upstream sale appears exactly once in `ACME.PERM.BSM30S1.DWSLS` | BR-005-01 |
| TC-005-02 | Sales applied to deal | Per-deal sales totals updated by `XXDM713` | BR-005-02 |
| TC-005-03 | Suppressed deal not updated | `XXDM713` skips deals flagged suppressed by `XXDLS01` | BR-005-03 |
| TC-005-04 | Idempotent reprocessing | Re-running window does not double-apply | BR-005-05 |
| TC-005-05 | Per-division split | Per-division consumers see only their division's records | BR-005-07 |

---

## 7. Open Questions

- `[CORPUS-EMPTY]` `xxdm713.md` has no business-level description. Either re-run codmod against the source to enrich the markdown, or capture an SME walkthrough and re-ingest it.
- `[CODMOD]` Enumerate **every** program reading or writing `ACME.PERM.BSM30S1.DWSLS`. This BP's blast radius cannot be assessed without that list.
- `[SME]` Confirm whether `XXDM713` is the only program that applies invoice-derived sales to deal totals, or whether other programs in the `XXDM*` family do the same.
- `[SME]` Tax handling — is `ACME.US_TAX_CU4U` included in or excluded from per-deal sales totals (BR-005-05)?
- `[SME]` Origin of the upstream sales feeds for `ACME.PERM.BSM30S1.DWSLS`.
- `[SME]` Cadence — daily? intraday? real-time?
- `[SME]` Reconciliation procedure if downstream Billing / AP detects a gap.

---

## 8. Modernization Notes

- **Blast radius:** Unknown until `[CODMOD]` enumeration is complete. Likely large given the dataset's described centrality.
- **Preservation:**
  - The link from this BP into BP-002's lifecycle (sales → accrual → billing) is the platform's revenue-recognition path; coexistence design must guarantee no in-flight sales are lost or double-counted at cutover.
  - Idempotency (BR-005-05) is the single most important property to validate during modernization.
- **Suggested cutover unit:** Out-of-the-box dual-write to legacy and modernized consolidation for a representative period; reconcile before flipping reads.
