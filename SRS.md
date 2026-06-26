# System Requirements Specification (SRS): Merchandising & Deal-Management Mainframe Platform (Legacy)

**Perspective:** Legacy / As-Is
**Opportunity:** `006UV00000aTYw1YAG`
**Version:** 1.0
**Status:** Draft — pending SME review
**Last Updated:** 2026-06-02
**Related:** [PRD](../01-product/PRD.md), [HLD](../03-technical/architecture/HLD.md), [BP specs](../03-technical/specs/)

---

## 1. Introduction

### 1.1 Purpose

Specifies the observed behaviour of the legacy COBOL / DB2 / CICS merchandising & deal-management platform, in terms the modernized system must reproduce. Functional requirements (FR-*) describe **what the system does today**; non-functional requirements (NFR-*) describe **how**.

### 1.2 Scope

Per the CAST snapshot (2026-05-21) under project codename `ACME`:

- **COBOL executables — 311 total** (123 Batch Programs + 143 Programs + 45 Transactional CICS Programs) across `acme-code/sclm.perm.prod.source/`.
- **612 COBOL CopyBooks** + 183 File Links across `acme-code/sclm.perm.prod.copy*/`.
- **62 JCL Jobs + 136 JCL Procedures** across `acme-code/{acme,sw}.perm.jcl/` and `acme-code/ds.perm.proclib/`.
- **CICS online surface — 45 transactional COBOL + 56 BMS Maps + 5 DataSets + 4 CICS Transactions + 3 Transient Data** queues.
- **DB2 schema** under `ACME.*` and `DS.*`, **220 Missing Tables** referenced (DDL outside scope), plus the **`ACME.PP_*`** price-protection cluster and **`ACME.INVC_*`** invoice (`BD*`) cluster.
- **VSAM:** 139 KSDS + 3 ESDS files (the divisional `<DIV>.MSTR.{SIM,VND,CAD}` families and others).
- **Datasets:** 889 JCL Data Sets + 173 PDS + 58 GDG + 37 Temp.
- **MQ:** 2 Subscribers + 3 MQ utility programs (consumed by CICS transactions `MCS4` / `MCS5`).
- **REXX:** utility scripts invoked through `IKJEFT01` / `IKJEFT1B`.
- **Inbound/outbound file contracts** with vendors, BI, billing, and stores.

Total platform footprint: **3,157 objects / 378,615 LoC**.

### 1.3 Definitions & Acronyms

- **ACME** — **Acme** (confirmed Round 5). Three in-corpus evidences: `XXDL742` constant `'ACME'` as `(System identifier)`; `MCDLS01` constant `'CST'` as `(ACME Business Type)`; `DCSMCA.ACME-COMPANY-SHORT-NAME PIC X(03)` field for centralised-buying short name. Customer is **Acme Company**, a major US convenience-store wholesale distributor.
- **Marwood** — The packaged COBOL/CICS centralised-buying / distribution-management product that this platform is built on. The DCS copybook family is Marwood framework code. Acme instance is at **Release 3.3.0** (per `MWD CONVERSION TO 3.3.0 RELEASE` and `SYSTEM RELEASE LEVEL.......: 3.3.0` copybook headers). The modernization is "replace a customised packaged product at a known customer", not "modernize bespoke code".
- **Quasar** — Downstream external system that consumes per-division `<DIV>.PERM.<DIV>ST2A.PP` price-change feeds produced by `MCRPR65J`. Out of scope for this modernization; the Quasar interface is fixed.
- **SCLM** — IBM Software Configuration & Library Manager — the legacy mainframe equivalent of a VCS. Production source lives in `sclm.perm.prod.*` libraries.
- **CAST** — Static-analysis tool whose 2026-05-21 export is the authoritative inventory for this assessment (see `_assess-cast-summary.md`).
- **CAD** — **Computerized Allowance Data** master family (`<DIV>.MSTR.CAD`). *(Corrected from "Cost-And-Deal" — see `mccbt07.md`.)*
- **SIM** — Item master family (`<DIV>.MSTR.SIM`).
- **VND** — Vendor master family (`<DIV>.MSTR.VND`).
- **DCS** — **Marwood Distribution System** shared copybook family — system control, file handlers, comm/error patterns. Includes `DCSMCA` (Marwood Control Area), `DCSWORK` (work fields + COMX), `DCSHVNDP/KVNDP/HITMP/FWHS/FOPR/FERR`, `DCST26`.
- **ACME** — **Marwood Control Area** — system-wide data structure passed program-to-program in CICS (per `DCSMCA`).
- **COMX** — Marwood communication exit calling codes (`COMX-LINK`, `COMX-XCTL`, `COMX-RETURN`, `COMX-CODE`, `COMX-PROGRAM`).
- **CIC** — **Cigarette Cost** (Corporate Cigarette Cost System) — `MCCIC*` job family handles cigarette cost change distribution, reporting, item-grouping cleanup. NOT "Customer Information Center".
- **CBT** — **Customer / Cigarette Batch Transactions** — `MCCBT*` is a **mixed** job family: `MCCBT03J` = customer billing; `MCCBT06/07` = cigarette deal batch; `MCCBT08J` = cigarette deal report; `MCCBT01J` = temp extract lifecycle (round 6).
- **MCM9 / M9*** — Data-integrity / reconciliation cycle (M9014 detects discrepancies; M9091 fills missing cost records).
- **DCLGEN** — DB2 column-descriptor copybooks generated from the DB2 catalog. Stored in `acme-code/DB2P.PERM.DCLGEN/`. Naming convention `DG*` (e.g. `DGDI1D` for `DIVMSTRDI1D`).
- **DLZB** — CICS transaction identified in the corpus (`dlzbstrt` JCL).
- **`MCxxx J`** — Naming pattern for orchestrating JCL jobs; usually executes `XXxxxP` COBOL program.
- **`MCS4` / `MCS5`** — CICS transaction IDs for MQ cost-update consumers (`MCCST50` / `MCCST51`).
- **`D21**`** — CICS transactional COBOL program family (41 programs `D2105` … `D2199`) forming the deal-domain online surface.
- **`DLBUYH` / `DLINVH` / `DLSHPH`** — Deal purchase / invoice / shipment dates used by archival rules (`XXDL960`).
- **`CDIC-LIST-AMT-TYPE`** — Cost list amount-type code that drives basis-code mapping in `XXCAD63`.
- **CPE-Legacy** — Context Preservation Engine corpus + graph for this legacy perspective.
- **CC / EC / IC** — Cyclomatic / Essential / Integration complexity (CAST metrics).
- **GDG** — Generation Data Group (z/OS dataset versioning convention; 58 GDG datasets in scope).
- **KSDS / ESDS** — VSAM Keyed-Sequence / Entry-Sequence data sets (139 KSDS + 3 ESDS in scope).

---

## 2. System Overview

### 2.1 System Context

```
            Vendors / EDI                    Buyers / Merchants
                  │                                │
                  │ vendor data, cost              │ CICS DLZB / MCCSM55
                  ▼                                ▼
        ┌────────────────────────────────────────────────────────┐
        │           Mainframe Merchandising Platform             │
        │                                                        │
        │     CICS (online)      JCL/COBOL (batch)               │
        │            │                  │                        │
        │            └────────┬─────────┘                        │
        │                     ▼                                  │
        │     DB2 (ACME.*)  +  VSAM/datasets (<DIV>.MSTR.*)        │
        │                     ▼                                  │
        │             Extracts / Reports                         │
        └────────────────────────────────────────────────────────┘
              │                       │                       │
              ▼                       ▼                       ▼
         Billing / AP            BI / Reporting            Stores / Repl.
```

### 2.2 Primary Actors

| Actor | Interaction |
|-------|-------------|
| Buyers / merchants | CICS — deal maintenance, price-protection setup. |
| Vendor master team | CICS + batch (`MCBSM52J`, `MCCST19`). |
| Costing / pricing | Batch (`MCCST24J`, `MCCST50J`, `MCCST63J`, `MCCST96J`, `XXCAD63`). |
| Operations | Run the JCL schedule, react to abends. |
| Downstream systems (Billing, BI, Stores) | Consume extracts and DB2 views. |
| External vendors / data providers | Push raw cost/vendor files into landing datasets. |

---

## 3. Functional Requirements

> Every FR-* describes **observed legacy behaviour**. The modernized system must preserve it unless explicitly waived.

### 3.1 Item Master Data Management

#### FR-ITEM-001 — Maintain divisional item master
- **Requirement:** The platform SHALL maintain a separate item master per division (`<DIV>.MSTR.SIM`) for 32 divisions, plus the corporate-level item entities in DB2 (`ACME.ITEM_MASTER_IM3I`, `ACME.UIN_ITEM_DE6C`, `ACME.DIV_ITEM_PACK_DE1I`, `ACME.ITEM_UPC_DE6Y`).
- **Acceptance (parity):** For every (`division`, `item`) tuple, the modernized read returns identical attribute values to the legacy read.
- **Spec:** [`BP-001-item-master-data-management.md`](../03-technical/specs/BP-001-item-master-data-management.md)

#### FR-ITEM-002 — Item-vendor relationship
- **Requirement:** The platform SHALL maintain the item-vendor relationship in `ACME.ITEM_VNDR_DE6V` (referenced by 35 programs).
- **Acceptance (parity):** Joining `ACME.ITEM_MASTER_IM3I` with `ACME.ITEM_VNDR_DE6V` yields the same set of authorised vendor-item pairs.

#### FR-ITEM-003 — UPC catalogue
- **Requirement:** The platform SHALL maintain UPC-to-item mappings in `ACME.ITEM_UPC_DE6Y` (referenced by 30 programs).
- **Acceptance:** UPC scans resolve to the same item identifier in both systems.

### 3.2 Vendor Data Management

#### FR-VEND-001 — Vendor master maintenance
- **Requirement:** The platform SHALL maintain vendor records in `ACME.VNDR_MSTR_VN1A` (46 referencing programs) and divisional vendor masters in `<DIV>.MSTR.VND`.
- **Acceptance (parity):** Vendor lookups (`MCCST19`-style "by short name") return identical result sets.

#### FR-VEND-002 — Vendor cross-reference
- **Requirement:** The platform SHALL maintain the corporate-vendor cross-reference (`CRP_VNDR_XREF_VN1X`) used by costing/deal jobs.
- **Acceptance:** Every active corporate vendor resolves to the same set of divisional vendor IDs.

#### FR-VEND-003 — Vendor synchronisation job
- **Requirement:** The platform SHALL run a vendor synchronisation pipeline (`MCBSM52J`) producing reconciled vendor data.
- **Behaviour:** Inbound `NEWVNDRS` and `CORPVNDR` sequential files are reconciled into the vendor masters.
- **Acceptance (parity):** The post-job state of `ACME.VNDR_MSTR_VN1A` is byte-identical given identical inputs.

#### FR-VEND-004 — Per-vendor specialised handlers
- **Requirement:** The platform SHALL execute vendor-specific handlers where they exist (`AUTHORKAY`, `AUTHORMCLANE`) for retrieving vendor item cost and deal information and for reporting missing cost records.
- **Acceptance:** Per-vendor reports remain identical.

### 3.3 Deal Lifecycle

#### FR-DEAL-001 — Pending deal capture and promotion to active
- **Requirement:** The platform SHALL capture pending deals in `ACME.PENDINGDEALSDM3P` (`DM3P`, 31 referencing programs) via the online (CICS) and batch (`MCCBT06`/`MCCBT07`) paths, and SHALL promote validated pending deals into the deal transaction table (`DM1T`) via the **`D8050` CICS polling transaction**.
- **Behaviour (verbatim from `D8050`):**
  - Polls `DM3P` periodically; branches by deal type — divisional → `ACME.DIVPENDDEALSDM3D` (`DM3D`); group → `ACME.GRPPENDDEALDM3G` (`DM3G`).
  - Performs six critical edits before promotion: overlapping deal date, item status, division, licensing limit, corporate vendor control, existing terminated deals.
  - Validation failures → `ACME.CAD_ERR_LOG_DM3E` (`DM3E`); successes → `DM1T`.
  - Maintains a restart record in `DM1E` keyed by `DM1E-SEQNCE` to recover from crashes.
  - Invokes `CM510ZO` during initialisation to load processing-data structures.
- **Acceptance:** All fields captured at submission survive every downstream join; for identical `DM3P` input, the modernized system produces identical `DM1T` output and identical `DM3E` rejection records.

#### FR-DEAL-002 — Active billing deals
- **Requirement:** The platform SHALL process active billing deals (`XXDL702`), bypassing deal types `group`, `DVRTR`, and `BRKT`, and SHALL move `ACEDL3` amounts into current deal fields and into `OUT-DEAL-CEDED-AMT` per the documented logic.
- **Acceptance:** Output records produced for a fixed input batch are byte-identical.

#### FR-DEAL-003 — Deal accrual
- **Requirement:** The platform SHALL run deal accrual processing (`XXDLC20`) over active deals.
- **Behaviour (verbatim from `XXDLC20`):** Reads an "end-of-processing date" from an input file (the accrual window driver); creates a temp DB2 table for intermediate accruals; aggregates `DEALDM1X` by **deal part**, **item number**, and **deal ID**; references `ACME.DIVMSTRDI1D` and `ACME.DT_JS1A`.
- **Acceptance (parity):** Accrual totals match the legacy run for the same input window.

#### FR-DEAL-004 — Deal transaction aggregation and output
- **Requirement:** The platform SHALL aggregate deal transactions (`XXDLC10`) and produce the aggregation output (`OUTDEAL`) consumed downstream.
- **Behaviour (verbatim from `XXDLC10`):** Loops `B1000-PROCESSING` until `SW-EOF-OF-TABLE = 'Y'`; complex aggregating join against `DEALDM1X` + master data; constructs a unique item key; `S1000-FINALIZATION` displays completion timestamp + division + total `OUTDEAL` written + total `DEALDM1X` fetched.
- **Edge case:** `E1000-FAILED-OPEN` displays `FAILED-FILE` + `FAILED-STATUS`; the `MOVE +16 TO RETURN-CODE` line is **commented out** in the legacy. The modernized system MUST decide whether to surface a non-zero RC on open failure — `[SME]` confirm.
- **Acceptance:** Aggregated records are identical in count and content.

#### FR-DEAL-005 — Deal suppression decision
- **Requirement:** The platform SHALL evaluate deal suppression (`XXDLS01`) using customer-, item-, and deal-level criteria.
- **Behaviour (verbatim from corpus):**
  - Initialise `DLS01-DEAL-SUPP-SW = 'N'`.
  - Reset `WS-CUST-TABLE` / `WS-ITEM-TABLE` caches when `WS-FIRST-SW = 'Y'` or the division code or division part changes.
  - Check customer suppression via `ACME.PROF_HDR_PR1P`, `ACME.PROF_CUS_PR3Q`, `ACME.DIVMSTRDI1D`, `ACME.CUST_XREF_CU1X`.
  - Check item suppression via `ACME.ITEM_SUPP_IT2A`, `ACME.ITEM_MASTER_IM3I`.
  - Check deal-specific suppression via `ACME.PROF_HDR_PR1P`, `ACME.PROF_DEAL_PR4D`, `ACME.DEAL_SUPP_DS1R` keyed by deal type, form of payment, and invoice date.
- **Acceptance:** Output switch `DLS01-DEAL-SUPP-SW ∈ {'Y','N'}` matches the legacy decision for every input tuple.
- **Spec:** [`BP-002-deal-management-and-analysis.md`](../03-technical/specs/BP-002-deal-management-and-analysis.md)

#### FR-DEAL-006 — Deal sales update
- **Requirement:** The platform SHALL update deal sales (`XXDM713`) into the deal data model.
- **Acceptance:** Updated sales totals match per (deal, period).

#### FR-DEAL-007 — Deal change and BI reporting
- **Requirement:** The platform SHALL produce deal-change details (`MCDLS17`), deal comparison reports (`XXDLS54`), and deal analysis for BI (`XXDLS50`).
- **Acceptance:** Reports are byte-identical when run on the same input.

#### FR-DEAL-008 — Group / mass deal maintenance
- **Requirement:** The platform SHALL support group-deal extraction and mass maintenance (`MCBSM70J`, `MCBSM72J`, `XXBSM70`, `XXBSM71`, `XXDLS51`, `XXDLS52`).
- **Acceptance:** Group-deal apply/extract operations are identical in their effects.

#### FR-DEAL-009 — Deal alerting and status tracking
- **Requirement:** The platform SHALL generate deal alert reports (`XXDL571`) and run batch-job status checks (`XXDLS59`).
- **Behaviour (round 6):** The six-step reporting + purge proc chain (`XXDL530P` → `XXDL570P` → `XXDL571P` → `XXDL995P` → `XXDL960P` → `XXDL980P`) is orchestrated by JCL job **`XXDLSWKY`** (CAST CC=414) — the weekly deal-cycle job.
- **Acceptance:** Alert and status outputs are identical.

#### FR-DEAL-011 — Weekly deal cycle and coupled purge orchestration
- **Requirement:** The platform SHALL run **`XXDLSWKY`** as the weekly deal-cycle JCL job, including the coupled purge/reporting procedure chain documented in BP-002.
- **Behaviour:** CAST shows all six procs invoke under `XXDLSWKY`; corpus prose for the job itself remains `[CORPUS-EMPTY]`.
- **Acceptance:** Post-run deal log, audit, and archive table states match legacy given identical inputs and parameter keys (`XXDL980_DM1A_DAYS`, etc.).

#### FR-DEAL-010 — Historical deal archival
- **Requirement:** The platform SHALL purge deal headers / archive deal records when older than two years (`XXDLS60`, `XXDL960`).
- **Behaviour:** Record age is determined by the maximum of `DLBUYH`, `DLINVH`, `DLSHPH` against *today − 2y*; eligible rows are removed from `DEALDM1X` via cursor read.
- **Acceptance:** Post-purge `DEALDM1X` state is identical given identical pre-state.

### 3.4 Costing & Price Protection

#### FR-COST-001 — Item cost validation
- **Requirement:** The platform SHALL validate and process item cost data (`XXCAD63` invoked by `MCCAD65J`) against the CAD masters and the corporate cost source.
- **Behaviour (verbatim from corpus):**
  - Active vendor requires `VNKY-RECORD-STATUS-CODE = 'A'`.
  - Cost control requires `VNRT-COST-CONTROL-FLAG = 'Y'`.
  - "Cost record found" when `WS-CUR-COST-SW = 'Y'` OR `WS-FUT-COST-SW = 'Y'`.
  - Basis-code mapping from `CDIC-LIST-AMT-TYPE`: `'F'` → `'ACME'`; `'1'` → `'CW'`; otherwise → `'SC'`.
  - Discrepancies (effective date or cost amount) generate report entries.
  - Exception items (`WS-EXC-ITEM-SW = 'Y'` AND `WS-AP1R-SW = 'Y'`) log `"ITEM EXCEPTION - NO UPDATES"` instead of failing.
  - File status `'23'` / `'10'` → abnormal termination with `RETURN-CODE = 16`.
- **Acceptance:** Discrepancy report content and abnormal-termination behaviour preserved byte-for-byte.

#### FR-COST-002 — Future cost-changes batch extract
- **Requirement:** The platform SHALL run `MCCST24J` (executing `MCCST24`) to produce the comma-delimited future-cost-changes extract `MCCST24O`.
- **Behaviour:** Reads `EDISENT` input; uses temp table `TEMP.ITEM_BILL_COST_T356`; queries `ACME.DIVMSTRDI1D`, `ACME.ITEMAUTHST1A`, `ACME.ITMCOSTCNTLAUDDE9A`, `ACME.ITM_BILL_COST_DE6E`. **Checkpoint:** reads `MCCST24_LAST_RUN` from `DS.APPL_SYS_PARM_AP1S` to derive the date filter; on successful completion `4000-UPDATE-AP1S-PARA` writes the current timestamp back to `MCCST24_LAST_RUN`.
- **Acceptance:** Output is byte-identical given identical input and identical `MCCST24_LAST_RUN`.

#### FR-COST-002b — Other batch costing cycles
- **Requirement:** The platform SHALL run `MCCST63J`, `MCCST96J` covering additional item costing, price-protection input prep, and deal-analysis cycles. `[RAG]` Document per-job behaviour at the fidelity of `MCCST24`.
- **Acceptance:** Per-job outputs are identical for the same inputs.

#### FR-COST-003a — Customer billing batch (`MCCBT03J`)
- **Requirement:** The platform SHALL run `MCCBT03J` (`XXCBT03P` / `MCCBT03`) to generate customer billing statements and update customer account balances from `CUSTOMER.MASTER.FILE` + `TRANSACTION.FILE`.
- **Acceptance:** Statement content and updated master balances match legacy for the same billing cycle.

#### FR-COST-003 — Cigarette-deals batch (two-stage)
- **Requirement:** The platform SHALL run the two-stage cigarette-deals batch `MCCBT06` → `MCCBT07` against sorted cigarette-deal extract files.
- **Behaviour (verbatim from `MCCBT07`):** Reads `BATCHID` from `RDRPARM` — **non-numeric `BATCHID` causes abend**. Queries `ACME.BAT_DEALERRLOGDM3B` for rows with `ERR_TYP = 'F'` (Fatal) for current `BATCH_ID`; surfaces informational message if found. Updates / inserts into `ACME.PENDINGDEALSDM3P`, `ACME.DIVPENDDEALSDM3D`, `ACME.DEALITEMDM2I`, `ACME.DEALDM1M`, `ACME.CAD_REMARK_DM3R`, `ACME.ITEM_UPC_DE6Y`. Status updates: `ACME.BAT_DEAL_HDR_DM3H`, `ACME.BAT_DEAL_DM3L`. Output rows in `ACME.PENDINGDEALSDM3P` become input to FR-DEAL-001.
- **Acceptance:** Resulting state is identical.

#### FR-COST-004 — CICS / MQ cost-message processing
- **Requirement:** The platform SHALL expose CICS transactions `MCS4` (program `MCCST50`) and `MCS5` (program `MCCST51`) as MQ consumers for cost updates, with `MCCST55` as the routing dispatcher.
- **Behaviour:**
  - `MCCST55` reads the incoming MQ message and routes by `COMM-Q-TYPE`: `'IC1X'` → `WS-TRANSID := 'MCS4'`, `WS-PROGRAM := 'MCCST50'`; `'CADMF'` → `WS-TRANSID := 'MCS5'`, `WS-PROGRAM := 'MCCST51'`; any other value → control to `STEP-X`.
  - `MCCST50` creates or updates cost records based on `COMM-ACTION`, verifies `DE9E` and `DE6E` data before write, and returns to CICS via `a6000-return-to-cics`. Enforces a retry cap via `b1100-get-max-retry`.
  - `MCCST51` logs errors to both `ACME.CMN_ERR_CM5A` and `ACME.COMNT_CM4A` with codes, descriptions, user IDs, and timestamps.
- **Acceptance:** Identical MQ message input → identical DB2 state. Online latency p95 MUST NOT regress vs legacy baseline `[SME]`.

#### FR-PRPR-001 — Price protection rule processing (full PP cycle)
- **Requirement:** The platform SHALL run the full PP cycle as 13 batch jobs: `MCRPR50J`, `MCRPR55J`, `MCRPR56J`, `MCRPR58J`, `MCRPR59J`, `MCRPR65J`, `MCRPR70J`, `MCRPR71J`, `MCRPR74J`, `MCRPR77J`, `MCRPR85J`, `MCRPR86J`, `MCRPR99J`. The underlying COBOL programs include `XXRPR50` … `XXRPR57`, `XXRPR65`, `XXRPR70`, `XXRPR71`, `XXRPR72`, `XXRPR74`, `XXRPR77P`, `XXRPR85`, `XXRPR86`, `XXRPR99`.
- **Critical risk:** `MCRPR50J` is the **highest-complexity job in the entire platform** (CC=553, LOC=20,938, ObjectCount=919). Modernization must treat it as its own milestone, not bundled.
- **Acceptance:** Request outcomes (apply, expire, archive) match legacy for identical inputs across every job. Each job's CC must not regress in the modernized re-implementation.

#### FR-PRPR-002 — MCRPR50J: PP request type `'2CE'` pipeline (CC=553)
- **Requirement:** The platform SHALL run `MCRPR50J` as a 6-program pipeline orchestrating the `'2CE'` PP request lifecycle:
  1. `XXRPR50` — identifies pending PP requests of type `'2CE'` linked to active rules; enriches with customer/item data; populates `ACME.PERM.RPR50S1`; logs to `ACME.PERM.RPR50.STAT`.
  2. `SORT1` — sorts `ACME.PERM.RPR50S1` per `DS.PERM.SORTPARM(XXRPR501)` → `ACME.PERM.RPR50S1.SRT`.
  3. `XXRPR51` — reads `ACME.PERM.RPR50S1.SRT`; applies **License Cost, Billing Cost, Invoice Price** to request lines; writes `ACME.PERM.RPR51S1`; logs `ACME.PERM.RPR51.STAT`.
  4. `SORT2/SORT3` — sorts `ACME.PERM.RPR51S1` (per `DS.PERM.SORTPARM(XXRPR511)`) and produces `ACME.PERM.RPR51S1.SRT` and `ACME.PERM.RPR51S3.SRT` (per `DS.PERM.SORTPARM(XXRPR512)`).
  5. `XXRPR52` — consumes the sorted output; updates core PP rules and customer/item associations; logs `ACME.PERM.RPR52.STAT`.
  6. `XXRPR53` — uses `ACME.PERM.RPR51S3.SRT` + consolidated `ACME.PERM.RPR.STAT` to generate email notifications to the job requester (config: `DS.PERM.RDRPARM(RPREML53)`).
- **Additional programs invoked:** `XXRPR54`, `XXRPR57`. Behaviour `[RAG]` for these two.
- **Rule files read:** `ACME.PERM.RPR72RUL`, `ACME.PERM.RPR73RUL`.
- **Item/customer scope files read:** `ACME.PERM.RPR71S1.ITEMS`, `ACME.PERM.RPR71S1.CUSTS`.
- **Acceptance:** Output of every intermediate dataset is byte-identical given identical inputs and identical rule files. The email notification's subject + payload structure preserved.

#### FR-PRPR-003 — MCRPR58J: Nightly PP rule application
- **Requirement:** The platform SHALL run `MCRPR58J` nightly to apply price-protection rules.
- **Behaviour:** `XXRPR58` reads a date parameter (`%%C1-%%M1-%%D1` format) from `READER`; filters DB2 by that date; inserts new effective customer-item configurations into **`ACME.PRC_CMPNT_ITM_ST3B`**. Tables referenced: `ACME.PRC_CMPNT_ITM_ST3B`, `ACME.PRC_CMPNT_ITM_ADTC` (audit), `ACME.PRC_COMP_DTC`, `ACME.PRC_CMPNT_ITM_PR1C`. **Conditional insertion:** if a configuration for the same customer-item-effective-date already exists, the insertion is skipped and counted as "record not inserted" (not an error). Critical errors → `RETURN-CODE = 16`. Standard linkage: `DS.EMER.LINK`, `DS.PERM.LINK`, `DS.PERM.SQLBATCH(SQLINFO)`.
- **Acceptance:** Run-summary counts (fetched, inserted, not-inserted) preserved byte-identical.

#### FR-PRPR-004 — MCRPR59J: Active-to-history archival for PP requests
- **Requirement:** The platform SHALL run `MCRPR59J` to archive completed/in-progress PP requests from the active table to the history table.
- **Behaviour:** `XXRPR59` reads retention days from `DS.PERM.RDRPARM(RPR59R1)` (a numeric retention parameter); calculates `target_date = today - retention_days`; selects rows from `ACME.PP_RQST_PP3C` with status `'CMP'` or `'INP'` and `CMPLTN_TS < target_date`; inserts each into **`ACME.PP_RQST_HIST_PP3H`** then deletes from `ACME.PP_RQST_PP3C`. **Duplicate-key behaviour:** if the history insert fails on a duplicate key, the active-table delete is suppressed (rollback). Database errors during insert/delete cause abnormal termination.
- **Acceptance:** Active-table row count post-run = pre-run count − successfully-archived count. Idempotent on re-run.

#### FR-PRPR-005 — MCRPR74J: Billing-to-allocation 2-step sync
- **Requirement:** The platform SHALL run `MCRPR74J` to update customer PP allocations based on recent billing data, in two sequential steps.
- **Behaviour:**
  - Step 1 (`XXRPR74`): reads billing data from `ACME.INVC_HDR_BD1H` + `ACME.INVC_DTL_COMN_BD1D` (the invoice / Bill Detail family); adjusts quantities and amounts at customer-group and item level; updates **`ACME.PP_ALLOC_DTL_PP4D`**, **`ACME.PP_ALLOC_SUM_PP4S`**, **`ACME.PP_ALLOC_HIST_PP4H`**. Logs execution timestamp.
  - Step 2 (`XXRPR75`): synchronises customer-specific pricing allocations with billing data; reads invoice details; adjusts pre-billing quantities for customer items; aggregates to customer-group level. Updates `ACME.PP_ALLOC_DTL_PP4D` and `ACME.PP_ALLOC_SUM_PP4S`.
  - JCL enforces strict step ordering — XXRPR75 must run after XXRPR74.
- **Acceptance:** Allocation totals at customer-item and customer-group levels match legacy byte-for-byte for identical invoice inputs.

#### FR-PRPR-006 — MCRPR86J: ST3B cleanup
- **Requirement:** The platform SHALL run `MCRPR86J` to remove outdated/invalid PP records from `ACME.PRC_CMPNT_ITM_ST3B`.
- **Behaviour:** `XXRPR86` identifies and deletes records meeting purge criteria (specific status codes, expiration dates, irrelevance). Paired with `MCRPR58J` which writes to the same table.
- **Acceptance:** Post-purge `ST3B` row count and content identical to legacy.

#### FR-PRPR-007 — Remaining PP cycle jobs (round 6)
- **Requirement:** The platform SHALL run `MCRPR65J` (Quasar price-change distribution), `MCRPR71J` (allocation expiry), `MCRPR77J` (conditional `XXRPR77P`), `MCRPR85J` (BI2 pending + MQ FTE), and `MCRPR99J` (pricing-record purge).
- **Behaviour:**
  - `MCRPR65J`: selects `ACME.PRC_CMPNT_ITM_ST3B` where `PRCS_TS = '1900-01-01-00.00.00.000000'`; outputs `ACME.PERM.ST2A.PP` + per-division `<DIV>ST2A.PP` files.
  - `MCRPR71J`: extract/sort/expire allocation working files (`ACME.PERM.RPR71S1.*`, `ACME.PERM.RPR721S1`); feeds `MCRPR50J`.
  - `MCRPR77J`: `XXRPR77P` executes only when prior step RC `< 4`.
  - `MCRPR85J`: processes BI2 pending requests; MQ FTE transfer via `&&TEMPFTE`.
  - `MCRPR99J`: purges aged pricing DB2 rows; CAST also shows deletes on `ACME.PP_RQST_HIST_PP3H`.
- **`MCRPR70J`:** still `[CORPUS-EMPTY]` for business narrative.
- **Acceptance:** Quasar file layouts, allocation expiry outcomes, BI2 extracts, and purge row counts match legacy.

#### FR-PRPR-002 — Price protection notifications
- **Requirement:** The platform SHALL handle email notifications and status updates for pricing requests (`XXRPR53`).
- **Acceptance:** Notification dispatch is recorded identically (recipients, subject, payload) — actual transport may be modernised under explicit waiver.

#### FR-PRPR-003 — AP1S end-of-job record update
- **Requirement:** The platform SHALL perform the AP1S record update at end-of-job (`XXRPR54`).
- **Acceptance:** AP1S state is identical post-run.

### 3.5 Sales Transaction Processing

#### FR-SALES-001 — Sales consolidation
- **Requirement:** The platform SHALL consolidate sales transaction data using `ACME.PERM.BSM30S1.DWSLS` as the central dataset, produced by `MCBSM50J` orchestrating `XXBSM30` → `XXBSM31` → `XXBSM32` (final write via `5000-WRITE-DATA`).
- **Acceptance:** Consolidated outputs (counts, totals, per-key sums) match legacy on the same inputs.
- **Spec:** [`BP-005-sales-transaction-processing.md`](../03-technical/specs/BP-005-sales-transaction-processing.md)

#### FR-SALES-002 — Invoice-driven deal-sales update
- **Requirement:** The platform SHALL run `XXDM713` (Deal Sales Update) to apply invoice-derived sales activity onto deal records.
- **Behaviour (graph-derived; corpus narrative empty for `XXDM713`):** Reads `ACME.INVC_HDR_BD1H`, `ACME.INVC_DTL_COMN_BD1D`, `ACME.INVC_DTL_ITEM_BD2D`, `ACME.INVC_TS_BD2T`. Joins with `ACME.CUST_XREF_CU1X`, `ACME.DIVMSTRDI1D`, `ACME.US_TAX_CU4U`, `DATECNTLCF1D`. Output: per-deal sales totals updated.
- **Acceptance:** Identical invoice input → identical updated deal-sales totals. Idempotent on re-run of the same window.
- **Caveat:** Earlier drafts of FR-SALES claimed `XXDM713` consumed `ACME.PERM.BSM30S1.DWSLS` directly. The dependency graph shows it consumes the `ACME.INVC_*` (`BD*`) family instead. `ACME.PERM.BSM30S1.DWSLS` is a separate consolidation plane (FR-SALES-001) whose consumers must be enumerated separately.

### 3.5b Temporal deal cycles (daily / weekly / EOP)

#### FR-DEAL-011 — Daily deal-cycle job
- **Requirement:** The platform SHALL run `XXDLSDLY` once per business day, processing the day's deal-side activity.
- **Complexity:** CC=375, LOC=11,646, ObjectCount=344 (top-5 platform complexity).
- **Acceptance:** Identical end-of-day deal state for identical day's input. Operational window `[SME]`.

#### FR-DEAL-012 — Weekly deal-cycle job
- **Requirement:** The platform SHALL run `XXDLSWKY` weekly, performing roll-ups and weekly reconciliations.
- **Complexity:** CC=414, LOC=14,693, ObjectCount=515 (top-4 platform complexity).
- **Acceptance:** Identical weekly summary state for identical inputs.

#### FR-DEAL-013 — End-of-Period deal-cycle job
- **Requirement:** The platform SHALL run `XXDLSEOP` at period-end (typically month-end), performing period-close activities.
- **Complexity:** CC=343, LOC=12,594, ObjectCount=485.
- **Acceptance:** Identical period-close state. Calendar of period boundaries `[SME]`.

#### FR-DEAL-014 — Additional deal-cycle batch jobs
- **Requirement:** The platform SHALL run `XXDL650J`, `XXDL664J`, `XXDL670J`, `XXDL745J`, `XXDLS02J`, `XXDLS06J`, `XXDLS07J`, and `XXD2137J` as part of the broader deal cycle.
- **Behaviour:** `[RAG]` to detail per job. CAST complexity already captured (HLD §6).

#### FR-DEAL-015 — Daily deal-reporting + purge cycle (6-step parent JCL)
- **Requirement:** The platform SHALL run the daily deal-reporting + purge cycle as a 6-step parent JCL job (CAST-normalized name `__name.md` — `[SME]` to recover actual job name):
  1. `XXDL530P` — Report on deals with upcoming / recent buy/ship dates.
  2. `XXDL570P` — Deal performance report (current fiscal year).
  3. `XXDL571P` — **Deal alert report.** Reads processing date from `RDR1`; calculates ±7-day range; queries `DEALDM1X` + `STARITEMCATST1I` + `ITEMAUTHST1A` + `CUST_CLS_GRP_CU2E` + `DIVMSTRDI1D` + `ACME.DIV_ITEM_PACK_DE1I` + `ACME.UIN_ITEM_DE6C`; categorises deals as active / future / inactive / terminated; produces alert report.
  4. `XXDL995P` — **Purge `DEALLOGDM5X` records older than 5 days.** Uses `FCURRF` timestamp for selection. Reads `ACME.DIVMSTRDI1D` for division context.
  5. `XXDL960P` — Archive `DEALDM1X` records older than 2 years (covered by FR-DEAL-010 / BR-002-20..23).
  6. `XXDL980P` — **Purge `DEALAUDITDM1A` records.** Retention period from `DS.APPL_SYS_PARM_AP1S.XXDL980_DM1A_DAYS` parameter, defaulting to **60 days** if not present.
- **Acceptance:** Post-run row counts for `DEALLOGDM5X`, `DEALDM1X`, `DEALAUDITDM1A` match the legacy retention windows (5d / 2y / `XXDL980_DM1A_DAYS` resp.). Reports are byte-identical for identical inputs.

### 3.5c Customer Information Center (CIC) and M9 families

#### FR-CIC-001 — Cigarette Cost (Corporate Cigarette Cost System) cycle
- **Requirement:** The platform SHALL run the `MCCIC*J` Cigarette Cost cycle as 4 jobs:
  - `MCCIC01J` — processes cigarette cost change transactions; splits by divisional GL codes; consolidates into per-division permanent files.
  - `MCCIC02J` — consolidates divisional report files into a single temp file; prints the cigarette-cost exception report.
  - `MCCIC20J` — generates detailed cost component report for linked cigarette items via the COBOL program `MCCIC20`. Reads DB2 tables `ACME.MCLANE_XREF_DI3X`, `ACME.UIN_ITEM_DE6C`, `ACME.ITEM_UPC_DE6Y`, **`ACME.CIG_ITEM_COST_PR1C`**, `ACME.DIVMSTRDI1D`.
  - `MCCIC90J` — cleanup/maintenance for "non-maintainable" and "default" item groupings. Inserts new cigarette items, deletes non-cigarette/inactive items, cleans orphans. Reads `ACME.ITEM_GRP_DE1E` and `ACME.CIG_ITEM_COST_PR1C`.
- **Acceptance:** Identical cigarette cost state per division after each cycle; identical reports byte-for-byte.
- **Dependency:** The CIC family is the cigarette-specific complement to BP-004 (general costing) and BP-002 (cigarette deals via `MCCBT06`/`07`).

#### FR-M9-001 — Data-integrity / reconciliation cycle (MCM9 family)
- **Requirement:** The platform SHALL run the `MCM9*J` data-integrity cycle as 2 jobs:
  - `MCM9014J` — calls `M901401` per corporate entity to compare deal data in `XX.DEALDM1X` against the related PowerBuilder CAD tables; calls `M901402` to generate a consolidated "Corporate Discrepancy Report" from sorted per-entity results.
  - `MCM9091J` — calls `M9093` per division to identify items in divisional files that lack corresponding cost records in CAD VSAM; validates items against SIM files; constructs and inserts the missing cost records; generates a summary report of created records.
- **Acceptance:** Discrepancy detection completeness and missing-record creation must remain identical to legacy. Modernization design should engineer the new platform such that the **MCM9 self-healing loop is unnecessary**, not reproduced.

#### FR-SW-001 — SW JCL jobs (sales/item update + costing)
- **Requirement:** The platform SHALL run `SWCST64J` (costing) and `SWDEI60J` (sales/item update) located under `acme-code/sw.perm.jcl/`.
- **Behaviour:** `SWDEI60J` processes sales and item data to update various system tables and generates extract files (delete old data → create new extracts → sort → update). `SWCST64J` mirrors `XXCST64J` for costing. `SW` scope confirmed via `swdei60j.md`.
- **`[SME]`:** Confirm whether `SW` denotes a specific division (out of the 32 division codes) or "Software" or another classification.

#### FR-QUASAR-001 — Quasar price-change distribution (downstream)
- **Requirement:** The platform SHALL produce price-change records for the downstream **Quasar** system via `MCRPR65J`.
- **Behaviour:** `MCRPR65J` queries DB2 to create a master file of price-change records, then splits it into per-division files **`<DIV>.PERM.<DIV>ST2A.PP`** (one dataset per division). Confirmed instances: `MS.PERM.MSST2A.PP`, `MN.PERM.MNST2A.PP`, `GM.PERM.GMST2A.PP`, `MP.PERM.MPST2A.PP`, `MW.PERM.MWST2A.PP`, `MG.PERM.MGST2A.PP`, `C3.PERM.C3ST2A.PP` (likely all 32 divisions).
- **Acceptance:** The Quasar interface contract (file layout + per-division splitting + cadence) MUST be preserved byte-for-byte. Quasar itself is out of scope for this modernization.

### 3.6 CICS Online Surface

#### FR-ONLINE-001 — DLZB transaction
- **Requirement:** The platform SHALL expose the CICS transaction `DLZB` (started by `dlzbstrt` JCL).
- **Acceptance:** Identical screen flow, identical record-write behaviour.

#### FR-ONLINE-002 — CICS BMS map surface (56 screens)
- **Requirement:** The platform SHALL render its full BMS map set — **56 CICS Maps** in total (CAST inventory). `MCCSM55` is the only one named in the corpus to date; the other 55 are referenced from the JCL/COBOL side but not yet enumerated.
- **Acceptance:** Field-level validations match per screen.
- **`[CAST]`:** Enumerate the 56 maps and map each to its owning CICS transactional COBOL program (likely most live in the `D21**` family).

#### FR-ONLINE-006 — `D21**` CICS transactional COBOL family
- **Requirement:** The platform SHALL expose the **41 CICS transactional COBOL programs** in the `D21**` family (`D2105` … `D2199`) as the deal-domain online surface. This is the largest contiguous program family in the platform.
- **Behaviour:** Per-program behaviour `[RAG]`. Programs include `D2138` (37 cross-references — top-30 most-referenced object), `D2112` (16 edges), `D2122` (corpus body empty — `[CORPUS-EMPTY]`).
- **Acceptance:** Identical CICS-transaction behaviour. The CAST `Transactions_Complexity` / `Relations_*` files plus a targeted RAG sweep on `d2***` markdown are the inputs.

#### FR-ONLINE-003 — Common CICS service primitives
- **Requirement:** The platform's online code SHALL use `EXEC CICS RETRIEVE`, `EXEC CICS XCTL`, and `EXEC CICS ENQ` consistently with the legacy idiom for transaction chaining and exclusivity.
- **Acceptance:** Concurrent-access semantics (e.g. ENQ scopes) match. `[CODMOD]` enumerate the named ENQ resources.

#### FR-ONLINE-004 — D8050 deal-capture polling transaction
- **Requirement:** The platform SHALL expose `D8050` as a CICS transaction that periodically polls `ACME.PENDINGDEALSDM3P` and promotes validated deals into `DM1T`. (See FR-DEAL-001 for the behavioural detail.)
- **Acceptance:** Polling cadence and validation outcomes match the legacy reference run.

#### FR-ONLINE-005 — MCS4 / MCS5 cost-message transactions
- **Requirement:** The platform SHALL expose `MCS4` (program `MCCST50`) and `MCS5` (program `MCCST51`) as CICS transactions consuming MQ cost-update messages, dispatched by `MCCST55`. (See FR-COST-004 for the behavioural detail.)
- **Acceptance:** Identical MQ message → identical DB2 effect; latency p95 not regressed.

### 3.7 Common Utilities

#### FR-UTIL-001 — Error message catalogue
- **Requirement:** Programs SHALL use `D0007` for extended error messages.
- **Acceptance:** Message catalogue text and codes preserved.

#### FR-UTIL-002 — Data dictionary lookup
- **Requirement:** Programs SHALL use `D0325` for data-dictionary access.

#### FR-UTIL-003 — Print copy routine
- **Requirement:** Programs SHALL use `D0703` for print-copy formatting.

#### FR-UTIL-004 — DB2 connection / stored procedure
- **Requirement:** Programs SHALL connect to DB2 via `SYSPROC.DSCON03`.

### 3.8 Operational Surface

#### FR-OPS-001 — Batch orchestration
- **Requirement:** The platform SHALL be driven by JCL jobs (`ACME*J` family). `[SME]` identify the scheduler (CA-7 / Control-M / OPC) and the runbook.

#### FR-OPS-002 — REXX utility scripting
- **Requirement:** The platform SHALL use REXX execs (`FTEREPL` for file transfer, via `IKJEFT01` / `IKJEFT1B`) for utility steps.

#### FR-OPS-003 — Abend handling
- **Requirement:** Programs SHALL exit with `RETURN-CODE = 16` on file-status `'23'` / `'10'` failures (`XXCAD63` is the canonical example). The modernized system SHALL surface equivalent fatal-vs-recoverable distinctions.

---

## 4. Non-Functional Requirements

### 4.1 Performance (as observed)

| Metric | Legacy baseline | Source |
|--------|-----------------|--------|
| End-of-day batch window | `[SME]` | Operations |
| Top job runtime (`XXDL960` archival) | `[SME]` per cycle | Operations |
| CICS transaction `DLZB` p95 latency | `[SME]` | Operations |

**Requirement:** Modernized system MUST meet or beat each baseline.

### 4.2 Reliability

- **NFR-REL-001:** A JCL job failure MUST NOT silently corrupt downstream state (file status `'23'`/`'10'` → `RETURN-CODE = 16` triggers abend and re-run).
- **NFR-REL-002:** Deal-suppression decisions MUST be reproducible — same input ⇒ same output `DLS01-DEAL-SUPP-SW`.
- **NFR-REL-003:** Archival (`XXDL960`) MUST be idempotent for already-deleted rows.

### 4.3 Data Integrity

- **NFR-DATA-001:** Divisional master writes and corporate DB2 writes MUST stay reconciled (the modernized DAL must encode this as a transactional invariant; today it is enforced procedurally in batch).
- **NFR-DATA-002:** Cost basis-code mapping (`'F'`→`'ACME'`, `'1'`→`'CW'`, otherwise→`'SC'`) is canonical and MUST NOT be re-interpreted.
- **NFR-DATA-003:** Deal lifecycle state transitions MUST be auditable end-to-end (pending → active → billed → accrued → archived).

### 4.4 Security

- **NFR-SEC-001:** All program execution today runs under z/OS RACF identities. `[SME]` capture the RACF groups per business process.
- **NFR-SEC-002:** No PII surfaces in the corpus excerpts inspected. `[SME]` confirm. If confirmed, the modernized system inherits the constraint.

### 4.5 Maintainability

- **NFR-MAINT-001:** All DB2 access in the modernized system MUST go through a single data-access layer per business process.
- **NFR-MAINT-002:** Every business rule listed in `BP-*.md` MUST be expressible as a named, testable function in the modernized code.
- **NFR-MAINT-003:** Naming convention from the legacy (JCL `ACME*J` → COBOL `XX*P`) MUST be either preserved as-is or replaced by a documented modernised convention; mixed conventions are not acceptable.

### 4.6 Portability

- **NFR-PORT-001:** The Plan/Design phase decides target runtime (the corpus surfaces **no** target preference). Until then, no FR-* implementation may assume a host-specific feature.

---

## 5. Data Requirements

### 5.1 DB2 entities of record

| Table | Programs referencing | Spec section |
|---|---:|---|
| `ACME.DIVMSTRDI1D` | 124 | BP-001 |
| `DS.APPL_SYS_AP1P` | 64 | BP-006 / common |
| `ACME.UIN_ITEM_DE6C` | 64 | BP-001 |
| `ACME.DIV_ITEM_PACK_DE1I` | 62 | BP-001 |
| `ACME.VNDR_MSTR_VN1A` | 46 | BP-003 |
| `DEALDM1X` | 40 | BP-002 |
| `DS.APPL_SYS_PARM_AP1S` | 39 | common |
| `ACME.ITEM_VNDR_DE6V` | 35 | BP-001 / BP-003 |
| `ACME.PENDINGDEALSDM3P` | 31 | BP-002 |
| `ACME.ITEM_UPC_DE6Y` | 30 | BP-001 |
| `ACME.PROF_HDR_PR1P`, `ACME.PROF_CUS_PR3Q`, `ACME.PROF_DEAL_PR4D`, `ACME.DEAL_SUPP_DS1R`, `ACME.ITEM_SUPP_IT2A`, `ACME.ITEM_MASTER_IM3I`, `ACME.CUST_XREF_CU1X` | (suppression cluster) | BP-002 |

### 5.2 Divisional dataset families

For each division in `{MZ, SW, SZ, SE, NC, ME, NE, PA, NW, SO, MS, MK, MP, MI, MO, HP, MW, MD, MY, MN, WJ, FE, MG, NT, GM, GA, GF, WK, C1, C2, C3, ACME}` there are three master families:

- `<DIV>.MSTR.SIM` — divisional item master.
- `<DIV>.MSTR.VND` — divisional vendor master.
- `<DIV>.MSTR.CAD` — divisional **Computerized Allowance Data** master.

Plus per-job extract families like `<DIV>.PERM.<DIV>DL656{1,2,3}` (seen in `mcdl656j.md` graph extracts).

### 5.3 Sequential files

`NEWVNDRS`, `CORPVNDR`, `XXOUT`, `EXTRACT-FILE`, plus the `iefbr02` / `iefbr03` placeholder steps that consume per-division datasets in `MCDL656J`.

### 5.4 Data retention

- **Deal records:** 2 years live in `DEALDM1X` before `XXDL960` purges them.
- **Divisional masters:** retained indefinitely (system of record).
- **Logs / extracts:** `[SME]`.

---

## 6. External Interfaces

| Interface | Direction | Mechanism | Spec |
|---|---|---|---|
| Vendor cost/vendor uploads | Inbound | Sequential files (`NEWVNDRS`, `CORPVNDR`) | BP-003 |
| Buyer / merchant maintenance | Inbound | CICS `DLZB` transaction, screen `MCCSM55` | BP-006 |
| Deal extracts to BI | Outbound | Per-division extracts via `MCDL656J` | BP-002 |
| Deal-sales updates | Outbound | `XXDM713` output | BP-002 |
| Billing / AP feeds | Outbound | `XXDLC10` / `XXDLC20` accrual outputs | BP-002 |
| Price-protection notifications | Outbound | `XXRPR53` (email) | BP-004 |
| Vendor master sync | Both | `MCBSM52J` reconciliation | BP-003 |

---

## 7. System Constraints

### 7.1 Technical

- **Languages:** COBOL today; JCL; REXX. No PL/I, no IMS DB observed.
- **DBs:** DB2 primary; VSAM / sequential datasets for divisional masters.
- **Online runtime:** CICS.

### 7.2 Business

- Divisional model and deal lifecycle are non-negotiable.
- File contracts with downstream consumers are frozen until those consumers are migrated.

### 7.3 Regulatory

- `[SME]` confirm whether the platform falls under SOX scope (likely true given cost/billing impact).

---

## 8. Assumptions & Dependencies

### 8.1 Assumptions

- The 312 programs in the graph constitute the platform's complete in-scope codebase. `[SME]` confirm.
- The `ACME.*` DB2 schema is the only DB2 schema this platform owns. `[SME]` confirm.

### 8.2 Dependencies

- **CPE-Legacy** activation already complete for this opportunity (corpus and graph populated).
- **yourbuddy-hybrid** available for fact lookup during Plan / Design refinement.
- `[SME]` access for division-code translation and runbook walkthroughs.

---

## 9. Acceptance Criteria (Document-Level)

This SRS is **approved** when:

1. Every `[SME]`, `[CODMOD]`, `[MAT]`, `[RAG]` placeholder is replaced with a sourced fact.
2. Every FR-* links to a `BP-*.md` spec covering it.
3. The 32 divisional codes are translated to business-friendly names in HLD §5.
4. Merchandising IT owner, Deal Management product owner, and DBA have signed off via GitLab MR.

---

**Document Control**
- Author: Assess Agent (yourbuddy-hybrid + RAG + Spanner Graph) + Human SME (pending)
- Reviewers: Merchandising IT owner, Deal Management PO, DBA, Operations
- Approval gate: Required before HLD §6 program inventory is finalised.
