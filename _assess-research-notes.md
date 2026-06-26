# Assess Phase Research Notes (Source Data)

> Internal artifact — not part of the approved Assess deliverables. Captures the queries and findings used to ground `docs/legacy/`.

## Opportunity

- **Opportunity ID:** `006UV00000aTYw1YAG`
- **GCP project:** `qatest-lmp` (project number `962673027835`)
- **yourbuddy-hybrid (qatest):** `https://yourbuddy-hybrid-962673027835.us-central1.run.app`
- **Session ID used:** `yb_hybrid_1780385956_cursor-a`
- **RAG corpus resolved:** `projects/qatest-lmp/locations/europe-west4/ragCorpora/2227030015734710272`

## System Identification

The opportunity's CPE-Legacy corpus contains a **COBOL / DB2 / CICS merchandising platform** (Acme Company's customised Marwood Distribution System Release 3.3.0). The documentation under `docs/legacy/` describes the system that the RAG and graph data represent.

Domain evidence from the corpus:

- DB2 tables under the `ACME.*` schema (`ACME.DIVMSTRDI1D`, `ACME.UIN_ITEM_DE6C`, `ACME.VNDR_MSTR_VN1A`, `ACME.PENDINGDEALSDM3P`, `ACME.DEAL_SUPP_DS1R`, `ACME.PROF_HDR_PR1P`, ...).
- Divisional VSAM/dataset families `<DIV>.MSTR.SIM` (item master), `<DIV>.MSTR.VND` (vendor master), `<DIV>.MSTR.CAD` (cost & deal).
- Programs covering deals (XXDL*, XXDLS*, XXDLC*), price protection (XXRPR*), costing (MCCST*, XXCAD*), vendor (MCCST19, MCBSM52J, AUTHORKAY, AUTHORMCLANE), and an online CICS surface (transaction `DLZB`, screen `MCCSM55`).

## Graph Snapshot

| Metric | Value |
|---|---|
| Distinct COBOL programs | **312** |
| Distinct divisions (item/vendor/cost-deal master file prefixes) | 32 (`MZ, SW, SZ, SE, NC, ME, NE, PA, NW, SO, MS, MK, MP, MI, MO, HP, MW, MD, MY, MN, WJ, FE, MG, NT, GM, GA, GF, WK, C1, C2, C3, ACME`) |
| Most-referenced data entity | `ACME.DIVMSTRDI1D` (124 referencing programs) |

### Top 20 most-connected programs (callers + callees combined)

| Program | Edges | Program | Edges |
|---|---|---|---|
| `MCBSM04` | 36 | `MCBSM03` | 19 |
| `MCBSM02` | 26 | `XXRPR70` | 19 |
| `XXRPR55` | 26 | `D2122` | 18 |
| `XXRPR85` | 25 | `MCRPR99` | 18 |
| `D8050` | 22 | `XXRPR72` | 17 |
| `XXDL740` | 21 | `XXBSM50` | 17 |
| `MCCST50` | 20 | `XXDL655` | 16 |
| `MCCST24` | 19 | `MCBSM30` | 16 |
| `XXDLS54` | 19 | `XXDLS50` | 16 |
| —       | —  | `D2112` / `MCCST51` | 16 |

### Top 10 datasets / DB2 tables by referencing program count

| Entity | Programs |
|---|---|
| `ACME.DIVMSTRDI1D` | 124 |
| `DS.APPL_SYS_AP1P` | 64 |
| `ACME.UIN_ITEM_DE6C` | 64 |
| `ACME.DIV_ITEM_PACK_DE1I` | 62 |
| `ACME.VNDR_MSTR_VN1A` | 46 |
| `DEALDM1X` | 40 |
| `DS.APPL_SYS_PARM_AP1S` | 39 |
| `ACME.ITEM_VNDR_DE6V` | 35 |
| `ACME.PENDINGDEALSDM3P` | 31 |
| `ACME.ITEM_UPC_DE6Y` | 30 |

## Naming Convention (inferred from corpus)

| Prefix | Meaning (inferred) | Examples |
|---|---|---|
| `ACME*` (job) | **M**erchandising **C**orporate JCL job; usually `ACME***J` orchestrates an `XX***P` COBOL program | `MCCAD65J`, `MCDL656J`, `MCDLS50J`, `MCRPR55J` |
| `XX*` | COBOL program / procedure invoked by an `ACME*J` job | `XXCAD63`, `XXDL656`, `XXDLS01`, `XXRPR55` |
| `D*` | Utility / CICS / data-dictionary modules | `D0007` (extended error msgs), `D0325` (data dictionary), `D0703` (print copy), `D8050`, `D2122` |
| `MCBSM` / `XXBSM` | **B**usiness **S**ervices / business **M**oves; group-deal processing | `MCBSM04`, `MCBSM52J`, `XXBSM70` |
| `MCCST` / `XXCST` | **C**o**ST**ing — item costing, price protection inputs | `MCCST24J`, `MCCST50J`, `MCCST96J` |
| `MCRPR` / `XXRPR` | Price **PR**otection / reporting | `MCRPR55J`, `XXRPR55`, `XXRPR77P` |
| `MCDLS` / `MCDL` / `XXDL` / `XXDLS` | **D**ea**L**s / **D**ea**L** **S**ervices | `MCDL656J`, `XXDL960`, `XXDLS01`, `XXDLS54` |
| `MCCAD` / `XXCAD` | **C**ost-**A**nd-**D**eal | `MCCAD65J`, `XXCAD63` |
| `MCCBT` | Cigarette-deals batch processing | `MCCBT07` |
| `AUTHOR*` | Vendor-specific cost/deal handlers | `AUTHORKAY`, `AUTHORMCLANE` |

## Technical Stack (inferred from corpus)

- **COBOL** application programs (no PL/I evidence).
- **JCL** for batch orchestration; JCL job names use the `*J` suffix.
- **REXX** for TSO utility scripting; `IKJEFT01` / `IKJEFT1B` invoke REXX execs (e.g., `FTEREPL` for file transfer).
- **DB2** as the primary RDBMS; stored procedures observed include `SYSPROC.DSCON03`.
- **CICS** online surface; transaction `DLZB`, screen `MCCSM55`, `EXEC CICS RETRIEVE / XCTL / ENQ` constructs.
- **Sequential files** (e.g., `NEWVNDRS`, `CORPVNDR`, `XXOUT`, `EXTRACT-FILE`) and **VSAM** for divisional master families.
- **Batch scheduler** not surfaced in the corpus (`[SME]` to confirm CA-7 / Control-M / OPC).

## Key Business Rules Captured (Anchor Examples)

### XXDL960 — Deal Records Archival (`DEALDM1X` housekeeping)
- BR-DL960-01: A row is **eligible for deletion when older than two years**.
- BR-DL960-02: Age is determined by the latest of `DLBUYH` (purchase date), `DLINVH` (invoice date), `DLSHPH` (shipment date) vs. *today − 2y*.
- BR-DL960-03: Cursor-based read; per-row evaluation; no bulk delete.

### XXDLS01 — Deal Suppression Module
- BR-DLS01-01: `DLS01-DEAL-SUPP-SW` is initialised to `'N'`.
- BR-DLS01-02: Per-call cache (`WS-CUST-TABLE`, `WS-ITEM-TABLE`) is cleared when `WS-FIRST-SW='Y'` **or** division code changes **or** division part changes.
- BR-DLS01-03: Customer suppression is checked via `ACME.PROF_HDR_PR1P` + `ACME.PROF_CUS_PR3Q` + `ACME.DIVMSTRDI1D` + `ACME.CUST_XREF_CU1X` when `WS-CUST-SW(WS00-CUST-NUM)` is not yet `'Y'`.
- BR-DLS01-04: Item suppression is checked via `ACME.ITEM_SUPP_IT2A` + `ACME.ITEM_MASTER_IM3I` when customer is not suppressed and `WS-ITEM-SW(WS00-ITEM-NUM)` is not yet `'Y'`.
- BR-DLS01-05: Deal-specific suppression uses `ACME.PROF_DEAL_PR4D` + `ACME.DEAL_SUPP_DS1R` keyed by deal type, form of payment, and invoice date.
- BR-DLS01-06: DB2 errors map to documented return codes.
- BR-DLS01-07: Returned switch `DLS01-DEAL-SUPP-SW ∈ {'Y','N'}`.

### XXCAD63 — Item Cost Validation
- BR-CAD63-01: A vendor is **active** when `VNKY-RECORD-STATUS-CODE = 'A'`.
- BR-CAD63-02: Cost control is enabled when `VNRT-COST-CONTROL-FLAG = 'Y'`.
- BR-CAD63-03: A cost record is *found* when either `WS-CUR-COST-SW='Y'` **or** `WS-FUT-COST-SW='Y'`.
- BR-CAD63-04: `CDIC-LIST-AMT-TYPE = 'F'` → cost basis code `'ACME'`.
- BR-CAD63-05: `CDIC-LIST-AMT-TYPE = '1'` → cost basis code `'CW'`.
- BR-CAD63-06: any other `CDIC-LIST-AMT-TYPE` → cost basis code `'SC'`.
- BR-CAD63-07: Effective-date discrepancy between current and future records → discrepancy report.
- BR-CAD63-08: Cost-amount discrepancy between current and future records → discrepancy report.
- BR-CAD63-09: For exception items (`WS-EXC-ITEM-SW='Y'` **and** `WS-AP1R-SW='Y'`) discrepancies log `"ITEM EXCEPTION - NO UPDATES"` instead of failing.
- BR-CAD63-10: File status `'23'` or `'10'` → abnormal termination with `RETURN-CODE = 16`.

### XXDL702 — Active Billing Deals (excerpt)
- BR-DL702-01: Deal types `group`, `DVRTR`, `BRKT` are bypassed by the active-billing handler.
- BR-DL702-02: Active deal is treated as expired when current date > deal end.
- BR-DL702-03: For non-`'Y'` type deals with non-zero `ACEDL3` amount: copy start/end into current deal fields, add `ACEDL3` into `OUT-DEAL-CEDED-AMT`, mark a rewrite.

## Outstanding Targets (Modernization)

- The corpus **does not** state a target stack (Java/Postgres/GCP/AWS/Azure). The Plan phase will define source-to-target mapping.

---

## Round 2 — Targeted Deep-Dive (2026-06-02)

Re-queried the seven `[RAG]`-flagged programs with prompts that explicitly asked for Business-Level Description / paragraph lists / file-and-table inventories. Results:

### Naming corrections

- **CAD = "Computerized Allowance Data"** (per `mccbt07.md`), not "Cost-and-Deal" as inferred. Update HLD §3.1 and all BP specs.
- The `D*` prefix is **not monolithic**. `D0007` / `D0325` / `D0703` are true utilities, but `D8050` is a **CICS deal-capture polling transaction** belonging to BP-002 (deal lifecycle), and `D2122` has no corpus content.

### Per-program findings

#### MCBSM04 — Customer / SA_CORP processing orchestrator
- Paragraphs: `0000-MAIN`, `1000-INIT`, `1500-PROCESS-CUSTS`, `3000-SET-SA-CORPS`, `8000-FINALIZE`, `9900-DISPLAY-HEADING-LINES`, `9910-DISPLAY-ERR-MSG-LINES`.
- Touches 30 DB2 tables — mostly the `ACME.CUST_*`, `ACME.CLS_GRP_*`, `ACME.DIV_CUST_*`, `ACME.CD_VAL_*`, `ACME.*_VAL_*` clusters.
- **No COBOL callers** — JCL-launched leaf orchestrator. The 36 graph edges are **data-entity edges**, not COBOL-call edges.

#### MCBSM02 — Customer record processor (deletion candidate selection)
- Paragraphs: `0000-MAIN`, `1000-INIT`, `2100-MOVE-CUSTOMERS`, `7000-READ-INFILE`, `8000-FINALIZE`, `9920-ERR-HAND`, plus the standard heading/error paragraphs.
- Concrete rules:
  - **Excluded user IDs:** `'MCBSM02'`, `'XXEBM39'` (records authored by these user IDs are excluded from deletion selection).
  - **Cursor selection threshold:** `CREATE_TS` older than **45 DAYS**.
  - Loops until `WS-EOF = 'Y'`.
  - DB2 errors call `DBDB2ER` + `DSWTO`; critical errors → `ROLLBACK WORK` + `RETURN-CODE = 16`.
- Output: `PRINT-FILE` deletion report (program name, run timestamp, salesman ID, from/to division, create timestamp, table-affected flags).

#### XXDL740 — Deal Statistics Update Report + Error Messages Report
- Paragraphs reference external subroutines: `DATETIME` (system date/time), `UT516XP` (fiscal period calculator), `DC502YP` (date format conversion), `XXDC608` (division ↔ common layout converter), `DSNTIAR` (SQL error translator), `ILBOABN0` (force abend).
- VSAM-driven dynamic configuration: input VSAM files control which subroutines get called and how output is formatted.
- Cursors / DB2 tables: `DL00-CUR1` on `DEAL_ANALYSIS_DM1L` (exclusive lock), `DM1X-CUR1` on `DEALDM1X` (exclusive lock), `DEALLOGDM5X` (exclusive lock), `DI3X-CUR1` on `ACME.MCLANE_XREF_DI3X`, `GL-CUR` on `DS.APPL_SYS_PARM_AP1S`.
- Files: `DT-FILE` (Optional Date Reader), `AP-FILE` (Vendor Master AP File, random by vendor #), `BY-FILE` (Buyer Number File, random by key), `SI-FILE` (SIM Master File, dynamic), `PMSG` (DB2 error messages), `QE-FILE` (error report).
- DCLGENs: `DGAP1S`, `DGCU2B`, `DGDE1E`, `DGDE1I`, `DGDE6C`, `DGDI1D`, `DGDI3X`, `DGDM5X`, `DGDS1H`.
- Has wait-and-retry logic for exclusive-mode locks.

#### XXDLC10 — Deal Transaction Aggregation and Output
- Paragraphs: `00000-MAIN-PROCESSING`, `A0000-INITIALIZATION`, `B1000-PROCESSING`, `S1000-FINALIZATION`, `E1000-FAILED-OPEN`.
- Output: `OUTDEAL` file.
- Inputs: "audit transaction records" + temp table for staging.
- Loops until `SW-EOF-OF-TABLE = 'Y'`.
- Queries `DEALDM1X` with a complex aggregation/join; constructs a unique item key.
- `E1000-FAILED-OPEN` displays `FAILED-FILE` + `FAILED-STATUS`; **`MOVE +16 TO RETURN-CODE` is commented out** (an interesting half-implemented abend behaviour worth confirming with SME).
- `S1000-FINALIZATION` displays timestamp + division + total OUTDEAL records written + total `DEALDM1X` records fetched.
- Tables: `DEALDM1X`, `ACME.DIVMSTRDI1D`, `ACME.DT_JS1A`.

#### XXDLC20 — Deal Accrual Processing
- Reads "end-of-processing date" from input file.
- Creates a temporary database table to stage intermediate accrual data.
- Queries `DEALDM1X` joined with the temp accrual table + other sources, aggregated by **deal part, item number, deal ID**.
- Tables: `DEALDM1X`, `ACME.DIVMSTRDI1D`, `ACME.DT_JS1A`.

#### XXRPR55 — Price Protection Modeler Report (request type `'BI1'`)
- Paragraphs: `0010-START-PROCESSING`, `0010-END-PROCESSING`, `1000-INIT-PARA`, `1100-OPEN-FILES`, `2000-PROCESS-PARA`, `2200-FORMAT-OUTREC`, `2300-GET-PRICE-AMT`, `2350-XXDIV50-CALL`, `4100-WRITE-OUTREC`, `4400-GET-GROUP`, `5000-SET-DB2-PACKAGE`, `5050-RESET-DB2-PACKAGE`.
- Rules:
  - Filters PP requests where request type = `'BI1'`.
  - Writes output record only **if either price is not zero**.
- Output files: `ACME.PERM.RPR55S1` (main report), `ACME.TEMP.RPR55S2` (email notifications).
- Caller: `MCRPR55J`.

#### XXRPR85 — Price Protection Modeler request processor (request type `'BI2'`, status `'PND'`)
- Filters: request type = `'BI2'`, status = `'PND'`.
- **Status transitions:** `'PND'` → `'INP'` (In Progress) → `'CMP'` (Completed).
- Extracts rule details + customer + item + vendor data by joining many tables; writes consolidated report records.
- Caller: `MCRPR85J`.

#### **PP DB2 schema (newly surfaced)**
Across XXRPR55/85 callees:
- `ACME.PP_RULE_PP1R` — price-protection rules.
- `ACME.PP_RQST_PP3C` — PP requests.
- `ACME.PP_CUST_PP1C` — PP customers.
- `ACME.PP_ITEM_PP1I` — PP items.
- `ACME.PP_CUST_ITEM_PP2C` — PP customer-item linkage.
- `ACME.PP_CUST_GRP_PP1G` — PP customer groups.
- `ACME.PP_ALLOC_PP2A` — PP allocation.
- `ACME.PPCUSTITEMAUD_PP7C` — PP customer-item audit.
- `ACME.PPA_MTHD_PP9M` — PP allocation method.
- `ACME.STAT_PP9S` — PP status master.
- `ACME.CST_CMPNT_TYP_PP9C` — cost component type.
- `ACME.CDATRBTVAL_CD_CU4G` — cost-data attribute values.

#### MCCST50 — **CICS MQ transaction MCS4** for `'IC1X'` cost-update messages
- **Reclassification:** Not batch costing. It is a CICS online transaction triggered by MQ messages.
- Paragraphs: `a0000-initialize`, `a2005-get-q-mssg`, `a2007-verify-de9e-data`, `a2009-verify-de6e-data`, `a3000-mqclose`, `a4000-error-message`, `a4100-no-data`, `a6000-return-to-cics`, `b1100-get-max-retry`, `c1000-open-item-detail-cur`, `c1500-fetch-item-detail-cur`, `c2000-process-catlg-num`, `c2005-build-ic1x-base-rec`, `c2010-create-cost-record`, `c2015-create-element-records`, `9000-process-error`, `9100-process-error`.
- Rule: When `COMM-Q-TYPE = 'IC1X'`, `WS-TRANSID := 'MCS4'`, `WS-PROGRAM := 'MCCST50'`.
- Tables touched: `ACME.ITM_COST_CNTL_DE9E`, `ACME.ITM_BILL_COST_DE6E`, `ACME.ITM_COST_DE8E`, `ACME.ITEM_GRP_DE1E`, `ACME.ITEM_VNDR_DE6V`, `ACME.UIN_ITEM_DE6C`, `ACME.VNDR_MSTR_VN1A`, `ACME.COMNT_CM4A` (error logging), `DS.APPL_SYS_AP1P`, `DS.APPL_SYS_PARM_AP1S`.
- Caller: `MCCST55` (the dispatcher).

#### MCCST51 — **CICS MQ transaction MCS5** for `'CADMF'` cost-update messages
- Sister to MCCST50: when `COMM-Q-TYPE = 'CADMF'`, `WS-TRANSID := 'MCS5'`, `WS-PROGRAM := 'MCCST51'`.
- Error logs into `ACME.CMN_ERR_CM5A` and `ACME.COMNT_CM4A`.

#### MCCST24 — Future Cost Changes Extract (batch)
- Paragraphs: `0010-MAIN-PROCESSING`, `0010-END-PROCESSING`, `1000-START-UP`, `1100-OPEN-FILES`, `1200-DELETE-ITM-TBL`, `2000-PROCESS`, `3000-WRAP-UP`, `3100-CLOSE-FILE`, `3200-EJOB-CONTROL`, `4000-UPDATE-AP1S-PARA`.
- Input file: `EDISENT`. Output: `MCCST24O` (comma-delimited extract).
- **Checkpoint pattern:** Reads `MCCST24_LAST_RUN` from `DS.APPL_SYS_PARM_AP1S` to determine date-range filter; on successful completion updates `MCCST24_LAST_RUN` to current timestamp.
- Uses temporary table `TEMP.ITEM_BILL_COST_T356`.
- Tables: `ACME.DIVMSTRDI1D`, `ACME.ITEMAUTHST1A`, `ACME.ITMCOSTCNTLAUDDE9A`, `ACME.ITM_BILL_COST_DE6E`, plus the temp table.
- Caller: `MCCST24J`.

#### D8050 — **CICS deal-capture polling transaction (BP-002, not utilities)**
- **Reclassification:** Not a utility. This is the deal-lifecycle bridge from pending to active.
- Behaviour: polls `DM3P` (Pending Deal table) periodically. For each pending deal, branches by deal type:
  - Divisional → retrieves from `DM3D` (Divisional Pointer).
  - Group → retrieves from `DM3G` (Group Pointer).
- Validates: overlapping deal dates, item status, division, licensing limits, corporate vendor control, existing terminated deals.
- Errors logged to `DM3E` (CAD Error Table).
- Successful deals inserted into `DM1T` (Deal Transaction table).
- Restart file `DM1E` maintained for recovery (sequence number `DM1E-SEQNCE`).
- Calls `CM510ZO` for processing-data load.

#### MCCBT07 — Cigarette Deals Batch (CAD update)
- Two-program pipeline: `MCCBT06` → `MCCBT07`.
- **Rule:** Read `BATCHID` from `RDRPARM`; if not numeric → **ABEND**.
- **Rule:** Query `ACME.BAT_DEALERRLOGDM3B` for rows with `ERR_TYP = 'F'` (Fatal) for current `BATCH_ID`; if any present, surface informational message that batch has fatal errors.
- Reads sorted `EXTRACT-FILE` from MCCBT06.
- Updates: `ACME.PENDINGDEALSDM3P`, `ACME.DIVPENDDEALSDM3D`, `ACME.DEALITEMDM2I`, `ACME.DEALDM1M`, `ACME.CAD_REMARK_DM3R`, `ACME.ITEM_UPC_DE6Y`.
- Status updates: `ACME.BAT_DEAL_HDR_DM3H`, `ACME.BAT_DEAL_DM3L`.

#### XXDM713 — Deal Sales Update
- **Corpus has no description** (`xxdm713.md` is empty / boilerplate only). Documented as `[CORPUS-EMPTY]`.
- Graph reveals it interacts with: `ACME.INVC_DTL_COMN_BD1D`, `ACME.INVC_DTL_ITEM_BD2D`, `ACME.INVC_HDR_BD1H`, `ACME.INVC_TS_BD2T`, `ACME.CUST_XREF_CU1X`, `ACME.DIVMSTRDI1D`, `ACME.US_TAX_CU4U`, `DATECNTLCF1D`.
- **Revision needed:** earlier draft assumed XXDM713 reads `ACME.PERM.BSM30S1.DWSLS`. The graph shows it actually reads the **`ACME.INVC_*` invoice family** (BD = Bill Detail). BP-005 needs to be revised.

#### D2122, D2112, XXRPR70, XXRPR72, XXRPR77P, MCCST51-internals
- **D2122:** `[CORPUS-EMPTY]`.
- **D2112, XXRPR70, XXRPR72, XXRPR77P:** still `[RAG]` (not re-queried in this round; lower-priority).

### Insight that changes how we read the graph

Edge counts in the dependency graph mix **COBOL→COBOL call edges** with **COBOL→DB2-entity edges**. `MCBSM04`'s "36 connections" is mostly data references, not call chains. Top-edge ≠ top-of-call-tree. The Plan phase should consume the edge data with this distinction in mind.

---

---

## Round 3 — CAST Analysis Import (2026-06-03)

Loaded the CAST static-analysis export `mcl_cast.zip` (snapshot **2026-05-21**, 12 JSON files, ~45 MB). CAST sees the source tree directly — no retrieval ranking, no chunking, no boilerplate noise. The full inventory and the verbatim CAST tables live in [`_assess-cast-summary.md`](./_assess-cast-summary.md); this section captures only the corrections it forces on the docs.

### Project identity confirmed

- **Project codename: `ACME`** (source path prefix `acme-code/`; CAST upload root `C:\Cast\cast-node\common-data\upload\ACME\…`). Likely Acme-related given `AUTHORMCLANE` in the corpus, but `[SME]` to confirm whether `ACME` is the customer name or an internal project code.
- **Source tree:** seven SCLM PDS libraries — `acme.perm.jcl`, `sw.perm.jcl`, `ds.perm.proclib`, `DB2P.PERM.DCLGEN`, `sclm.perm.prod.copy`, `sclm.perm.prod.copyproc`, `sclm.perm.prod.source`.

### Inventory corrections

| Quantity | Round-2 RAG view | CAST view |
|---|---:|---:|
| COBOL executables (top-level) | "312 programs" | **311** (= 123 Batch + 143 Program + 45 Transactional) |
| COBOL CopyBooks | not surfaced | **612** |
| Cobol File Links | not surfaced | 183 |
| JCL Jobs | not surfaced | **62** |
| JCL Procedures | not surfaced | 136 |
| **CICS Maps (screens)** | 1 (`MCCSM55`) | **56** |
| CICS Transactional COBOL Programs | implicit | **45** (41 `D21**` + `D8050` + `MCCST50/51/55`) |
| KSDS VSAM Files | not surfaced | **139** |
| ESDS VSAM Files | not surfaced | 3 |
| GDG Datasets | not surfaced | 58 |
| PDS Datasets | not surfaced | 173 |
| JCL Data Sets | partial | **889** |
| Missing Tables (referenced but not defined in scope) | partial | **220** |
| Cobol Paragraphs | not surfaced | **34,405** |
| **Total LoC** | not surfaced | **378,615** |

### Architectural reclassifications

1. **The `D21**` family (`D2105` … `D2199`, 41 programs) is the deal-domain CICS online surface**, not utilities. Programs `D2112`, `D2122`, `D2138` were on my `[RAG]`/`[CORPUS-EMPTY]` list — they're actually CICS transactional programs.
2. **`MCRPR50J` is the platform's highest-complexity job** (CC=553, LOC=20,938, ObjectCount=919). It outranks `MCRPR55J` and `MCRPR85J`, which the RAG round elevated as PP anchors.
3. **PP cycle is 13 jobs**, not the 4 I documented: `MCRPR50J, 55J, 56J, 58J, 59J, 65J, 70J, 71J, 74J, 77J, 85J, 86J, 99J`.
4. **Daily / Weekly / End-of-Period deal cycles** (`XXDLSDLY`, `XXDLSWKY`, `XXDLSEOP`) are top-5 complexity — they are the temporal heartbeat of the deal lifecycle; RAG never surfaced them.
5. **`MCCIC*J` (4 jobs) and `MCM9*J` (2 jobs)** are new business families not previously documented. `[SME]` to identify.
6. **`SW*` JCL jobs** (`SWCST64J`, `SWDEI60J`) live in `sw.perm.jcl` — division-`SW`-specific (or "Software"?) batch path.
7. **`DBDB2ERL` is the canonical DB2-error copybook**, 241 call sites — the single most-referenced shared artifact in the platform.
8. **`DCS*` copybook family** is a 12+-member cross-cutting library used by hundreds of programs — second only to `DB*` in cross-cutting use. `[SME]` to expand "DCS".
9. **`DG*` copybook family** is DCLGENs (DB2 column descriptors) generated from the DB2 catalog and stored in `DB2P.PERM.DCLGEN`. Modernizing the DB2 schema means regenerating these.
10. **`DS.EMER.LINK` and `DS.PERM.LINK`** are the **DB2 plan-linkage members** (emergency and permanent), referenced from every SQL-using job. They are not domain data — they are infrastructure. The earlier RAG ranking that put `DIVMSTRDI1D` at the top was directionally right; CAST confirms `DIVMSTRDI1D` is the most-referenced domain table (229 call sites; **7,465 SELECT transactions** with zero writes — pure RO reference).

### Data-access pattern (new, only available from CAST)

CAST's per-(transaction, data-source) row gives SELECT / INSERT / UPDATE / DELETE counts. Highlights:

- `PP_RQST_PP3C` — **1,192 UPDATEs** vs 760 SELECTs and 56 DELETEs — the most heavily mutated table on the platform. The PP request lifecycle is the platform's primary write-load contributor.
- `PP_CUST_ITEM_PP2C` — 122 INSERTs + 664 UPDATEs + **184 DELETEs** — the highest insert+delete churn; PP customer×item linkages are continuously reshaped.
- `DIVMSTRDI1D`, `DIV_ITEM_PACK_DE1I`, `UIN_ITEM_DE6C`, `VNDR_MSTR_VN1A`, `CUST_XREF_CU1X`, `CLS_GRP_DESC_CU2B`, `DT_JS1A`, `SYSDUMMY1` — **near-zero or zero write traffic** — pure RO reference tables. Excellent caching candidates in any modernized DAL.
- `APPL_SYS_PARM_AP1S` — 496 UPDATEs (vs 2,040 SELECTs) — confirms the **checkpoint pattern** observed in `MCCST24` is industry-wide across batch jobs.

### Dead-code candidates

- **201 unreferenced `JCL Included` members** (e.g., `//INCLUDE` snippets) — first batch of removal candidates for the Plan phase.

### Why CAST + RAG together is the right model

| Question shape | Best source |
|---|---|
| "What does this program do business-wise?" | RAG (yourbuddy) — natural-language descriptions. |
| "How many of X exist?" | CAST — exact counts. |
| "What's the call/data graph?" | CAST — relations files, no chunking loss. |
| "What rules / paragraph names / cursor names live in this program?" | RAG with **anchor-prompt** technique (round-2 lesson). |
| "Which DB2 table has the most write churn?" | CAST — SELECT/INSERT/UPDATE/DELETE counts. |
| "Which programs reference X?" | CAST — exact, complete. |
| "What's *all* the dead code?" | CAST — `Unreferenced Objects` report. |

The next Plan phase should consume both — CAST for the deterministic structural facts, RAG for the explanatory business prose.

---

---

## Round 4 — Targeted RAG Deep-Dives Driven by CAST Complexity (2026-06-03)

Eight prompts issued (Q4.1–Q4.8 in `_assess-prompts.md`) against a fresh session (`yb_hybrid_1780479025_cursor-a`), prioritised by CAST complexity ranking and by the new business families CAST surfaced. Below: the high-impact findings.

### **Headline finding: the platform is a customised Marwood Distribution System (REL 3.3.0)**

- The `DCS*` copybook family — second-most-referenced cross-cutting layer after the DB2 helpers — uses headers that explicitly say `"MARWOOD CONTROL AREA"` (DCSMCA) and `"MARWOOD SOURCE BOOK DELIMITER PROGRAM"` (DCSWORK).
- Copybook metadata records `SYSTEM RELEASE LEVEL.......: 3.3.0` and `MWD CONVERSION TO 3.3.0 RELEASE`. So **the platform sits on Release 3.3.0 of Marwood Distribution System**, a packaged COBOL/CICS distribution-management product.
- This **reframes the modernization** from "modernize a bespoke mainframe app" to "replace a customer-customised, 30-year-old packaged product." Different problem statement — Plan phase needs to know whether the target is (a) lift-and-shift Marwood, (b) replace Marwood entirely, or (c) extract just the customer-specific business rules and re-implement around a new core.
- DCS copybook roles documented (BP-006 + HLD §7 will be updated):
  - `DCSWORK` — system work fields, common flags, CICS attention key names, COMX (Communication eXit) calling codes (`COMX-LINK`, `COMX-XCTL`, `COMX-RETURN`, `COMX-CODE`, `COMX-PROGRAM`).
  - `DCSMCA` — Marwood Control Area: system data, sequencing fields, program names, comm/security (passed program-to-program in CICS).
  - `DCSHVNDP` / `DCSKVNDP` — Host / Key Vendor Parameters (vendor master access; key conversion).
  - `DCSHITMP` — Host Item Parameters (item master access).
  - `DCSFWHS` — File WareHouSe (warehouse file layout).
  - `DCSFOPR` — File OPeRator (operator/buyer info).
  - `DCSFERR` — File ERRor (error message handling).
  - `DCST26` — file set table (additional Marwood copybook seen in `d2116.md` source).
- Implication for `DBDB2ERL` (241 call sites): this is the **DB2 access pattern Marwood prescribes**, not bespoke customer code.

### **`MCRPR50J` (CC=553) — full pipeline characterised**

- 6-program PP pipeline: `XXRPR50` (intake) → SORT1 → `XXRPR51` (cost apply) → SORT2/SORT3 → `XXRPR52` (rule mgmt) → `XXRPR53` (notify) + `XXRPR54`, `XXRPR57`.
- **New filter constant:** XXRPR50 processes **request type `'2CE'`** (distinct from XXRPR55's `'BI1'` and XXRPR85's `'BI2'`).
- Cost types applied by XXRPR51: **License Cost, Billing Cost, Invoice Price**.
- Intermediate datasets: `ACME.PERM.RPR50S1`, `ACME.PERM.RPR50S1.SRT`, `ACME.PERM.RPR51S1`, `ACME.PERM.RPR51S1.SRT`, `ACME.PERM.RPR51S3.SRT`.
- Status logs: `ACME.PERM.RPR50.STAT`, `ACME.PERM.RPR51.STAT`, `ACME.PERM.RPR52.STAT`, consolidated to `ACME.PERM.RPR.STAT` for XXRPR53 email notifications.
- Rule files: `ACME.PERM.RPR73RUL`, `ACME.PERM.RPR72RUL`; item/customer files: `ACME.PERM.RPR71S1.ITEMS`, `ACME.PERM.RPR71S1.CUSTS`.
- Sort params live in `DS.PERM.SORTPARM(XXRPR501/511/512/71I/71C)`; email config in `DS.PERM.RDRPARM(RPREML53)`.

### **`MCBSM01J` (CC=486) — 8-step business-moves / license / salesman-ID pipeline**

- Steps: `MCBSM01P` (extract business moves) → `MCBSM02P` (process + print) → `MCBSM03P` (stores-to-inactivate / sales-corp extract) → `MCBSM04P` (process + print) → `LICPRT` (`SORT` for license backup, `COND=(4,LT)` gated) → `MCBSM05P` (reclamation transfers) → `MCBSM07P` (more reclamation) → `XXBSM20P` (salesman ID create/delete).
- Key datasets: `DS.PERM.MCBSM04.LICDATA`, `ACME.PERM.LICENSE.DAILY` (read + written), `DS.PERM.SORTPARM(SORTCOPY)`.
- MCBSM01 filters by **date/time conditions** + **attribute values** + **exclusion criteria on business-move status flags**.

### **`XXD2137J` (CC=416) — D2137 is a BATCH PURGE PROGRAM, not CICS**

- Important reclassification: despite the `D21**` family being CAST-tagged "Cobol Transactional Program", `D2137` is described in the corpus as a **"Batch purge program for Cost/Deal and PO files"**. The `D21**` family is **mixed batch + CICS**, not pure CICS as round-3 HLD §3 claimed.
- D2137 flow: parameter cards → housekeeping → Cost/Deal record loop (per-record keep/purge decision) → end-of-job summary.
- Files: `READER1` (params), `KCADX2` (main Cost/Deal), `KPODTX1` (PO Detail), `PRTFILE`, `HISTOUT` (history tape), `HISTDSK` (history disk), `LOADDSK`.
- Subroutines called: `D0173R` (date manipulation), `DCSHITMB` (item access), `DCSHVNDB` (vendor access), `D0402BR` (warehouse options).
- Purge rule: deletion-date check on the record's date vs a calculated deletion date.
- **`D2133.md` (sister program)** described as: "ensure that the item master file accurately stores the latest cost and deal information" — confirms the `D21**` family is **cost/deal management** at its core.

### **Temporal cycles (XXDLSDLY/WKY/EOP)**

- `XXDLSDLY` (Daily, CC=375) and `XXDLSWKY` (Weekly, CC=414) have **no direct corpus document**. Their behaviour must be derived from CAST graph + SME interview.
- `XXDLSEOP` (End-of-Period, CC=343) details were surfaced via `mcdl656j.md`:
  - Processes **historical deal data for end-of-period analysis**.
  - Calls `XXDL656P` running `XXDL656` per division.
  - Categorizes data into 3 historical buckets: "previous period / current fiscal year", "previous periods / current fiscal year", "previous period / previous/current year".
  - **Explicitly excludes current period and current year data**.
  - Outputs per-division files like `WJ.PERM.WJDL6563`, then sorted/aggregated by buyer and vendor.
  - Cleanup via `IEFBR14`.

### **Purge cycle (from `__name.md` source — name `[RAG]`)**

- A JCL job (name not captured) orchestrates the purge chain:
  - `XXDL995P` → purges `DEALLOGDM5X` (deal log records).
  - `XXDL960P` → purges `DEALDM1X` (main deal table — confirms BP-002 BR-002-20..23).
  - `XXDL980P` → purges `DEALAUDITDM1A` (deal audit records, **new entity**).
  - `XXDL571P` → produces audit data input to `XXDL980P`.
- The purge job also generates "report on deals with upcoming or recent buy/ship dates" + deal-alert report + deal performance report for current fiscal year.

### **CIC family = "Cigarette Cost" (Corporate Cigarette Cost System)**

- **CIC ≠ Customer Information Center** (my round-3 guess). CAST `MCCIC*` jobs handle **cigarette cost management**.
- `MCCIC01J` — processes cigarette cost change transactions; splits by divisional GL codes into division-specific files.
- `MCCIC02J` — consolidates divisional report files; prints exception report.
- `MCCIC20J` — generates detailed cost component report for linked cigarette items. Reads: `ACME.MCLANE_XREF_DI3X`, `ACME.UIN_ITEM_DE6C`, `ACME.ITEM_UPC_DE6Y`, **`ACME.CIG_ITEM_COST_PR1C`** (new DB2 table — Cigarette Item Cost), `ACME.DIVMSTRDI1D`.
- `MCCIC90J` — cleanup/maintenance for "non-maintainable" and "default" item groupings; inserts new cigarette items, deletes non-cigarette/inactive items, cleans orphans.
- The platform actually has **TWO cigarette families**:
  - `MCCBT*` (Cigarette **Batch** Transactions) — `MCCBT06`/`07` for cigarette deal batch + `MCCBT03J` for cigarette billing (or general billing — `[SME]`).
  - `MCCIC*` (Cigarette **Cost**) — cost management of cigarette items.

### **MCM9 family = data-integrity / reconciliation**

- `MCM9014J` calls `M901401` + `M901402`: identifies discrepancies between `XX.DEALDM1X` and PowerBuilder CAD tables for multiple corporate entities; outputs "Corporate Discrepancy Report".
- `MCM9091J` calls `M9093` per division: identifies items in divisional files lacking cost records in CAD VSAM; constructs the missing cost records; validates items against SIM files.

### **Quasar = downstream price-change distribution system**

- New external system: **Quasar** — receives price-change records from this platform.
- `MCRPR65J` (a PP cycle job not previously documented) generates the master price-change file from DB2 and splits into divisional files for Quasar consumption.
- Per-division datasets feeding Quasar: `<DIV>.PERM.<DIV>ST2A.PP` (e.g. `MS.PERM.MSST2A.PP`, `MN.PERM.MNST2A.PP`, `GM.PERM.GMST2A.PP`, `MP.PERM.MPST2A.PP`, `MW.PERM.MWST2A.PP`, `MG.PERM.MGST2A.PP`, `C3.PERM.C3ST2A.PP`). Confirmed for at least 7 divisions; presumably all 32.
- Implication: the ACME ↔ Quasar boundary is a **fixed external interface** to preserve in modernization.

### **`ACME.PERM.BSM30S1.DWSLS` consumer enumeration (BP-005 unknown closed)**

- `MCDLS50J` reads `ACME.PERM.BSM30S1.DWSLS` via its **`SORTSUM`** step (a sort-summary aggregation).
- Producer programs **still not surfaced** by corpus — `[CODMOD]` to derive from CAST relations.
- This corrects BP-005's earlier framing: DWSLS feeds **deal profitability analysis** (via XXDLS50, which `mcdls50j.md` describes as finding profitable deals using future expiration dates, unit deal amounts, deal types).

### Other miscellaneous side-finds from round 4

- **`MCRPR59J`** — produces summary statistics on "records processed, inserted, not inserted, and deleted" (NEW PP job documented).
- **`SWDEI60J`** — processes sales/item data, updates system tables, generates extract files (`SW*` JCL confirmed as sales/item processing, not site-specific).
- **`MCCBT03J`** — "process customer billing data, generates customer billing statements and updates customer account" — so `MCCBT*` is not exclusively cigarettes; this one is general customer billing. **Family is mixed**; `[SME]` to map each MCCBT** job to its function.
- **`MCBSM52J`** detailed join chain (read in side-data, not asked): cursor over `TEMP.VNDR_ID_DIV_T312` joined with `ACME.DIV_VNDR_XREF_VN1Y`, `ACME.VNDR_MSTR_VN1A` (vendor names), `ACME.BUYR_MSTR_VN4B` (buyer IDs) → comma-delimited `ACME.PERM.BSM55S1` (from/to divisions, buyer IDs, vendor IDs, cleaned vendor names with special chars replaced).

### Open after Round 4

- `MCRPR58J`, `MCRPR59J`, `MCRPR74J`, `MCRPR86J` deeper retrieval (some surfaced sideways).
- Daily/Weekly XXDLSDLY / XXDLSWKY — `[CORPUS-EMPTY]` for direct docs; need SME or codmod re-run.
- Full enumeration of the 41-program `D21**` family by batch vs CICS (D2137 batch, D8050 CICS — at least these are mixed).
- `DCSCOMX`, `DCSLINK` as standalone copybooks not found; `COMX-` prefix is shared across DCSWORK/DCSMCA.
- 56 CICS Maps enumeration (`[CAST]` — extract from inventory file with `ObjectType='Unknown CICS Map'` filter that works on the schema).
- Marwood vendor identity / current support status — is there still a Marwood support contract? Is REL 3.3.0 EOL? `[SME]`.

---

---

## Round 5 — Mixed CAST-deterministic + Targeted RAG (2026-06-03)

Round 5 combined **CAST relations queries** (deterministic structural answers) with 5 RAG prompts to close the highest-value remaining gaps.

### 5a. CAST relations queries (deterministic)

#### DWSLS producer identified

`ACME.PERM.BSM30S1.DWSLS` chain (from `MCL_Relation_between_Objects_And_DataSources` + `…DataSources_And_Transactions`):

| Object | Type | Role |
|---|---|---|
| `MCBSM50J` | JCL Job | **Producer** — owns the DWSLS write per `AccessLinkTypes: ['CALL ', 'DEFINE ', 'WRITE ']` |
| `XXBSM32P` | JCL Procedure | Invoked from MCBSM50J |
| `XXBSM32` | Cobol Batch Program | The actual COBOL that writes DWSLS |
| `5000-WRITE-DATA` | Cobol Paragraph | The specific paragraph that does the WRITE |
| `MCDLS50J` | JCL Job | **Consumer** (round 4 finding — reads DWSLS via `SORTSUM` step) |

This closes BP-005 BR-005-07.

#### D21** family runtime classification — CAST evidence

A key round-4 conclusion needed sharpening. CAST data shows:

- The **45 "Cobol Transactional Program"** entries in the inventory are `D2105–D2199` (41 progs, with gaps) + `D8050` + `MCCST50/51/55`. **None** of them have a JCL Job as caller per `MCL_Relation_between_Transactions_And_Objects` — so they are **genuinely CICS-only** transactional programs (started by CICS transactions, not by JCL EXEC).
- `D2137` is a **separate** CAST entry tagged `Cobol Batch Program` (source `D2137.cbl`), invoked by `XXD2137J` (the CC=416 batch purge job from round 4). It is **not** in the "Cobol Transactional Program" group.
- Likely the same is true of `D2133` (round 4: "ensures item master file accurately stores the latest cost and deal information").
- Conclusion: **the D21NN number range is shared by two distinct program populations** — the CICS-only application family (~41 programs) and a handful of same-numbered batch programs (D2137, D2133, possibly others). Round-4's "mixed batch+CICS within a single family" wording is technically wrong; each individual program is one or the other, not both.

D21** programs that reference at least one DataSource per CAST (21 of 41): `D2137` (9 refs — likely the batch one), `D2122` (6), `D2112` (4), `D2118` (3), `D2128` (3), `D2184` (2), `D2135` (2), `D2171` (2), `D2119` (2), `D2131` (2), `D2142` (2), `D2101` (2), `D2178` (1), `D2136` (1), `D2174` (1). The other 20 D21** programs have **zero direct DataSource references** in CAST — they delegate all I/O to the DCS file handlers (DCSHVNDP/HITMP/etc.), and the data access shows up against those handlers, not the calling program.

#### CICS Maps enumeration — not in the inventory file

The 56 CICS Maps shown in `MCL_Lines_of_Code_Per_Technology_Artifact_*` do **not** appear in `MCL_Added,_Modified_or_Deleted_Objects` under any `ObjectType` containing "Map". They likely live in a CAST-internal table not exported. The COBOL source does reference maps via paragraphs containing `MAP` in their names (127 Cobol Paragraph rows; e.g. `SUPPRESS-ENTRY-MAP1`, `PROTECT-MAP1-ENTRY`, `ERRX-CICS-MAPFAIL`, `ERRX-RESEND-MAP1`) — these confirm at least `D2146`, `D2147`, `D2149`, `D2157`, `D2161`, `D2162`, `D2199` host CICS map I/O logic. The actual BMS mapset names (~8-char) would need to be parsed from `EXEC CICS SEND MAP / RECEIVE MAP` statements in the COBOL — `[CODMOD]` follow-up.

### 5b. RAG prompts Q5.1 – Q5.5

#### Q5.1 / Q5.2 — Remaining MCRPR* jobs

- **`MCRPR58J`** — *Nightly PP rule application*. `XXRPR58` reads date param `%%C1-%%M1-%%D1` from `READER`; filters DB2 by date; inserts new effective configurations into **`ACME.PRC_CMPNT_ITM_ST3B`**; conditional insertion (skip duplicates → counted as "not inserted"); RC=16 on critical errors. Other tables: `ACME.PRC_CMPNT_ITM_ADTC` (audit), `ACME.PRC_COMP_DTC`, `ACME.PRC_CMPNT_ITM_PR1C`.
- **`MCRPR59J`** — *Active → History archival for PP requests*. `XXRPR59` reads retention days from `DS.PERM.RDRPARM(RPR59R1)`; calculates `target_date = today - retention_days`; selects `ACME.PP_RQST_PP3C` rows with status `'CMP'` or `'INP'` and `CMPLTN_TS < target_date`; inserts into **`ACME.PP_RQST_HIST_PP3H`** then deletes from active. Duplicate-key on history insert ⇒ do NOT delete from active (rollback).
- **`MCRPR74J`** — *2-step billing-to-allocation sync*. Step 1 `XXRPR74` reads `ACME.INVC_HDR_BD1H` + `ACME.INVC_DTL_COMN_BD1D`; updates **`ACME.PP_ALLOC_DTL_PP4D`**, **`ACME.PP_ALLOC_SUM_PP4S`**, **`ACME.PP_ALLOC_HIST_PP4H`**. Step 2 `XXRPR75` synchronises customer-specific pricing allocations with billing data; aggregates to customer group level. JCL enforces step ordering.
- **`MCRPR86J`** — *ST3B purge*. `XXRPR86` deletes outdated/invalid PP records from `ACME.PRC_CMPNT_ITM_ST3B` (paired with MCRPR58J's writes).

#### Q5.3 — Purge cycle parent JCL identified (6 steps)

A single JCL job (CAST normalized as `__name.md`, so the actual JCL job name is `[CAST-NORMALIZED]` — `[SME]` to recover the real name; possibly `XXDLPURG` or similar) orchestrates the daily deal-reporting + purge cycle:

1. `XXDL530P` — Report on Deals with Upcoming/Recent Buy/Ship Dates.
2. `XXDL570P` — Deal Performance Report (current fiscal year).
3. `XXDL571P` — **Deal Alert Report.** Reads processing date from `RDR1`; calculates ±7-day range; queries `DEALDM1X` + `STARITEMCATST1I` + `ITEMAUTHST1A` + `CUST_CLS_GRP_CU2E` + `DIVMSTRDI1D` + `ACME.DIV_ITEM_PACK_DE1I` + `ACME.UIN_ITEM_DE6C`; categorises deals as active / future / inactive / terminated.
4. `XXDL995P` — **Purge `DEALLOGDM5X` records older than 5 days** (using `FCURRF` timestamp).
5. `XXDL960P` — Archive `DEALDM1X` records older than 2 years (BR-002-20..23, confirmed).
6. `XXDL980P` — **Purge `DEALAUDITDM1A` records.** Retention period is configurable via `DS.APPL_SYS_PARM_AP1S` parameter **`XXDL980_DM1A_DAYS`** (default 60 days).

New tables surfaced: `STARITEMCATST1I` (STAR item category), `ITEMAUTHST1A` (item authorisation).

#### Q5.4 — Daily / Weekly cycles confirmed corpus-empty

`XXDLSDLY` (Daily) and `XXDLSWKY` (Weekly) have **no document** in the corpus by either exact name. `XXDLSEOP` (End-of-Period) gets its behaviour entirely via `mcdl656j.md` (round 4 finding). These two genuinely require corpus re-ingest or SME — round-5 retrieval confirms they're not just below the chunking floor.

#### Q5.5 — **ACME codename confirmed: ACME = Acme**

Three independent evidences from the corpus:

1. `XXDL742.md` lists `'ACME'` as the explicit `(System identifier)` constant.
2. `MCDLS01.md` references `'CST'` as the `(ACME Business Type)`.
3. `DCSMCA` (Marwood Control Area) defines `ACME-COMPANY-SHORT-NAME PIC X(03)` — the "COMPANY SHORT NAME FOR CENTRALIZED BUYING" field — which is populated with the 3-char `ACME` for this customer instance.

Plus `ACME-CORP-LOCAL-SW` ("CORPORATE-LOCAL SWITCH FOR CENTRALIZED BUYING") confirms the **centralised buying / wholesale distribution** business pattern characteristic of Acme Company (a major US convenience-store wholesale distributor). The `AUTHORMCLANE` per-vendor handler is consistent: Acme is itself a distributor that re-distributes vendor goods.

Implication for assessment scope:
- **The customer is Acme Company.** Plan/Design phases can cite the actual customer's industry, regulatory profile (PCI for store payments, tobacco regulations for cigarette distribution, etc.), and competitor systems.
- Combined with HS-017 (platform is customised Marwood DS Release 3.3.0), the picture is: **Acme runs a customer-customised Marwood Distribution System for centralised buying / merchandising / deal management.**

### Open after Round 5

- `XXDLSDLY` / `XXDLSWKY` — confirmed corpus-empty. Need codmod re-run on those JCL files OR SME walkthrough.
- The purge JCL job's actual name (CAST normalised to `__name.md`) — `[SME]` or `[CAST]` to recover from JCL relation rows.

---

## Round 6 + Round 6b — Final retrieval-side completion (2026-06-03)

Round 6 ran **7 RAG prompts** (Q6.1–Q6.7) on session `yb_hybrid_1780479025_cursor-a` plus **4 deterministic CAST queries** (C6.1–C6.4). Goal: close remaining `[RAG]` / `[CAST]` markers at the retrieval floor.

### 6a. RAG prompts (Q6.1 – Q6.7)

#### Q6.1 — `D2133` + `D2138`

- **`D2133` / `D2138` prose:** corpus-empty for Business-Level Description, runtime evidence, datasets, and rules.
- **Graph:** `D2133` calls `D2138B`, `D2139B`; no callers to `D2133`. **`D2138` is a hub subroutine** — callers include `D2109`, `D2112`, `D2116`, `D2118`, `D2122`, `D2128`, `D2131`, `D2135`, `D2143`, `D2147`, `D2157` (11 programs). No callees from `D2138` in graph. Behaviour must be inferred from caller context until `d2138.md` is re-ingested.

#### Q6.2 — CICS-map-heavy `D21**` (`D2146`, `D2147`, `D2149`, `D2161`, `D2162`, `D2199`)

| Program | Corpus outcome |
|---|---|
| `D2147` | Deal Control **item/deal file** subroutine — add/change/delete/inquire single deal record; uses `DCSHITMP`, `DCSHVNDP`, `DCSFCAD`, `DCSPCAD`, `DCSFPOD`, calendar + control tables. BMS map names still empty. |
| `D2149` | **User Deal Information** inquiry/maintenance — two-phase screen flow; reads `DCSHITMP`, `DCSHVNDP`, `DCSFCAD`. |
| `D2162` | **Truckload discount** maintenance — inquiry/update; reads `DCSHVNDP`, `DCSHITMP`; parameters include group number, allowance, control type, load size, unit type. |
| `D2146`, `D2161`, `D2199` | Corpus-empty (graph still shows CICS map paragraphs exist per round 5 CAST paragraph names). |

#### Q6.3 — Rule-bearing `D21**` (`D2105`, `D2109`, `D2111`, `D2118`, `D2119`, `D2122`)

- **`D2122`:** PO header inquiry/update in Marwood Source Book — `DCSHPOHP`, `DCSFVND`, `DCSFSHP`; maintenance subject to validations; calls `D2138` + several `ACME.*` tables (`CHGALLWIC2A`, `CLS_GRP_DESC_CU2B`, `EDI_PO_TRK_BE5M`, `MARWOODEDIPOBE1M`, `PO_SPLT_TRM_VN3C`).
- **`D2109`:** only `STEP-A` program-directory paragraph surfaced.
- **`D2119`:** `STEP A` initialization (control blocks, calling parameters, print command).
- **`D2118`:** graph shows calls `D2138` + `ACME.DIV_ITEM_PACK_DE1I`, `ACME.DIVMSTRDI1D`, `ACME.UIN_ITEM_DE6C`; prose empty.
- **`D2105`, `D2111`:** corpus-empty.

#### Q6.4 — Vendor/item-handler `D21**` (`D2128`, `D2131`, `D2135`, `D2142`, `D2171`, `D2178`)

- **`D2135`:** Item cost inquiry by vendor/date — `DCSHITMP`, `DCSHVNDP`, `DCSFCAD`; links to Vendor Dictionary `D2201` / Vendor-Item Index `D3025`.
- **`D2171`:** Vendor Maintenance — multi-screen (`DCSFVND`, `DCSFCAR`, `DCSFBKR`, `DCSFVPL`); calls `D2201` for lookups.
- **`D2178`:** Vendor Return Information maintenance — `DCSFVND`, `DCSFRTV`.
- **`D2142`:** Removes cost information from vendor file via vendor handler (`DCSHVND` / `DCSFVND`).
- **`D2128`:** prose empty; graph + `D2122` context ⇒ **P.O. Add Module**; calls `D2138` + divisional item tables.
- **`D2131`:** inferred **P.O. Cost Maintenance** from `D2122`; uses `DCSKVNDP`, `DCSKITMP`, extensive PO/cost tables (prior round + graph).

#### Q6.5 — `MCRPR65J`, `MCRPR70J`, `MCRPR71J`

- **`MCRPR65J`:** **Quasar price-change producer.** `XXRPR65` builds `ACME.PERM.ST2A.PP` master + per-division `<DIV>.PERM.<DIV>ST2A.PP` files. Selection: `PRCS_TS = '1900-01-01-00.00.00.000000'` on `ACME.PRC_CMPNT_ITM_ST3B`; non-deleted `ACME.CUST_XREF_CU1X`; output fields `ST2A-DIVCD`, `ST2A-ITEM-NUM`, `ST2A-CUST-NUM`.
- **`MCRPR70J`:** corpus-empty (no JCL/business detail in assessment docs).
- **`MCRPR71J`:** **PP allocation expiry pipeline** — `XXRPR70P`/`XXRPR71P`/`XXRPR72P` extract; `SORT7/8/9`; `XXRPR57P`, `XXRPR78P`, `YYRPR78P`; datasets `ACME.PERM.RPR71S1.{RULE,ITEMS,CUSTS}`, `ACME.PERM.RPR721S1`, sorted variants; rule files `ACME.PERM.RPR72RUL`, `ACME.PERM.RPR73RUL`. Graph notes `MCRPR50J` consumes `ACME.PERM.RPR721S1.SRT`.

#### Q6.6 — `MCRPR77J`, `MCRPR85J`, `MCRPR99J`

- **`MCRPR77J`:** Conditional rule processing — executes `XXRPR77P` only when prior step RC `< 4` (`COND=(4,LT)`).
- **`MCRPR85J`:** **BI2 pending PP requests** — `XXRPR85` extracts type `'BI2'` / status Pending; report + zip + backup + **MQ FTE** transfer (`&&TEMPFTE`).
- **`MCRPR99J`:** Pricing-record purge — `MCRPR99` removes aged pricing rows from DB2 (retention configurable; table list not in corpus snippet).

#### Q6.7 — `MCCBT*` family + Marwood status + purge parent

| Job | Function (round 6) |
|---|---|
| `MCCBT01J` | Temporary batch-extract file lifecycle (cleanup → create → transform → sorted consolidate). Not cigarette-specific. |
| `MCCBT03J` | **Customer billing** — `XXCBT03P` / `MCCBT03`; reads `CUSTOMER.MASTER.FILE` + `TRANSACTION.FILE`; writes statements + updated master + `ERROR.REPORT`. |
| `MCCBT06J` | Batch deal-header processing for a deal type — extract/sort batch IDs (deal-management, not billing). |
| `MCCBT07J` | Corpus-empty in Q6.7 (already documented in round 4 as cigarette pending-deal loader). |
| `MCCBT08J` | **Cigarette batch deal report** — `MCCBT08` → `ACME.PERM.MCCBT8S1.COMMA` + header merge for email distribution; active/future/terminated deals. |

- **Marwood support:** corpus cites Marwood Maintenance Agreement — unmodified base-system portions maintained; **customer modifications forfeit maintenance on that program** (`d2105.md`). No explicit product EOL date found.
- **Purge parent JCL (corpus):** still referenced as `%%NAME` / `__name.md` in index. **CAST (C6.4) resolves the real name: `XXDLSWKY`.**

### 6b. CAST deterministic queries (C6.1 – C6.4)

#### C6.1 — Upstream `XXBSM32` / DWSLS chain (extended)

`MCBSM50J` is a **multi-step sales-consolidation orchestrator**, not a single-program writer:

| Step / object | Role |
|---|---|
| `XXBSM30P` → `XXBSM30` | Earlier pipeline stage; writes `ACME.PERM.BSM30S1.DWITM` |
| `XXBSM31P` → `XXBSM31` | Intermediate; touches `ACME.PERM.BSM31S1.DWSLS.SRT` |
| `XXBSM32P` → `XXBSM32` | Final writer to `ACME.PERM.BSM30S1.DWSLS` via `5000-WRITE-DATA` |
| `2100-ITM-DWSLS-CHECK`, `2200-DWSLS-FILE-READ` | Paragraphs that read/compare DWSLS inside `MCBSM50J` |
| `XXBSM32` (batch) | Also `SELECT` on `WD2D_DT_TBL` (date table) |

Upstream POS/e-commerce feeds remain `[SME]` — CAST shows the **in-platform** producer chain only.

#### C6.2 — Per-table PP writers (CAST `Objects↔DataSources`)

| Table | Writers (programs / paragraphs) |
|---|---|
| `ACME.PRC_CMPNT_ITM_ST3B` | `XXRPR58` (insert), `XXRPR65` (update), `XXRPR86` (delete), paragraphs `5400-INSERT-ST3B`, `5400-UPDATE-ST3B-REC`, `5300-DEL-ST3B-TBL` |
| `ACME.PP_RQST_PP3C` | `XXRPR50/53/55/70/71/85` (update), `XXRPR59` (delete), `MCRPR25` (insert), paragraphs across request-processing |
| `ACME.PP_RQST_HIST_PP3H` | `XXRPR59` (insert), `MCRPR99` (delete), `5500-INSERT-PP3H`, `3100-DELETE-PP3H` |
| `ACME.PP_ALLOC_DTL_PP4D` / `PP_ALLOC_SUM_PP4S` | **0 direct writers** in CAST export (allocation updates may be via `303.ACME.PP_ALLOC_PP2A` alias — `[CODMOD]`) |

Cross-validates round-5 `MCRPR58/59/74/86` findings; adds explicit writer map for `PP3C` mutation surface.

#### C6.3 — `D21**` batch vs CICS (locked)

- **26 programs** tagged `Cobol Transactional Program` in CAST inventory (`D2105`–`D2199` subset).
- **`D2137`** = `Cobol Batch Program`, invoked only by **`XXD2137J`** (CC=416 purge job).
- **`D2133`** has **zero** rows in `Objects↔DataSources` — not present as a standalone analysed object in this CAST export (unlike `D2137`). Classification remains **likely batch** from round-4 prose only.
- **Zero JCL Job callers** for any `D21**` transactional program — CICS-only entry confirmed for the 26.

#### C6.4 — High-CC JCL jobs (CC>100) absent from docs

Only **4** jobs with CC>100 were missing from the `docs/legacy/` text at round-6 start:

| Job | CC | Notes |
|---|---:|---|
| `XXDLSWKY` | 414 | **Weekly deal cycle** — also **parent of purge/reporting proc chain** (`XXDL530P`…`XXDL980P`) |
| `XXDLSDLY` | 375 | Daily deal cycle — still no corpus narrative |
| `XXDLSEOP` | 343 | End-of-period — partial narrative via `mcdl656j.md` |
| `XXDL702P` | 110 | Procedure wrapper for `XXDL702` active billing |

### Open after Round 6 (retrieval floor)

- **BMS mapset names** for all 56 maps — still require `EXEC CICS SEND MAP` source parse (`[CODMOD]`).
- **`MCRPR70J`** — corpus-empty; CAST shows CC=73 but no business narrative.
- **`D2133`, `D2138`, `D2146`, `D2161`, `D2199`, `D2105`, `D2111`** — corpus-empty prose; graph/CAST give structure only.
- **`XXDLSDLY`** — high complexity (CC=375) but still no assessment markdown; CAST + job name only.
- **Marwood vendor EOL / current support contract** — maintenance-policy text only; `[SME]` for commercial status.
- **~100 `[SME]` markers** — unchanged; human interviews required.
- **11 `[CORPUS-EMPTY]`** — unchanged; need HTML-stripped re-ingest.
- Per-program runtime classification for `D2133`, `D2148`, and other D21NN numbers that exist as both CICS-transactional and batch entries.
- Full enumeration of the 56 BMS Maps with names (requires extracting from CICS source `EXEC CICS SEND MAP / RECEIVE MAP` statements).
- Whether Marwood (the vendor) still exists or is supported — material to lift-and-shift option of HS-017.

---

## Reproducing These Findings

Start a fresh session and replay the questions in `BP-*.md` headers — they cite the exact prompts used. yourbuddy resolves the corpus from Spanner via `opportunityId`, so no manual corpus ID is needed:

```bash
curl -s -X POST \
  "https://yourbuddy-hybrid-962673027835.us-central1.run.app/api/v1/yourbuddy-hybrid/chat/start" \
  -H "Content-Type: application/json" \
  -d '{"opportunityId":"006UV00000aTYw1YAG","userId":"<your-id>"}'
```
