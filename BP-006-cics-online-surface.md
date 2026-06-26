# BP-006 — CICS Online Surface

**Status:** Draft — surface confirmed in corpus, internals require `[CODMOD]`
**Owners:** Buyers / Merchants (users), Merchandising IT (operators)
**Anchor artefacts:** CICS transaction `DLZB` (started by `dlzbstrt` JCL), screen `MCCSM55`, plus the `D*` utility family invoked from online code
**Related SRS:** FR-ONLINE-001, FR-ONLINE-002, FR-ONLINE-003

---

## 1. Overview

The platform's **CICS online surface** has grown larger than the first draft of this spec assumed. Round-2 retrieval surfaced **at least three transaction families**:

- **`DLZB`** — initiated by JCL `dlzbstrt`. Naming is consistent with the deals (`DL*`) family; full screen flow needs `[CODMOD]`.
- **Cost-message processors `MCS4` (program `MCCST50`) and `MCS5` (program `MCCST51`)** — MQ-driven CICS transactions that consume cost-update queue messages. **These are documented in BP-004 §2.2** because they are cost-functional, not generic online utilities.
- **D8050** was previously documented here as a utility. Round-2 retrieval shows it is actually the **CICS deal-capture polling transaction** for the pending → active lifecycle bridge. **D8050 has moved to BP-002** (deal lifecycle). It is removed from this BP.
- **The `D21**` CICS COBOL family (41 programs)** — CAST (round 3) reveals that the `D21**` series (`D2105`…`D2199`) are **CICS transactional COBOL programs**, not utilities. They form the bulk of the platform's online application code. Programs in this family were previously mis-marked `[CORPUS-EMPTY]` (`D2122`) or `[RAG]` (`D2112`).

Screen `MCCSM55` is explicitly named in the corpus as one of the online screens. Online code uses three CICS primitives consistently: `EXEC CICS RETRIEVE`, `EXEC CICS XCTL` (transfer control), and `EXEC CICS ENQ` (serialise access).

This BP now covers:
1. The `DLZB` transaction and `MCCSM55` screen (deal/back-office maintenance).
2. The **shared utility modules** invoked by online code (`D0007` extended error messages, `D0325` data dictionary, `D0703` print copy).
3. Cross-cutting concurrency / serialisation behaviour shared across CICS transactions in this platform (used here, in BP-002 by D8050, and in BP-004 by MCCST50/51).

---

## 2. Programs / Artefacts Involved

### 2.1 Owned by this BP

| Type | Name | CAST count / metric | Purpose |
|---|---|---|---|
| CICS transaction | `DLZB` | — | Online transaction (deals-family naming). |
| JCL | `DLZBSTRT` | 1 job | Starts the `DLZB` transaction. |
| **CICS BMS Maps (screens)** | (56 total per CAST) | **56** | `MCCSM55` is one. Full enumeration `[CAST]` — extract from `MCL_Added,_Modified_or_Deleted_Objects` with `ObjectType='Unknown CICS Map'`. |
| **CICS Transactional COBOL — `D21**` family** | `D2105`, `D2109`, `D2110`, `D2111`, **`D2112`**, `D2113`, `D2115`, `D2116`, `D2118`, `D2119`, `D2121`, **`D2122`**, `D2123`, `D2126`, `D2127`, `D2128`, `D2131`, `D2135`, `D2136`, **`D2138`** (37 cross-refs — top-30 most-referenced), `D2139`, `D2141`, `D2142`, `D2143`, `D2145`, `D2146`, `D2147`, `D2149`, `D2157`, `D2161`, `D2162`, `D2171`, `D2173`, `D2174`, `D2175`, `D2178`, `D2184`, `D2187`, `D2188`, `D2198`, `D2199` | **41 programs** | Deal-domain online application code. Per-program behaviour `[RAG]` (the corpus has files for each). |
| Utility | `D0007` | — | Extended error message display. |
| Utility | `D0325` | — | Data dictionary lookups. |
| Utility | `D0703` | — | Print copy routine. |
| Utility | `D0173P`, `D0402P`, `D0402R` | top-30 most-referenced | `D*` proc/record copybooks shared across the platform. |
| Utility — CICS DataSet | (5 CICS DataSets) | 5 | `[CAST]`/`[CODMOD]` to enumerate. |
| Utility — CICS Transient Data | (3 TD queues) | 3 | `[CAST]`/`[CODMOD]` to enumerate. |
| Utility — Cobol Transaction | (4 CICS-tagged Cobol transactions) | 4 | `[CAST]` distinct artefact type from "Cobol Transactional Program"; enumeration `[CODMOD]`. |
| **DCS\* copybook family** | `DCSWORK` (62 calls), `DCSHVNDP` (44), `DCSFERR` (41), `DCSCOMX` (41), `DCSMCA` (41), `DCSERRX` (40), `DCSLINK` (40), `DCSFWHS` (38), `DCSFOPR` (38), `DCSKVNDP` (35), `DCSHITMP` (34), plus more | 12+ members in top-30 most-referenced | Cross-cutting shared subsystem; used by both online and batch. Acronym expansion `[SME]`. |
| **DB2 error copybook** | `DBDB2ERL` | **241 calls (most-referenced object in entire platform)**; companions `DB2ERRW2` (103), `DB2ERRP2` (96), `DB2GDW1` (58), `DB2GDP1` (50) | Canonical DB2-error idiom used by every DB2-touching program. |
| **DCLGEN copybook family** | `DG*` — `DGDI1D`, `DGAP1P`, `DGAP1S`, `DGDE6C`, `DGDE1I`, `DGVN1A`, `DGDM1X`, plus 600+ more | 612 copybooks total platform-wide | DB2 column descriptors generated from the DB2 catalog, stored in `acme-code/DB2P.PERM.DCLGEN/`. |

### 2.2 Cross-references (owned by other BPs)

| Type | Name | Owning BP | Note |
|---|---|---|---|
| CICS transaction | **`D8050`** | **BP-002** | Reclassified: this is the deal-capture polling transaction (polls `DM3P`, inserts into `DM1T`). |
| CICS transaction | **`MCS4` (`MCCST50`)** | **BP-004** | Processes `'IC1X'` MQ cost messages. |
| CICS transaction | **`MCS5` (`MCCST51`)** | **BP-004** | Processes `'CADMF'` MQ cost messages. |
| CICS dispatcher | **`MCCST55`** | **BP-004** | Routes by `COMM-Q-TYPE` to `MCCST50` / `MCCST51`. |

### 2.3 Corpus-empty / unresolved utilities

| Name | Status |
|---|---|
| `D2122` | RAG corpus body is empty; **CAST confirms `D2122` is a CICS transactional COBOL program in the `D21**` family** — so we know *what kind of thing it is*, but not its specific business behaviour. Either re-run codmod on this program's source, or capture an SME description. |
| `D2112` | RAG body thin; **CAST confirms it is a CICS transactional COBOL program in the `D21**` family**. `[RAG]` deeper. |

> The `D*` prefix on these programs led to an earlier mis-assumption that they were utilities. They are not — they are part of the 41-program `D21**` family.

### 2.4 D21** family — Round 5 sharpening (CICS-only vs separate batch entries)

Round 5 CAST relations clarified the round-4 "mixed batch+CICS" wording. The actual picture:

- **45 CAST "Cobol Transactional Program" entries** (`D2105`, `D2109`, `D2110`, `D2111`, `D2112`, `D2113`, `D2115`, `D2116`, `D2118`, `D2119`, `D2121`, `D2122`, `D2123`, `D2126`, `D2127`, `D2128`, `D2131`, `D2135`, `D2136`, `D2138`, `D2139`, `D2141`, `D2142`, `D2143`, `D2145`, `D2146`, `D2147`, `D2149`, `D2157`, `D2161`, `D2162`, `D2171`, `D2173`, `D2174`, `D2175`, `D2178`, `D2184`, `D2187`, `D2188`, `D2198`, `D2199` + `D8050` + `MCCST50` + `MCCST51` + `MCCST55`) are **all genuinely CICS-only** per CAST relations — **zero** of them have a JCL Job as caller in `MCL_Relation_between_Transactions_And_Objects`.
- `D2137` is a **separate CAST artefact** classified `Cobol Batch Program`, source `D2137.cbl`, invoked by JCL job `XXD2137J`. It is **not** in the 45-program transactional list.
- `D2133` is similarly likely a separate batch entry — `[RAG]` to verify.
- So the D21NN number range is shared by **two distinct populations**: the CICS-only application family and a small set of same-numbered batch programs. **Each individual program is one or the other, not both.** The round-4 "mixed family" framing has been sharpened in this revision.

Known per-program classifications:

| Program | Runtime | Source |
|---|---|---|
| 41 D21** programs in the Transactional list (see §2.1) | **CICS-only** (no JCL caller per CAST) | round 5 (CAST relations) |
| `D8050` | **CICS** (deal-capture polling transaction → `DM1T`) | round 2, BP-002 |
| `MCCST50` / `MCCST51` / `MCCST55` | **CICS** (MQ cost transactions `MCS4`/`MCS5` + dispatcher) | round 2, BP-004 |
| `D2137` | **Batch** — separate `Cobol Batch Program` entry, invoked by `XXD2137J` | round 4 (`d2137.md`) + round 5 (CAST inventory dup) |
| `D2133` | **Not in CAST `Objects↔DataSources`** — round 4 prose suggested batch item-master update; round 6 RAG corpus-empty; graph shows calls `D2138B`/`D2139B` only | round 4–6 |

#### Round 6 — per-program behaviour (RAG + graph)

| Program | Business function (round 6) | Status |
|---|---|---|
| `D2138` | Shared subroutine hub — **11 callers** (`D2109`…`D2157`); no corpus prose | `[CORPUS-EMPTY]` + graph |
| `D2147` | Deal Control item/deal file subroutine — add/change/delete/inquire; `DCSHITMP`, `DCSHVNDP`, `DCSFCAD`, PO/calendar tables | documented |
| `D2149` | User Deal Information inquiry/maintenance — two-phase screens | documented |
| `D2162` | Truckload discount maintenance — `DCSHVNDP`, `DCSHITMP` | documented |
| `D2122` | PO header inquiry/update — `DCSHPOHP`, `DCSFVND`, `DCSFSHP`; calls `D2138` | documented |
| `D2135` | Item cost inquiry by vendor/date — `DCSHITMP`, `DCSHVNDP`, `DCSFCAD` | documented |
| `D2171` | Vendor Maintenance — multi-screen; `DCSFVND`; calls `D2201` | documented |
| `D2178` | Vendor Return Information — `DCSFVND`, `DCSFRTV` | documented |
| `D2142` | Removes cost from vendor file | documented |
| `D2128` | P.O. Add Module (inferred from `D2122`); calls `D2138` | `[CORPUS-EMPTY]` prose |
| `D2146`, `D2161`, `D2199`, `D2105`, `D2111` | — | `[CORPUS-EMPTY]` |

D21** programs that have DataSource references in CAST (21 of the 41 transactional + the D2137 batch):

| Program | # DataSource refs (CAST) |
|---|--:|
| D2137 (batch) | 9 |
| D2122 | 6 |
| D2112 | 4 |
| D2118 | 3 |
| D2128 | 3 |
| D2184 | 2 |
| D2135 | 2 |
| D2171 | 2 |
| D2119 | 2 |
| D2131 | 2 |
| D2142 | 2 |
| D2101 | 2 |
| D2178 | 1 |
| D2136 | 1 |
| D2174 | 1 |

The **other 20 transactional D21** programs have zero direct DataSource references** in CAST — they delegate all I/O to the DCS file handler copybooks (DCSHVNDP/DCSHITMP/etc.), and CAST attributes the data access to those handlers, not the calling program.

CICS-handling paragraphs observed in source: `D2146`, `D2147`, `D2149`, `D2157`, `D2161`, `D2162`, `D2199` all contain `ERRX-CICS-*` and `COMX-CICS-XCTL` paragraphs (confirming they're CICS). `D2147` has `SUPPRESS-ENTRY-MAP1`, `PROTECT-MAP1-ENTRY`, `PROTECT-MAP2-ENTRY` — confirming it drives at least 2 BMS maps. (Full BMS map enumeration still `[CODMOD]`.)

### 2.5 Marwood Distribution System cross-cutting layer (round 4)

DCS copybook semantics confirmed (per source headers "MARWOOD CONTROL AREA" in `DCSMCA` and "MARWOOD SOURCE BOOK DELIMITER PROGRAM" in `DCSWORK`):

| Copybook | Role |
|---|---|
| `DCSWORK` | System work fields — common flags, return codes, CICS attention key names, **COMX-prefixed communication calling codes** (`COMX-LINK`, `COMX-XCTL`, `COMX-RETURN`, `COMX-CODE`, `COMX-PROGRAM`). |
| `DCSMCA` | **Marwood Control Area** — system data, sequencing fields, program names, comm/security info; passed program-to-program in CICS. |
| `DCSHVNDP` / `DCSKVNDP` | Host / Key Vendor Parameters — vendor master file access; key conversion (vendor number in/out). |
| `DCSHITMP` | Host Item Parameters — item master file access (key in/out, read item records). |
| `DCSFWHS` | File WareHouSe — warehouse file layout / handler parameters. |
| `DCSFOPR` | File OPeRator — operator/buyer info file. |
| `DCSFERR` | File ERRor — error message handling. |
| `DCST26` | File Set Table (additional Marwood copybook observed in `d2116.md`). |
| `DCSCOMX` / `DCSLINK` | Not found as standalone copybooks; `COMX-` and `LINK` semantics live inside `DCSWORK` / `DCSMCA`. |

**Implication for modernization (HS-017):** The DCS layer is **Marwood framework code, not bespoke customer code**. ~80% of the cross-cutting CICS/file plumbing this BP relies on is replaceable wholesale if the modernization target abandons Marwood. The customer-unique behaviour lives in the `D21**` business programs and the BMS map flows, not in DCS.

---

## 3. Business Rules

### 3.1 Transaction / screen behaviour

| ID | Rule | Source |
|---|---|---|
| BR-006-01 | Transaction `DLZB` is started from the JCL `dlzbstrt`. | corpus |
| BR-006-02 | Online code uses `EXEC CICS RETRIEVE` to recover input data passed by the start. | corpus |
| BR-006-03 | Online code uses `EXEC CICS XCTL` to transfer control between modules. | corpus |
| BR-006-04 | Online code uses `EXEC CICS ENQ` to serialise access to shared resources. | corpus |
| BR-006-05 | `[CODMOD]` Enumerate the named resources held under `EXEC CICS ENQ` (these become the modernized system's exclusivity contract). |  |
| BR-006-06 | `MCCSM55` screen's field-level validations are uniform across all entry points. | `[CODMOD]` confirm |

### 3.2 Utility module contracts

| ID | Rule | Source |
|---|---|---|
| BR-006-10 | `D0007` is the single source of extended error-message text. | corpus |
| BR-006-11 | `D0325` is the single source of data-dictionary lookups. | corpus |
| BR-006-12 | `D0703` is the standard print-copy routine. | corpus |
| BR-006-13 | `D2112` is a common utility cited across many programs; semantics MUST be preserved or replaced by a named modernized equivalent. Detailed behaviour `[RAG]`. | corpus |
| BR-006-14 | `D2122` has **no corpus content** (`[CORPUS-EMPTY]`); preservation contract is unknown. Block this on either a re-ingest of cleaner source markdown or an SME description before approving modernization. | corpus state |

### 3.3 Online concurrency / integrity

| ID | Rule | Source |
|---|---|---|
| BR-006-20 | Online updates against entities also written by batch (e.g. deal capture into `ACME.PENDINGDEALSDM3P`) MUST not conflict with concurrent batch writers. The `EXEC CICS ENQ` resources are the legacy mechanism. | derived |
| BR-006-21 | DB2 access is via the same `SYSPROC.DSCON03` stored procedure used by batch. | corpus |

---

## 4. Data Structures

The online surface acts on the same DB2 entities documented in BP-001 / BP-002 / BP-003 / BP-004. Notable on-screen-bound entities:

- `ACME.PENDINGDEALSDM3P` — pending deals captured online before they flow into BP-002's lifecycle.
- Vendor and item masters when buyers maintain deals tied to specific (vendor, item).

`[CODMOD]` provide the field map for `MCCSM55` (and any other identified screens).

---

## 5. Sequence Diagrams

### 5.1 Transaction start

```
Buyer terminal ──► CICS region ──► dlzbstrt (JCL) starts DLZB
                                      │
                                      ▼
                              Initial online module (CICS RETRIEVE)
                                      │
                                      ▼
                              Screen MCCSM55 displayed
```

### 5.2 Buyer action with ENQ

```
Buyer ──► MCCSM55 ──► CICS module
                          │
                          ├── EXEC CICS ENQ on (resource)   [BR-006-04]
                          ├── DB2 update via SYSPROC.DSCON03 [BR-006-21]
                          ├── EXEC CICS DEQ on (resource)
                          └── EXEC CICS XCTL to next module  [BR-006-03]
```

### 5.3 Error display

```
CICS module ── error code ──► D0007 (extended error)
                                 │
                                 ▼
                         Formatted message back to MCCSM55
```

---

## 6. Test Cases

| ID | Scenario | Expected | Maps to |
|---|---|---|---|
| TC-006-01 | DLZB starts from dlzbstrt | Transaction active in CICS region; first screen displayed | BR-006-01/02 |
| TC-006-02 | XCTL between modules preserves state | Common area passed correctly | BR-006-03 |
| TC-006-03 | Two concurrent buyers update the same record | One blocks on ENQ, succeeds after the other releases | BR-006-04 |
| TC-006-04 | Batch writer touches an ENQ'd resource | Batch waits or is rolled back per legacy behaviour | BR-006-20 |
| TC-006-05 | MCCSM55 invalid field | Validation error surfaced via `D0007` text | BR-006-06, BR-006-10 |
| TC-006-06 | Data dictionary lookup | `D0325` returns expected entry | BR-006-11 |
| TC-006-07 | Print copy | `D0703` formats per legacy baseline | BR-006-12 |
| TC-006-08 | DB2 unavailable | Online surface returns the documented user-facing error | BR-006-21 |

---

## 7. Open Questions

- `[CODMOD]` Enumerate the full set of CICS transactions and screens. Known so far: `DLZB` (this BP), `D8050` (BP-002), `MCS4` (BP-004), `MCS5` (BP-004). Others likely exist.
- `[CODMOD]` Enumerate every named resource under `EXEC CICS ENQ` and the programs that ENQ them — this is the platform-wide serialisation contract.
- `[RAG]` Deep-dive `D2112` (top-edge utility).
- `[CORPUS-EMPTY]` `D2122` — either re-run codmod on the source or capture an SME description, then re-ingest.
- `[SME]` Number of concurrent online users at peak; user-population split between buyers/merchants (DLZB) and corporate/AP users (cost-message-driven).
- `[SME]` Authentication / authorisation model (RACF groups → screen access).
- `[SME]` Decision: modernize the online surface in place (CICS preserved) or migrate to a modern web UI? Affects every modernization sequencing decision for this BP.

---

## 8. Modernization Notes

- **Blast radius:** Modest in code volume but **broad in user impact**. Online surface is the buyer/merchant's daily tool.
- **Preservation:**
  - ENQ semantics (BR-006-04..20) are the platform's concurrency contract; the modernized stack must encode equivalent semantics (DB row locks, distributed locks, etc.) before any cutover.
  - Screen field validations must remain identical (BR-006-06) — a regression here is immediately user-visible.
  - The `D*` utility family is shared with batch; modernizing them is a cross-BP change and should be sequenced after BP-002 and BP-004 cutovers, when usage patterns are best understood.
- **Suggested cutover unit:** Per-screen modernization with a thin compatibility shim on the back-end so the online tier can keep calling `SYSPROC.DSCON03` semantics during transition.
