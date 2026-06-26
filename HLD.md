# High-Level Design (HLD): Merchandising & Deal-Management Mainframe Platform (Legacy)

**Perspective:** Legacy / As-Is
**Opportunity:** `006UV00000aTYw1YAG`
**Version:** 1.0
**Status:** Draft — pending SME inputs
**Last Updated:** 2026-06-02
**Inputs:** RAG corpus `…/ragCorpora/2227030015734710272`, Spanner dependency graph (qatest-lmp), **CAST analysis export `mcl_cast.zip`** (snapshot 2026-05-21, source tree `acme-code/`), `_assess-research-notes.md`, `_assess-cast-summary.md`, `_assess-prompts.md`.

> **Critical context (rounds 4 + 5):** The platform is **Acme Company's customer-customised installation of Marwood Distribution System, Release 3.3.0** — a packaged COBOL/CICS centralised-buying / merchandising / distribution-management product. The DCS copybook family (DCSMCA = "Marwood Control Area", DCSWORK = "Marwood Source Book Delimiter Program") and per-copybook "SYSTEM RELEASE LEVEL.......: 3.3.0" headers confirm Marwood. The ACME codename is Acme: `XXDL742` lists `'ACME'` as `(System identifier)`; `MCDLS01` uses `'CST'` as `(ACME Business Type)`; `DCSMCA.ACME-COMPANY-SHORT-NAME PIC X(03)` carries the 3-char company short name; `ACME-CORP-LOCAL-SW` is the "corporate-local switch for centralised buying" — all consistent with Acme Company (a major US convenience-store wholesale distributor). The modernization is **replacing a packaged product at a known customer**, not bespoke code — a different problem class than originally framed.

---

## 1. Purpose & Scope

This HLD describes the **as-is** architecture of the mainframe merchandising / deal-management platform at the level needed to plan its modernization. File-by-file internals belong in the per-business-process specs (`../specs/BP-*.md`).

It is the source for:

- The system context view (external actors, file/transaction boundaries).
- The runtime view (online vs batch, JCL/CICS structure).
- The module/component view (program families and how they cluster by business process).
- The data view (DB2 schema, divisional dataset families, sequential files).
- The dependency view (graph statistics that drive the modernization sequencing).
- The cross-cutting concerns (utilities, error handling, locking, REXX scripting).
- The findings / hot-spots that the Plan phase must address.

---

## 2. System Context

```
                     ┌────────────────────────────────────────┐
                     │   "ACME" — Customer-customised          │
                     │   MARWOOD DISTRIBUTION SYSTEM REL 3.3.0 │
                     │   (packaged COBOL/CICS product)        │
                     │                                        │
                     │  ~311 COBOL executables (123 Batch +   │
                     │  143 Program + 45 "Transactional")     │
                     │  612 CopyBooks (incl. DCS* Marwood),   │
                     │  198 JCL jobs/procs, 56 CICS Maps,     │
                     │  139 KSDS VSAM, 58 GDG, 220 missing-   │
   Vendors / EDI ───►│  table DB2 references, ~32 divisions   │── Extracts ───► BI / Reporting
                     │  DB2 (ACME.*, DS.*) + VSAM/datasets      │── Sales Upd ─► Billing / AP
   Buyers (CICS) ───►│  CICS: DLZB, D21** family (mixed       │── PP notif ──► Email
                     │   batch+CICS), D8050, MCS4/MCS5/MCCST55│── Price chg ─► QUASAR  (per-div .ST2A.PP)
                     │  Batch JCL (ACME*J → XX*P), 62 jobs      │── Per-div ───► Downstream readers
                     │  Total LoC: 378,615 across 3,157 obj   │
                     └────────────────────────────────────────┘
                                       ▲
                                       │
                       Operations (JCL schedule, abend handling)
```

External boundaries:

- **Inbound files:** `NEWVNDRS`, `CORPVNDR`, vendor/cost upload datasets per division.
- **Inbound interactive:** CICS transaction `DLZB` (entry via `dlzbstrt` JCL), screen `MCCSM55`, and related online maintenance modules in the `D*` family.
- **Outbound files:** per-division deal extracts written through the `iefbr02` / `iefbr03` flow in `MCDL656J`; deal-sales updates from `XXDM713`; price-protection notifications via `XXRPR53`.
- **Outbound DB2:** downstream consumers (Billing/AP, BI) read `ACME.*` directly per the corpus excerpts.

---

## 3. Logical Module View

Counts in this section are from CAST (see `_assess-cast-summary.md`).

| Layer | What lives here | Count | Examples |
|-------|-----------------|------:|----------|
| **Source tree** | SCLM PDS libraries under `acme-code/`. | 7 libraries | `acme.perm.jcl`, `sw.perm.jcl`, `ds.perm.proclib`, `DB2P.PERM.DCLGEN`, `sclm.perm.prod.copy`, `sclm.perm.prod.copyproc`, `sclm.perm.prod.source` |
| **Orchestration (JCL Jobs)** | `ACME*J` / `XX*J` / `SW*J` / `DLZBSTRT` etc. — wire DDs and invoke COBOL programs. | **62 jobs** | `MCCAD65J`, `MCDL656J`, `MCBSM01J`, `MCBSM50J`, `MCBSM52J`, `MCBSM70J/72J`, `MCCST24J/50J/62J/63J/70J/96J`, `MCDLS50J–60J`, `MCRPR50J/55J/56J/58J/59J/65J/70J/71J/74J/77J/85J/86J/99J`, `MCCBT01J/03J/06J/08J`, `MCCIC01J/02J/20J/90J`, `MCM9014J/9091J`, `XXDLSDLY/WKY/EOP`, `XXDL650J/664J/670J/745J`, `XXDLS02J/06J/07J`, `XXD2137J`, `XXCST64J`, `XXDEI60J`, `SWCST64J`, `SWDEI60J`, `DLZBSTRT`. |
| **JCL Procedures** | Reusable JCL fragments invoked by jobs. | **136** | `XXCAD64P`, `XXCAD66P`, `XXDL702P`, `XXDL701P`, `XXDLV01P`. |
| **Business COBOL — batch (`XX*` etc.)** | Validation, transformation, reporting, archival; invoked by `ACME*J`/`XX*J` orchestrators. | **123 batch + 143 program = 266** | `XXCAD63/65`, `XXDL656/960/702/740/750`, `XXDLS01/50–60`, `XXDLC10/20`, `XXDM713`, `XXRPR50–56/65/70/72/77P/85`, `XXBSM70/71`, `XXDLV01`. |
| **CICS Transactional COBOL** | CAST tags as "Cobol Transactional Program". **Mixed batch + CICS** within a single cost/deal domain — *not* CICS-only despite the CAST tag. Use `EXEC CICS RETRIEVE / XCTL / ENQ` when online; standard batch I/O when batch. | **45** | **`D21**` family (41 progs: `D2105` … `D2199`)** — mixed (e.g., `D2137` batch purge, `D2133` item-master update, `D8050` CICS polling). Plus `MCCST50` (CICS `MCS4`), `MCCST51` (CICS `MCS5`), `MCCST55` (dispatcher). |
| **CICS Maps (screens)** | BMS maps. | **56** | `MCCSM55` is one of these. Others `[CODMOD]` to enumerate. |
| **Copybooks** | DCLGENs (`DG*`), DB2 error library (`DB*`/`DBDB2ERL`), DCS shared subsystem (`DCS*` — Marwood framework), `D*` proc/record copybooks, etc. | **612** | `DBDB2ERL` (241 calls), `DGDI1D`, `DGAP1P/1S`, `DGDE6C/1I/8E/9E`, `DGDM1X`, `DCSWORK`, `DCSMCA`, `DCSHVNDP/KVNDP`, `DCSHITMP`, `DCSFWHS`, `DCSFOPR`, `DCSFERR`, `DCST26`, `D0173P/R`, `D0402P/R/BR` (note: `DCSCOMX` and `DCSLINK` are *not* standalone copybooks — `COMX-`/`LINK` semantics live inside `DCSWORK`/`DCSMCA`). |
| **JCL data objects** | Datasets the JCL references. | 889 JCL Data Set + 173 PDS + 139 KSDS VSAM + 3 ESDS VSAM + 58 GDG + 37 Temp + 171 unknown control cards | `DS.EMER.LINK`, `DS.PERM.LINK`, `DS.PERM.SQLBATCH(SQLINFO)`, `&DI2..MSTR.VND`, `<DIV>.MSTR.SIM/VND/CAD`. |
| **External DB2 tables** | Referenced but not defined in scope. | **220 Missing Tables** | `ACME.*` + `DS.*` + `T*` families. |
| **MQ** | MQ subscribers + utilities. | 2 + 3 | Consumed by CICS transactions `MCS4`/`MCS5`. |
| **REXX utilities** | Invoked through TSO `IKJEFT01` / `IKJEFT1B`. | (handful) | `FTEREPL` exec referenced from `MCBSM70J`. |
| **Vendor-specific handlers** | Per-vendor handlers (cost + missing-record reporting). | (few) | `AUTHORKAY`, `AUTHORMCLANE`. |
| **Unreferenced** | Dead-code candidates. | **201 JCL Included members** | Removal candidates pending SME confirmation. |

**Total platform footprint:** **3,157 objects / 378,615 LoC** (CAST snapshot 2026-05-21).

### 3.1 Naming convention

| Prefix | Meaning (from corpus) | Example |
|--------|------------------------|---------|
| `ACME*J` | JCL orchestration job (corporate / merchandising) | `MCDL656J` |
| `XX*` | COBOL program invoked by the JCL | `XXDL656` |
| `MCBSM` / `XXBSM` | Business Services / business Moves (group deals) | `MCBSM52J`, `XXBSM70` |
| `MCCST` / `XXCST` | Costing | `MCCST24J`, `XXCST*` |
| `MCRPR` / `XXRPR` | Price PRotection / reporting | `MCRPR55J`, `XXRPR55` |
| `MCDLS` / `MCDL` / `XXDL` / `XXDLS` / `XXDLC` | Deals / Deal Services / Deal C-something (Cancellation? Compensation?) | `MCDLS17`, `XXDL960`, `XXDLS01`, `XXDLC10` |
| `MCCAD` / `XXCAD` | **Computerized Allowance Data** (CAD) — per `mccbt07.md` | `MCCAD65J`, `XXCAD63` |
| `MCCBT` | Cigarette deals batch | `MCCBT07` |
| `D*` | **Heterogeneous** — (a) true utilities (`D0007`/`D0325`/`D0703`/`D0173P/R`/`D0402P/R/BR`), (b) **CICS deal-capture transaction** `D8050` (polls `DM3P`, inserts into `DM1T`), (c) **the `D21**` family — 41 COBOL programs (`D2105`…`D2199`)** in the cost/deal domain. Don't assume `D*` ≡ utility. | `D0325`, `D8050`, `D2138`, `D2137`, `D2133`, `D2122` |
| `D21**` | **Cost/deal/item-master domain COBOL.** Round-5 CAST relations clarify: the 45 entries CAST tags as "Cobol Transactional Program" (`D2105`–`D2199` minus gaps, plus `D8050` / `MCCST50` / `MCCST51` / `MCCST55`) are **all genuinely CICS-only** — none has a JCL Job caller per CAST. `D2137` (and likely `D2133`) are **separate `Cobol Batch Program` entries** in the same number range — distinct CAST artefacts compiled from different source. So the D21NN number range is shared by two distinct populations; each individual program is one or the other. `D8050` is CICS-online (deal capture). | CICS: `D2138` (37 callers), `D2147`, `D2161`, `D2199`, `D8050`. Batch: `D2137` (purge), `D2133` (item-master update). |
| `DCS*` | **Marwood Distribution System** shared copybook library — system control, file handlers, comm/error patterns. **NOT bespoke customer code** — it's the Marwood framework. 12+ members in top-30 most-referenced. | `DCSMCA` (Marwood Control Area), `DCSWORK` (work fields + COMX comm codes), `DCSHVNDP/KVNDP` (vendor handlers), `DCSHITMP` (item handler), `DCSFWHS/FOPR/FERR` (warehouse/operator/error handlers), `DCST26` (file set table) |
| `MCRPR*` (round 4) | Price Protection — full 13-job cycle. **`MCRPR50J` is platform's largest** (CC=553). `MCRPR65J` produces price-change records for Quasar. | `MCRPR50J`, `MCRPR65J`, `MCRPR99J` |
| `MCCIC*` (round 4) | **Cigarette Cost** (Corporate Cigarette Cost System) — cigarette cost change management, distribution to divisional files, reporting, item-grouping cleanup. | `MCCIC01J/02J/20J/90J`, programs `MCCIC01`, `MCCIC20`, `MCCIC90` |
| `MCCBT*` (round 4) | **Mixed** — `MCCBT06`/`MCCBT07` are cigarette deals batch (BP-004); `MCCBT03J` is customer billing statements; `MCCBT01J` is batch extract management. The prefix is not monolithic — `[SME]` to confirm exact grouping. | `MCCBT01J`, `MCCBT03J`, `MCCBT06J`, `MCCBT07J`, `MCCBT08J` |
| `MCM9*` (round 4) | **Data-integrity / reconciliation.** `MCM9014J` detects discrepancies between `XX.DEALDM1X` and PowerBuilder CAD tables. `MCM9091J` fills missing cost records in divisional CAD VSAM. | `MCM9014J`, `MCM9091J`; programs `M901401/402`, `M9093` |
| `AUTHOR*` | Vendor-specific handlers | `AUTHORKAY`, `AUTHORMCLANE` |

`[SME]` confirm the exact expansion for each prefix and add to the modernized code's naming convention.

---

## 4. Runtime View

```
┌──────────────────────────────────────────────────────────────────────┐
│                              z/OS host                               │
│                                                                      │
│   ┌─────────────────────────────┐   ┌──────────────────────────────┐ │
│   │  CICS region                │   │  Batch initiator(s)          │ │
│   │  ─ Transaction DLZB         │   │  ─ JCL schedule via [SME]    │ │
│   │  ─ Screen MCCSM55           │   │  ─ ACME*J jobs invoke XX*P     │ │
│   │  ─ EXEC CICS RETRIEVE /     │   │  ─ Sort / utility steps      │ │
│   │    XCTL / ENQ               │   │  ─ REXX (FTEREPL) via        │ │
│   │  ─ Calls into D* utilities  │   │    IKJEFT01 / IKJEFT1B       │ │
│   └─────────────────────────────┘   └──────────────────────────────┘ │
│                  │                              │                    │
│                  └───────────┬──────────────────┘                    │
│                              ▼                                       │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │                        DB2 subsystem                        │    │
│   │   ─ Schema ACME.* and DS.* (table catalogue)                  │    │
│   │   ─ Stored procedures (e.g. SYSPROC.DSCON03)                │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │              VSAM / sequential / partitioned datasets       │    │
│   │   ─ <DIV>.MSTR.SIM / .VND / .CAD  (32 divisions × 3 = 96)   │    │
│   │   ─ <DIV>.PERM.* extract families                           │    │
│   │   ─ NEWVNDRS, CORPVNDR, XXOUT, EXTRACT-FILE                 │    │
│   └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

**Runtime characteristics**

- Majority of work is **batch**, but the **CICS online surface is wider than the first draft assumed** — at least 4 distinct transaction families are documented:
  - `DLZB` (BP-006) — deal/back-office maintenance, started by `dlzbstrt` JCL.
  - `D8050` (BP-002) — **deal-capture polling transaction**: periodically polls `ACME.PENDINGDEALSDM3P`, validates, promotes to `DM1T`.
  - `MCS4` / `MCCST50` (BP-004) — **MQ-driven** cost-update consumer for `COMM-Q-TYPE = 'IC1X'`.
  - `MCS5` / `MCCST51` (BP-004) — **MQ-driven** cost-update consumer for `COMM-Q-TYPE = 'CADMF'`.
  - `MCCST55` (BP-004) — CICS dispatcher routing by `COMM-Q-TYPE` to `MCCST50` / `MCCST51`.
- Programs use a uniform DB2 idiom (`SET CURRENT PACKAGE`, cursors, stored proc `SYSPROC.DSCON03` for connectivity).
- File / DB2 errors map to `RETURN-CODE` — the canonical example is `XXCAD63` returning `16` on file status `'23'` / `'10'`. **Counter-example:** `XXDLC10.E1000-FAILED-OPEN` has `MOVE +16 TO RETURN-CODE` **commented out** — a possible latent issue noted in BP-002.
- **MQ subsystem topology:** `MCCST40` references `MQTA` for DB2T and `MQPA` for DB2P; the production MQ topology and failover need SME confirmation.
- **Checkpoint / last-run state** is persisted in `DS.APPL_SYS_PARM_AP1S` (e.g. `MCCST24_LAST_RUN` set by `MCCST24.4000-UPDATE-AP1S-PARA` on success).

---

## 5. Data View

### 5.1 DB2 schema — most-active tables (CAST `Number_of_linked_transactions_per_data_source`)

The CAST view shows full SELECT / INSERT / UPDATE / DELETE breakdowns per (program, table). This **supersedes** the round-1 RAG "programs touching it" estimate.

| Table | Total tx | SEL | INS | UPD | DEL | Pattern |
|---|--:|--:|--:|--:|--:|---|
| `DIVMSTRDI1D` | **7,465** | 7,465 | 0 | 0 | 0 | Pure RO reference |
| `DIV_ITEM_PACK_DE1I` | 4,112 | 4,112 | 0 | 0 | 0 | Pure RO reference |
| `SYSDUMMY1` | 4,044 | 4,044 | 0 | 0 | 0 | DB2 utility |
| `APPL_SYS_PARM_AP1S` | 2,536 | 2,040 | 0 | 496 | 0 | Config + checkpoint writes |
| `DT_JS1A` | 2,262 | 2,262 | 0 | 0 | 0 | Date master (RO) |
| `UIN_ITEM_DE6C` | 2,107 | 2,069 | 0 | 38 | 0 | Reference + rare update |
| `PP_RQST_PP3C` | 2,008 | 760 | 0 | **1,192** | 56 | **Heaviest mutation on platform** |
| `DEALDM1X` | 1,874 | 1,514 | 0 | 302 | 58 | Deal data mart |
| `PP_CUST_ITEM_PP2C` | 1,667 | 697 | 122 | 664 | **184** | Highest insert+delete churn |
| `PP_RULE_PP1R` | 1,623 | 856 | 0 | 727 | 40 | PP rules |
| `PP_CUST_GRP_PP1G` | 1,335 | 753 | 0 | 542 | 40 | PP customer groups |
| `VNDR_MSTR_VN1A` | 1,272 | 1,272 | 0 | 0 | 0 | Vendor master (RO) |
| `DIV_CUST_CU1Q` | 1,149 | 673 | 0 | 340 | 136 | Division customer |
| `ITM_COST_CNTL_DE9E` | 1,072 | 986 | 28 | 0 | 58 | Cost control |
| `CUST_CLS_CU2A` | 1,003 | 867 | 0 | 136 | 0 | Customer classification |
| `ITM_COST_DE8E` | 999 | 913 | 0 | 0 | 86 | Item cost |
| `ITM_BILL_COST_DE6E` | 958 | 900 | 0 | 0 | 58 | Item bill cost |
| `CLS_GRP_DESC_CU2B` | 941 | 941 | 0 | 0 | 0 | Class group description |
| `CUST_XREF_CU1X` | 831 | 695 | 0 | 0 | 136 | Customer xref |
| `PP_ALLOC_PP2A` | 815 | 635 | 0 | 140 | 40 | PP allocation |

**Caching candidates (zero or near-zero writes):** `DIVMSTRDI1D`, `DIV_ITEM_PACK_DE1I`, `DT_JS1A`, `UIN_ITEM_DE6C`, `VNDR_MSTR_VN1A`, `CLS_GRP_DESC_CU2B`, `CUST_XREF_CU1X` — natural read-through cache targets in any modernized DAL.

### 5.1b Round-4 DB2 additions

| Entity | Role | Surfaced from |
|---|---|---|
| `ACME.CIG_ITEM_COST_PR1C` | Cigarette item cost (Corporate Cigarette Cost System) | `MCCIC20J`, `MCCIC90J` |
| `DEALAUDITDM1A` | Deal audit (purged by `XXDL980P`) | `__name.md` purge job |
| `XX.DEALDM1X` (qualified form of `DEALDM1X`) | Same as DEALDM1X; appears qualified in `MCM9014J` context | `MCM9014J` |
| `ACME.PERM.RPR50S1` / `RPR50S1.SRT` / `RPR51S1` / `RPR51S1.SRT` / `RPR51S3.SRT` | PP-50 cycle intermediate files | `MCRPR50J` |
| `ACME.PERM.RPR50.STAT` / `RPR51.STAT` / `RPR52.STAT` / `RPR.STAT` | PP-50 cycle status logs (consolidated to `RPR.STAT` for email) | `MCRPR50J` |
| `ACME.PERM.RPR72RUL` / `RPR73RUL` | PP rule files (read by MCRPR50J) | `MCRPR50J` |
| `ACME.PERM.RPR71S1.ITEMS` / `RPR71S1.CUSTS` | PP item / customer scope files | `MCRPR50J` |
| `DS.PERM.SORTPARM(XXRPR501/511/512/71I/71C)` | Sort param PDS members for MCRPR50J SORT steps | `MCRPR50J` |
| `DS.PERM.RDRPARM(RPREML53/PRTONLY1)` | Reader/printer param PDS members | `MCRPR50J` |
| `DS.PERM.MCBSM04.LICDATA`, `ACME.PERM.LICENSE.DAILY` | License data (read+written by `MCBSM01J`) | `MCBSM01J` |
| `<DIV>.PERM.<DIV>ST2A.PP` | **Per-division price-change feed to QUASAR** (external system). Confirmed for MS, MN, GM, MP, MW, MG, C3 divisions. | `MCRPR65J`, divisional dataset docs |
| **Round-5 additions (PP + PRC clusters):** `ACME.PRC_CMPNT_ITM_ST3B` (PP rule application target — MCRPR58J writes / MCRPR86J purges), `ACME.PRC_CMPNT_ITM_ADTC` (audit), `ACME.PRC_CMPNT_ITM_PR1C`, `ACME.PRC_COMP_DTC`, **`ACME.PP_RQST_HIST_PP3H`** (PP request history — target of MCRPR59J archival), `ACME.PP_ALLOC_DTL_PP4D` / `ACME.PP_ALLOC_SUM_PP4S` / `ACME.PP_ALLOC_HIST_PP4H` (PP allocation cluster — written by MCRPR74J/XXRPR74/75 from invoice data), `STARITEMCATST1I` (STAR item category — read by XXDL571P deal-alert), `ITEMAUTHST1A` (item authorisation), `ACME.CIG_ITEM_COST_PR1C` (cigarette item cost — CIC family). | round 5 |
| **DWSLS producer chain** | `ACME.PERM.BSM30S1.DWSLS` is written by **`MCBSM50J`** (JCL) → `XXBSM32P` (proc) → `XXBSM32` (COBOL batch) → `5000-WRITE-DATA` paragraph. Read by `MCDLS50J`'s `SORTSUM` step. | round 5 (CAST relations) |
| `ACME.PERM.BSM55S1` | Vendor cleansed-name report (cleaned commas etc.); produced by `MCBSM52J` | `MCBSM52J` |
| `TEMP.VNDR_ID_DIV_T312` (cursor source in MCBSM52J) | Vendor temp table feeding the BSM55S1 join | `MCBSM52J` |

### 5.2 Infrastructure / linkage datasets (the "Most-Referenced Data Objects" perspective)

By **distinct call-site count**, the highest-referenced data objects include shared infrastructure:

| DataSource | Type | Calls | Role |
|---|---|--:|---|
| `DS.EMER.LINK` | JCL Data Set | 250 | DB2 plan linkage — **emergency** |
| `DS.PERM.LINK` | JCL Data Set | 250 | DB2 plan linkage — **permanent** |
| `DS.PERM.SQLBATCH(SQLINFO)` | PDS Dataset | 226 | SQL batch toolkit info member |
| `&DI2..MSTR.VND` | **KSDS VSAM** | 62 | Divisional VSAM vendor master (substituted by division at run-time) |

`DS.EMER.LINK` / `DS.PERM.LINK` appear in nearly every SQL-using JCL job; they are the DB2-plan linkage convention, not domain data.
| `ACME.PROF_HDR_PR1P`, `ACME.PROF_CUS_PR3Q`, `ACME.PROF_DEAL_PR4D`, `ACME.DEAL_SUPP_DS1R`, `ACME.ITEM_SUPP_IT2A`, `ACME.ITEM_MASTER_IM3I`, `ACME.CUST_XREF_CU1X` | (suppression cluster) | Deal/customer/item suppression model used by `XXDLS01` |
| `CRP_VNDR_XREF_VN1X` | (corpus) | Corporate-to-division vendor cross-reference |
| **Deal-lifecycle (added in round 2):** `DM1T` (Deal Transaction — target of `D8050`), `DM1E` (D8050 restart), `ACME.CAD_ERR_LOG_DM3E` (CAD error log), `ACME.DIVPENDDEALSDM3D` (divisional pending pointer), `ACME.GRPPENDDEALDM3G` (group pending pointer), `DEAL_ANALYSIS_DM1L`, `DEALLOGDM5X`, `ACME.MCLANE_XREF_DI3X`, `ACME.DT_JS1A` | (lifecycle cluster) | Used across `D8050`, `XXDL740`, `XXDLC10`, `XXDLC20`. |
| **Cigarette-deal batch (added in round 2):** `ACME.BAT_DEALERRLOGDM3B`, `ACME.BAT_DEAL_HDR_DM3H`, `ACME.BAT_DEAL_DM3L`, `ACME.DEALITEMDM2I`, `ACME.DEALDM1M`, `ACME.CAD_REMARK_DM3R` | (CAD batch cluster) | Used by `MCCBT06` / `MCCBT07`. |
| **Price-Protection (added in round 2):** `ACME.PP_RULE_PP1R`, `ACME.PP_RQST_PP3C`, `ACME.PP_CUST_PP1C`, `ACME.PP_ITEM_PP1I`, `ACME.PP_CUST_ITEM_PP2C`, `ACME.PP_CUST_GRP_PP1G`, `ACME.PP_ALLOC_PP2A`, `ACME.PPA_MTHD_PP9M`, `ACME.PPCUSTITEMAUD_PP7C`, `ACME.STAT_PP9S`, `ACME.CST_CMPNT_TYP_PP9C`, `ACME.CDATRBTVAL_CD_CU4G` | (PP cluster) | Used across `XXRPR50`..`XXRPR65`, `XXRPR55`, `XXRPR85`. |
| **Invoice / Bill Detail (added in round 2):** `ACME.INVC_HDR_BD1H`, `ACME.INVC_DTL_COMN_BD1D`, `ACME.INVC_DTL_ITEM_BD2D`, `ACME.INVC_TS_BD2T`, `DATECNTLCF1D`, `ACME.US_TAX_CU4U` | (invoice cluster) | Read by `XXDM713` (BP-005). |
| **CICS / MQ cost (added in round 2):** `ACME.ITM_COST_CNTL_DE9E`, `ACME.ITM_BILL_COST_DE6E`, `ACME.ITM_COST_DE8E`, `ACME.ITMCOSTCNTLAUDDE9A`, `ACME.ITEM_GRP_DE1E`, `ACME.COMNT_CM4A`, `ACME.CMN_ERR_CM5A`, `TEMP.ITEM_BILL_COST_T356` | (cost cluster) | Used by `MCCST50`, `MCCST51`, `MCCST24`. |

### 5.2 Divisional dataset families

The platform is organised around **32 divisions** identified by two-letter codes. For each division `<DIV>` the platform owns three master families:

| Family | Purpose | Example |
|---|---|---|
| `<DIV>.MSTR.SIM` | Item master | `MZ.MSTR.SIM`, `SW.MSTR.SIM`, `SZ.MSTR.SIM`, … |
| `<DIV>.MSTR.VND` | Vendor master | `WJ.MSTR.VND`, `HP.MSTR.VND`, … |
| `<DIV>.MSTR.CAD` | **Computerized Allowance Data** master | `MZ.MSTR.CAD`, `NC.MSTR.CAD`, `FE.MSTR.CAD`, … |

Plus per-job extract families such as `<DIV>.PERM.<DIV>DL656{1,2,3}` (visible in the `MCDL656J` graph).

The 32 division codes captured by the graph are: `MZ, SW, SZ, SE, NC, ME, NE, PA, NW, SO, MS, MK, MP, MI, MO, HP, MW, MD, MY, MN, WJ, FE, MG, NT, GM, GA, GF, WK, C1, C2, C3, ACME`. `[SME]` translate each code into a human-readable division name and add to this table.

### 5.3 Sequential / utility files

`NEWVNDRS` (inbound new vendors), `CORPVNDR` (corporate vendors), `XXOUT` (generic XX-program output), `EXTRACT-FILE` (used by several `XX*` programs), and the `iefbr02` / `iefbr03` no-op placeholder steps that consume per-division datasets in `MCDL656J`.

---

## 6. Program Inventory & Complexity (CAST `Transactions_Complexity`)

CAST gives precise cyclomatic / essential / integration complexity per **JCL transaction** (i.e., per job entry-point). The full 86-row table is in `_assess-cast-summary.md` §"Transaction complexity"; top-15 by CC reproduced here as the **modernization risk heatmap**:

| Rank | JCL Job / Proc | CC | EC | IC | LOC | Obj# | Owning BP |
|--:|---|--:|--:|--:|--:|--:|---|
| 1 | **`MCRPR50J`** | **553** | 320 | 42 | 20,938 | 919 | BP-004 |
| 2 | `MCBSM01J` | 486 | 71 | 32 | 14,552 | 645 | BP-002/003 |
| 3 | `XXD2137J` | 416 | 138 | 9 | 8,785 | 218 | BP-002 |
| 4 | `XXDLSWKY` | 414 | 179 | 24 | 14,693 | 515 | BP-002 (weekly) |
| 5 | `XXDLSDLY` | 375 | 83 | 31 | 11,646 | 344 | BP-002 (daily) |
| 6 | `XXDLSEOP` | 343 | 180 | 20 | 12,594 | 485 | BP-002 (EOP) |
| 7 | `XXDL745J` | 334 | 113 | 9 | 10,083 | 404 | BP-002 |
| 8 | `MCRPR71J` | 302 | 183 | 24 | 12,972 | 549 | BP-004 |
| 9 | `MCDL656J` | 298 | 126 | 16 | 13,268 | 594 | BP-002 |
| 10 | `MCBSM50J` | 274 | 98 | 21 | 9,415 | 444 | BP-002/003 |
| 11 | `XXDL664J` | 264 | 79 | 8 | 8,123 | 328 | BP-002 |
| 12 | `XXDL650J` | 247 | 127 | 26 | 8,287 | 384 | BP-002 |
| 13 | `MCBSM52J` | 196 | 18 | 20 | 6,157 | 371 | BP-003 |
| 14 | `MCCAD65J` | 149 | 41 | 9 | 6,448 | 450 | BP-001/004 |
| 15 | `MCDLS54J` | 115 | 65 | 5 | 5,457 | 196 | BP-002 |

Totals across the 86-transaction set: **CC = 7,591**, **LOC = 306,022**, **ObjectCount = 14,146**.

`CC` (cyclomatic) > 100 is the conventional "high-risk / hard-to-test" threshold; **22 of 86 transactions** exceed it. The two jobs above 400 (`MCRPR50J`, `MCBSM01J`, `XXD2137J`, `XXDLSWKY`) are the system's true heavy-lifters and the primary modernization sequencing constraints.

> The earlier "20 most-connected programs by edge count" table was a graph-edges proxy from the Spanner dependency graph; it mixed call edges and data-entity references and ranked **subroutines** (`XXRPR55`, `MCCST50` etc.) rather than the **transactions** that drive them. The CAST view above is the right modernization risk metric.

---

## 7. Cross-Cutting Concerns

### 7.1 DB2 access

- All programs follow the same idiom: set DB2 package, open cursor(s), fetch rows, evaluate, write outputs, close cursor(s), terminate. Connectivity goes through `SYSPROC.DSCON03`.
- DB2 errors are translated to documented return codes; `XXDLS01` and `XXCAD63` both demonstrate the pattern.

### 7.2 Concurrency (CICS)

- Online uses `EXEC CICS ENQ` to serialise access to shared resources. `[CODMOD]` enumerate the named ENQ resources so the modernized system can implement equivalent semantics.

### 7.3 Error handling

- File-status `'23'` (record not found) and `'10'` (end-of-file unexpected) → `RETURN-CODE = 16` and abnormal termination (`XXCAD63`).
- Exception items receive a softer treatment: `WS-EXC-ITEM-SW = 'Y'` AND `WS-AP1R-SW = 'Y'` → log `"ITEM EXCEPTION - NO UPDATES"` and continue (`XXCAD63`).

### 7.4 Logging & observability

- Programs log via standard COBOL `DISPLAY` statements; logs collect in SYSOUT and the JCL listing. There is no central observability stack identifiable from the corpus. Modernization must preserve the **content** of these logs while changing the format to structured logging.

### 7.5 Time handling

- Programs read the current timestamp at initialisation, parse it into date/time components, and use them for downstream date math (e.g. *today − 2y* in `XXDL960`; date-discrepancy checks in `XXCAD63`).
- Day information for some programs comes from a `CM1G` reference table (`XXDLV01`).

### 7.6 Configuration

- `DS.APPL_SYS_AP1P` (64 programs) and `DS.APPL_SYS_PARM_AP1S` (39 programs) are the de-facto runtime configuration tables. Any modernized configuration store must absorb both.

### 7.7 Security

- z/OS RACF identities govern execution today. `[SME]` produce the RACF group → business process mapping.

### 7.8 Operations

- Batch is JCL-driven; the scheduler is not surfaced in the corpus. `[SME]` confirm CA-7 / Control-M / OPC and provide the runbook.
- Vendor-specific operations (`AUTHORKAY`, `AUTHORMCLANE`) likely have dedicated runbook entries.

---

## 8. Dependency Graph (Modernization-Driver View)

The graph tells us:

- **Centrality.** `MCBSM04` (36 edges) is the single most-connected program. Whatever it does (business services / group deals — `[CODMOD]` extract) is on the critical path for at least every group-deal pipeline.
- **Data-entity centrality.** `ACME.DIVMSTRDI1D` (124 referencing programs) and `DS.APPL_SYS_AP1P` (64) are the heaviest data dependencies — modernizing them touches the broadest blast radius.
- **Business-process clustering.** Programs cluster cleanly by prefix into the five BPs in `../specs/`. A *per-BP cutover plan* is the natural sequencing.

A precise blast-radius report (per DB2 entity → affected programs → affected BPs) is generated by joining the graph with the per-BP spec catalogue. Plan/Design phase consumes this directly.

---

## 9. Modernization Hot-Spots (Findings Surfaced by the Assessment)

- **HS-001 — `MCBSM04` and the BSM cluster are a bottleneck.** Highest-edge program plus four other `MCBSM*` / `XXBSM*` in the top 20. Plan must address group-deal pipeline modernization carefully.
- **HS-002 — `ACME.DIVMSTRDI1D` saturates the system.** 124 programs depend on it. A wrapper / facade should be introduced early to decouple readers from the storage layer.
- **HS-003 — Suppression decision is rich rule logic in a single COBOL module.** `XXDLS01` is a critical decision module reading from 7+ DB2 tables. Extract into a named, testable rules service in the target.
- **HS-004 — Cost basis mapping is canonical and small.** `XXCAD63` basis-code mapping (`'F'`→`'ACME'`, `'1'`→`'CW'`, default `'SC'`) is a clear candidate for an early modernised rule with byte-exact test coverage.
- **HS-005 — Divisional model.** 32 divisions × 3 master families = 96 long-lived dataset families. The Design phase must define how this is represented (single multi-tenant table with `division_code` column? per-division schemas? per-division pipelines?).
- **HS-006 — Vendor-specific code paths.** `AUTHORKAY` / `AUTHORMCLANE` suggest a per-vendor handler pattern that may apply more broadly. Identify all such handlers before designing the vendor module.
- **HS-007 — Configuration is two DB2 tables.** `DS.APPL_SYS_AP1P` + `DS.APPL_SYS_PARM_AP1S` together are the config store. Replace deliberately; misconfiguration in this layer cascades everywhere.
- **HS-008 — Lifecycle preservation.** The pending → captured → active → billed → accrued → archived chain has visible programs in every stage (`ACME.PENDINGDEALSDM3P` → `D8050` → `DM1T` → `XXDL702` → `XXDLC10` → `XXDLC20` → `XXDL960`). All stages must be cut over together or behind a coexistence layer. **`D8050` is the gateway** — pending deals only become deal transactions through it.
- **HS-009 — CICS + MQ event-driven costing.** `MCCST50` / `MCCST51` / `MCCST55` form an MQ-driven event consumer pipeline inside CICS, not a batch costing one. Modernization target must support an equivalent event-driven runtime — this is qualitatively different from porting the batch `MCCST24J` family.
- **HS-010 — Edge counts in the graph are data + call edges combined.** A program's "edge count" mixes COBOL→COBOL call edges with COBOL→DB2-entity references. `MCBSM04`'s 36 edges = 30 DB2-table references + a small number of call edges. Plan-phase blast-radius reports must split these two views.
- **HS-011 — Corpus boilerplate dominates RAG chunks.** Many program markdown files have ~800 tokens of CSS / `tooltip` boilerplate at the top, starving the top-k retrieval of business content. The corpus needs an HTML-stripping re-ingest before Plan-phase analysts depend on it.
- **HS-012 — `MCRPR50J` is the single largest modernization risk.** CC=553, LOC=20,938, references 919 distinct objects — by every CAST metric the most complex transaction in the platform. It is in the PP cycle (BP-004). Any "PP modernization" plan that treats `MCRPR55J` (CC=23 per RAG-anchored anchor) as representative is mis-scoping by an order of magnitude.
- **HS-013 — `DBDB2ERL` is the canonical DB2-error idiom.** 241 copybook call sites. Any modernization of DB2 access starts with a strategy for this copybook — preserve as-is via a shim, or replace as part of the new DAL. There is no third option.
- **HS-014 — `DCS*` is an undeclared shared subsystem.** 12+ `DCS*` copybooks appear in the top-30 most-referenced. The `DCS*` library acts as the platform's common runtime services layer, but its scope and naming expansion is not yet recovered. `[SME]` to characterise.
- **HS-015 — Temporal cycles (`XXDLSDLY` / `XXDLSWKY` / `XXDLSEOP`) are the deal lifecycle's heartbeat.** Each is top-6 in complexity. Modernizing the deal cycle without first understanding the daily/weekly/EOP boundary semantics will break operational expectations.
- **HS-016 — 220 "Missing Tables" are external DB2 dependencies.** The DDL for these tables is outside the analysed scope. Plan phase needs an explicit step to obtain it (DBA-supplied schema dump, or scope-extension of CAST analysis).
- **HS-017 — The platform is a customised Marwood Distribution System (Release 3.3.0).** This is the most important architectural reframe surfaced by round 4. Implications:
  - The DCS copybook layer is **Marwood framework code**, not customer business logic. Up to ~80% of the cross-cutting plumbing (vendor handlers, item handlers, CICS comm exits, file handlers, error handlers) may be removable if the modernization target replaces Marwood wholesale.
  - The business logic *unique* to this installation likely lives in the `ACME*` jobs, the `XX*` programs, the `D21**` family, and the divisional CAD files — the "what the customer did on top of Marwood" layer.
  - Plan-phase modernization options become: (a) **lift-and-shift Marwood** onto a modern host (preserves everything; lowest risk; no functional change), (b) **replace Marwood entirely** with a modern distribution-management platform (highest value; highest risk), (c) **extract-and-rebuild** — keep the business-logic specs, abandon the Marwood framework, rebuild on a modern stack (middle path).
  - REL 3.3.0 dating: `MWD CONVERSION TO 3.3.0 RELEASE` is the most recent version footprint. `[SME]` confirm whether Marwood as a product still exists / is supported, and what the customer's contractual obligations are.
- **HS-018 — Quasar is a fixed external interface.** Per-division `<DIV>.PERM.<DIV>ST2A.PP` price-change feeds go from ACME to a downstream system named "Quasar" — confirmed for at least 7 of the 32 divisions. The Quasar boundary file format and cadence MUST be preserved in modernization (Quasar is out of scope to modernize in this engagement).
- **HS-019 — The platform self-heals via the MCM9 cycle.** `MCM9014J` detects deal/CAD discrepancies; `MCM9091J` fills missing cost records in CAD VSAM. These are not domain features — they are integrity guardrails compensating for legacy distribution patterns. Modernization should engineer the new platform so this self-healing is **unnecessary** (rather than reproducing it).
- **HS-020 — Cigarette cost is a first-class business domain.** Two whole job families — `MCCBT*` (cigarette deals batch) and `MCCIC*` (cigarette cost) — exist for cigarettes alone. Plus the `ACME.CIG_ITEM_COST_PR1C` table. Round-5 confirms the customer is **Acme Company** — a major US convenience-store wholesale distributor — so cigarette is a top-3 SKU category. Plan phase: confirm cigarette-specific regulatory and reporting requirements (state-by-state tobacco tax, master settlement, distributor reporting) that the modernized platform must continue to satisfy.
- **HS-021 — Centralised buying / corporate-local split is built into Marwood's data model.** `DCSMCA.ACME-CORP-LOCAL-SW` switches each record between "centralised buying" and "local buying" modes. Modernization must preserve this two-mode operation; collapsing it would break Acme's purchasing operations across the 32 divisions.
- **HS-022 — Purge cycle parent JCL = `XXDLSWKY` (resolved round 6).** CAST `Transactions↔Objects` shows all six purge/reporting procedures invoke under **`XXDLSWKY`** (CC=414). The corpus still references normalised `__name.md` / `%%NAME` — schedule modernization against the CAST name. `XXDLSWKY` is also the **weekly deal-cycle** job, so weekly boundary semantics and purge/reporting are coupled in one JCL transaction.
- **HS-023 — `D2138` is a hidden shared subroutine hub.** Eleven `D21**` programs call `D2138` but `d2138.md` is corpus-empty. Modernizing any caller without reverse-engineering `D2138` risks breaking PO, deal, and cost-inquiry flows. Treat `D2138`/`D2139` as a **framework submodule** alongside the DCS handlers.
- **HS-024 — Temporal deal cycles remain high-risk with thin documentation.** `XXDLSDLY` (CC=375) still has no assessment markdown; `XXDLSEOP` (CC=343) is only partially described via `mcdl656j.md`. CAST names them; corpus does not explain boundary semantics — `[SME]` + codmod re-run required before cutover planning.

---

## 10. Quality Attributes (Architectural)

| Attribute | Current mechanism | Modernization stance |
|-----------|------------------|----------------------|
| Availability | Single z/OS host with regional DR. | Preserved or improved per target stack (Plan-phase). |
| Performance | Batch-window-sensitive; CICS for online. | Preserved; specific p95s captured per BP. |
| Maintainability | **Low** — 312 COBOL programs with implicit business rules. | **Primary improvement target.** Named rules, tested DAL, per-BP modular structure. |
| Auditability | Strong (JCL listings, DB2 audit). | Preserve. Add structured logging. |
| Recoverability | DB2 + dataset backups + RACF. | Match DB2 durability semantics in target. |

---

## 11. Open Architectural Questions

- `[SME]` What scheduler runs the JCL (CA-7 / Control-M / OPC)?
- `[SME]` What human-readable name maps to each two-letter division code?
- `[SME]` Are there programs that bypass the standard `ACME*J → XX*P` orchestration pattern?
- `[CODMOD]` Enumerate all `EXEC CICS ENQ` resources to capture the system's serialisation semantics.
- `[RAG]` Document the remaining top-edge programs not yet detailed (`MCBSM03`, `MCBSM30`, `MCRPR99`, `XXRPR70`, `XXRPR72`, `XXBSM50`, `XXDL655`, `XXDLS50`, `XXDLS54`, `MCCST51` internals, `MCCST96`).
- `[CAST]` The `D21**` CICS family (41 programs) and the 56 CICS Maps need to be classified into per-business-process specs — they were entirely missed by the RAG-only first pass.
- `[CORPUS-EMPTY]` `D2122`, `XXDM713` — need corpus re-ingest or SME walkthrough; CAST confirms they exist as CICS-trans / batch COBOL, but the descriptive markdown is empty.
- `[SME]` Confirm Marwood vendor commercial status / EOL (round 6 surfaced maintenance-agreement policy only; ACME = Acme confirmed round 5).
- `[SME]` Expand `DCS*` copybook family — what does "DCS" stand for? It's a 12+-member shared library hit by hundreds of programs.
- `[SME]` Identify the business function of the `MCCIC*J` family (4 jobs) and the `MCM9*J` family (2 jobs).
- `[SME]` Identify the `SW*` JCL job semantics — division-specific (SW = a regional code in the divisional list) or "Software"?
- `[SME]` Stated modernization target stack (not present in any data source so far).
- `[SME]` MQ subsystem topology (`MQTA`/DB2T vs `MQPA`/DB2P observed in `MCCST40`).
- `[SME]` Confirm whether the 201 unreferenced `JCL Included` members are dead code or intentional templates.

---

**Document Control**
- Author: Assess Agent (yourbuddy-hybrid + RAG + Spanner Graph) + Human SME (pending)
- Inputs: codmod (when available) + MAT dependency graph (Spanner) + SME interviews.
- Approval gate: Required before any per-BP spec is approved.
