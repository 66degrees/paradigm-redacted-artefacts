# BP-002 — Deal Management & Analysis (Lifecycle)

**Status:** Draft — pending SME review
**Owners:** Deal Management product team, Merchandising IT
**Anchor programs:** `XXDLS01`, `XXDL702`, `XXDL960`, `XXDM713`, `XXDLC10`, `XXDLC20`, `XXDLS50`, `XXDLS54`, `MCDLS17`, `MCBSM04`, `MCBSM02`, `MCDL656J`
**Related SRS:** FR-DEAL-001 … FR-DEAL-010
**Related HLD:** §5.1 (deal entities), §6 (top connected programs), §9 (hot-spots HS-001/HS-003/HS-008)

---

## 1. Overview

This is the **lifecycle-bearing** business process of the platform. Deals progress through:

```
pending  ──►  capture / promote  ──►  active billing  ──►  accruals  ──►  aggregation  ──►  archival
   │                  │                       │                  │                │              │
   │                  │                       │                  │                │              │
ACME.PENDING            D8050               XXDL702            XXDLC20          XXDLC10        XXDL960
DEALSDM3P          (CICS polling)        (active)           (accrual)      (aggregation     (purge >2y
+ MCCBT07          → DM1T (Deal           rolls ACEDL3      stages temp     → OUTDEAL)      from DEALDM1X)
(cigarette         Transaction)           into OUT-DEAL-    accrual table
batch)             + DM1E (restart)       CEDED-AMT
                   + DM3E (errors)
```

with **suppression** (`XXDLS01`) layered on top to gate which deals participate in each downstream step, and **reporting** (`XXDL740` Deal Statistics, `XXDL571` Deal Alerts, `XXDLS50` BI, `XXDLS54` comparison, `MCDLS17` change details) consuming the lifecycle state.

Because group-deal pipelines (`MCBSM02`, `MCBSM04`, `XXBSM*`, `XXDLS51`, `XXDLS52`) feed into the lifecycle at multiple points, this BP also covers **group/mass deal maintenance** and the **customer-side processing** orchestrators (`MCBSM02` / `MCBSM04`) that the BSM cluster depends on.

> **Important:** `D8050` was previously documented in BP-006 (CICS utilities). Round-2 retrieval shows it is in fact the **CICS polling transaction that captures pending deals from `DM3P` into the `DM1T` deal-transaction table**. It is the pending→active bridge of the lifecycle and lives in this BP.

---

## 2. Programs Involved

| Type | Name | Purpose |
|---|---|---|
| **CICS txn** | **`D8050`** | **Pending-deal capture polling transaction.** Polls `ACME.PENDINGDEALSDM3P`, validates and enriches, inserts into `DM1T` (Deal Transaction). Logs errors to `DM3E`, restart in `DM1E`. Calls `CM510ZO`. |
| COBOL | `XXDLS01` | **Deal Suppression Module** — gates whether a deal proceeds. Heart of the BP. |
| COBOL | `XXDL702` | Active billing-deal handler. Bypasses `group`/`DVRTR`/`BRKT` types; manages `ACEDL3` rolling into current deal fields and `OUT-DEAL-CEDED-AMT`. |
| COBOL | `XXDLC10` | **Deal Transaction Aggregation and Output.** Reads audit transactions, joins `DEALDM1X` + master data, writes aggregated `OUTDEAL` file. Paragraphs `A0000-INITIALIZATION` → `B1000-PROCESSING` (loop until `SW-EOF-OF-TABLE = 'Y'`) → `S1000-FINALIZATION`. `E1000-FAILED-OPEN` handler. |
| COBOL | `XXDLC20` | **Deal Accrual Processing.** Reads "end-of-processing date" from input file; stages intermediate accruals in a temp table; aggregates `DEALDM1X` by deal part / item number / deal ID. |
| COBOL | `XXDM713` | **Deal Sales Update.** Invoice-driven (reads `ACME.INVC_*` family — see BP-005 for the cross-reference). Updates per-deal sales totals. Corpus description currently empty. |
| COBOL | `XXDL960` | Data Archival for Deal Records (`DEALDM1X`). |
| COBOL | `XXDL740` | **Deal Statistics Update Report + Error Messages Report.** VSAM-config-driven orchestrator; calls `DATETIME`, `UT516XP` (fiscal periods), `DC502YP` (date format), `XXDC608` (division ↔ common layout), `DSNTIAR` (SQL err), `ILBOABN0` (force abend). Exclusive locks on `DEAL_ANALYSIS_DM1L`, `DEALDM1X`, `DEALLOGDM5X` with wait-and-retry. |
| COBOL | `XXDLS50` | Deal Analysis for BI Reporting. |
| COBOL | `XXDLS54` | Deal Comparison Report Generator. |
| COBOL | `XXDLS59` | Batch Job Status Checker and Updater. |
| COBOL | `XXDLS60` / `XXDLS60P` | Data purge utility for deal headers. |
| COBOL | `XXDL571` | Deal Alert Report Generation. |
| COBOL | `XXDL655`, `XXDL656`, `XXDL660` | Deal-side companions to the JCL orchestrators below. |
| COBOL | `XXDLV01` | Customer Delivery Window maintenance. |
| COBOL | `XXBSM70`, `XXBSM71` | Group-deal / business-moves processors. |
| COBOL | `XXDLS51`, `XXDLS52` | Mass Maintenance for Group Deals. |
| JCL job | `MCDLS17` | Fetch Deal Details for Deal Change. |
| JCL job | `MCDLS50J` | Processes deal data, references `SZ.MSTR.SIM`. |
| JCL job | `MCDLS54J` | Processes deal-related data and manages migrated information. |
| JCL job | `MCDL656J` | Per-division enrichment fan-out (see BP-001 §5.2). |
| JCL job | `MCDLS60J` | Executes `XXDLS60P` for data processing. |
| JCL job | `MCBSM70J`, `MCBSM72J` | Group-deal extract and processing. Also uses REXX `FTEREPL` via `IKJEFT01`. |
| **JCL job** | **`MCBSM01J`** | **CAST CC=486 (rank 2 in platform).** LOC=14,552, ObjectCount=645. Master BSM job; behaviour `[RAG]`. |
| **JCL job** | **`XXDLSDLY`** | **Daily deal-cycle.** CC=375, LOC=11,646, ObjectCount=344. CAST-named; assessment markdown `[CORPUS-EMPTY]` for business steps (round 6). |
| **JCL job** | **`XXDLSWKY`** | **Weekly deal-cycle** (CC=414, LOC=14,693, ObjectCount=515). **Also the purge/reporting parent** (round 6 CAST): orchestrates `XXDL530P` → `XXDL570P` → `XXDL571P` → `XXDL995P` → `XXDL960P` → `XXDL980P`. Corpus still references normalised `__name.md` / `%%NAME`. |
| **JCL job** | **`XXDLSEOP`** | **End-of-Period deal-cycle.** CC=343, LOC=12,594, ObjectCount=485. Per `mcdl656j.md`: processes historical deal data; **explicitly excludes current period and current year**; calls `XXDL656P` running `XXDL656` per division; outputs categorized into 3 historical buckets ("previous period / current fiscal year", "previous periods / current fiscal year", "previous period / previous/current year"); sorted/aggregated by buyer and vendor; cleanup via `IEFBR14`. |
| **JCL proc** | **`XXDL702P`** | Procedure wrapper for active billing (`XXDL702`). CC=110 — referenced in CAST round 6. |
| **CICS** | **`D8050`** | Already covered in §2 above as the pending-deal capture polling transaction. |
| **JCL job** | **`XXDL745J`** | CC=334, LOC=10,083, ObjectCount=404. `[RAG]` to detail. |
| **JCL job** | **`XXDL650J`** | CC=247, LOC=8,287, ObjectCount=384. `[RAG]` to detail. |
| **JCL job** | **`XXDL664J`** | CC=264, LOC=8,123, ObjectCount=328. `[RAG]` to detail. |
| **JCL job** | **`XXDL670J`** | CC=101, LOC=4,928, ObjectCount=195. `[RAG]` to detail. |
| **JCL job** | **`XXD2137J`** | CC=416 (rank 3 in platform), LOC=8,785, ObjectCount=218. Likely deal-related given `D2137` naming. `[RAG]` to confirm. |
| **JCL job** | **`XXDLS02J`, `XXDLS06J`, `XXDLS07J`** | Deal-services batches (XXDLS07J CC=80). `[RAG]` to detail. |
| **JCL proc** | **`XXDL702P`, `XXDL701P`, `XXDLV01P`** | CC=110 / 84 / 63. Reusable procedures invoked from multiple jobs. |
| COBOL | `MCBSM04` | **Customer / SA_CORP processing orchestrator.** Paragraphs `0000-MAIN → 1000-INIT → 1500-PROCESS-CUSTS → 3000-SET-SA-CORPS → 8000-FINALIZE`. Touches 30 customer / classification / division-customer DB2 tables. JCL-launched leaf (no COBOL callers). |
| COBOL | `MCBSM02` | **Customer deletion-candidate selector.** Loops `2100-MOVE-CUSTOMERS` until `WS-EOF = 'Y'`. Excludes user IDs `'MCBSM02'`, `'XXEBM39'`. Cursor threshold `CREATE_TS` older than **45 DAYS**. DB2 errors → `DBDB2ER` + `DSWTO`; critical → `ROLLBACK WORK` + `RETURN-CODE = 16`. Reports to `PRINT-FILE`. |
| COBOL | `MCBSM03`, `MCBSM30` | Companion BSM programs (top-connected; deeper rules `[RAG]` next round). |
| JCL pipeline | `MCCBT06` → `MCCBT07` | Cigarette-deal batch (see BP-004). MCCBT07 inserts/updates into `ACME.PENDINGDEALSDM3P`, feeding this BP's lifecycle. |

---

## 3. Business Rules

### 3.1 From `XXDLS01` — Deal Suppression

| ID | Rule | Source |
|---|---|---|
| BR-002-01 | `DLS01-DEAL-SUPP-SW` initialised to `'N'`. | `XXDLS01` |
| BR-002-02 | Reset per-call caches (`WS-CUST-TABLE`, `WS-ITEM-TABLE`) when `WS-FIRST-SW = 'Y'` OR division code changes OR division part changes; then set `WS-FIRST-SW = 'N'`. | `XXDLS01` |
| BR-002-03 | Customer suppression check (when `WS-CUST-SW(WS00-CUST-NUM)` ≠ `'Y'`) queries `ACME.PROF_HDR_PR1P`, `ACME.PROF_CUS_PR3Q`, `ACME.DIVMSTRDI1D`, `ACME.CUST_XREF_CU1X`. | `XXDLS01` |
| BR-002-04 | Item suppression check (when customer not suppressed AND `WS-ITEM-SW(WS00-ITEM-NUM)` ≠ `'Y'`) queries `ACME.ITEM_SUPP_IT2A`, `ACME.ITEM_MASTER_IM3I`. | `XXDLS01` |
| BR-002-05 | Deal-specific suppression (when neither customer nor item suppressed) queries `ACME.PROF_HDR_PR1P`, `ACME.PROF_DEAL_PR4D`, `ACME.DEAL_SUPP_DS1R` keyed by **deal type**, **form of payment**, **invoice date**. | `XXDLS01` |
| BR-002-06 | DB2 errors during these checks map to documented return codes (do not silently swallow). | `XXDLS01` |
| BR-002-07 | Final returned switch `DLS01-DEAL-SUPP-SW ∈ {'Y','N'}`. | `XXDLS01` |

### 3.2 From `XXDL702` — Active Billing Deals

| ID | Rule | Source |
|---|---|---|
| BR-002-10 | Bypass deal types `group`, `DVRTR`, `BRKT`. | `XXDL702` |
| BR-002-11 | An active deal is treated as expired when current date > deal end. | `XXDL702` |
| BR-002-12 | For a non-`'Y'` type deal with non-zero `ACEDL3`: copy deal start/end into current-deal fields, add `ACEDL3` into `OUT-DEAL-CEDED-AMT`, mark a rewrite. | `XXDL702` |
| BR-002-13 | Increment counters, update output record fields, and write `OUT-RCD` only when modifications were made. | `XXDL702` |
| BR-002-14 | Blank date fields are initialised before the next iteration. | `XXDL702` |
| BR-002-15 | If no deals were processed, the program exits cleanly. | `XXDL702` |

### 3.3 From `XXDL960` — Deal Archival

| ID | Rule | Source |
|---|---|---|
| BR-002-20 | A row in `DEALDM1X` is eligible for deletion when older than **2 years**. | `XXDL960` |
| BR-002-21 | "Older than 2 years" = at least one of `DLBUYH` (purchase date), `DLINVH` (invoice date), `DLSHPH` (shipment date) is ≤ (today − 2y). `[SME]` confirm AND vs OR semantics. | `XXDL960` |
| BR-002-22 | Eligible rows are retrieved via a cursor and evaluated row-by-row (no bulk delete). | `XXDL960` |
| BR-002-23 | Archival is idempotent — re-running on an already-purged window MUST be a no-op. | `[SME]` confirm |

### 3.4 From `D8050` — Pending-Deal Capture (CICS polling)

| ID | Rule | Source |
|---|---|---|
| BR-002-40 | Periodically polls `ACME.PENDINGDEALSDM3P` (`DM3P`) for new pending deals. | `D8050` |
| BR-002-41 | Branches by deal type — divisional pending deals fetched from `DM3D` (`ACME.DIVPENDDEALSDM3D`); group pending deals fetched from `DM3G` (`ACME.GRPPENDDEALDM3G`). | `D8050` |
| BR-002-42 | **Critical edits performed before promotion to `DM1T`:** (a) overlapping deal date check, (b) item-status check, (c) division check, (d) licensing-limit check, (e) corporate-vendor-control check, (f) existing-terminated-deals check. | `D8050` |
| BR-002-43 | Validation failures are written to the **CAD Error Table `DM3E` (`ACME.CAD_ERR_LOG_DM3E`)** with full diagnostic context. | `D8050` |
| BR-002-44 | Successful promotions are inserted into the **Deal Transaction table `DM1T`** (the system-of-record handoff out of "pending"). | `D8050` |
| BR-002-45 | Restart records are written to `DM1E` with a unique sequence number `DM1E-SEQNCE`; restart record contains program ID, division code, timestamp, CICS error info, SQL error info. | `D8050` |
| BR-002-46 | On a restart, the program reads `DM1E` to determine where to resume. | `[SME]` confirm restart protocol |
| BR-002-47 | Calls `CM510ZO` during initialization to load processing-data structures. | `D8050` |

### 3.5 From `XXDL740` — Deal Statistics Update Report

| ID | Rule | Source |
|---|---|---|
| BR-002-50 | Dynamic configuration is read from input VSAM files; subroutine selection and output formatting are driven by these configs, not by code constants. | `XXDL740` |
| BR-002-51 | Calls `DATETIME` for system date/time, `UT516XP` for fiscal period boundaries, `DC502YP` for date format conversion, `XXDC608` for division ↔ common layout transformation, `DSNTIAR` for SQL-code translation. | `XXDL740` |
| BR-002-52 | Exclusive locks on `DEAL_ANALYSIS_DM1L`, `DEALDM1X`, `DEALLOGDM5X` — must wait-and-retry if another process holds the lock. | `XXDL740` |
| BR-002-53 | On fatal error, `ILBOABN0` is invoked to force an abend with system dump. | `XXDL740` |
| BR-002-54 | Random reads: `AP-FILE` keyed by vendor number; `BY-FILE` keyed by buyer key; `SI-FILE` dynamic. | `XXDL740` |

### 3.6 From `XXDLC10` — Deal Transaction Aggregation

| ID | Rule | Source |
|---|---|---|
| BR-002-60 | Reads audit-transaction input file; loops `B1000-PROCESSING` until `SW-EOF-OF-TABLE = 'Y'`. | `XXDLC10` |
| BR-002-61 | Stages audit data in a temp area populated with: division, transaction type, item, deal identifier, quantity, amount, date. | `XXDLC10` |
| BR-002-62 | Executes a complex aggregating join against `DEALDM1X` and master data; calculates amounts per transaction type; constructs a unique item key. | `XXDLC10` |
| BR-002-63 | On a file-open failure, displays `FAILED-FILE` + `FAILED-STATUS` via `E1000-FAILED-OPEN`. | `XXDLC10` |
| BR-002-64 | **Inconsistency:** `MOVE +16 TO RETURN-CODE` in `E1000-FAILED-OPEN` is **commented out**, so a failed open currently does not return a non-zero RC. `[SME]` confirm whether this is intentional. | `XXDLC10` |
| BR-002-65 | `S1000-FINALIZATION` displays total `OUTDEAL` records written + total `DEALDM1X` records fetched + completion timestamp + division. | `XXDLC10` |

### 3.7 From `XXDLC20` — Deal Accrual Processing

| ID | Rule | Source |
|---|---|---|
| BR-002-70 | Reads an "end-of-processing date" from an input file — drives the filter window for accruals. | `XXDLC20` |
| BR-002-71 | Creates a temporary DB2 table for intermediate accrual data. | `XXDLC20` |
| BR-002-72 | Aggregates by **deal part**, **item number**, and **deal ID**. | `XXDLC20` |

### 3.8 From `MCBSM02` — Customer Deletion-Candidate Selection

| ID | Rule | Source |
|---|---|---|
| BR-002-80 | Excludes records authored by user IDs `'MCBSM02'` and `'XXEBM39'` from the deletion-candidate set. | `MCBSM02` |
| BR-002-81 | Cursor selects records where `CREATE_TS` is older than **45 DAYS**. | `MCBSM02` |
| BR-002-82 | Main loop runs until `WS-EOF = 'Y'`. | `MCBSM02` |
| BR-002-83 | DB2 errors logged via `DBDB2ER` + `DSWTO`; critical errors → `ROLLBACK WORK` + `RETURN-CODE = 16`. | `MCBSM02` |
| BR-002-84 | Output is a `PRINT-FILE` report: program name, run timestamp, salesman ID, from/to division, create timestamp, table-affected flags. | `MCBSM02` |

### 3.8b Daily deal-reporting + purge cycle (parent JCL — name `[SME]`)

The 6-step parent JCL job orchestrating the daily reporting + purge cycle:

| Step | Program | Function | Tables / Datasets |
|---|---|---|---|
| 1 | `XXDL530P` | Report on deals with upcoming / recent buy/ship dates | `DEALDM1X` |
| 2 | `XXDL570P` | Deal performance report (current fiscal year) | `DEALDM1X` |
| 3 | `XXDL571P` | **Deal Alert Report.** Reads processing date from `RDR1`; calculates ±7-day range; categorises deals as active / future / inactive / terminated; produces alert report. | `DEALDM1X`, `STARITEMCATST1I`, `ITEMAUTHST1A`, `CUST_CLS_GRP_CU2E`, `DIVMSTRDI1D`, `ACME.DIV_ITEM_PACK_DE1I`, `ACME.UIN_ITEM_DE6C` |
| 4 | `XXDL995P` | **Purge `DEALLOGDM5X` records older than 5 days.** Uses `FCURRF` timestamp for selection. | `ACME.DIVMSTRDI1D`, `DEALLOGDM5X` |
| 5 | `XXDL960P` | Archive `DEALDM1X` records older than 2 years (already covered by BR-002-20..23 / FR-DEAL-010). | `DEALDM1X` |
| 6 | `XXDL980P` | **Purge `DEALAUDITDM1A` records.** Retention period configurable via `DS.APPL_SYS_PARM_AP1S.XXDL980_DM1A_DAYS` parameter (**default 60 days** if not found). | `ACME.DIVMSTRDI1D`, `DS.APPL_SYS_PARM_AP1S`, `DEALAUDITDM1A` |

The parent JCL job's actual name is not in the corpus (CAST normalised it to `__name.md`); `[SME]` to recover. New rules:

| ID | Rule | Source |
|---|---|---|
| BR-002-90 | `XXDL995P` purge threshold = **5 days** from current date, using `FCURRF` timestamp. | Round 5 (`__name.md`) |
| BR-002-91 | `XXDL980P` retention = `DS.APPL_SYS_PARM_AP1S.XXDL980_DM1A_DAYS` parameter, default **60 days**. Demonstrates the same parameterised-checkpoint pattern as `MCCST24_LAST_RUN`. | Round 5 (`__name.md`) |
| BR-002-92 | `XXDL571P` deal categorisation states: **active / future / inactive / terminated**. Computed against `DEALDM1X` joined with item/division/customer-class masters using a ±7-day window around the processing date from `RDR1`. | Round 5 (`__name.md`) |

### 3.9 Lifecycle invariants (cross-program)

| ID | Rule | Source |
|---|---|---|
| BR-002-30 | A deal not present in `ACME.PENDINGDEALSDM3P` cannot become an active billing deal. | `[SME]` confirm |
| BR-002-31 | A suppressed deal (output `'Y'` from `XXDLS01`) MUST NOT contribute to accruals (`XXDLC20`) or sales updates (`XXDM713`). | `[SME]` confirm |
| BR-002-32 | A deal removed by `XXDL960` MUST already be reflected in downstream BI exports (i.e. archive after report cycle). | `[SME]` confirm |

---

## 4. Data Structures

### 4.1 DB2 entities

| Entity | Role | Programs |
|---|---|---:|
| `DEALAUDITDM1A` | **Deal audit table — purged by `XXDL980P`** (retention via `XXDL980_DM1A_DAYS` param, default 60d). Note: round-5 finding shows `XXDL571P` does NOT feed `DEALAUDITDM1A` (corrects an earlier inference); XXDL571P reads `DEALDM1X` + masters and produces the Deal Alert Report. | (purge chain) |
| `STARITEMCATST1I` | STAR item category — read by `XXDL571P` for deal classification (active/future/inactive/terminated). | round 5 |
| `ITEMAUTHST1A` | Item authorisation — read by `XXDL571P` and by `MCCST24` (BP-004). | round 5 |
| `ACME.PENDINGDEALSDM3P` (`DM3P`) | Pending deals capture (polled by `D8050`) | 31 |
| `ACME.DIVPENDDEALSDM3D` (`DM3D`) | Divisional pending-deal pointer | (cluster) |
| `ACME.GRPPENDDEALDM3G` (`DM3G`) | Group pending-deal pointer | (cluster) |
| `DM1T` | **Deal Transaction table** — system of record after `D8050` promotes from `DM3P` | (cluster) |
| `DM1E` | **Restart file** for `D8050` recovery (`DM1E-SEQNCE` sequence key) | `D8050` |
| `ACME.CAD_ERR_LOG_DM3E` (`DM3E`) | **CAD Error Table** populated by `D8050` validation failures | `D8050` |
| `DEALDM1X` | Deal data mart (archival target) | 40 |
| `DEAL_ANALYSIS_DM1L` | Deal analysis (exclusive-lock target of `XXDL740`) | `XXDL740` |
| `DEALLOGDM5X` | Deal log (exclusive-lock target of `XXDL740`) | `XXDL740` |
| `ACME.PROF_HDR_PR1P` | Profile header (suppression cluster) | (cluster) |
| `ACME.PROF_CUS_PR3Q` | Customer profile (suppression cluster) | (cluster) |
| `ACME.PROF_DEAL_PR4D` | Deal profile (suppression cluster) | (cluster) |
| `ACME.DEAL_SUPP_DS1R` | Deal suppression rules | (cluster) |
| `ACME.ITEM_SUPP_IT2A` | Item suppression | (cluster) |
| `ACME.ITEM_MASTER_IM3I` | Item master (read for suppression) | (cluster) |
| `ACME.CUST_XREF_CU1X` | Customer cross-reference | (cluster) |
| `ACME.DIVMSTRDI1D` | Division master (read) | 124 |
| `ACME.MCLANE_XREF_DI3X` | Vendor cross-reference (read by `XXDL740` via `DI3X-CUR1`) | `XXDL740` |
| `ACME.DT_JS1A` | Date control (read by `XXDLC10`/`XXDLC20`) | `XXDLC10/20` |
| **Customer cluster** (touched by MCBSM02/04) | `ACME.CUST_MSTR_CU1A`, `ACME.CUST_CLS_CU2A`, `ACME.CUST_CLS_GRP_CU2E`, `ACME.DIV_CUST_CU1Q`, `ACME.DIV_CUST_TYP_CU2I`, `ACME.DIVCUSTAPPRVL_CU4Q`, `ACME.US_TAX_CU4U`, `ACME.UNIT_CNTRCT_CU2U`, `ACME.CUST_LICNS_CU1K`, `ACME.CUST_LOAD_RTG_SO1L`, `ACME.RTG_CU2L`, `ACME.CUST_CNTCT_CU2C`, `ACME.CNTCT_ID_AT2B`, `ACME.CD_VAL_CU4V/6V`, `ACME.DT_VAL_CU5D/6D`, `ACME.SW_VAL_CU5W/6W/4W`, `ACME.TXT_VAL_CU5T/6T`, `ACME.INTGR_VAL_CU6I`, `ACME.TIME_VAL_CU5M/6M`, `ACME.ENTITYCLS_GRP_CM1E`, `ACME.ITEMAUTHST1A` | MCBSM02/04 |

### 4.2 Working-storage anchors

`DLS01-DEAL-SUPP-SW`, `WS-FIRST-SW`, `WS-CUST-SW(*)`, `WS-ITEM-SW(*)`, `WS-CUST-TABLE`, `WS-ITEM-TABLE`, `WS00-CUST-NUM`, `WS00-ITEM-NUM`, `ACEDL3`, `OUT-DEAL-CEDED-AMT`, `OUT-RCD`, `DLBUYH`, `DLINVH`, `DLSHPH`.

### 4.3 Sequence-bearing dataset families

`<DIV>.PERM.<DIV>DL656{1,2,3}` extracts produced by `MCDL656J` (per-division output channels via `iefbr02` / `iefbr03` placeholders).

---

## 5. Sequence Diagrams

### 5.1 Suppression decision (`XXDLS01`)

```
Caller            XXDLS01                 DB2
  │ call(deal,cust,item) ──►│
  │                         │── init DLS01-DEAL-SUPP-SW='N'
  │                         │── if WS-FIRST-SW='Y' or div changes → reset caches
  │                         │── query ACME.PROF_HDR_PR1P  ──────────►│
  │                         │── query ACME.PROF_CUS_PR3Q  ──────────►│
  │                         │── query ACME.DIVMSTRDI1D    ──────────►│
  │                         │── query ACME.CUST_XREF_CU1X ──────────►│
  │                         │── (if cust ok) query ACME.ITEM_SUPP_IT2A
  │                         │                      ACME.ITEM_MASTER_IM3I
  │                         │── (if cust+item ok) query ACME.PROF_DEAL_PR4D
  │                         │                          ACME.DEAL_SUPP_DS1R
  │                         │── on DB2 error → set documented RC
  │ ◄── DLS01-DEAL-SUPP-SW──│
```

### 5.2 Full lifecycle

```
Buyer/CICS ───► ACME.PENDINGDEALSDM3P ───► XXDL702 (active) ───► XXDLC20 (accrual) ───► XXDM713 (sales)
                                              │                       │
                                              └──► XXDLS01 (gate) ◄───┘
                                                          │
                                                          ▼
                                                 XXDLS50 / XXDLS54 / MCDLS17 (BI / change reporting)
                                                          │
                                                          ▼
                                                   XXDL960 (purge >2y from DEALDM1X)
```

---

## 6. Test Cases

| ID | Scenario | Expected | Maps to |
|---|---|---|---|
| TC-002-01 | First-call cache initialisation | `WS-FIRST-SW='Y'` → caches reset, then `WS-FIRST-SW='N'` | BR-002-02 |
| TC-002-02 | Division-code change mid-stream | Caches reset on next call | BR-002-02 |
| TC-002-03 | Customer pre-marked suppressed | No DB2 calls for customer; suppression set | BR-002-03 |
| TC-002-04 | Customer not suppressed, item suppressed | Item DB2 calls made; suppression set | BR-002-04 |
| TC-002-05 | Neither cust nor item suppressed, deal-rule hit | Deal-suppression cluster queried; `'Y'` returned | BR-002-05 |
| TC-002-06 | DB2 SQLCODE non-zero | Return code reflects documented mapping | BR-002-06 |
| TC-002-07 | Active billing of `group` deal | Bypassed | BR-002-10 |
| TC-002-08 | Active deal expired by current date | Treated as expired | BR-002-11 |
| TC-002-09 | Non-`'Y'` deal, non-zero `ACEDL3` | Start/end copied; `OUT-DEAL-CEDED-AMT += ACEDL3`; rewrite mark | BR-002-12 |
| TC-002-10 | No modifications in iteration | `OUT-RCD` NOT written | BR-002-13 |
| TC-002-11 | Archival of row 2y+1d old by `DLBUYH` | Row removed from `DEALDM1X` | BR-002-20/21 |
| TC-002-12 | Archival idempotency | Re-run on same input is a no-op | BR-002-23 |
| TC-002-13 | Suppressed deal in accrual cycle | `XXDLC20` does NOT roll it into accruals | BR-002-31 |

---

## 7. Open Questions

- `[SME]` `XXDL960` — AND vs OR semantics across `DLBUYH`/`DLINVH`/`DLSHPH` for the 2y test.
- `[SME]` `XXDL702` — what is the exact rule that classifies a deal as `'Y'` type vs non-`'Y'`?
- `[SME]` `XXDLC10` — `MOVE +16 TO RETURN-CODE` is commented out in `E1000-FAILED-OPEN`. Confirm whether this is intentional or a latent bug.
- `[SME]` `D8050` polling cadence and restart protocol details (BR-002-46).
- `[SME]` `MCBSM02` 45-day threshold — business rationale and any plans to parameterise.
- `[SME]` Confirm BR-002-31 (suppression gates accrual) and BR-002-32 (purge follows report cycle).
- `[RAG]` Document `MCBSM03`, `MCBSM30` (the remaining top-connected `MCBSM*` programs).
- `[RAG]` Document `XXDL655`, `XXDL656`, `XXDL660`, `XXDL702`, `XXDLV01` at the same fidelity as `XXDL740` and `XXDLC10`.

---

## 8. Modernization Notes

- **Blast radius (revised with CAST):** this BP is **massive**. CAST shows the deal-cycle jobs occupy 7 of the top-10 platform complexity slots: `MCBSM01J` (CC=486), `XXD2137J` (416), `XXDLSWKY` (414), `XXDLSDLY` (375), `XXDLSEOP` (343), `XXDL745J` (334), `MCDL656J` (298). Plus `XXDL664J` (264), `XXDL650J` (247), `MCDLS54J` (115), `MCDLS52J` (112). RAG-derived edge counts (`MCBSM04`=36, etc.) are mostly data-entity references — useful for *data*-level blast-radius but not for *call*-level blast-radius.
- **Temporal cycles are the heartbeat (HS-015):** `XXDLSDLY` / `XXDLSWKY` / `XXDLSEOP` together are 38,933 LoC and reference 1,344 distinct objects. Modernizing the deal cycle without first capturing the daily/weekly/EOP boundary semantics (when do period totals freeze? which artefacts are immutable between cycles?) will break operational expectations.
- **Preservation:**
  - `XXDLS01` is a **pure decision function** of (customer, item, deal). Cleanest candidate to extract first into the modernized stack as a tested rules service.
  - `XXDL960` purge logic is a small, dateable function — encode and certify byte-exact early.
  - `D8050` is the **lifecycle gateway** — replacing it without keeping `DM3P` → `DM1T` semantics intact will break every downstream step.
  - The 6 critical edits in `D8050` (BR-002-42) are the **deal-promotion contract** — they should be expressed as named, testable validators in the modernized stack.
  - `MCBSM02`'s 45-day cursor threshold (BR-002-81) is a hard-coded constant — modernization should expose it as configuration.
  - The lifecycle invariants (BR-002-30..32) MUST be preserved across cutover; coexistence between legacy and modernized lifecycles requires explicit dual-write or one-way-write strategy (Plan-phase decision).
- **Suggested cutover units (ranked by lowest risk first):**
  1. Suppression (`XXDLS01`) — pure function, smallest blast radius.
  2. Archival (`XXDL960` / `XXDLS60`) — small, dateable, idempotent.
  3. Reporting (`XXDLS50` / `XXDLS54` / `XXDL740`) — read-only consumers.
  4. Aggregation / accrual (`XXDLC10` / `XXDLC20`) — derived data, can run in parallel for parity validation.
  5. Active billing (`XXDL702`) — mutates active deal state; needs lock-coordinated cutover.
  6. **Capture (`D8050`)** — last, because it is the gateway and breaking it stalls every downstream step.
