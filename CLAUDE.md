# SAP Payment Method Format Migration — Domain Knowledge

This file contains SAP FI/AP domain knowledge required to reason about phased, safe migration of payment formats in a production SAP environment.

---

## 0. ISO 20022 Migration Context

### 0.1 Key Deadlines

| Deadline | Requirement |
|---|---|
| November 2025 | SWIFT ceases MT format support (MT940, MT942, MT100); migration to ISO 20022 camt/pain messages required |
| November 15, 2026 | SEPA mandate: unstructured postal addresses no longer accepted; pain.001 version 2019 (pain.001.001.09) required |
| Ongoing | LEI (Legal Entity Identifier) and UETR (Unique End-to-End Transaction Reference) adoption expanding |

### 0.2 Format Version Mapping (Legacy → ISO 20022 2019)

| Message Type | Old Version | New Version (2019) |
|---|---|---|
| SEPA Credit Transfer | pain.001.001.03 | pain.001.001.09 |
| Payment Status Report | pain.002.001.03 | pain.002.001.10 |
| SEPA Direct Debit | pain.008.001.02 | pain.008.001.08 |
| Account Statements | camt.053.001.02 | camt.053.001.08 |

### 0.3 Key ISO 20022 (2019 Version) Enhancements

- **UETR** (Unique End-to-End Transaction Reference) for end-to-end payment tracking
- **LEI** (Legal Entity Identifier) for enhanced transparency
- **Structured Remittance Information** enabling automated posting
- **Structured/Hybrid Addresses** mandatory; unstructured formats deprecated after November 2026
- **Supplementary Data Elements** for additional information capture

### 0.4 Master Data Implications

The new format has stricter data requirements than legacy formats:
- **IBAN** is mandatory (not bank account number alone)
- **Structured addresses** require separate fields: street (`LFA1-STRAS`), city (`LFA1-ORT01`), postal code (`LFA1-PSTLZ`), country (`LFA1-LAND1`) — single-line address formats are rejected
- **BIC** may be required for cross-border payments
- Vendors with incomplete master data will cause PMW format generation errors

---

## 1. Payment Method Architecture in SAP

### 1.1 Configuration Transaction: FBZP

`FBZP` (Automatic Payment Transactions configuration) is the central transaction for payment method setup. It has four sections:

| Section | Scope | Key Fields |
|---|---|---|
| **All Company Codes** | Global defaults | Sending/paying company code, tolerance days |
| **Paying Company Codes** | Per company code | Minimum amounts, forms, sender details |
| **Payment Methods in Country** | Per country | Payment method character, classification (outgoing/incoming), required master data fields, payment medium programs/formats |
| **Payment Methods in Company Code** | Per company code + payment method | Amount limits, foreign currency settings, **format program/PMW format**, house bank assignment |
| **Bank Determination** | Per company code + payment method | Ranking of house banks, account IDs, available amounts |

### 1.2 Payment Method Code Constraint

- A Payment Method is a **single alphanumeric character** (A-Z, 0-9, and some special chars).
- This means a maximum of ~36-40 usable codes per country/company code.
- With 10 already in use per company code, headroom is limited but not zero.

### 1.3 Payment Method Supplements (Zahlwegergänzung)

- Configured in `OBPM` or directly in `FBZP` → Payment Methods in Company Code.
- A supplement is a **2-character extension** to the base Payment Method code (e.g., Payment Method `T` with supplement `01`, `02`, etc.).
- Supplements allow **different formats, forms, or print programs** for the same base Payment Method.
- The supplement is assigned at the **vendor/customer master** level (field `UZAWE` in `LFB1`/`KNB1`) or can be derived via BAdI logic.
- This is one of the most powerful levers for phased migration: same Payment Method code, different format per supplement.

### 1.4 Key Tables

| Table | Content | Key Fields |
|---|---|---|
| `T042` | Company code payment control | |
| `T042A` | Payment methods per company code | Amount limits, house bank |
| `T042B` | House bank details (account determination) | |
| `T042C` | Payment methods per country | `XZAWE` (supplement allowed indicator) |
| `T042D` | Payment method supplements | Supplement code, format assignment |
| `T042E` | Bank determination / ranking | House bank, account ID, ranking |
| `T042F` | Payment method supplements (check table) | |
| `T042I` | Payment method classification | |
| `T042Z` | Payment methods in company code (format assignment) | PMW format key |
| `TFPM042FZ` | PMW format supplements | `FORMI` (format), `FORMZ` (supplement) |
| `REGUH` | Payment run header data (generated during F110) | `RZAWE` (PM), `UZAWE` (supplement), amounts |
| `REGUP` | Payment run item data (generated during F110) | Line item details |
| `LFB1` | Vendor master company code segment | `ZWELS` (payment method), `UZAWE` (supplement) |
| `LFBK` | Vendor master — bank details | IBAN, BIC, bank key |
| `LFA1` | Vendor master — general data (address) | `STRAS`, `ORT01`, `PSTLZ`, `LAND1` |
| `CDPOS` / `CDHDR` | Change documents | Audit trail for `LFB1-UZAWE` changes (object class `KRED`) |

---

## 2. Format Assignment — Where and How

### 2.1 Classic Payment Programs (RFFO*)

Historically, SAP shipped country-specific ABAP payment programs (e.g., `RFFOUS_T` for US ACH, `RFFOD__T` for DE domestic). These are assigned in:

- `FBZP` → Payment Methods in **Country** → "Payment medium program" field
- `FBZP` → Payment Methods in **Company Code** → can override with company-specific program

### 2.2 Payment Medium Workbench (PMW)

Modern approach using **format tree** concept:

- Configured via `OBPM1` (format assignment) or in `FBZP` → Payment Methods in Company Code → "Payment Medium Format" field.
- PMW formats are identified by a **format key** (e.g., `PMW_CGI_XML`, `SEPA_CT`, custom Z-variants).
- PMW uses **Selection Variants** (transaction `OBPM4`) to control format-specific parameters.
- The format can be assigned at multiple levels with the following **resolution priority**:
  1. Payment Method Supplement level (highest priority)
  2. Payment Method in Company Code level
  3. Payment Method in Country level (fallback)

### 2.3 Format Resolution at Runtime (F110 → SAPFPAYM)

During a payment run (`F110`), the payment program `SAPFPAYM` resolves the format:

1. Read `REGUH` header for Payment Method (`REGUH-RZAWE`) + Supplement (`REGUH-UZAWE`)
2. Look up format in `T042Z` (company code level), falling back to `T042C` (country level)
3. If supplement is populated and has a separate format entry in `T042D` / `TFPM042FZ`, use that (highest priority)
4. Call PMW or classic program accordingly

**Detailed call chain:**

```
F110 (Payment Run)
  │
  ├─ Proposal Run (SAPFPAYM_SCHEDULE → SAPFPAYM)
  │   ├─ Reads open items per vendor
  │   ├─ Determines Payment Method from vendor master (LFB1-ZWELS)
  │   ├─ Reads Payment Method Supplement from vendor master (LFB1-UZAWE)
  │   ├─ Writes REGUH header: RZAWE = Payment Method, UZAWE = Supplement
  │   └─ Writes REGUP items: line items grouped under REGUH
  │
  ├─ Payment Run (executes payments, creates documents)
  │
  └─ Payment Medium Generation (SAPFPAYM → PMW / Classic Program)
      │
      ├─ For each REGUH entry:
      │   ├─ Read REGUH-RZAWE (e.g., "T") and REGUH-UZAWE (e.g., "01" or blank)
      │   │
      │   ├─ IF UZAWE is populated (e.g., "01"):
      │   │   ├─ Look up T042D for Payment Method T + Supplement 01
      │   │   ├─ Look up TFPM042FZ for supplement-specific PMW format
      │   │   └─ Call PMW with supplement-assigned format tree + selection variant
      │   │
      │   └─ IF UZAWE is blank:
      │       ├─ Look up T042Z for Payment Method T in company code
      │       └─ Call classic program or base PMW format
      │
      └─ Generate DME file(s) — separate files per format
```

**Key insight:** Within a single F110 run, vendors with a supplement produce DME output via the supplement's format, while vendors without a supplement produce DME output via the base format. Two separate DME files are generated — one per format. Both are available in `FDTA` for review and bank transmission.

This layered resolution is the key enabler for phased migration.

---

## 3. Migration Levers — Standard SAP Mechanisms

### 3.1 Payment Method Supplements (Primary Lever)

**How it works for migration:**

1. Keep existing Payment Method (e.g., `T` for bank transfer) with old format as default.
2. Create a supplement (e.g., `01`) and assign the **new format** to it.
3. Assign supplement `01` to a small pilot group of vendors in their master data (`LFB1-UZAWE`).
4. Run `F110` — vendors with supplement `01` get the new format; all others stay on the old format.
5. Gradually expand the vendor population with supplement `01`.
6. When 100% migrated, swap the base Payment Method format to the new one and remove the supplement.

**Advantages:**
- Same Payment Method code, no character exhaustion (up to 100 supplement values 00-99 per PM)
- Granular vendor-level control
- Fully reversible (remove supplement from vendor master → falls back to base format)
- No impact on F110 selection parameters or authorization
- Standard SAP functionality — no custom code; supported since R/3 4.6C
- Auditable — `UZAWE` changes logged in change documents (`CDPOS`/`CDHDR`, object class `KRED`)

**Disadvantages:**
- Requires vendor master data changes (can be mass-updated via `LSMW` / `XK02` / BDC)
- Supplement field (`LFB1-UZAWE`) may already be in use for other purposes — always audit via `SE16` → `LFB1` filter `UZAWE <> ''` before starting
- In S/4HANA, the `UZAWE` field may not be visible in BP transaction without configuration (see SAP Note 3107250)

**Mass update methods for vendor assignment:**
- **LSMW** with BDC recording of `XK02`: create project → record transaction → map vendor number + company code + supplement value from CSV → run batch session
- **Direct table update** via `SE16N` or custom ABAP: emergency only — does not create change documents
- In S/4HANA: use `BP` transaction or BAPI `BUPA_SUPPLIER_CHANGE`

### 3.2 House Bank / Account ID Override

- In `FBZP` → Bank Determination, different house bank + account ID combinations can be configured with different formats.
- By creating a new account ID under the same house bank with the new format, and adjusting ranking/priority in bank determination, you can route specific payment runs through the new format.
- **Not recommended as primary migration mechanism** because:
  - No vendor-level granularity — bank determination operates at PM + Company Code level, not per vendor
  - Cannot selectively route pilot vendors within a single F110 run
  - Manipulating ranking logic is fragile, especially with amount-based rules
  - Reverting requires careful re-ranking
- Useful as a supplementary mechanism when the new format also requires a different bank account.

### 3.3 BAdI-Based Dynamic Format Selection

| BAdI | Purpose |
|---|---|
| `FI_PAYMEDIUM` | Controls payment medium generation; can dynamically switch format based on custom logic (vendor group, date, amount, etc.) |
| `SAMPLE_PROCESS_00001820` | PMW process BAdI for format-specific processing |
| `FI_BL_PAYMENT` | Bank-level payment processing |
| `BADI_PAYMETHOD_SELECTION` | Influences payment method selection during proposal run |

**Custom BAdI approach:**
1. Implement `FI_PAYMEDIUM` (or relevant BAdI) with logic: "If vendor in pilot list → use new format; else → use old format."
2. Maintain the pilot list via a custom Z-table or a vendor master field.
3. No master data changes needed if using a Z-table.
4. Full programmatic control and logging.

**Risk:** Custom code in payment processing requires thorough testing and carries maintenance cost.

**Evaluation for format migration:**
- **Not recommended as the sole format-routing mechanism** — custom code in `SAPFPAYM` critical path creates risk of payment file corruption, duplicate payments, or run failures. Must be regression-tested with every SAP support package.
- **Recommended as a secondary read-only logging tool:** implement a lightweight `FI_PAYMEDIUM` BAdI that only logs which format was selected per vendor per run (vendor, PM, format used, date, run ID) without altering payment behavior. Actual format routing is handled by the supplement mechanism.
- Auditors and bank operations teams may require additional sign-off for custom code in the payment chain.

### 3.4 Parallel Payment Runs with Separate Variants

- Run two `F110` runs for the same date:
  - Run 1: Old format, excludes pilot vendors (via additional selection)
  - Run 2: New format (temporary Payment Method or supplement), includes only pilot vendors
- Operationally complex but provides complete separation.
- **Not recommended as primary mechanism:**
  - Doubles operational burden (scheduling, bank file transmission, reconciliation, cash management)
  - Changes F110 user workflows
  - Risk of duplicate payments if vendor selection between runs is not perfectly exclusive
  - Cash position fragmentation across two bank files
- Useful for initial smoke-testing in QAS before going live with supplements in PRD.

### 3.5 Temporary Duplicate Payment Method (Limited Use)

- Create a new Payment Method (e.g., `X`) as a copy of the original (e.g., `T`) but with the new format.
- Assign `X` to pilot vendors in master data.
- After full migration, switch all vendors back to `T` (now with new format) and decommission `X`.
- Consumes a character code but only temporarily.
- **Not recommended when migrating many Payment Methods:**
  - 10 PMs require 10 extra characters (20 total) — character exhaustion risk
  - Every FBZP setting, bank determination, house bank, form, authorization must be duplicated per temporary PM, per company code
  - Vendors must be changed twice (to temp PM, then back to original) — double mass-update effort and error risk
  - F110 selection variants may need adjustment to include new PM codes — changes user workflows
- Only viable when supplements are unavailable AND the number of PMs to migrate is very small (1-2) AND sufficient free characters exist.

### 3.6 Other Mechanisms Evaluated

**PMW Selection Variant Switching (`OBPM4`):**
Selection variants control format-specific parameters (grouping, house bank criteria) but do not provide vendor-level format routing. Not suitable as a primary migration lever.

**Business Transaction Event 00001820:**
Can influence payment medium processing but is designed for correspondence customization, not format-level switching. Not recommended.

**S/4HANA Output Determination / Condition-Based Format:**
Output management can theoretically route different formats based on conditions, but this is not mature for payment medium formats (as of S/4HANA 2023) and adds unnecessary complexity.

---

## 4. Risk Mitigation Patterns

### 4.1 F110 Test Run vs. Productive Run

- Every `F110` run has a **proposal run** (test) and a **payment run** (productive).
- Always execute the proposal first, review the payment list, check format output in spool, and only then execute the payment run.
- Use `F110` → Environment → Payment Medium → DME Administration to preview generated files.

### 4.2 DME File Validation

- After payment medium generation, inspect the DME (Data Medium Exchange) file via `DMEE` or transaction `FDTA`.
- Compare old-format and new-format outputs side by side for the same vendor/payment.
- Validate against bank specifications before first productive run.

### 4.3 Reconciliation

- Compare `REGUH`/`REGUP` entries for old vs. new format runs.
- Verify totals, bank details, payment amounts, and vendor coverage.
- Use `FPY1`/`FPY2`/`FPY3` (Payment Medium Workbench monitor) for tracking.

### 4.4 Rollback Strategy

- **Supplements approach:** Remove supplement from vendor master → instant rollback to base format.
  - **Per-vendor:** `XK02` → clear `UZAWE` → save. Takes effect on next F110 run.
  - **Per-wave (mass):** LSMW with same vendor list, `UZAWE` = blank. Time: < 2 hours.
  - **Full rollback:** LSMW for ALL vendors with `UZAWE` = supplement value, set to blank. No FBZP changes needed — supplement config can remain dormant.
- **BAdI approach:** Deactivate BAdI implementation or set pilot list to empty → instant rollback.
- **Duplicate PM approach:** Switch vendors back to original Payment Method code.
- **In-place swap:** No easy rollback — requires re-transporting old FBZP configuration. This is why it should be the last step (Phase 4), not the first.

Rollback can be executed at any time, including mid-day between F110 runs.

### 4.5 Communication and Change Management

- Notify treasury/bank operations of format changes (banks may need to enable new format acceptance).
- Coordinate with business users running F110.
- Document old vs. new format mapping per Payment Method per company code.

---

## 5. Key Transactions Reference

| Transaction | Purpose |
|---|---|
| `FBZP` | Payment method configuration (all sections) |
| `F110` | Automatic payment program execution |
| `F111` | Automatic payment program — cross-company code (note: SAP KBA 3061862 — `UZAWE` may not update in `REGUH` after F111) |
| `OBPM` | Payment method supplements |
| `OBPM1` | PMW format assignment |
| `OBPM4` | PMW selection variants |
| `DMEE` | DME format tree editor |
| `FDTA` | DME file administration / download |
| `XK02` / `FK02` | Vendor master change (supplement field) |
| `BP` | Business Partner maintenance (S/4HANA vendor master — `UZAWE` visibility may require config per Note 3107250) |
| `LSMW` | Legacy System Migration Workbench — mass vendor master updates via BDC recording |
| `SM30` → `T042Z` | Direct table maintenance for format assignment |
| `SE16` / `SE16N` | Table data browser — verification of `LFB1`, `T042D`, etc. |
| `SE18` / `SE19` | BAdI definition / implementation |
| `FPY1`-`FPY3` | Payment Medium Workbench monitor |
| `SM37` | Background job monitoring (F110 job logs) |
| `ST05` | SQL trace — performance monitoring for F110 runtime |
| `SA38` | ABAP program execution (e.g., `RFKKIB00` bank data report) |
| `STMS` | Transport management system |
| `SU01` / `PFCG` | User / role maintenance — authorization review |

---

## 6. Relevant SAP Notes

| Note | Title / Purpose |
|---|---|
| 2816629 | ISO 20022 pain.001.001.09 / pain.002.001.10 format trees (2019 version) |
| 2665657 | ISO 20022 format notes and corrections |
| 1682833 | SEPA format corrections and updates |
| 1541877 | PMW format migration guidance |
| 986210 | Payment method supplements — functional documentation |
| 3107250 | `UZAWE` field visibility in BP transaction (S/4HANA) |
| 2175218 | Structured address handling in payment formats |
| 3061862 | `UZAWE` field not updated in `REGUH` after F111 — relevant for cross-company code payments |

> Always check the SAP Support Portal for the latest versions of these notes and any dependent/successor notes.

---

## 7. Authorization Objects

| Object | Description | Relevant For |
|---|---|---|
| `F_LFB1_BUK` | Vendor master — company code data | `UZAWE` field maintenance in vendor master |
| `F_REGU_BUK` | Payment run — company code authorization | F110 execution |
| `F_PMTFMT` | Payment medium format authorization | Execution of new PMW formats |

During migration, verify that the designated team has authorization for `LFB1-UZAWE` changes across all target company codes, and that F110 operators have authorization for the new PMW format.

---

## 8. Glossary

| Term | Meaning |
|---|---|
| **Payment Method** | Single-char code (e.g., `T`, `U`, `C`) representing a type of payment (transfer, check, etc.) |
| **Payment Method Supplement** | 2-char extension (`UZAWE`) allowing sub-variants of a Payment Method with different formats/forms |
| **PMW** | Payment Medium Workbench — modern framework for payment file generation |
| **DME** | Data Medium Exchange — the output file sent to the bank |
| **DMEEX** | Extended DME — newer format tree tool used with CGI-XML format trees |
| **Format Tree** | PMW structure defining the XML/flat file layout |
| **House Bank** | Bank through which the company makes payments (configured with bank key + account ID) |
| **F110** | Transaction for automatic payment program (proposal → payment → printout/medium) |
| **REGUH / REGUP** | Runtime tables holding payment header/item data during F110 execution |
| **ISO 20022** | International standard for financial messaging (XML-based); replaces SWIFT MT formats and older domestic formats |
| **pain.001** | ISO 20022 Customer Credit Transfer Initiation message (the payment file sent to the bank) |
| **pain.002** | ISO 20022 Payment Status Report (bank's response confirming/rejecting payments) |
| **camt.053** | ISO 20022 Bank-to-Customer Statement (account statements replacing MT940) |
| **UETR** | Unique End-to-End Transaction Reference — mandatory tracking ID in ISO 20022 2019 formats |
| **LEI** | Legal Entity Identifier — 20-char code identifying legal entities in financial transactions |
| **LSMW** | Legacy System Migration Workbench — tool for mass data updates via BDC recordings |
| **EBICS / H2H** | Bank communication protocols for DME file transmission (Electronic Banking Internet Communication Standard / Host-to-Host) |
| **XSD** | XML Schema Definition — used to validate ISO 20022 XML payment files against the official schema |
