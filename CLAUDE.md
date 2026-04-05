# SAP Payment Method Format Migration — Domain Knowledge

This file contains SAP FI/AP domain knowledge required to reason about phased, safe migration of payment formats in a production SAP environment.

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

| Table | Content |
|---|---|
| `T042` | Company code payment control |
| `T042A` | Payment methods per company code |
| `T042B` | House bank details (account determination) |
| `T042C` | Payment methods per country |
| `T042D` | Payment method supplements |
| `T042E` | Bank determination / ranking |
| `T042I` | Payment method classification |
| `T042Z` | Payment methods in company code (format assignment) |
| `REGUH` | Payment run header data (generated during F110) |
| `REGUP` | Payment run item data (generated during F110) |
| `LFB1` | Vendor master company code segment (payment method, supplement) |

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

1. Read `REGUH` header for Payment Method + Supplement
2. Look up format in `T042Z` (company code level), falling back to `T042C` (country level)
3. If supplement is populated and has a separate format entry in `T042D`, use that
4. Call PMW or classic program accordingly

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
- Same Payment Method code, no character exhaustion
- Granular vendor-level control
- Fully reversible (remove supplement from vendor master → falls back to base format)
- No impact on F110 selection parameters or authorization

**Disadvantages:**
- Requires vendor master data changes (can be mass-updated via `LSMW` / `XK02` / BDC)
- Supplement field may already be in use for other purposes

### 3.2 House Bank / Account ID Override

- In `FBZP` → Bank Determination, different house bank + account ID combinations can be configured with different formats.
- By creating a new account ID under the same house bank with the new format, and adjusting ranking/priority in bank determination, you can route specific payment runs through the new format.
- Less granular than supplements (bank-level, not vendor-level).

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

### 3.4 Parallel Payment Runs with Separate Variants

- Run two `F110` runs for the same date:
  - Run 1: Old format, excludes pilot vendors (via additional selection)
  - Run 2: New format (temporary Payment Method or supplement), includes only pilot vendors
- Operationally complex but provides complete separation.

### 3.5 Temporary Duplicate Payment Method (Limited Use)

- Create a new Payment Method (e.g., `X`) as a copy of the original (e.g., `T`) but with the new format.
- Assign `X` to pilot vendors in master data.
- After full migration, switch all vendors back to `T` (now with new format) and decommission `X`.
- Consumes a character code but only temporarily.
- Works well if you have enough free characters and few company codes.

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
- **BAdI approach:** Deactivate BAdI implementation or set pilot list to empty → instant rollback.
- **Duplicate PM approach:** Switch vendors back to original Payment Method code.
- **In-place swap:** No easy rollback — this is why it should be the last step, not the first.

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
| `OBPM` | Payment method supplements |
| `OBPM1` | PMW format assignment |
| `OBPM4` | PMW selection variants |
| `DMEE` | DME format tree editor |
| `FDTA` | DME file administration / download |
| `XK02` / `FK02` | Vendor master change (supplement field) |
| `SM30` → `T042Z` | Direct table maintenance for format assignment |
| `SE18` / `SE19` | BAdI definition / implementation |
| `FPY1`-`FPY3` | Payment Medium Workbench monitor |

---

## 6. Glossary

| Term | Meaning |
|---|---|
| **Payment Method** | Single-char code (e.g., `T`, `U`, `C`) representing a type of payment (transfer, check, etc.) |
| **Payment Method Supplement** | 2-char extension allowing sub-variants of a Payment Method |
| **PMW** | Payment Medium Workbench — modern framework for payment file generation |
| **DME** | Data Medium Exchange — the output file sent to the bank |
| **Format Tree** | PMW structure defining the XML/flat file layout |
| **House Bank** | Bank through which the company makes payments (configured with bank key + account ID) |
| **F110** | Transaction for automatic payment program (proposal → payment → printout/medium) |
| **REGUH / REGUP** | Runtime tables holding payment header/item data during F110 execution |
