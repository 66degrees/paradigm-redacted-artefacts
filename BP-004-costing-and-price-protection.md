# BP-004 — Costing & Price Protection

**Status:** Draft — pending SME review
**Owners:** Costing / Pricing Analysts, Merchandising IT
**Anchor programs:** `XXCAD63`, `MCCAD65J`, `MCCST24J`, `MCCST50J`, `MCCST63J`, `MCCST96J`, `MCCBT07`, `XXRPR50` … `XXRPR56`, `XXRPR65`, `MCRPR55J`, `XXRPR77P` (via `MCRPR77J`), and the highest-edge `XXRPR55`/`XXRPR85`/`MCCST50`/`MCCST24`
**Related SRS:** FR-COST-001 … FR-COST-003, FR-PRPR-001 … FR-PRPR-003
**Related HLD:** §5 (CAD masters), §6 (top programs), §9 (HS-002/HS-004/HS-007)

---

## 1. Overview

This BP governs **cost** and **price protection**. It has both **batch** and **CICS / MQ online** sub-pipelines:

- **Costing batch** validates and propagates item-vendor costs through the `<DIV>.MSTR.CAD` (**Computerized Allowance Data**) divisional masters and the corporate cost stores. The `MCCST*J` jobs orchestrate; `XXCAD63` is the validation core. `MCCST24J` produces the future-cost-change extract used downstream.
- **Costing online (CICS + MQ)** processes incoming cost-update messages from MQ queues. **`MCCST50` (transaction `MCS4`)** consumes `'IC1X'` queue messages; **`MCCST51` (transaction `MCS5`)** consumes `'CADMF'` queue messages. **`MCCST55`** is the dispatcher that routes by `COMM-Q-TYPE` to MCCST50 or MCCST51.
- **Price Protection (PP)** is a rule-driven mechanism where vendor cost changes propagate compensating adjustments. The `MCRPR*` / `XXRPR*` family implements request capture, rule application, expiration, archival, and modeller reporting. There is a dedicated `ACME.PP_*` DB2 schema (catalogued in §4).
- **`DS.APPL_SYS_AP1P`** (64 referencing programs) and **`DS.APPL_SYS_PARM_AP1S`** (39 programs) are the platform's configuration backbone for these flows (HS-007). Batch jobs use named parameter keys (e.g. `MCCST24_LAST_RUN`) for **checkpoint / last-run tracking**.
- **Cigarette deals** have their own two-stage batch path (`MCCBT06` → `MCCBT07`) updating CAD tables from sorted extract files — a category-specific exception to the generic cost cycle. MCCBT07 feeds rows back into `ACME.PENDINGDEALSDM3P`, joining BP-002's lifecycle.

> **Naming correction:** `CAD` in this codebase stands for **Computerized Allowance Data** (per `mccbt07.md`), not "Cost-and-Deal". The `<DIV>.MSTR.CAD` divisional family stores allowance data and feeds the deal-allowance and price-protection processes. Documents in this folder use the corrected expansion from this revision onward.

---

## 2. Programs Involved

### 2.1 Costing — batch

| Type | Name | Purpose |
|---|---|---|
| JCL job | `MCCAD65J` | Orchestrates item cost data processing; executes `XXCAD63`. |
| COBOL | `XXCAD63` | Item cost validation (anchor of BP-001 rules; same anchor in BP-004 for vendor + basis-code rules). |
| COBOL | `XXCAD65` | Cursor-driven matching of `XXCAD` vs `XXDE9E` records. |
| JCL job | `MCCST24J` | **Future Cost Changes Extract.** Executes `MCCST24`. |
| COBOL | `MCCST24` | Generates comma-delimited `MCCST24O` extract of future cost changes. Reads `EDISENT` input. **Checkpoint:** reads/writes `MCCST24_LAST_RUN` in `DS.APPL_SYS_PARM_AP1S` to derive the filter window. Uses temp table `TEMP.ITEM_BILL_COST_T356`. Paragraphs `1000-START-UP → 1100-OPEN-FILES → 1200-DELETE-ITM-TBL → 2000-PROCESS → 3000-WRAP-UP → 3100-CLOSE-FILE → 3200-EJOB-CONTROL → 4000-UPDATE-AP1S-PARA`. |
| JCL job | `MCCST62J`, `MCCST63J`, `MCCST70J` (CC=105), `MCCST96J` | Other batch costing cycles (anchor description; `[RAG]` deeper). |
| JCL job | `MCCAD01J`, `MCCAD62J` (CC=64), `MCCAD65J` (CC=149), `MCCAD66J`, `MCCAD98J` | Full CAD cycle (Computerized Allowance Data). `MCCAD65J` invokes `XXCAD63` (see BP-001). |
| JCL job | `XXCST64J`, `SWCST64J` | Standalone costing batches (`SW*` lives in `acme-code/sw.perm.jcl/`). Behaviour `[RAG]`/`[SME]`. |
| JCL job | `MCCBT01J` | **Temporary batch-extract file lifecycle** — cleanup, create batch-id file, transform extracts, consolidate sorted output. Not cigarette-specific. |
| JCL job | `MCCBT03J` | **Customer billing** — `XXCBT03P` / `MCCBT03`; reads `CUSTOMER.MASTER.FILE` + `TRANSACTION.FILE`; writes `BILLING.STATEMENTS`, `UPDATED.CUSTOMER.MASTER`, `ERROR.REPORT`. |
| JCL job | `MCCBT06J` | **Deal batch-header processing** — extracts/sorts batch identifiers for a deal type (feeds deal pipelines, not customer billing). |
| JCL job | `MCCBT07J` | **Cigarette pending-deal loader** (round 4) — updates CAD + `ACME.PENDINGDEALSDM3P`; see BR-004-20..25. |
| JCL job | `MCCBT08J` | **Cigarette batch-deal report** — `MCCBT08` → `ACME.PERM.MCCBT8S1.COMMA` + header merge (`ICEGENER`) for email distribution of active/future/terminated cigarette deals. |

### 2.2 Costing — CICS / MQ online

| Type | Name | Purpose |
|---|---|---|
| CICS dispatcher | `MCCST55` | Reads MQ message; sets `WS-TRANSID` + `WS-PROGRAM` by `COMM-Q-TYPE`; routes to MCCST50 or MCCST51. Unknown `COMM-Q-TYPE` → `STEP-X`. |
| **CICS txn `MCS4`** | **`MCCST50`** | Processes `'IC1X'` MQ messages → cost create/update via `COMM-ACTION`. Paragraphs `a0000-initialize`, `a2005-get-q-mssg`, `a2007-verify-de9e-data`, `a2009-verify-de6e-data`, `a3000-mqclose`, `a4000-error-message`, `a4100-no-data`, `a6000-return-to-cics`, `b1100-get-max-retry`, `c1000-open-item-detail-cur`, `c1500-fetch-item-detail-cur`, `c2000-process-catlg-num`, `c2005-build-ic1x-base-rec`, `c2010-create-cost-record`, `c2015-create-element-records`, `9000-process-error`, `9100-process-error`. |
| **CICS txn `MCS5`** | **`MCCST51`** | Processes `'CADMF'` MQ messages → CAD-master cost updates. Error logs to `ACME.CMN_ERR_CM5A` + `ACME.COMNT_CM4A`. |
| COBOL | `MCCST51`'s `1200-debug-msg-check`, `1100-get-max-retry` | Internal control helpers for online cost message handling. |

### 2.2 Price Protection (PP) — full 13-job cycle (per CAST)

> CAST shows the PP cycle is **13 JCL jobs**, far larger than the 4 surfaced by RAG. `MCRPR50J` is the single largest job in the entire platform.

| Type | Name | CAST CC | LOC | Obj# | Purpose |
|---|---|--:|--:|--:|---|
| **JCL job** | **`MCRPR50J`** | **553** | 20,938 | 919 | **Highest-complexity job on the platform.** PP primary cycle. Behaviour `[RAG]`. |
| JCL job | `MCRPR55J` | (mid) | — | — | PP Modeler Report driver (`XXRPR55`, request type `'BI1'`). |
| JCL job | `MCRPR56J` | 108 | 3,425 | 159 | PP rule expiration/archival driver (`XXRPR56`). |
| JCL job | `MCRPR58J` | `[RAG]` | — | — | **Nightly PP rule application.** `XXRPR58` reads date param `%%C1-%%M1-%%D1`; inserts new customer-item configs into `ACME.PRC_CMPNT_ITM_ST3B`; conditional insert (skip duplicates → "not inserted" count); RC=16 on critical. |
| JCL job | `MCRPR59J` | `[RAG]` | — | — | **Active → history archival for PP requests.** `XXRPR59` reads retention days from `DS.PERM.RDRPARM(RPR59R1)`; archives `PP_RQST_PP3C` rows (status `'CMP'` or `'INP'` and `CMPLTN_TS` older than `today - retention_days`) into `ACME.PP_RQST_HIST_PP3H`. Duplicate-key on history insert → don't delete from active. |
| JCL job | `MCRPR65J` | `[RAG]` | — | — | **Quasar producer (round 6).** `XXRPR65` builds `ACME.PERM.ST2A.PP` + per-division `<DIV>.PERM.<DIV>ST2A.PP`. Selects `ACME.PRC_CMPNT_ITM_ST3B` where `PRCS_TS='1900-01-01-00.00.00.000000'`; filters `ACME.CUST_XREF_CU1X` with `DELT_SW='N'`. |
| JCL job | `MCRPR70J` | 73 | 5,263 | 149 | PP cycle job (`XXRPR70`). **Business narrative `[CORPUS-EMPTY]`** (round 6). |
| JCL job | `MCRPR71J` | 302 | 12,972 | 549 | **PP allocation expiry (round 6).** `XXRPR70P`/`71P`/`72P` extract → `SORT7/8/9` → `XXRPR57P`/`78P`/`YYRPR78P`; datasets `ACME.PERM.RPR71S1.*`, `ACME.PERM.RPR721S1`; feeds `MCRPR50J` via `ACME.PERM.RPR721S1.SRT`. |
| JCL job | `MCRPR74J` | 105 | 4,126 | 184 | **2-step billing → allocation sync.** Step 1 `XXRPR74` reads invoice `ACME.INVC_HDR_BD1H` + `ACME.INVC_DTL_COMN_BD1D`; updates `ACME.PP_ALLOC_DTL_PP4D`, `ACME.PP_ALLOC_SUM_PP4S`, `ACME.PP_ALLOC_HIST_PP4H`. Step 2 `XXRPR75` synchronises customer-specific pricing allocations; aggregates to customer-group level. JCL enforces step ordering. |
| JCL job | `MCRPR77J` | `[RAG]` | — | — | **Conditional rule processing (round 6).** Executes `XXRPR77P` only when prior step RC `< 4` (`COND=(4,LT)`). |
| JCL job | `MCRPR85J` | `[RAG]` | — | — | **BI2 pending requests (round 6).** `XXRPR85` extracts type `'BI2'` / status Pending; report + zip + backup + **MQ FTE** via `&&TEMPFTE`. |
| JCL job | `MCRPR86J` | `[RAG]` | — | — | **`ST3B` cleanup.** `XXRPR86` deletes outdated/invalid PP records from `ACME.PRC_CMPNT_ITM_ST3B` (paired with MCRPR58J's writes). |
| JCL job | `MCRPR99J` | 92 | 2,306 | 108 | **Pricing-record purge (round 6).** `MCRPR99` removes aged pricing rows from DB2 per configurable retention. Also deletes `ACME.PP_RQST_HIST_PP3H` per CAST writer map. |
| COBOL | `XXRPR50` | — | — | — | Processes price protection requests. |
| COBOL | `XXRPR51` | — | — | — | Applies cost amounts to request details. |
| COBOL | `XXRPR52` | — | — | — | Manages price protection rules. |
| COBOL | `XXRPR53` | — | — | — | Email notifications and status updates for pricing requests. |
| COBOL | `XXRPR54` | — | — | — | AP1S record update for end-of-job processing. |
| COBOL | `XXRPR55` | — | — | — | Generates Price Protection Modeler Reports (`'BI1'`). |
| COBOL | `XXRPR56` | — | — | — | Handles price protection rule expiration and archival. |
| COBOL | `XXRPR65` | — | — | — | Price protection rule processing and status updates, including error handling. |
| COBOL | `XXRPR70`, `XXRPR71`, `XXRPR72`, `XXRPR74`, `XXRPR86`, `XXRPR99` | — | — | — | Additional rule/request processors invoked by their `MCRPR*J` driver. Detailed behaviour `[RAG]`. |
| COBOL | `XXRPR77P` (proc) | — | — | — | Rule-processing procedure with conditional execution (under `MCRPR77J`). |
| COBOL | `XXRPR85` | — | — | — | PP Modeler `'BI2'` processor. |

---

## 3. Business Rules

### 3.1 Costing (recap from `XXCAD63`)

`BR-001-01` … `BR-001-10` from BP-001 apply unchanged here. They are reproduced as `BR-004-01` … `BR-004-10` for traceability:

| ID | Rule |
|---|---|
| BR-004-01 | Active vendor: `VNKY-RECORD-STATUS-CODE = 'A'`. |
| BR-004-02 | Cost control: `VNRT-COST-CONTROL-FLAG = 'Y'`. |
| BR-004-03 | Cost record found: `WS-CUR-COST-SW='Y'` OR `WS-FUT-COST-SW='Y'`. |
| BR-004-04 | `CDIC-LIST-AMT-TYPE='F'` → cost basis `'ACME'`. |
| BR-004-05 | `CDIC-LIST-AMT-TYPE='1'` → cost basis `'CW'`. |
| BR-004-06 | Other `CDIC-LIST-AMT-TYPE` → cost basis `'SC'`. |
| BR-004-07 | Effective-date discrepancy → discrepancy report entry. |
| BR-004-08 | Cost-amount discrepancy → discrepancy report entry. |
| BR-004-09 | Exception items (`WS-EXC-ITEM-SW='Y'` AND `WS-AP1R-SW='Y'`) → log `"ITEM EXCEPTION - NO UPDATES"`. |
| BR-004-10 | File status `'23'` / `'10'` → `RETURN-CODE = 16` abend. |

### 3.2 Cigarette deals (`MCCBT06` → `MCCBT07`)

| ID | Rule | Source |
|---|---|---|
| BR-004-20 | Two-stage pipeline — `MCCBT06` sorts and edits the extract; `MCCBT07` validates and updates CAD + pending-deal tables. | `mccbt06j.md` / `mccbt07.md` |
| BR-004-21 | `MCCBT07` reads `RDRPARM` to obtain `BATCHID`; **if `BATCHID` is not numeric, abend**. | `mccbt07.md` |
| BR-004-22 | `MCCBT07` queries `ACME.BAT_DEALERRLOGDM3B` for rows matching the current `BATCH_ID` with `ERR_TYP = 'F'` (Fatal). If found, surface the informational message that the batch has fatal errors. | `mccbt07.md` |
| BR-004-23 | MCCBT07 updates / inserts into: `ACME.PENDINGDEALSDM3P`, `ACME.DIVPENDDEALSDM3D`, `ACME.DEALITEMDM2I`, `ACME.DEALDM1M`, `ACME.CAD_REMARK_DM3R`, `ACME.ITEM_UPC_DE6Y`. | `mccbt07.md` |
| BR-004-24 | MCCBT07 maintains deal-header status in `ACME.BAT_DEAL_HDR_DM3H` and pending-deal-batch entries in `ACME.BAT_DEAL_DM3L`. | `mccbt07.md` |
| BR-004-25 | **Cross-BP handoff:** MCCBT07 inserts into `ACME.PENDINGDEALSDM3P`, which then becomes input to BP-002's `D8050` deal-capture polling. The cigarette path is a feeder of the main lifecycle, not a sibling of it. | derived |

### 3.3 Price protection — anchor rules

| ID | Rule | Source |
|---|---|---|
| BR-004-30 | A price-protection request is captured into the rule store before `XXRPR50` processes it. | `XXRPR50` |
| BR-004-31 | `XXRPR51` applies cost amounts to request details — request totals reflect per-line cost deltas. | `XXRPR51` |
| BR-004-32 | `XXRPR52` manages rule lifecycle (create/update/deactivate). | `XXRPR52` |
| BR-004-33 | `XXRPR53` issues email notifications and status updates for pricing requests. The notification's payload includes the request identifier and the resulting status. | `XXRPR53` |
| BR-004-34 | `XXRPR54` performs the AP1S record update at end-of-job (per-job ledger entry). | `XXRPR54` |
| BR-004-35 | `XXRPR55` generates the Price Protection Modeler Report. | `XXRPR55` |
| BR-004-36 | `XXRPR56` archives expired rules; expiration is rule-date-driven. | `XXRPR56` |
| BR-004-37 | `XXRPR65` handles status updates and error reclassification during rule processing. | `XXRPR65` |
| BR-004-39 | `[SME]` Confirm whether notifications (`XXRPR53`) MUST be retried on transport failure or are fire-and-forget. |  |

### 3.3b Price protection — `MCRPR50J` (request type `'2CE'`, full pipeline)

| ID | Rule | Source |
|---|---|---|
| BR-004-50-01 | **Request-type filter:** XXRPR50 identifies pending PP requests of type `'2CE'` linked to active rules (distinct from XXRPR55's `'BI1'` and XXRPR85's `'BI2'`). | `MCRPR50J` |
| BR-004-50-02 | XXRPR50 enriches requests with customer + item data from various source tables, populates intermediate `ACME.PERM.RPR50S1`, logs status to `ACME.PERM.RPR50.STAT`. | `MCRPR50J` |
| BR-004-50-03 | `SORT1` step sorts `ACME.PERM.RPR50S1` per `DS.PERM.SORTPARM(XXRPR501)` → `ACME.PERM.RPR50S1.SRT`. | `MCRPR50J` |
| BR-004-50-04 | **XXRPR51 applies three cost types** to the sorted request lines: **License Cost, Billing Cost, Invoice Price** — based on component type, customer, item. Writes `ACME.PERM.RPR51S1`; logs `ACME.PERM.RPR51.STAT`. | `MCRPR50J` |
| BR-004-50-05 | `SORT2/SORT3` steps produce `ACME.PERM.RPR51S1.SRT` and `ACME.PERM.RPR51S3.SRT` (per `DS.PERM.SORTPARM(XXRPR511)`/`XXRPR512`). | `MCRPR50J` |
| BR-004-50-06 | XXRPR52 consumes the sorted output; updates core PP rules and customer/item associations in the system; logs `ACME.PERM.RPR52.STAT`. | `MCRPR50J` |
| BR-004-50-07 | XXRPR53 uses `ACME.PERM.RPR51S3.SRT` + consolidated `ACME.PERM.RPR.STAT` to generate email notifications to the job requester (config: `DS.PERM.RDRPARM(RPREML53)`). | `MCRPR50J` |
| BR-004-50-08 | XXRPR54 and XXRPR57 are also invoked; specific roles `[RAG]`. | `MCRPR50J` |
| BR-004-50-09 | Rule files read: `ACME.PERM.RPR72RUL`, `ACME.PERM.RPR73RUL`. Item/customer scope: `ACME.PERM.RPR71S1.ITEMS`, `ACME.PERM.RPR71S1.CUSTS`. | `MCRPR50J` |
| BR-004-50-10 | Standard linkage: `DS.EMER.LINK`, `DS.PERM.LINK`, `DS.PERM.SQLBATCH(SQLINFO)`, `DS.PERM.RDRPARM(PRTONLY1)` for IDCAMS checks. | `MCRPR50J` |

### 3.4 Price protection — `XXRPR55` (Modeler Report, request type `'BI1'`)

| ID | Rule | Source |
|---|---|---|
| BR-004-55-01 | Processes only PP requests where request type = `'BI1'`. | `XXRPR55` |
| BR-004-55-02 | For each qualifying request: retrieves rule details, then customer / division / vendor / item enrichment via the `ACME.PP_*` cluster + `ACME.CUST_*` / `ACME.DIV_*` / `ACME.VNDR_*` masters. | `XXRPR55` |
| BR-004-55-03 | **Conditional output:** an output record is written to `ACME.PERM.RPR55S1` **only if either price is not zero**. | `XXRPR55` |
| BR-004-55-04 | A second output `ACME.TEMP.RPR55S2` is staged for email notification of report completion. | `XXRPR55` |
| BR-004-55-05 | Paragraphs include `2300-GET-PRICE-AMT` (price extraction), `2350-XXDIV50-CALL` (division-aware enrichment), `4400-GET-GROUP` (group lookup). DB2 access bracketed by `5000-SET-DB2-PACKAGE` / `5050-RESET-DB2-PACKAGE`. | `XXRPR55` |
| BR-004-55-06 | Caller: `MCRPR55J`. | graph |

### 3.5 Price protection — `XXRPR85` (Modeler request processor, request type `'BI2'`, status `'PND'`)

| ID | Rule | Source |
|---|---|---|
| BR-004-85-01 | Processes only PP requests where request type = `'BI2'` **and** status = `'PND'` (Pending). | `XXRPR85` |
| BR-004-85-02 | **Status transitions enforced:** `'PND'` → `'INP'` (In Progress) at start of work, then → `'CMP'` (Completed) on successful finish. | `XXRPR85` |
| BR-004-85-03 | Extracts rule details + customer + item + vendor data by joining the `ACME.PP_*` cluster + masters; writes consolidated rows to a structured output file. | `XXRPR85` |
| BR-004-85-04 | Caller: `MCRPR85J`. | graph |

### 3.5b Price protection — Quasar interface (`MCRPR65J`)

| ID | Rule | Source |
|---|---|---|
| BR-004-65-01 | `MCRPR65J` generates and distributes **price change records for the external Quasar system**. | `mcrpr65j.md` |
| BR-004-65-02 | The job first queries DB2 to create a master price-change file, then splits it into per-division files. | `mcrpr65j.md` |
| BR-004-65-03 | Per-division output dataset naming: `<DIV>.PERM.<DIV>ST2A.PP` (e.g. `MS.PERM.MSST2A.PP`, `MN.PERM.MNST2A.PP`, `GM.PERM.GMST2A.PP`, `MP.PERM.MPST2A.PP`, `MW.PERM.MWST2A.PP`, `MG.PERM.MGST2A.PP`, `C3.PERM.C3ST2A.PP`). Confirmed for 7 divisions; presumed all 32. | divisional dataset docs |
| BR-004-65-04 | The Quasar boundary is a **fixed external interface**: file layout, splitting, and cadence MUST be preserved exactly through any modernization. Quasar is out of scope to modernize in this engagement. | derived |

### 3.5c Cigarette Cost (CIC) cycle — new BP-004 sub-pipeline

The `MCCIC*J` family handles cigarette-cost management as a discrete sub-process complementing the general-cost cycle.

| ID | Job | Rule |
|---|---|---|
| BR-CIC-01 | `MCCIC01J` | Processes cigarette cost change transactions; splits by divisional GL codes; consolidates into per-division permanent files. |
| BR-CIC-02 | `MCCIC02J` | Consolidates divisional cigarette-cost report files into a single temp; prints the exception report. |
| BR-CIC-03 | `MCCIC20J` | Generates detailed cost component report for linked cigarette items. Reads `ACME.MCLANE_XREF_DI3X`, `ACME.UIN_ITEM_DE6C`, `ACME.ITEM_UPC_DE6Y`, **`ACME.CIG_ITEM_COST_PR1C`** (new DB2 table — Cigarette Item Cost), `ACME.DIVMSTRDI1D`. Performs cost calculations and UPC construction. |
| BR-CIC-04 | `MCCIC90J` | Cleanup/maintenance for "non-maintainable" and "default" item groupings. Inserts new cigarette items; deletes non-cigarette/inactive items; cleans orphaned link items; updates orphaned cigarette items; generates a detailed report. Reads `ACME.ITEM_GRP_DE1E` and `ACME.CIG_ITEM_COST_PR1C`. |
| BR-CIC-05 | (cross-BP) | Cigarette is a first-class business domain — two families exist: `MCCBT*` (cigarette deals batch, BP-002) and `MCCIC*` (cigarette cost, this BP). The `ACME.CIG_ITEM_COST_PR1C` and `ACME.MCLANE_XREF_DI3X` tables anchor this domain. |

### 3.5d Data-integrity cycle (MCM9 family) — companion to BP-004

| ID | Job | Rule |
|---|---|---|
| BR-M9-01 | `MCM9014J` | Detects discrepancies between deal data in `XX.DEALDM1X` and PowerBuilder CAD tables across multiple corporate entities. Calls `M901401` per entity, then `M901402` to produce a sorted "Corporate Discrepancy Report". |
| BR-M9-02 | `MCM9091J` | Identifies items in divisional files lacking corresponding cost records in CAD VSAM; validates items against SIM files; constructs and inserts missing cost records; updates CAD VSAM; generates summary report. Calls `M9093` per division. |
| BR-M9-03 | (design implication) | The MCM9 cycle is a **self-healing loop compensating for legacy distribution inconsistencies**. Modernization design should make this loop unnecessary in the new platform (transactional integrity by construction), rather than reproducing it. |

### 3.6 PP status master and rule-status state machine

| ID | Rule | Source |
|---|---|---|
| BR-004-90 | The canonical PP status set lives in `ACME.STAT_PP9S`; programs reference statuses by symbolic name (e.g. `'PND'`, `'INP'`, `'CMP'`). | `XXRPR85` graph |
| BR-004-91 | The legal transitions observed from `XXRPR85` are `PND → INP → CMP`. Other transitions (e.g. expiration / archival) live in `XXRPR56` and need targeted retrieval to enumerate. | derived, `[RAG]` |

### 3.7 Configuration

| ID | Rule | Source |
|---|---|---|
| BR-004-50 | Per-application configuration is read from `DS.APPL_SYS_AP1P` (64 program references). | corpus |
| BR-004-51 | Per-parameter configuration is read from `DS.APPL_SYS_PARM_AP1S` (39 program references). | corpus |
| BR-004-52 | A misconfigured AP1P / AP1S row affects every BP-004 cycle (HS-007). | derived |
| BR-004-53 | **Checkpoint pattern:** batch jobs persist their last successful run via named keys in `DS.APPL_SYS_PARM_AP1S` (e.g. `MCCST24_LAST_RUN` set by `XXMCCST24.4000-UPDATE-AP1S-PARA` on successful completion). | `MCCST24` |

### 3.8 CICS / MQ online cost flow

| ID | Rule | Source |
|---|---|---|
| BR-004-60 | Cost-update MQ messages arrive on queues whose `COMM-Q-TYPE` field routes the dispatcher (`MCCST55`). | `mccst55.md` |
| BR-004-61 | `COMM-Q-TYPE = 'IC1X'` → `WS-TRANSID := 'MCS4'`, `WS-PROGRAM := 'MCCST50'`. | `mccst55.md` |
| BR-004-62 | `COMM-Q-TYPE = 'CADMF'` → `WS-TRANSID := 'MCS5'`, `WS-PROGRAM := 'MCCST51'`. | `mccst55.md` |
| BR-004-63 | Any other `COMM-Q-TYPE` → control transfers to `STEP-X` (a documented bypass). | `mccst55.md` |
| BR-004-64 | `MCCST50` decides between **create-cost-record** and **update-existing** based on `COMM-ACTION`. | `MCCST50` |
| BR-004-65 | Data verification is mandatory before write: `a2007-verify-de9e-data` (DE9E) and `a2009-verify-de6e-data` (DE6E). | `MCCST50` |
| BR-004-66 | `MCCST50` returns to CICS via `a6000-return-to-cics`; it does not loop. Each MQ message is one transaction invocation. | `MCCST50` |
| BR-004-67 | A retry cap is enforced via `b1100-get-max-retry`. | `MCCST50` |
| BR-004-68 | `MCCST51` logs errors to **two** tables: `ACME.CMN_ERR_CM5A` (error log) and `ACME.COMNT_CM4A` (comment log) — both include error codes, descriptions, user IDs, timestamps. | `MCCST51` |
| BR-004-69 | `[SME]` Confirm the MQ subsystems (`MQTA` for DB2T, `MQPA` for DB2P observed in `MCCST40`) and how the program selects between them. |  |

---

## 3.7b Round-5 PP cycle rules — MCRPR58/59/74/86

### MCRPR58J (nightly rule application)
| ID | Rule | Source |
|---|---|---|
| BR-004-58-01 | Reads date parameter from `READER` in `%%C1-%%M1-%%D1` format. | `MCRPR58J` |
| BR-004-58-02 | Filters DB2 by the parameter date; identifies PP rules starting or ending on that date, plus audit records indicating expiration. | `MCRPR58J` |
| BR-004-58-03 | Inserts new effective customer-item configurations into `ACME.PRC_CMPNT_ITM_ST3B`. | `MCRPR58J` |
| BR-004-58-04 | If a configuration for the same customer-item-effective-date already exists, the insert is **skipped** and counted as a "record not inserted" — **not** treated as an error. | `MCRPR58J` |
| BR-004-58-05 | Critical errors → `RETURN-CODE = 16`. | `MCRPR58J` |

### MCRPR59J (active → history archival for PP requests)
| ID | Rule | Source |
|---|---|---|
| BR-004-59-01 | Reads retention days from `DS.PERM.RDRPARM(RPR59R1)` (numeric value). | `MCRPR59J` |
| BR-004-59-02 | Calculates `target_date = today - retention_days`. | `MCRPR59J` |
| BR-004-59-03 | Selects `ACME.PP_RQST_PP3C` rows where `status ∈ {'CMP','INP'}` AND `CMPLTN_TS < target_date`. | `MCRPR59J` |
| BR-004-59-04 | Inserts each selected row into `ACME.PP_RQST_HIST_PP3H`; on success, deletes the original from `ACME.PP_RQST_PP3C`. | `MCRPR59J` |
| BR-004-59-05 | **Duplicate-key on history insert** → the active-table delete is **suppressed** (no rollback chain; the record stays in active). | `MCRPR59J` |
| BR-004-59-06 | DB2 errors during insert/delete → abnormal termination. | `MCRPR59J` |

### MCRPR74J (billing → allocation 2-step sync)
| ID | Rule | Source |
|---|---|---|
| BR-004-74-01 | Step 1 (`XXRPR74`): reads invoice data from `ACME.INVC_HDR_BD1H` + `ACME.INVC_DTL_COMN_BD1D`; updates `ACME.PP_ALLOC_DTL_PP4D`, `ACME.PP_ALLOC_SUM_PP4S`, `ACME.PP_ALLOC_HIST_PP4H`. Logs execution timestamp. | `MCRPR74J` |
| BR-004-74-02 | Step 2 (`XXRPR75`): synchronises customer-specific pricing allocations; reads invoice details; adjusts pre-billing quantities for customer items; aggregates to customer-group level. Updates `ACME.PP_ALLOC_DTL_PP4D` and `ACME.PP_ALLOC_SUM_PP4S`. | `MCRPR74J` |
| BR-004-74-03 | JCL enforces strict step ordering — XXRPR75 must run after XXRPR74 (conditional execution). | `MCRPR74J` |

### MCRPR86J (ST3B cleanup, paired with MCRPR58J writes)
| ID | Rule | Source |
|---|---|---|
| BR-004-86-01 | `XXRPR86` identifies records in `ACME.PRC_CMPNT_ITM_ST3B` meeting purge criteria (status codes, expiration dates, irrelevance — specific logic `[RAG]`) and deletes them. | `MCRPR86J` |
| BR-004-86-02 | Paired-cycle invariant: every `MCRPR58J` insert into `ST3B` may eventually be deleted by `MCRPR86J` per the purge logic. Modernization must preserve the insert/purge pairing. | derived |

## 3.7c Round-6 PP + MCCBT rules — MCRPR65/71/77/85/99 + CAST writer map

### MCRPR65J (Quasar producer — round 6 detail)
| ID | Rule | Source |
|---|---|---|
| BR-004-65-05 | Selects `ACME.PRC_CMPNT_ITM_ST3B` rows where `PRCS_TS = '1900-01-01-00.00.00.000000'` (pending processing). | round 6 RAG |
| BR-004-65-06 | Only non-deleted customer cross-references (`DELT_SW = 'N'` on `ACME.CUST_XREF_CU1X`). | round 6 RAG |
| BR-004-65-07 | Master output `ACME.PERM.ST2A.PP`; per-division `<DIV>.PERM.<DIV>ST2A.PP` splits via ICETOOL (`DS.PERM.SORTPARM(XXRPR651)`). | round 6 RAG |

### MCRPR71J (allocation expiry)
| ID | Rule | Source |
|---|---|---|
| BR-004-71-01 | Identifies and expires PP allocations that met duration/limit criteria; uses `XXRPR70P`/`XXRPR71P`/`XXRPR72P` extract + `SORT7/8/9` + `XXRPR57P`/`XXRPR78P`/`YYRPR78P`. | round 6 RAG |
| BR-004-71-02 | Working datasets: `ACME.PERM.RPR71S1.{RULE,ITEMS,CUSTS}` (+ `.SRT` variants), `ACME.PERM.RPR721S1`; rule files `ACME.PERM.RPR72RUL`, `ACME.PERM.RPR73RUL`. | round 6 RAG |
| BR-004-71-03 | Downstream: `MCRPR50J` consumes `ACME.PERM.RPR721S1.SRT` (graph side-find). | round 6 graph |

### MCRPR77J / MCRPR85J / MCRPR99J
| ID | Rule | Source |
|---|---|---|
| BR-004-77-01 | `XXRPR77P` runs only when prior step return code `< 4` (`COND=(4,LT)`). | round 6 RAG |
| BR-004-85-07 | `MCRPR85J` packages BI2 pending requests for **MQ FTE** file transfer (`&&TEMPFTE`). | round 6 RAG |
| BR-004-99-01 | `MCRPR99` purges aged pricing DB2 rows per configurable retention. | round 6 RAG |
| BR-004-99-02 | CAST writer map: `MCRPR99` also deletes `ACME.PP_RQST_HIST_PP3H`. | round 6 CAST |

### CAST PP table writer map (round 6b C6.2)
| Table | Primary writers |
|---|---|
| `ACME.PRC_CMPNT_ITM_ST3B` | `XXRPR58` (insert), `XXRPR65` (update), `XXRPR86` (delete) |
| `ACME.PP_RQST_PP3C` | `XXRPR50`, `XXRPR53`, `XXRPR55`, `XXRPR59` (delete), `XXRPR70`, `XXRPR71`, `XXRPR85` (update), `MCRPR25` (insert) |
| `ACME.PP_RQST_HIST_PP3H` | `XXRPR59` (insert), `MCRPR99` (delete) |
| `ACME.PP_ALLOC_DTL_PP4D` / `PP_ALLOC_SUM_PP4S` | No direct writers in CAST export — allocation updates may route via `303.ACME.PP_ALLOC_PP2A` (`[CODMOD]`) |

### MCCBT family (round 6)
| ID | Rule | Source |
|---|---|---|
| BR-004-MCCBT-01 | `MCCBT03J` is **customer billing**, not cigarette — generates statements and updates customer balances from `CUSTOMER.MASTER.FILE` + `TRANSACTION.FILE`. | round 6 RAG |
| BR-004-MCCBT-02 | `MCCBT08J` is **cigarette batch-deal reporting** — comma-delimited deal extract + header for email. | round 6 RAG |
| BR-004-MCCBT-03 | `MCCBT06J` processes **deal batch headers** (batch ID extract/sort), distinct from MCCBT07 cigarette pending-deal load. | round 6 RAG |
| BR-004-MCCBT-04 | `MCCBT01J` manages **temporary batch extract files** (cleanup → create → transform → consolidate). | round 6 RAG |
| BR-004-MCCBT-05 | Marwood Maintenance Agreement: unmodified base-system portions maintained; **customer modifications forfeit Marwood maintenance on that program** (`d2105.md`). | round 6 RAG |

## 4. Data Structures

### 4.1 PP (`ACME.PP_*`) DB2 cluster

| Entity | Role |
|---|---|
| `ACME.PP_RULE_PP1R` | Price-protection rule definitions. |
| `ACME.PP_RQST_PP3C` | Price-protection requests. |
| `ACME.PP_CUST_PP1C` | PP customer scope. |
| `ACME.PP_ITEM_PP1I` | PP item scope. |
| `ACME.PP_CUST_ITEM_PP2C` | PP customer × item association. |
| `ACME.PP_CUST_GRP_PP1G` | PP customer groups. |
| `ACME.PP_ALLOC_PP2A` | PP allocation. |
| `ACME.PPA_MTHD_PP9M` | PP allocation methods. |
| `ACME.PPCUSTITEMAUD_PP7C` | PP customer-item audit. |
| `ACME.STAT_PP9S` | PP status master (states: `'PND'`, `'INP'`, `'CMP'`, plus expiration / archival). |
| `ACME.CST_CMPNT_TYP_PP9C` | Cost component type. |
| `ACME.CDATRBTVAL_CD_CU4G` | Cost-data attribute values. |
| **Round-5 additions:** |  |
| `ACME.PRC_CMPNT_ITM_ST3B` | **Pricing-component item** — heart of PP-58 (writes) and PP-86 (purges). |
| `ACME.PRC_CMPNT_ITM_ADTC` | PP-58 audit table — `XXRPR58` writes audit entries. |
| `ACME.PRC_CMPNT_ITM_PR1C` | Pricing-component item primary (referenced by `XXRPR58`). |
| `ACME.PRC_COMP_DTC` | Component date table (referenced by `XXRPR58`). |
| `ACME.PP_RQST_HIST_PP3H` | **PP request history** — target of `MCRPR59J` / `XXRPR59` archival from `ACME.PP_RQST_PP3C`. |
| `ACME.PP_ALLOC_DTL_PP4D` | PP allocation detail — written by both `XXRPR74` and `XXRPR75`. |
| `ACME.PP_ALLOC_SUM_PP4S` | PP allocation summary — written by both `XXRPR74` and `XXRPR75`. |
| `ACME.PP_ALLOC_HIST_PP4H` | PP allocation history — written by `XXRPR74`. |
| `ACME.INVC_HDR_BD1H` / `ACME.INVC_DTL_COMN_BD1D` | Read by `XXRPR74` (and by `XXDM713` in BP-005) — the invoice / Bill-Detail family is shared between PP-74 (billing→allocation sync) and BP-005 (deal-sales update). |

### 4.2 Cost (`ACME.ITM_COST_*` / `ACME.ITM_BILL_*` / `DE9E` / `DE6E`) DB2 cluster

| Entity | Role |
|---|---|
| `ACME.ITM_COST_CNTL_DE9E` (`DE9E`) | Item cost control. |
| `ACME.ITM_BILL_COST_DE6E` (`DE6E`) | Item bill cost. |
| `ACME.ITM_COST_DE8E` | Item cost. |
| `ACME.ITMCOSTCNTLAUDDE9A` | Item cost control audit. |
| `ACME.ITEM_GRP_DE1E` | Item group. |
| `ACME.COMNT_CM4A` | Comment / error log (used by `MCCST50` / `MCCST51`). |
| `ACME.CMN_ERR_CM5A` | Common error log (used by `MCCST51`). |
| `TEMP.ITEM_BILL_COST_T356` | Temporary staging table for `MCCST24`. |

### 4.3 Cigarette-deal batch entities

`ACME.BAT_DEALERRLOGDM3B` (batch deal error log), `ACME.BAT_DEAL_HDR_DM3H` (batch deal header), `ACME.BAT_DEAL_DM3L` (batch deal line), `ACME.DEALITEMDM2I`, `ACME.DEALDM1M`, `ACME.CAD_REMARK_DM3R`.

### 4.4 Configuration

`DS.APPL_SYS_AP1P` (per-application config), `DS.APPL_SYS_PARM_AP1S` (per-parameter values). Named keys observed: `MCCST24_LAST_RUN` (BR-004-53).

### 4.5 Divisional dataset families

`<DIV>.MSTR.CAD` (Computerized Allowance Data) for each of the 32 divisions.

### 4.6 Working-storage anchors

`VNKY-RECORD-STATUS-CODE`, `VNRT-COST-CONTROL-FLAG`, `CDIC-LIST-AMT-TYPE`, `WS-CUR-COST-SW`, `WS-FUT-COST-SW`, `WS-EXC-ITEM-SW`, `WS-AP1R-SW`, `COMM-Q-TYPE`, `COMM-ACTION`, `WS-TRANSID`, `WS-PROGRAM`, `WS-COMM-AREA`, `WS-COMM-LENGTH`, `BATCHID`, `ERR_TYP`.

---

## 5. Sequence Diagrams

### 5.1 End-to-end PP request lifecycle

```
Buyer/CICS ──► PP request store
                      │
                      ▼
                XXRPR50 (capture/intake)
                      │
                      ▼
                XXRPR51 (apply cost amounts)
                      │
                      ▼
                XXRPR52 (rule mgmt)  ──► XXRPR65 (status / error reclass)
                      │
                      ▼
                XXRPR53 (notify)
                      │
                      ▼
                XXRPR54 (AP1S end-of-job update)
                      │
                      ▼
                XXRPR55 (modeler report)
                      │
                      ▼
                XXRPR56 (expire/archive)
```

### 5.2 Cigarette-deal CAD update (`MCCBT07`)

```
Sorted extract file ──► MCCBT07
                          │
                          └── per division ──► <DIV>.MSTR.CAD update
```

---

## 6. Test Cases

| ID | Scenario | Expected | Maps to |
|---|---|---|---|
| TC-004-01..10 | (mirror TC-001-01..10 from BP-001 for the costing rules) | — | BR-004-01..10 |
| TC-004-20 | Cigarette-deal extract for a single division | Only that division's `<DIV>.MSTR.CAD` is updated | BR-004-21 |
| TC-004-30 | PP request capture | Row created in rule store | BR-004-30 |
| TC-004-31 | Apply cost amounts | Request line totals reflect delta cost | BR-004-31 |
| TC-004-32 | Rule lifecycle transitions | Status transitions per allowed graph (TBD via `[RAG]`) | BR-004-32, BR-004-38 |
| TC-004-33 | Notification dispatch | Email payload contains request ID + status | BR-004-33 |
| TC-004-34 | End-of-job AP1S update | Single ledger row per job | BR-004-34 |
| TC-004-35 | Modeler report generation | Report layout identical to legacy baseline | BR-004-35 |
| TC-004-36 | Rule expiration | Expired rule archived; not selected by `XXRPR50` | BR-004-36 |
| TC-004-37 | Error reclassification | Error in mid-rule processing reclassified by `XXRPR65`; pipeline continues | BR-004-37 |
| TC-004-50 | Misconfigured AP1P row | All BP-004 jobs in the cycle reflect the misconfig identically (parity test) | BR-004-50/52 |

---

## 7. Open Questions

- `[RAG]` Document `XXRPR77P` (under `MCRPR77J`) — rule-processing logic and the conditional execution criterion.
- `[RAG]` Document `XXRPR70`, `XXRPR72` (remaining top-edge `XXRPR*` programs not yet detailed).
- `[RAG]` Enumerate the **full** PP status state machine — `XXRPR56`'s expiration / archival transitions are still pending retrieval.
- `[RAG]` Document `MCCST51`'s full paragraph list (similar fidelity to `MCCST50`).
- `[SME]` Notification retry semantics (BR-004-39).
- `[SME]` Cigarette-deal categorisation — is the category code carried on the item or on the deal, and where is the routing decision made into the MCCBT pipeline?
- `[SME]` Authoritative cost source: when costs are present in both vendor uploads and the corporate cost store, which wins?
- `[SME]` MQ subsystem topology (`MQTA` for DB2T / `MQPA` for DB2P observed in `MCCST40` debug code) — production layout and failover (BR-004-69).

---

## 8. Modernization Notes

- **Blast radius (revised with CAST):** This BP contains the **single highest-complexity job in the platform**: `MCRPR50J` (CC=553, LOC=20,938, ObjectCount=919). `MCRPR71J` (CC=302) is also top-10. The full PP cycle is 13 jobs. Modernization must treat `MCRPR50J` as a discrete milestone — it cannot be bundled.
- **PP DB2 access is mutation-heavy** (HS unique to CAST): `PP_RQST_PP3C` sees 1,192 UPDATEs and 56 DELETEs (against 760 SELECTs); `PP_CUST_ITEM_PP2C` sees 184 DELETEs and 122 INSERTs. The PP cluster is the write-load hot-spot of the entire platform. Any modernized DAL for PP must support high-throughput writes; treating PP tables as read-mostly will starve the cycle.
- **Online vs batch split is wider than first thought.** The MCCST50/51 family is **CICS + MQ online**, the MCCST24 family is batch. Modernization must treat them as distinct sub-pipelines with potentially different target runtimes (e.g. event-driven service vs. scheduled job).
- **Preservation:**
  - Basis-code mapping (BR-004-04..06) is canonical and must move byte-exactly.
  - The dispatcher logic in `MCCST55` (`COMM-Q-TYPE` → transaction) is small but high-leverage — encode it as a first-class router in the modernized stack.
  - PP rule semantics are date-driven; calendar handling (timezone, fiscal calendar) MUST be locked down before any modernised rule executes.
  - PP status transitions (`PND → INP → CMP` + expiration / archival) are the price-protection contract — implement as an explicit state machine, not as scattered status assignments.
  - Configuration (`AP1P` / `AP1S`) is a shared dependency with the rest of the platform; replace deliberately (HS-007). The **checkpoint pattern** (`MCCST24_LAST_RUN`) needs an equivalent durable mechanism in the target.
- **Suggested cutover units (lowest risk first):**
  1. PP modeler reports (`XXRPR55`, `XXRPR85`) — read-only, byte-exact validatable, no upstream cutover dependency.
  2. Future-cost extract (`MCCST24`) — small batch with a clear checkpoint and idempotency story.
  3. Cigarette deals (`MCCBT06` → `MCCBT07`) — isolated category pipeline; only downstream consumer is `ACME.PENDINGDEALSDM3P` which BP-002 already understands.
  4. **CICS / MQ cost flow (`MCCST55` + `MCCST50` + `MCCST51`)** — last among costing pieces; introducing a new event-driven target while live MQ traffic flows requires careful dual-consumer design.
