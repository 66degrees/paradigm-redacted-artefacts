# Product Requirements Document (PRD): Merchandising & Deal-Management Mainframe Platform (Legacy)

**Perspective:** Legacy / As-Is
**Opportunity:** `006UV00000aTYw1YAG`
**Source corpus:** `projects/qatest-lmp/locations/europe-west4/ragCorpora/2227030015734710272` (CPE-Legacy)
**Status:** Draft — pending SME review
**Last Updated:** 2026-06-02
**See also:** [`_assess-research-notes.md`](../_assess-research-notes.md) for the raw research log.

---

## 1. Executive Summary

The legacy system under assessment is a **z/OS mainframe platform** that runs a major retailer's **merchandising, deal management, costing, and vendor data** functions. It consists of **312 distinct COBOL programs** orchestrated by JCL jobs (and a smaller CICS online surface), reading and writing **DB2 tables under the `ACME.*` schema** alongside **divisional VSAM/dataset families** keyed by a two-letter division code (32 divisions in scope: `MZ, SW, SZ, SE, NC, ME, NE, PA, NW, SO, MS, MK, MP, MI, MO, HP, MW, MD, MY, MN, WJ, FE, MG, NT, GM, GA, GF, WK, C1, C2, C3, ACME`).

Functionally, it is the system of record for:

- **Item master data** (`<DIV>.MSTR.SIM` families and DB2 `ACME.ITEM_MASTER_IM3I`, `ACME.UIN_ITEM_DE6C`, `ACME.DIV_ITEM_PACK_DE1I`, `ACME.ITEM_UPC_DE6Y`, `ACME.ITEM_VNDR_DE6V`).
- **Vendor master data** (`<DIV>.MSTR.VND` families and DB2 `ACME.VNDR_MSTR_VN1A`, `CRP_VNDR_XREF_VN1X`).
- **Deals — pending, active, billed, accrued, archived** (`ACME.PENDINGDEALSDM3P`, `DEALDM1X`, `ACME.DEAL_SUPP_DS1R`, `ACME.PROF_DEAL_PR4D`).
- **Costing & price protection** (`<DIV>.MSTR.CAD` families, the `XXCAD*` / `MCCST*` / `MCRPR*` program families).
- **Customer / profile suppression** (`ACME.PROF_HDR_PR1P`, `ACME.PROF_CUS_PR3Q`, `ACME.CUST_XREF_CU1X`, `ACME.ITEM_SUPP_IT2A`).
- A small **CICS online surface** for deal/back-office maintenance (transaction `DLZB`, screen `MCCSM55`).

## 2. Business Context

### 2.1 Why the system exists

| Driver | Description |
|--------|-------------|
| Single source of truth for merchandising | Items, vendors, deals, costs, and the suppression rules that govern them are all owned by this platform. Downstream systems (BI, billing, replenishment, store systems) depend on its data and its outbound reports. |
| Multi-divisional accounting | The 32 divisional `MSTR` families exist because each business division has its own item, vendor, and cost/deal master with shared corporate-level keys; the platform reconciles divisional vs. corporate views. |
| Deal lifecycle integrity | Deals progress through pending → active → billed → accrued → archived; each transition has audit, suppression, and accrual implications. Breaking that lifecycle breaks vendor billing and rebate accounting. |
| Cost-control governance | Costs are not free-form: vendor flags (`VNRT-COST-CONTROL-FLAG`), record-status codes (`VNKY-RECORD-STATUS-CODE = 'A'`), and basis-code mappings (`CDIC-LIST-AMT-TYPE` → `ACME` / `CW` / `SC`) act as gates on what can be priced and how. |
| Stability over decades | The naming conventions (`ACME*J` jobs orchestrating `XX*P` programs, `D*` utilities) and the breadth of the program graph (top program `MCBSM04` alone has 36 graph edges) demonstrate a system that has accreted three decades of business logic with very limited rewrite. |

### 2.2 Strategic position

The platform sits at the **centre** of the retailer's merchandising data plane:

```
       ┌────────────────┐                              ┌─────────────────┐
       │ Vendors / EDI  │ ── new costs / vendor data ─▶│  Mainframe      │
       └────────────────┘                              │  Merchandising  │── master files / extracts ──▶ Stores / BI / Billing
       ┌────────────────┐ ── deal proposals / pending ▶│  Platform       │
       │ Merchants /    │                              │  (this system)  │
       │ Buyers (CICS)  │ ── price-protection / cost  ▶│                 │── DB2 ───────────────────────▶ Downstream readers
       └────────────────┘                              └─────────────────┘
```

It is **not customer-facing**, but the integrity of every customer-facing pricing, billing, and replenishment process depends on what it stores and emits.

## 3. Goals & Non-Goals (of the Legacy System)

### 3.1 Goals the legacy system meets today

- Maintain item, vendor, deal, and cost master data with **divisional + corporate** views always reconciled.
- Run a **gated deal lifecycle** with explicit suppression, accrual, and archival rules (the most-cited business rules in the corpus live here — see `XXDLS01`, `XXDL702`, `XXDL960`).
- Validate **vendor costs** end-to-end (status code, control flag, basis-code mapping, discrepancy reporting — see `XXCAD63`).
- Produce extracts and reports for downstream consumers (BI, billing, vendor reconciliation).
- Maintain a CICS online surface for buyers / merchants to maintain deals and price-protection rules.

### 3.2 Non-goals of the legacy system

- Real-time pricing decisions at the register (handled by store systems consuming this system's outputs).
- E-commerce or web channel logic.
- Direct integrations with cloud SaaS — all integrations are file-based or DB2-table-based.

## 4. Users & Stakeholders

| Stakeholder | Relationship to the platform |
|-------------|------------------------------|
| Buyers / merchants | Direct — interact via CICS screens (e.g. `MCCSM55`, transaction `DLZB`) to maintain deals and price-protection rules. |
| Vendor master team | Direct — maintain vendor records via online and batch paths (`MCBSM52J` vendor sync, `MCCST19` vendor lookup, `AUTHORKAY` / `AUTHORMCLANE` vendor-specific handlers). |
| Costing / pricing analysts | Direct — drive `MCCST24J`, `MCCST50J`, `MCCST63J`, `MCCST96J`, `XXCAD63` to validate and reconcile costs. |
| Billing & accounts-payable | Indirect downstream — consume deal-accrual and active-billing outputs (`XXDLC10`, `XXDLC20`, `XXDL702`). |
| BI / analytics | Indirect downstream — consume `XXDLS50` (Deal Analysis for BI Reporting) and the deal extracts. |
| Operations | Run the JCL schedule, monitor abends, recover failed jobs (file-status `'23'` / `'10'` → `RETURN-CODE = 16` in `XXCAD63`). |
| Vendor-specific operations | `AUTHORKAY` and `AUTHORMCLANE` indicate specialised handlers for at least two named vendors (Kay; Acme). |

## 5. Strategic Pillars for the Modernization

These pillars constrain what the Plan and Design phases may propose. They are derived from observed behaviour, not stated preferences (the corpus contains **no explicit target stack**).

1. **Behavioural parity.** Every documented rule in `docs/legacy/03-technical/specs/` must execute identically in the target. Examples: `XXDLS01` suppression decision; `XXCAD63` basis-code mapping; `XXDL960` two-year deletion window.
2. **Lifecycle preservation.** The pending → active → billed → accrued → archived deal lifecycle, and the cost-control gates around it, are not negotiable.
3. **Divisional model preservation.** The 32-division `<DIV>.MSTR.{SIM,VND,CAD}` model is a load-bearing business abstraction; flattening it without a defined divisional dimension would lose downstream report semantics.
4. **File-contract preservation.** Any file consumed by downstream systems (extracts written by `MCDL656J`, deal sales updates by `XXDM713`, etc.) must remain bit-compatible until those consumers are themselves migrated.
5. **Risk-aware cutover.** The size of the program graph (312 programs; top program `MCBSM04` has 36 dependencies) implies a **per-business-process** cutover, not a big-bang. The BP specs in `03-technical/specs/` are the cutover units.

## 6. Success Criteria for the Assess Phase

| # | Criterion | Evidence |
|---|-----------|----------|
| 1 | All 312 graph programs are classified into a business process and linked to a `BP-*.md`. | HLD §6 program inventory complete; zero `[CODMOD]` rows |
| 2 | All 32 divisional master families are documented with their member dataset families. | HLD §5 division catalogue |
| 3 | The top 10 DB2 tables and top 20 connected programs are explained at row-purpose / call-purpose level. | `_assess-research-notes.md` + HLD §5/§6 |
| 4 | Each business process has business rules, data structures, sequence diagrams, and test cases. | `BP-*.md` files |
| 5 | CPE-Legacy CRUD reflects all merged docs and `yourbuddy-hybrid` answers stay coherent. | Live re-query against opportunity |

## 7. Out of Scope

- The retailer's e-commerce stack.
- Store-side POS / replenishment systems (they are consumers, not part of the assessed platform).
- Building a new front-end for buyers (the CICS surface is preserved or replaced one screen at a time in the Plan/Design phase).

## 8. Constraints

- **Language constraint:** The platform is COBOL today. Whether the target stays COBOL or moves to a modern language is a Plan-phase decision; the assessment must remain language-agnostic on rules so they can be implemented in any host.
- **Risk constraint:** Multi-divisional financial data → no destructive cutovers. Every cutover unit must be reversible.
- **Operational constraint:** Downstream file contracts and DB2 read patterns of external consumers cannot change inside this engagement.

## 9. Open Questions for SMEs

- `[SME]` What does each two-letter division code stand for (MZ, SW, SZ, …)? Required to make HLD §5 human-readable.
- `[SME]` What is the batch schedule and who owns it? No scheduler (CA-7 / Control-M / OPC) is identifiable from the corpus.
- `[SME]` Which downstream systems consume the deal-sales updates from `XXDM713` and the extract files from `MCDL656J`? They become explicit cutover dependencies.
- `[SME]` Are `AUTHORKAY` / `AUTHORMCLANE` part of a broader pattern (per-vendor handlers) or one-offs? Determines whether `AUTHOR*` deserves its own prefix in the naming model.
- `[SME]` What is the stated modernization target stack? The corpus does not surface one.

---

**Document Control**
- Author: Assess Agent (yourbuddy-hybrid + RAG + Spanner Graph) + Human SME (pending)
- Reviewer: Merchandising IT owner, Deal Management product owner, DBA
- Approval gate: This PRD must be approved before SRS sign-off.
