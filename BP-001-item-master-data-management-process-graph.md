# BP-001 — Item Master Data Management: Business-Process Graph

**Status:** Draft — structured-BPMN process graph derived from the extracted business logic.
**Conforms to:** [process-graph-meta-model.md](../../../../../reference/process-graph-meta-model.md) (the canonical meta-model: element vocabulary, purity axioms, rule→element taxonomy, and the FSM-projection definition).
**Companion to:** [BP-001 business logic](BP-001-item-master-data-management-business-logic.md), [BP-001 call graph](BP-001-item-master-data-management-call-graph.md), and [BP-001 overview](../BP-001-item-master-data-management.md).
**Scope:** The two BP-001 processes — Process A (Cost Out-of-Sync, `MCCAD65J`) and Process B (Deal-Analysis End-of-Period, `MCDL656J`) — re-expressed as pure BPMN processes with a derived finite-state-machine projection.

---

## 1. Meta-model (reference)

This graph is a faithful re-projection of the companion business-logic rules (`BL-001-NN`) into the **structured BPMN** meta-model defined in [process-graph-meta-model.md](../../../../../reference/process-graph-meta-model.md). That reference is authoritative; only a working summary is repeated here.

- **Governing axiom (separation).** Work and routing are disjoint: Activities never branch, Gateways never do work, and a branch condition lives on the sequence flow leaving a gateway.
- **Execution semantics.** The token (Petri-net) game of §2 of the meta-model; the model is sound and block-structured, which is what makes the §-FSM projection well-defined.
- **Element legend (Mermaid rendering, per meta-model §3.4):**

| Element | Shape | Meaning |
|---|---|---|
| Event | `(( ))` circle | Start / End / Intermediate milestone; Error End is red |
| Task | `[ ]` rectangle | a unit of work (`Script` / `Service` / `Business Rule` / `Send`) |
| Gateway | `{ }` rhombus | routing only (`XOR` / `IOR` / `AND`); condition on outgoing flows |
| Sub-Process | `[[ ]]` | iteration (`Loop` / `Multi-Instance`), e.g. control breaks and merges |
| Data | `[( )]` cylinder | Data Object (transient) / Data Store (persistent); carries a typed id (`DSN:`/`DB2:`/`REC:`/`WS:`) or `[derived]`/`[sink]` per meta-model §3.3.1 |
| External participant | `{{ }}` hexagon | an external system/integration endpoint (`MQ:`/`API:`/`HOOK:`/`MAIL:`/`RPT:`/`FT:`), reached only by Message Flow (dotted `msg ▷ out / ◁ in`), per meta-model §3.5 |

- **Coverage rule.** Every `BL-001-NN` maps to **exactly one** flow node (see the conformance tables in §2.7 and §3.4).
- **FSM.** Each process carries a derived Mealy projection (§2.8, §3.5) per meta-model §7.

A single self-contained Mermaid source for Process A is also kept in the `diagrams/` subfolder at `diagrams/BP-001-A-process-graph.mmd`.

---

## 2. Process A — Cost Out-of-Sync (`MCCAD65J`)

Three stages: the divisional cost extract (`XXCAD63`, `BL-001-01..11`) and the corporate cost-basis extract (`XXCAD64`, `BL-001-12..16`) build independent per-item cost pictures; the comparison stage (`XXCAD65`, `BL-001-17..23`) merges them and gates report distribution. The two extracts share no data and are modelled as a concurrent `AND` block (the legacy sequential job-step order is an implementation choice).

### 2.1 Orchestration

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef sub fill:#d7ccc8,stroke:#3e2723,color:#000;

  st(("None Start<br/>MCCAD65J invoked")):::ev
  asplit{"AND split"}:::gw
  s1[["Sub-Process · XXCAD63<br/>Build divisional cost extract"]]:::sub
  s2[["Sub-Process · XXCAD64<br/>Build corporate cost-basis extract"]]:::sub
  ajoin{"AND join"}:::gw
  s3[["Sub-Process · XXCAD65<br/>Compare extracts and distribute report"]]:::sub
  en(("None End<br/>job complete")):::ev

  st --> asplit
  asplit --> s1 --> ajoin
  asplit --> s2 --> ajoin
  ajoin --> s3 --> en
```

### 2.2 Stage 1 — Divisional cost extract (`XXCAD63`, BL-001-01..11)

Reader-switch validation (01) runs once at entry; the per-item logic is a Multi-Instance sub-process whose instance boundary *is* the control break (02). Per-row classification is a Loop sub-process expanded in §2.3.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef err fill:#ffcdd2,stroke:#b71c1c,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;
  classDef sub fill:#d7ccc8,stroke:#3e2723,color:#000;

  start(("Start · XXCAD63<br/>read reader card MCCAD631")):::ev
  g01{"XOR · BL-001-01<br/>reader switch valid?<br/>(exactly 1 card AND value in Y,N)"}:::gw

  subgraph MI["MI Sub-Process — per division then per item   (control break · BL-001-02)"]
    direction TB
    is(("Start item instance")):::ev
    rl[["Loop Sub-Process<br/>Classify each cost row (see §2.3)"]]:::sub
    t07["Script Task · BL-001-07<br/>resolve cost picture at flush"]:::task
    rdI["Service Task<br/>read item master"]:::task
    g09{"XOR · BL-001-09<br/>item found AND active?"}:::gw
    rdV["Service Task<br/>read owning vendor"]:::task
    g10{"XOR · BL-001-10<br/>vendor active AND cost-controlled?"}:::gw
    wr["Service Task<br/>write divisional extract record"]:::task
    idone{"XOR join · item processed"}:::gw

    is --> rl --> t07 --> rdI --> g09
    g09 -- "no (skip)" --> idone
    g09 -- "yes" --> rdV --> g10
    g10 -- "no (skip)" --> idone
    g10 -- "yes" --> wr --> idone
    idone -- "next item" --> is
  end

  dcs[("Data Store · REC:DCS-RCD<br/>Divisional cost extract")]:::data
  ferr((("Error End · BL-001-11<br/>RC16 hard-fail"))):::err
  done(("None End · divisional extract complete")):::ev

  start --> g01
  g01 -- "invalid (abandoned, no output)" --> done
  g01 -- "valid" --> MI
  idone -- "EOF" --> done
  wr -. "writes" .-> dcs
  rdI -. "access error" .-> ferr
  rdV -. "access error" .-> ferr
```

### 2.3 Stage 1 — row-classification loop body (BL-001-03/04/05/06/08)

A classification rule that branches *and* seeds is a Gateway (the test) whose true branch carries its transform Tasks; the sub-transforms (04, 05, 08) keep their own nodes. BL-001-08 appears at both date seeds.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;

  rs(("Start row")):::ev
  g03{"XOR · BL-001-03<br/>current cost row?<br/>(orderLast ≥ today AND listOrder ≤ today ≤ orderLast)"}:::gw
  t08a["Script · BL-001-08<br/>Julian→ISO (current eff date)"]:::task
  t04["Script · BL-001-04<br/>map list amount type → basis"]:::task
  t05["Script · BL-001-05<br/>track cost sameness"]:::task
  j03{"XOR join"}:::gw
  g06{"XOR · BL-001-06<br/>future cost row?<br/>(NOT futureFound AND listOrder > today)"}:::gw
  t08b["Script · BL-001-08<br/>Julian→ISO (future eff date)"]:::task
  j06{"XOR join"}:::gw
  re(("End row → next / EOF")):::ev

  rs --> g03
  g03 -- "yes (seed current)" --> t08a --> t04 --> t05 --> j03
  g03 -- "no" --> j03
  j03 --> g06
  g06 -- "yes (seed future)" --> t08b --> j06
  g06 -- "no" --> j06
  j06 --> re
```

### 2.4 Stage 2 — Corporate cost-basis extract (`XXCAD64`, BL-001-12..16)

Control break (14) is the MI instance boundary (per catalog item). Eligibility (12) filters rows; selection (13), mapping (15), defaulting (16) are Tasks.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef err fill:#ffcdd2,stroke:#b71c1c,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;

  s(("Start · XXCAD64")):::ev

  subgraph MIc["MI Sub-Process — per division then per catalog item   (control break · BL-001-14)"]
    direction TB
    cs(("Start catalog item")):::ev
    g12{"XOR · BL-001-12<br/>eligible row?<br/>(partition AND ACT/DIS AND cost-controlled AND ITMCST/BASCOST AND not deleted)"}:::gw
    t15["Script · BL-001-15<br/>map cost-basis code → canonical"]:::task
    t13["Script · BL-001-13<br/>choose current/future cost"]:::task
    jr{"XOR join"}:::gw
    t16["Script · BL-001-16<br/>default missing current/future"]:::task
    wr2["Service Task<br/>write corporate extract record"]:::task
    cdone{"XOR join · catalog processed"}:::gw

    cs --> g12
    g12 -- "no (skip row)" --> jr
    g12 -- "yes" --> t15 --> t13 --> jr
    jr -- "next row" --> g12
    jr -- "rows done" --> t16 --> wr2 --> cdone
    cdone -- "next catalog" --> cs
  end

  de9e[("Data Store · REC:DE9E-RCD<br/>Corporate cost-basis extract")]:::data
  ferr2((("Error End · BL-001-11<br/>RC16 hard-fail"))):::err
  done2(("None End · corporate extract complete")):::ev

  s --> MIc
  cdone -- "EOF" --> done2
  wr2 -. "writes" .-> de9e
  t13 -. "access error" .-> ferr2
```

### 2.5 Stage 3 — Comparison and report orchestration (`XXCAD65`, BL-001-17..23)

Exception-set load (17) at entry; merge as a Loop sub-process (18, expanded in §2.6); report gate (23) with a single purge on both branches (23 purges regardless).

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef err fill:#ffcdd2,stroke:#b71c1c,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;
  classDef sub fill:#d7ccc8,stroke:#3e2723,color:#000;
  classDef ext fill:#ffe0b2,stroke:#e65100,color:#000;

  s3(("Start · XXCAD65")):::ev
  t17["Service Task · BL-001-17<br/>load OOS exception-item set"]:::task
  exc[("Data Object · WS:WS-EXC-ITEM-SW + MCCAD651 switches<br/>exception set + 6 comparison switches")]:::data
  merge[["Loop Sub-Process · BL-001-18<br/>match/merge both extracts on division+item (see §2.6)"]]:::sub
  g23{"XOR · BL-001-23<br/>report non-empty? (rows ≥ 1)"}:::gw
  rpt[("Data Store · DSN: report dataset [SME]<br/>Cost Discrepancy report")]:::data
  dist["Send Task · BL-001-23<br/>distribute Cost Discrepancy report"]:::task
  pc{{"RPT:PageCenter<br/>print / distribution"}}:::ext
  ml{{"MAIL:cost-oos-distribution<br/>email"}}:::ext
  jrep{"XOR join"}:::gw
  purge["Service Task<br/>purge temporary datasets"]:::task
  e3(("None End · job complete")):::ev
  ferr3((("Error End · BL-001-11<br/>RC16 hard-fail"))):::err

  s3 --> t17 --> merge --> g23
  t17 -. "writes / read by 19,20,21" .-> exc
  g23 -- "yes" --> dist --> jrep
  g23 -- "no (suppressed)" --> jrep
  jrep --> purge --> e3
  rpt -. "reads" .-> dist
  dist -. "msg ▷ out" .-> pc
  dist -. "msg ▷ out" .-> ml
  t17 -. "access error" .-> ferr3
```

The report distribution is an external integration (§3.5 of the meta-model): `RPT:PageCenter` and `MAIL:cost-oos-distribution` are external participants reached by Message Flow (out, fire-and-forget, at-least-once); the report dataset itself remains data-at-rest. Confirm the report DD name `[SME]`.

### 2.6 Stage 3 — merge loop body (BL-001-18/19/20/21/22)

`g18a` is a 3-way exclusive gateway (`=`, `>`, `<` — mutually exclusive and exhaustive). The exception soft-line task (21) is shared by orphan and discrepancy paths.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;

  rd(("Start pair<br/>read div row + corp row")):::ev
  g18a{"XOR · BL-001-18<br/>compare keys"}:::gw
  g18b{"XOR · BL-001-18<br/>data identical?"}:::gw
  sync(("in sync — no line")):::ev
  t20["Business Rule Task · BL-001-20<br/>compare enabled fields → mismatch set"]:::task
  g20e{"XOR · BL-001-20<br/>exception item AND report switch?"}:::gw
  g20m{"XOR · BL-001-20<br/>any enabled field mismatch?"}:::gw
  t20w["Task<br/>write field-discrepancy line(s)"]:::task
  t22["Service Task · BL-001-22<br/>enrich discrepant item"]:::task
  t22w["Service Task<br/>write discrepancy data record"]:::task
  outds[("Data Store · DSN:XXCAD65.OUT (REC:OUT-RCD)<br/>discrepancy data extract")]:::data
  g19c{"XOR · BL-001-19<br/>corp orphan: exception AND report switch?"}:::gw
  g19d{"XOR · BL-001-19<br/>div orphan: exception AND report switch?"}:::gw
  t19c["Task · BL-001-19<br/>write 'ITEM NOT FOUND AT DCS'"]:::task
  t19d["Task · BL-001-19<br/>write 'ITEM NOT FOUND AT CBR'"]:::task
  t21["Task · BL-001-21<br/>write 'ITEM EXCEPTION - NO UPDATES'"]:::task
  jm{"XOR join · next pair / EOF"}:::gw

  rd --> g18a
  g18a -- "keys equal" --> g18b
  g18b -- "identical" --> sync --> jm
  g18b -- "differ" --> t20 --> g20e
  g20e -- "yes" --> t21
  g20e -- "no" --> g20m
  g20m -- "no enabled mismatch" --> jm
  g20m -- "yes" --> t20w --> t22 --> t22w --> jm
  t22w -. "writes" .-> outds
  g18a -- "div.key > corp.key (corp orphan)" --> g19c
  g18a -- "div.key < corp.key (div orphan)" --> g19d
  g19c -- "yes" --> t21
  g19c -- "no" --> t19c --> jm
  g19d -- "yes" --> t21
  g19d -- "no" --> t19d --> jm
  t21 --> jm
```

The hard-fail convention (BL-001-11) wraps every Service Task here per §4 / meta-model §6.4.

### 2.7 Process A — rule → element conformance

| BL-001 | Title | Logic type | BPMN element | Where |
|---|---|---|---|---|
| 01 | Validate reader switch | validation | XOR Gateway (+ None End "abandoned") | §2.2 |
| 02 | Group cost rows by item | control break | MI Sub-Process boundary (per item) | §2.2 |
| 03 | Recognise current cost row | classification | XOR Gateway (test) + seed branch | §2.3 |
| 04 | Map amount type → basis | transformation | Script Task | §2.3 |
| 05 | Track cost sameness | classification | Script Task | §2.3 |
| 06 | Recognise future cost row | classification | XOR Gateway (test) + seed branch | §2.3 |
| 07 | Resolve cost picture at flush | transformation | Script Task | §2.2 |
| 08 | Julian → calendar | transformation | Script Task (2 placements) | §2.3 |
| 09 | Require active item | validation | XOR Gateway | §2.2 |
| 10 | Require active cost-controlled vendor | validation | XOR Gateway | §2.2 |
| 11 | Hard-fail on file errors | error handling | Error Boundary → Error End | §4 / all stages |
| 12 | Select eligible cost-basis rows | selection | XOR Gateway (filter) | §2.4 |
| 13 | Choose current/future cost | selection | Script Task | §2.4 |
| 14 | Assemble record per item | control break | MI Sub-Process boundary (per catalog) | §2.4 |
| 15 | Map cost-basis code → canonical | transformation | Script Task | §2.4 |
| 16 | Default missing current/future | transformation | Script Task | §2.4 |
| 17 | Load OOS exception set | data load | Service Task | §2.5 |
| 18 | Match/merge extracts | matching/merge | Loop Sub-Process + 2 XOR Gateways | §2.6 |
| 19 | Report orphan item | reporting | 2 XOR Gateways + 2 Tasks | §2.6 |
| 20 | Compare enabled fields | validation | Business Rule Task + 2 XOR Gateways | §2.6 |
| 21 | Emit exception soft line | control | Task (shared) | §2.6 |
| 22 | Enrich discrepant item | enrichment | Service Task | §2.6 |
| 23 | Gate report distribution | control | XOR Gateway + Send Task → `RPT:PageCenter` / `MAIL:` (Message Flow) + purge | §2.5 |

### 2.8 Process A — derived FSM projection (Mealy)

**Orchestration** — anchors {`START`, `EXTRACTS_READY`, `MERGE_COMPLETE`, `END`}:

```
START          --[valid switch]  / { run XXCAD63 ∥ XXCAD64 } --> EXTRACTS_READY
START          --[invalid switch]/ { }                        --> END   (abandoned, no output)
EXTRACTS_READY --[true]          / { 17, merge-loop }         --> MERGE_COMPLETE
MERGE_COMPLETE --[rows ≥ 1]      / { 23:distribute, purge }   --> END
MERGE_COMPLETE --[rows = 0]      / { purge }                  --> END
```
`∥` is the AND region; expand to the Petri reachability product (meta-model §7.1) for interleavings.

**Per merge pair** — states {`PAIR_READ`, `PAIR_DONE`} (`PAIR_DONE` loops to `PAIR_READ` until EOF → `MERGE_COMPLETE`):

```
PAIR_READ --[keysEq ∧ identical]                                  / { }                       --> PAIR_DONE
PAIR_READ --[keysEq ∧ ¬identical ∧ exc ∧ sw]                      / {20, 21}                  --> PAIR_DONE
PAIR_READ --[keysEq ∧ ¬identical ∧ ¬(exc∧sw) ∧ ¬mismatch]         / {20}                      --> PAIR_DONE
PAIR_READ --[keysEq ∧ ¬identical ∧ ¬(exc∧sw) ∧ mismatch]          / {20, fieldLine, 22}       --> PAIR_DONE
PAIR_READ --[¬keysEq ∧ exc ∧ sw]                                  / {21}                      --> PAIR_DONE
PAIR_READ --[¬keysEq ∧ ¬(exc∧sw)]                                 / {19}                      --> PAIR_DONE
```
Guards are mutually exclusive and exhaustive (deterministic Mealy); effects are the `BL-001` ids fired.

---

## 3. Process B — Deal-Analysis End-of-Period (`MCDL656J`)

Two stages: divisional enrichment (`XXDL656`, `BL-001-24..36`) filters and enriches settled deals and routes each into the period / year-to-date / deal-to-date streams; corporate aggregation/reporting (`MCDL656J` + `XXDL658/660/662`, `BL-001-37..39`) sums across divisions and prints loss reports.

### 3.1 Orchestration

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef sub fill:#d7ccc8,stroke:#3e2723,color:#000;

  bst(("Timer Start<br/>end-of-period trigger")):::ev
  bmi[["MI Sub-Process · per division<br/>XXDL656 — divisional deal enrichment & routing"]]:::sub
  bcorp[["Sub-Process<br/>MCDL656J — corporate aggregation & reporting"]]:::sub
  bend(("None End<br/>job complete")):::ev

  bst --> bmi --> bcorp --> bend
```

### 3.2 Stage 1 — Divisional deal enrichment (`XXDL656`, BL-001-24..36)

Initialization (24, 25) once per division; per-deal-row filters (26/27/28), gain/loss (29), enrichment (31/32/33), and routing (34, expanded in §3.3). The control break is the per-row loop boundary.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef err fill:#ffcdd2,stroke:#b71c1c,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;
  classDef sub fill:#d7ccc8,stroke:#3e2723,color:#000;

  s(("Start · XXDL656 (per division)")):::ev
  t24["Script Task · BL-001-24<br/>resolve reporting period/year"]:::task
  t25["Service Task · BL-001-25<br/>build vendor short-name table"]:::task
  vt[("Data Object · WS:VT00<br/>vendor name table")]:::data

  subgraph ROWS["Loop — for each deal row"]
    direction TB
    rs(("Start deal row")):::ev
    g26{"XOR · BL-001-26<br/>settled deal? (any amount > 0)"}:::gw
    g27{"XOR · BL-001-27<br/>future year? (periodYear > reportingYear)"}:::gw
    g28{"XOR · BL-001-28<br/>current open period? (periodYear=CY AND period=RP)"}:::gw
    t29["Script Task · BL-001-29<br/>compute deal gain/loss"]:::task
    g31{"XOR · BL-001-31<br/>group number ≠ 0?"}:::gw
    t31["Service Task · BL-001-31<br/>attach group description"]:::task
    j31{"XOR join"}:::gw
    t32["Business Rule Task · BL-001-32<br/>item-attribute lookup"]:::task
    g32{"XOR · BL-001-32<br/>catalog in attribute table?"}:::gw
    tinv["Task<br/>mark INVALID ITEM (CATLG NUM NOT IN TABLE)"]:::task
    t33["Service Task · BL-001-33<br/>enrich buyer + buyer name"]:::task
    route[["IOR fan-out · BL-001-34<br/>route to streams (see §3.3)"]]:::sub
    rdone{"XOR join · deal row done"}:::gw
  end

  done656(("None End · all deal rows processed")):::ev
  t36["Service Task · BL-001-36<br/>write per-division deal summary"]:::task
  sumds[("Data Store · DSN:DL6560<br/>divisional summary")]:::data
  berr((("Error End · BL-001-35<br/>RC16 hard-fail"))):::err

  s --> t24 --> t25 --> rs
  t25 -. "writes" .-> vt
  rs --> g26
  g26 -- "no (no monetary activity)" --> rdone
  g26 -- "yes" --> g27
  g27 -- "yes (future year)" --> rdone
  g27 -- "no" --> g28
  g28 -- "yes (open period)" --> rdone
  g28 -- "no" --> t29 --> g31
  g31 -- "yes" --> t31 --> j31
  g31 -- "no (blank group)" --> j31
  j31 --> t32 --> g32
  g32 -- "no" --> tinv --> route
  g32 -- "yes" --> t33 --> route
  route --> rdone
  rdone -- "next row" --> rs
  rdone -- "EOF" --> done656
  done656 --> t36 --> sumds
  t25 -. "access error" .-> berr
```

### 3.3 Stage 1 — routing fan-out (BL-001-34, with BL-001-30)

The three stream tests are independent — a deal can land in 1–3 streams — so they are one Inclusive (`IOR`) split/join, not a serial chain. Period-total accumulation (30) is the data effect of the period write.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;

  rdy(("Deal ready to route")):::ev
  ior{"IOR split · BL-001-34"}:::gw
  tdtd["Service Task<br/>write deal-to-date → DL6563"]:::task
  tper["Service Task<br/>write period stream → DL6561"]:::task
  t30["Script Task · BL-001-30<br/>accumulate period totals"]:::task
  tytd["Service Task<br/>write year-to-date → DL6562"]:::task
  iorj{"IOR join"}:::gw
  routed(("Deal routed")):::ev
  streams[("Data Store · DSN:DL6561/DL6562/DL6563 (REC:XXDL670C)<br/>period / YTD / deal-to-date streams")]:::data

  rdy --> ior
  ior -- "¬(prd=RP ∧ yr=CY)" --> tdtd --> iorj
  ior -- "prd=RP ∧ yr=RY" --> tper --> t30 --> iorj
  ior -- "yr=RY ∧ (RP=1 ∨ prd<RP)" --> tytd --> iorj
  iorj --> routed
  tdtd -. "writes" .-> streams
  tper -. "writes" .-> streams
  tytd -. "writes" .-> streams
```

### 3.4 Stage 2 — Corporate aggregation and reporting (`MCDL656J`, BL-001-37..39)

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart TD
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef data fill:#e1bee7,stroke:#4a148c,color:#000;
  classDef ext fill:#ffe0b2,stroke:#e65100,color:#000;

  cs(("Start · MCDL656J (corporate)")):::ev
  t37["Service Task · BL-001-37<br/>aggregate divisional streams"]:::task
  agg[("Data Store · DSN:ACME.PERM.MCDL6561..6566<br/>corporate vendor / buyer-vendor summaries")]:::data

  subgraph VEND["Loop — per aggregated vendor"]
    direction TB
    vs(("Start vendor")):::ev
    g38{"XOR · BL-001-38<br/>loss ≤ -threshold? (RDRAMT 050000-)"}:::gw
    tprint["Task<br/>print vendor report line"]:::task
    vomit(("vendor omitted (within tolerance)")):::ev
    vj{"XOR join · next vendor / EOF"}:::gw

    vs --> g38
    g38 -- "yes (material loss)" --> tprint --> vj
    g38 -- "no" --> vomit --> vj
    vj -- "next vendor" --> vs
  end

  t39["Send Task · BL-001-39<br/>distribute deal-analysis reports"]:::task
  infopac{{"RPT:InfoPac<br/>report distribution"}}:::ext
  purge["Service Task<br/>purge staged datasets"]:::task
  cend(("None End · corporate reporting complete")):::ev

  cs --> t37 --> vs
  t37 -. "writes" .-> agg
  vj -- "EOF" --> t39 --> purge --> cend
  t39 -. "msg ▷ out" .-> infopac
```

Per meta-model §3.5, `RPT:InfoPac` is an external participant reached by Message Flow (out, fire-and-forget); the six formatted report streams remain data-at-rest upstream of the Send Task.

### 3.5 Process B — rule → element conformance

| BL-001 | Title | Logic type | BPMN element | Where |
|---|---|---|---|---|
| 24 | Resolve reporting period/year | transformation | Script Task | §3.2 |
| 25 | Build vendor short-name table | data load | Service Task | §3.2 |
| 26 | Select settled deal rows | selection | XOR Gateway (filter) | §3.2 |
| 27 | Skip future-year rows | filter | XOR Gateway | §3.2 |
| 28 | Skip current open period | filter | XOR Gateway | §3.2 |
| 29 | Compute deal gain/loss | calculation | Script Task | §3.2 |
| 30 | Accumulate period totals | aggregation | Script Task (period branch) | §3.3 |
| 31 | Enrich with group description | enrichment | XOR Gateway + Service Task | §3.2 |
| 32 | Enrich item attributes | enrichment | Business Rule Task + XOR Gateway | §3.2 |
| 33 | Enrich buyer + buyer name | enrichment | Service Task | §3.2 |
| 34 | Route into period/YTD/deal-to-date | routing | IOR split/join + 3 Service Tasks | §3.3 |
| 35 | Hard-fail on file/SQL errors | error handling | Error Boundary → Error End | §4 / all stages |
| 36 | Per-division deal summary | reporting | Service Task | §3.2 |
| 37 | Aggregate divisional streams | aggregation | Service Task | §3.4 |
| 38 | Report vendors exceeding loss threshold | filter | XOR Gateway | §3.4 |
| 39 | Distribute reports to InfoPac | control | Send Task → `RPT:InfoPac` (Message Flow) | §3.4 |

### 3.6 Process B — derived FSM projection (Mealy)

**Orchestration** — anchors {`START`, `DIVISIONS_DONE`, `END`}:

```
START          --[timer] / { 24, 25, per-division enrich loop } --> DIVISIONS_DONE
DIVISIONS_DONE --[true]  / { 37, per-vendor threshold filter, 39, purge } --> END
```

**Per deal row** — states {`ROW_READ`, `ROW_DONE`} (`ROW_DONE` loops to `ROW_READ` until EOF → division summary 36):

```
ROW_READ --[¬settled]                              / {26}                                   --> ROW_DONE  (skipped)
ROW_READ --[settled ∧ futureYear]                  / {26, 27}                               --> ROW_DONE  (skipped)
ROW_READ --[settled ∧ ¬futureYear ∧ openPeriod]    / {26, 27, 28}                           --> ROW_DONE  (skipped)
ROW_READ --[settled ∧ ¬futureYear ∧ ¬openPeriod]   / {29, 31, 32, (33 | INVALID), 34:set, [30 if period]} --> ROW_DONE
```
The `34:set` effect is the subset of {deal-to-date, period(+30), year-to-date} whose guards held (big-step per meta-model §7.2); `(33 | INVALID)` is the enrichment outcome depending on the item-found gateway.

---

## 4. Cross-cutting rule — operational hard-fail (`BL-001-11`, `BL-001-35`)

Every data-access Task in both processes is guarded by the reusable hard-fail boundary (meta-model §6.4): `NOT-FOUND` (status 23/10) is an ordinary soft-skip XOR branch; any other unexpected status raises an Error caught by an Error Boundary Event → `Error End · RC16`, which the job-level guard turns into a pipeline stop. It is modelled once rather than wired into each access.

```mermaid
%%{init: {'theme': 'base'}}%%
flowchart LR
  classDef ev fill:#c8e6c9,stroke:#1b5e20,color:#000;
  classDef task fill:#bbdefb,stroke:#0d47a1,color:#000;
  classDef gw fill:#fff9c4,stroke:#f9a825,color:#000;
  classDef err fill:#ffcdd2,stroke:#b71c1c,color:#000;

  acc["Service Task<br/>read / write data access"]:::task
  xnf{"XOR · BL-001-11/35<br/>status = NOT-FOUND? (23/10)"}:::gw
  skip["Task<br/>skip / blank attribute"]:::task
  use(("continue main flow")):::ev
  eb(("Error Boundary<br/>status ∉ {00,97,04,23,10}")):::err
  ferr((("Error End · BL-001-11/35<br/>RC16 — instance terminates"))):::err

  acc --> xnf
  xnf -- "yes (soft skip)" --> skip
  xnf -- "no (status OK)" --> use
  acc -. "error" .-> eb --> ferr
```

---

## 5. Conformance and traceability

This graph conforms to [process-graph-meta-model.md](../../../../../reference/process-graph-meta-model.md):

- **Total coverage.** All 39 rules map to exactly one flow node — §2.7 (`BL-001-01..23`) and §3.5 (`BL-001-24..39`). Rules previously absent or smeared onto edges in the prior draft are now first-class nodes, notably **BL-001-17** (exception-set load, §2.5) and the transform/enrichment rules **04, 05, 07, 08, 13, 15, 16, 22, 24, 25, 29, 30, 33, 36, 37** (Tasks).
- **Purity.** Each diagram is block-structured (matched split/join gateways), loops are confined to Loop/MI sub-processes, exceptions exit via Error End, and XOR guards are mutually exclusive and total — so each process is sound (meta-model P1–P9 ⇒ P6).
- **Concurrency.** Process A's two extracts are an `AND` block; Process B's stream routing is an `IOR` fan-out (the one place the prior serial chain misrepresented the logic).
- **External integrations.** Report distribution is modelled as external participants reached by Message Flow (meta-model §3.5): Process A's `RPT:PageCenter` + `MAIL:cost-oos-distribution` (§2.5) and Process B's `RPT:InfoPac` (§3.4); report datasets remain data-at-rest.
- **FSM.** Derived Mealy projections are in §2.8 and §3.6; the exact reachability-graph FSM is obtainable per meta-model §7.1.
- **Source artifacts.** Standalone Mermaid sources are in the `diagrams/` subfolder at `diagrams/BP-001-A-process-graph.mmd` and `diagrams/BP-001-B-process-graph.mmd`.
