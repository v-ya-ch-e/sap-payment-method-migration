# SAP Payment Method Format Migration — Phased Strategy

---

## Executive Summary

This document presents a production-safe, phased migration strategy for transitioning all Payment Methods across multiple company codes from legacy payment formats (classic RFFO* programs or older PMW format trees) to modern ISO 20022 formats (pain.001.001.09 / CGI XML / custom PMW format trees).

**Recommended primary approach: Payment Method Supplements (Zahlwegergänzung).**

The supplement mechanism (`UZAWE` field in `LFB1`, configuration in `T042D`) allows assigning a new payment format to a 2-character extension of the existing Payment Method code. This provides vendor-level granularity, requires no new Payment Method characters, preserves all existing FBZP configuration, and offers instant rollback by simply clearing the supplement from the vendor master. The entire migration can be executed within a single F110 payment run — old-format and new-format vendors coexist transparently.

**Estimated timeline:** 12-16 weeks for 10 Payment Methods across multiple company codes.
**Estimated effort:** 45-60 person-days for a team of 2 SAP FI consultants.

### Key ISO 20022 Deadlines

| Deadline | Requirement |
|---|---|
| November 2025 | SWIFT ceases MT format support (MT940, MT942, MT100); migration to camt/pain messages required |
| November 15, 2026 | SEPA mandate: unstructured postal addresses no longer accepted; pain.001 version 2019 (pain.001.001.09) required |
| Ongoing | LEI (Legal Entity Identifier) and UETR (Unique End-to-End Transaction Reference) adoption expanding |

These deadlines mean that organizations still on legacy formats face hard cutoff dates from banks and clearing houses. A phased migration approach is not merely desirable — it is the only responsible path given the compressed timeline and the severity of payment disruption if the new format fails.

---

## Part A — Recommended Approach

### A.1 Evaluation of Migration Mechanisms

#### A.1.1 Payment Method Supplements (Zahlwegergänzung)

**Verdict: RECOMMENDED — Primary mechanism.**

Payment Method Supplements extend a base Payment Method (single character, e.g., `T`) with a 2-character suffix (e.g., `01`). The supplement is stored in the vendor master record (`LFB1-UZAWE`) and read during F110 processing. When supplement-specific format configuration exists in `T042D` / `TFPM042FZ`, the PMW format assigned to the supplement overrides the base Payment Method format.

**Why it fits this scenario:**

1. **No character exhaustion.** The base Payment Method code remains unchanged. Supplements use a separate 2-character field with up to 100 combinations (00-99) per Payment Method, far exceeding migration needs.
2. **Vendor-level granularity.** Each vendor can be individually assigned to the new format (supplement `01`) or left on the old format (no supplement / blank `UZAWE`). This enables precise 5-vendor, 50-vendor, 500-vendor rollout waves.
3. **Standard SAP functionality.** No custom code. Configured entirely through `FBZP`, `OBPM`, and vendor master transactions. Supported by SAP since R/3 4.6C.
4. **Transparent to F110 users.** Payment run parameters, selection variants, and user workflows do not change. The same F110 run processes both old-format and new-format vendors — the format is resolved per `REGUH` line item based on the vendor's supplement.
5. **Instant rollback.** Clearing `LFB1-UZAWE` (setting it to blank) for any vendor immediately reverts that vendor to the base Payment Method format. No FBZP reconfiguration needed.
6. **Auditable.** The `UZAWE` field change is logged in the vendor master change history (table `CDPOS` / `CDHDR`, object class `KRED`). Combined with `REGUH` data, a complete audit trail exists showing which vendor used which format on which payment run date.

**Limitations to address:**

- Requires vendor master data changes (mitigated by LSMW / BDC mass update — see Section A.2.4).
- The `UZAWE` field may already be in use for other purposes (check current usage via `SE16` on `LFB1` before starting; see Risk D.4).

#### A.1.2 House Bank / Account ID Level Format Override

**Verdict: NOT RECOMMENDED as primary mechanism.**

This approach creates a parallel Account ID under the same House Bank, assigns the new format to it, and adjusts bank determination ranking in `FBZP` → Bank Determination (`T042E`).

**Why it does not fit:**

- **No vendor-level granularity.** Bank determination operates at the Payment Method + Company Code level, not per vendor. All vendors using the same Payment Method and company code are routed through the same house bank ranking. You cannot selectively send 10 pilot vendors through the new Account ID while keeping others on the old one within a single F110 run.
- **Complex ranking logic.** Manipulating bank determination ranking to partially route traffic is fragile and error-prone, especially with amount-based ranking rules.
- **Limited rollback.** Reverting bank determination changes requires careful re-ranking and risks payment routing errors.

**When it is useful:** As a supplementary mechanism when the new format also requires a different bank account (e.g., a new clearing bank), which is orthogonal to the format migration itself.

#### A.1.3 BAdI-Based Dynamic Format Switching

**Verdict: RECOMMENDED — Secondary / optional enhancement, not primary.**

Implementing BAdI `FI_PAYMEDIUM` (or `BADI_PAYMETHOD_SELECTION` for proposal-phase logic) allows runtime format selection based on arbitrary criteria: vendor number, vendor group, date range, amount threshold, or a custom Z-table containing the pilot vendor list.

**Why it is attractive but not primary:**

- **Maximum flexibility.** Any selection logic can be coded: date-based cutover, vendor-group-based waves, percentage-based random sampling for A/B testing.
- **No master data changes.** The pilot list can live in a Z-table, avoiding vendor master updates entirely.
- **Full logging.** Custom code can write detailed migration logs (vendor, payment method, format used, date, run ID).

**Why it should not be primary:**

- **Custom code in the payment critical path.** Any bug in BAdI logic can cause payment file corruption, duplicate payments, or payment run failures. The audit and testing burden for custom code in `SAPFPAYM` processing is significantly higher than for standard configuration.
- **Maintenance cost.** The BAdI implementation must be maintained, transported across environments, regression-tested with every SAP support package, and eventually decommissioned. For a temporary migration tool, this is disproportionate effort.
- **Not "standard SAP."** Auditors and bank operations teams may require additional sign-off for custom code in the payment chain.

**Recommended hybrid use:** Implement a lightweight `FI_PAYMEDIUM` BAdI that only logs which format was selected per vendor per run (read-only, no format-switching logic). This provides the audit trail without the risk of modifying payment behavior. The actual format routing is handled by the supplement mechanism.

#### A.1.4 Temporary Duplicate Payment Method

**Verdict: NOT RECOMMENDED for this scenario.**

Creating a temporary Payment Method (e.g., `X` as a copy of `T`) with the new format assigned, then moving pilot vendors to `X` in their master data.

**Why it does not fit:**

- **Character exhaustion.** With 10 Payment Methods already in use, duplicating all 10 requires 10 additional characters (20 total). If other characters are already taken by inactive or country-level Payment Methods, this may be impossible.
- **Configuration duplication.** Every FBZP setting, bank determination entry, house bank assignment, form assignment, authorization object value, and selection variant must be duplicated for each temporary Payment Method. For multiple company codes, this multiplies further.
- **Master data churn.** Vendors must be changed twice: once to the temporary Payment Method, and again back to the original (with new format) after migration. Double the mass-update effort, double the error risk.
- **F110 impact.** Selection variants and payment run parameters may need adjustment to include the new Payment Method codes. This changes user workflows.

**When it is useful:** Only when supplements are not available (e.g., the `UZAWE` field is locked by another process) AND the number of Payment Methods to migrate is very small (1-2), AND sufficient free characters exist.

#### A.1.5 Parallel F110 Runs

**Verdict: NOT RECOMMENDED as primary mechanism.**

Running two separate F110 payment runs per payment date — one for old-format vendors, one for new-format vendors — using different selection criteria or Payment Methods.

**Why it does not fit:**

- **Operational complexity.** Doubles the number of payment runs. Payment run scheduling, bank file transmission, reconciliation, and cash management all become more complex.
- **User workflow changes.** F110 operators must manage two runs instead of one, with vendor-level selection logic that must be maintained and updated as the pilot group expands.
- **Duplicate payment risk.** If vendor selection between the two runs is not perfectly exclusive, a vendor could be processed in both runs, resulting in duplicate payments.
- **Cash position fragmentation.** Two separate payment runs generate two separate bank files, potentially complicating cash forecasting and bank reconciliation.

**When it is useful:** For initial smoke-testing in QAS (run a parallel F110 with 2-3 vendors on the new format to validate the DME output before going live with supplements in PRD).

#### A.1.6 Other Mechanisms Considered

**PMW Selection Variant Switching (`OBPM4`):**
Selection variants control format-specific parameters (e.g., grouping, house bank criteria) but do not provide vendor-level format routing. Not suitable as the primary migration lever.

**Customer/Vendor-Specific Format via Correspondence (BTE 00001820):**
Business Transaction Event 00001820 can influence payment medium processing but is designed for correspondence customization, not format-level switching. Not recommended.

**Output Determination / Condition-Based Format (S/4HANA only):**
In S/4HANA, output management can theoretically route different formats based on conditions. However, this is not mature for payment medium formats as of S/4HANA 2023 and adds unnecessary complexity.

---

### A.2 Detailed Walkthrough — Payment Method Supplement Approach

#### A.2.1 FBZP Configuration Steps

The following configuration must be performed for each Payment Method (e.g., `T`) in each company code (e.g., `1000`). Repeat for all 10 Payment Methods across all company codes.

**Step 1: Define the Supplement in Payment Methods per Country**

1. Transaction: `FBZP`
2. Navigate to: **Payment Methods in Country** → select country (e.g., `DE`)
3. Select Payment Method (e.g., `T`)
4. Verify that the "Payment method supplement" checkbox / indicator is enabled for this Payment Method. If not, activate it.
5. Save and note the transport request.

> Table affected: `T042C` — field `XZAWE` (supplement allowed indicator).

**Step 2: Create the Supplement Configuration**

1. Transaction: `FBZP` → **Payment Methods in Company Code**
2. Select company code `1000`, Payment Method `T`
3. Navigate to the **Supplements** section (or use transaction `OBPM`)
4. Create a new supplement: `01`
5. In the supplement configuration, assign the **new PMW format**:
   - Field: "Payment Medium Format" → enter the new format key (e.g., `SEPA_CT`, `CGI_XML_V3`, or your custom `Z_ISO20022_CT`)
   - Field: "Format Supplement" (in `TFPM042FZ`) → if using a PMW format supplement, enter it here
6. Assign the corresponding **selection variant** (created in `OBPM4`) if the new format requires one
7. Optionally assign new **form** for payment advice / accompanying sheet if the format change requires a different form layout
8. Save. Transport to QAS and PRD.

> Tables affected: `T042D` (supplement definition), `TFPM042FZ` (format-to-supplement mapping), `T042Z` (company code format assignment for supplement level).

**Configuration example:**

```
FBZP → Payment Methods in Company Code
  Company Code:     1000
  Payment Method:   T (Bank Transfer)
  Supplement:       01
  PMW Format:       SEPA_CT  (new ISO 20022 pain.001.001.09 format)
  Selection Variant: SEPA_CT_1000_T01 (created in OBPM4)
  Form:             F110_SEPA_AVIS (if different from base)
```

The base Payment Method `T` (without supplement) retains the old format assignment:

```
FBZP → Payment Methods in Company Code
  Company Code:     1000
  Payment Method:   T (Bank Transfer)
  Supplement:       (blank / base)
  PMW Format:       RFFOD__T  (old classic program — unchanged)
```

**Step 3: Create PMW Selection Variant**

1. Transaction: `OBPM4`
2. Create a new selection variant for the new format (e.g., `SEPA_CT_1000_T01`)
3. Set format-specific parameters: house bank, payment grouping criteria, currency, structured address flags
4. Assign this variant to the supplement `01` in `FBZP` / `OBPM`

**Step 4: Validate Format Tree**

1. Transaction: `DMEE`
2. Open the target format tree (e.g., `SEPA_CT` or your custom tree)
3. Verify all required fields are mapped: IBAN, BIC, structured address blocks, remittance information, UETR
4. Verify XML schema version matches the bank's requirements (pain.001.001.09 for 2026 mandate)

#### A.2.2 Runtime Format Resolution Trace

When a payment run executes, the following logic determines which format is used for each payment line item:

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
      │   │   ├─ Found: SEPA_CT → use new format
      │   │   └─ Call PMW with format tree SEPA_CT, selection variant SEPA_CT_1000_T01
      │   │
      │   └─ IF UZAWE is blank:
      │       ├─ Look up T042Z for Payment Method T in company code 1000
      │       ├─ Found: RFFOD__T → use old format
      │       └─ Call classic program RFFOD__T (or old PMW format)
      │
      └─ Generate DME file(s) — separate files per format
```

**Key insight:** Within a single F110 run, vendors with supplement `01` produce DME output via the new format, while vendors without a supplement produce DME output via the old format. Two separate DME files are generated — one per format. Both are available in `FDTA` for review and bank transmission.

#### A.2.3 Vendor Assignment to the Pilot Group

**Individual vendor update:**

1. Transaction: `XK02` (or `FK02` for FI vendors, `MK02` for MM vendors; in S/4HANA use `BP`)
2. Select vendor (e.g., `100001`), company code (`1000`)
3. Navigate to: **Payment transactions** section
4. Field: `Payment method supplement` (`UZAWE`) → enter `01`
5. Save

> Table affected: `LFB1` — field `UZAWE`.

**Mass update via LSMW:**

1. Transaction: `LSMW`
2. Create a new project/subproject/object
3. Recording method: BDC recording of transaction `XK02`
4. Record the following steps:
   - Enter vendor number and company code
   - Navigate to payment transactions
   - Change field `UZAWE` to `01`
   - Save
5. Prepare input file (CSV/Excel) with columns: vendor number, company code, supplement value (`01`)
6. Map source fields to recording fields
7. Execute: Read data → Convert → Create batch input session → Run session

**Mass update via direct table update (emergency only):**

Transaction `SE16N` or custom ABAP program to update `LFB1-UZAWE` directly. Not recommended for production due to lack of change document creation. Use only if LSMW/BDC is not feasible and supplement the update with manual change document entries.

**Vendor selection criteria for pilot group:**

Select 5-10 vendors per Payment Method that meet ALL of the following:

- Low payment frequency (weekly or monthly, not daily) — limits blast radius if format fails
- Small payment amounts — reduces financial exposure
- Domestic payments only (initially) — avoids cross-border complexity
- Known-good master data: IBAN populated, structured address, bank details verified
- Cooperative relationship — vendor will confirm receipt and flag issues quickly
- Spread across different house banks (if multiple) — validates bank-side acceptance

#### A.2.4 Gradual Expansion (10% → 50% → 100%)

| Wave | Vendor Population | Method | Validation Period |
|---|---|---|---|
| Pilot (Phase 1) | 5-10 per Payment Method (50-100 total) | Manual `XK02` or small LSMW batch | 2-4 weeks |
| Expansion (Phase 2) | ~50% of vendors per Payment Method | LSMW batch run | 2-3 weeks |
| Full migration (Phase 3) | Remaining 50% | LSMW batch run | 1-2 weeks |
| Cleanup (Phase 4) | 0% on supplement (all on base) | In-place format swap on base PM, then clear all supplements | 1 week |

**Expansion procedure for each wave:**

1. Prepare vendor list (CSV) for the wave: vendor number, company code, supplement = `01`
2. Execute LSMW in QAS first → verify via `SE16` on `LFB1` that `UZAWE` is set correctly
3. Transport / re-execute LSMW in PRD
4. Execute next F110 run (proposal only first) → verify in proposal log that supplement vendors are identified
5. Check DME preview via `F110` → Environment → Payment Medium → DME Administration
6. If DME output is correct → execute payment run
7. Monitor bank confirmations for the new-format DME files
8. Wait for validation period to pass before next wave

#### A.2.5 Rollback Procedure at Each Phase

Rollback is the same at every phase and can be executed at three levels of granularity:

**Per-vendor rollback:**

1. Transaction: `XK02` → vendor → company code → Payment transactions
2. Clear field `UZAWE` (set to blank)
3. Save
4. Next F110 run: this vendor reverts to the base Payment Method format (old format)

**Per-wave rollback (mass):**

1. Prepare LSMW input file with the same vendor list used for the wave, but with `UZAWE` = blank
2. Execute LSMW
3. All vendors in the wave revert to old format

**Full rollback (all phases):**

1. Run LSMW with ALL vendors that have `UZAWE` = `01`, setting it to blank
2. Verify via `SE16` → `LFB1` → filter `UZAWE = '01'` → count should be 0
3. No FBZP changes needed — the supplement configuration (`T042D`) can remain in place (dormant) or be removed later

**Rollback timing:** Rollback can be executed at any time, including mid-day between F110 runs. The change takes effect on the next payment run that processes the affected vendor.

---

## Part B — Migration Phases

### Phase 0: Preparation

**Duration:** 3-4 weeks
**Effort:** 12-15 person-days

#### Configuration Actions

| # | Action | Transaction | Details |
|---|---|---|---|
| 1 | Audit current Payment Method usage | `SE16` → `T042Z`, `T042C`, `T042A` | Document all 10 PMs per company code: current format, house bank, forms, supplements already in use |
| 2 | Audit `UZAWE` field usage | `SE16` → `LFB1` filter `UZAWE <> ''` | Identify any vendors already using supplements. If the field is in use, determine if supplement `01` is available or choose a different value (e.g., `N1`) |
| 3 | Verify supplement support per PM | `SE16` → `T042C` field `XZAWE` | Confirm the supplement indicator is active for each Payment Method in each country. If not, activate in `FBZP` → Payment Methods in Country |
| 4 | Import / create new PMW format tree | `DMEE` | Import the target format tree (e.g., SAP standard `SEPA_CT` or custom `Z_ISO20022_CT`). Verify XML schema, field mapping, namespace declarations |
| 5 | Create PMW selection variants | `OBPM4` | One variant per Payment Method per company code (e.g., `SEPA_CT_1000_T01`, `SEPA_CT_1000_C01`, etc.). Set house bank, grouping, currency parameters |
| 6 | Configure supplements in FBZP | `FBZP` → PM in Company Code → Supplements; `OBPM` | For each PM in each company code: create supplement `01`, assign new PMW format, assign selection variant. See Section A.2.1 |
| 7 | Master data quality check | `SE16` → `LFB1`, `LFBK`; `SA38` → `RFKKIB00` (bank data report) | Verify: IBAN populated for all vendors, bank details valid, structured addresses present (street, city, postal code, country as separate fields). Fix any gaps before pilot |
| 8 | Bank communication | External | Notify each house bank that new-format DME files will begin arriving. Confirm bank's acceptance of the target format version (e.g., pain.001.001.09). Obtain bank's test file submission channel if available |
| 9 | DEV/QAS end-to-end test | `F110` in QAS | Run full F110 cycle in QAS: proposal → payment → medium generation. Validate DME output in `FDTA`. Compare field-by-field with old-format output. Test with at least 5 vendors per PM |
| 10 | DME file comparison | `FDTA`, external diff tool | Download both old-format and new-format DME files from QAS. Perform structural comparison: header fields, payment records, totals, XML schema validation. Document discrepancies and resolve |
| 11 | Authorization review | `SU01` / `PFCG` | Verify that F110 operators, payment approvers, and vendor master maintainers have the necessary authorizations for `UZAWE` field maintenance. Check authorization object `F_LFB1_BUK` (vendor master company code data) |
| 12 | Transport to PRD | `STMS` | Transport all FBZP configuration (supplement definitions, format assignments, selection variants, format trees) from DEV → QAS → PRD. Do NOT transport vendor master data — that is done separately per phase |

#### Validation Criteria Before Proceeding to Phase 1

- [ ] All 10 PMs across all company codes have supplement `01` configured with the new format in PRD
- [ ] PMW selection variants created and assigned for all PM/company code combinations
- [ ] New format tree imported and validated in `DMEE` (schema-valid XML, all required fields mapped)
- [ ] QAS end-to-end test passed: F110 run completed, DME file generated, file passes bank-side validation (or bank test channel)
- [ ] No existing `UZAWE` usage conflicts identified (or mitigated with alternative supplement value)
- [ ] All target vendors have valid IBAN and structured address data
- [ ] Banks have confirmed acceptance of the new format
- [ ] Authorization roles updated to allow `UZAWE` maintenance by designated team members

#### Rollback

No vendor-facing changes have been made. Rollback = delete supplement configuration from `FBZP` via transport (or simply leave it dormant).

---

### Phase 1: Pilot (5-10 Vendors per Payment Method)

**Duration:** 2-4 weeks
**Effort:** 8-10 person-days

#### Configuration Actions

| # | Action | Transaction | Details |
|---|---|---|---|
| 1 | Select pilot vendors | Manual / spreadsheet | 5-10 vendors per PM, meeting criteria in Section A.2.3. Document selection rationale |
| 2 | Assign supplement to pilot vendors | `XK02` (individual) or small LSMW batch | Set `LFB1-UZAWE` = `01` for each pilot vendor in each relevant company code |
| 3 | Verify assignment | `SE16` → `LFB1` filter `UZAWE = '01'` | Confirm correct vendor/company code combinations. Expected count: 50-100 (5-10 per PM × 10 PMs) |
| 4 | Execute F110 proposal run | `F110` | Run proposal with standard parameters (no changes to selection). Review proposal log — pilot vendors should appear with their normal Payment Method (e.g., `T`) |
| 5 | Review payment medium preview | `F110` → Environment → Payment Medium | Verify that two sets of DME files are generated: one for old-format vendors, one for new-format (supplement `01`) vendors |
| 6 | Inspect DME files | `FDTA` | Download new-format DME file. Validate: XML schema, field mapping (IBAN, BIC, amount, currency, remittance info, structured address), header totals. Compare with old-format file for the same vendors (use QAS test data or historical PRD data) |
| 7 | Execute F110 payment run | `F110` | Execute payment run. Monitor for errors in job log (`SM37`) and payment medium log (`FPY1`) |
| 8 | Transmit DME to bank | Bank communication channel | Send new-format DME file to bank via standard channel (file upload, SWIFT, EBICS, H2H). Monitor for bank-side rejection or acknowledgment |
| 9 | Monitor bank processing | External / bank portal | Confirm: files accepted, payments processed, no rejection codes. Check payment status reports (pain.002 / camt.054) if available |
| 10 | Reconcile | `REGUH`, `REGUP`, `FPY1`-`FPY3` | Verify: all pilot vendors' payments are posted correctly, amounts match, no duplicate or missing payments. Cross-reference with bank statements |
| 11 | Document results | Spreadsheet / change log | Record per vendor: payment date, run ID, format used, DME file ID, bank acceptance status. This is the audit trail |

#### Validation Criteria Before Proceeding to Phase 2

- [ ] At least 2 full F110 cycles (e.g., 2 weekly runs) completed successfully with pilot vendors on new format
- [ ] Zero bank rejections for new-format DME files
- [ ] DME file field-level validation passed (XML schema valid, all mandatory ISO 20022 elements present)
- [ ] Payment amounts and totals reconcile between `REGUH`/`REGUP` and bank statements
- [ ] No errors in payment medium generation logs (`FPY1`)
- [ ] Pilot vendors confirmed payment receipt (for critical/cooperative vendors)
- [ ] Old-format vendors continue to process normally — no regression

#### Rollback Procedure

1. Identify affected vendors (from pilot list)
2. Execute `XK02` or LSMW: set `LFB1-UZAWE` = blank for all pilot vendors
3. Verify via `SE16` → `LFB1` → filter `UZAWE = '01'` → count = 0
4. Next F110 run: all vendors revert to old format
5. Time to rollback: < 1 hour for manual `XK02` (5-10 vendors), < 30 minutes for LSMW

---

### Phase 2: Expansion (50% of Vendors)

**Duration:** 2-3 weeks
**Effort:** 8-10 person-days

#### Configuration Actions

| # | Action | Transaction | Details |
|---|---|---|---|
| 1 | Prepare expansion vendor list | Spreadsheet / SQL on `LFB1` | Select ~50% of remaining vendors per PM. Prioritize: domestic before international, high-IBAN-quality before others, medium-frequency vendors |
| 2 | Execute LSMW mass update | `LSMW` | Set `LFB1-UZAWE` = `01` for all expansion vendors. Run in QAS first to validate, then in PRD |
| 3 | Verify assignment | `SE16` → `LFB1` | Confirm supplement is set for the correct vendors. Expected: ~50% of total vendor base per PM |
| 4 | Execute F110 runs | `F110` | Execute standard payment runs. Monitor: proposal log, payment medium log, DME file generation |
| 5 | Validate DME volume | `FDTA`, `FPY1` | New-format DME file should now contain ~50% of payment items. Verify file size, record count, and totals are proportionally larger than Phase 1 |
| 6 | Monitor bank processing | External | Confirm bank acceptance for the larger volume. Watch for: batch size limits, file size limits, processing time increases |
| 7 | Cross-border payments | `FDTA` | If international vendors are included in this wave, validate: BIC routing, currency conversion, cross-border XML elements (charges, regulatory reporting codes) |
| 8 | Performance monitoring | `SM37`, `ST05` (SQL trace if needed) | Monitor F110 runtime — with two format streams, the payment medium generation step may take slightly longer. Ensure it stays within the payment processing window |

#### Monitoring Plan

During Phase 2, implement daily monitoring for 2 weeks:

| Check | Frequency | Tool | Action if Failed |
|---|---|---|---|
| DME file generated without errors | Every F110 run | `FPY1`, `SM37` | Investigate error log; rollback affected vendors if critical |
| Bank acceptance confirmation | Within 24h of DME transmission | Bank portal / pain.002 | Contact bank; rollback if systematic rejection |
| Payment amount reconciliation | Daily | `REGUH` vs. bank statement | Investigate discrepancy; pause expansion if pattern found |
| Vendor complaints | Ongoing | Helpdesk / email | Investigate per vendor; rollback individual vendor if needed |
| F110 runtime within SLA | Every F110 run | `SM37` job duration | Optimize selection variant or split run if timeout risk |

#### Validation Criteria Before Proceeding to Phase 3

- [ ] At least 2 full F110 cycles completed with 50% vendor population on new format
- [ ] Bank acceptance rate: 100% (zero rejections)
- [ ] No payment discrepancies in reconciliation
- [ ] F110 runtime within acceptable SLA (no degradation > 15%)
- [ ] Cross-border payments (if included) processed correctly
- [ ] No systemic vendor complaints related to format change

#### Rollback Procedure

1. Prepare LSMW reversal file: all Phase 2 vendors with `UZAWE` = blank
2. Execute LSMW
3. Verify count
4. Phase 1 pilot vendors can remain on new format (they are proven) or be reverted too, depending on the nature of the issue
5. Time to rollback: < 2 hours including verification

---

### Phase 3: Full Migration (100% of Vendors)

**Duration:** 1-2 weeks
**Effort:** 6-8 person-days

#### Configuration Actions

| # | Action | Transaction | Details |
|---|---|---|---|
| 1 | Prepare final vendor list | `SE16` → `LFB1` filter `UZAWE <> '01'` AND `ZWELS` = relevant PM | All remaining vendors not yet on supplement `01` |
| 2 | Execute LSMW mass update | `LSMW` | Set `LFB1-UZAWE` = `01` for all remaining vendors |
| 3 | Verify 100% coverage | `SE16` → `LFB1` | For each PM and company code: all vendors with that PM should have `UZAWE` = `01`. Cross-check count against total vendor population per PM |
| 4 | Execute F110 runs | `F110` | Standard payment runs. The old-format DME stream should now be empty or near-empty |
| 5 | Confirm old format stream is empty | `FDTA`, `FPY1` | Verify that no DME files are generated in the old format. If any remain, investigate — likely a vendor that was missed or a new vendor created without the supplement |
| 6 | Monitor for 1-2 full payment cycles | All monitoring from Phase 2 | Confirm 100% new-format processing is stable |

#### Validation Criteria Before Proceeding to Phase 4

- [ ] 100% of vendors across all PMs and company codes have `UZAWE` = `01`
- [ ] At least 2 full F110 cycles with 100% new-format output
- [ ] Zero old-format DME files generated
- [ ] Zero bank rejections
- [ ] Full reconciliation passed
- [ ] Sign-off from Treasury / AP operations

#### Rollback Procedure

Same as Phase 2 but larger scope. Full rollback of all vendors to blank `UZAWE`:

1. Prepare LSMW reversal file: ALL vendors, `UZAWE` = blank
2. Execute LSMW
3. Time to rollback: < 4 hours including verification for large vendor populations

> **Important:** After Phase 3 is validated, the rollback window begins to close. Once Phase 4 (in-place format swap) is executed, rollback requires reversing the FBZP configuration change, which is more complex.

---

### Phase 4: Cleanup

**Duration:** 1-2 weeks
**Effort:** 5-7 person-days

#### Configuration Actions

| # | Action | Transaction | Details |
|---|---|---|---|
| 1 | Swap base Payment Method format | `FBZP` → PM in Company Code | For each PM in each company code: change the base format (non-supplement) from the old format to the new format. Example: PM `T` in company code `1000` → change "Payment Medium Format" from `RFFOD__T` to `SEPA_CT` |
| 2 | Remove supplement format assignment | `FBZP` → PM in Company Code → Supplements; `OBPM` | Remove the format assignment from supplement `01` (or delete the supplement definition entirely from `T042D`). Now the base PM format handles everything |
| 3 | Clear vendor supplement fields | `LSMW` | Mass update ALL vendors: set `LFB1-UZAWE` = blank. The supplement is no longer needed because the base PM now uses the new format |
| 4 | Verify base format processing | `F110` | Execute payment run. Verify that all vendors produce DME output in the new format via the base PM (no supplement involved) |
| 5 | Remove old format artifacts | `DMEE` (optional) | Mark old format trees as inactive or delete if no longer needed by any PM in any company code. Remove old classic programs from FBZP country-level assignment if fully replaced by PMW |
| 6 | Clean up selection variants | `OBPM4` | Remove supplement-specific selection variants that are no longer needed. Keep base-level variants for the new format |
| 7 | Update documentation | External | Update: FBZP configuration documentation, F110 runbooks, payment format specification documents, bank communication records |
| 8 | Archive audit trail | External | Export the migration log: per vendor, per PM, per company code — when the supplement was assigned, when payments first ran on the new format, when the supplement was cleared. Store per audit retention policy |
| 9 | Deactivate BAdI (if used) | `SE19` | If a logging BAdI was implemented in Phase 0, deactivate or remove it. Transport the deactivation |
| 10 | New vendor process update | Organizational | Update vendor master creation procedures: new vendors should NOT receive a supplement value (the base PM format is now the new format). Update any automated vendor creation interfaces (IDocs, BAPI wrappers) |

#### Validation Criteria — Migration Complete

- [ ] All PMs in all company codes: base format = new format (verified in `T042Z`)
- [ ] All vendors: `UZAWE` = blank (verified in `LFB1`)
- [ ] Supplement configuration removed or deactivated in `T042D`
- [ ] At least 2 full F110 cycles completed post-cleanup with base format only
- [ ] Old format trees deactivated / archived
- [ ] Documentation updated
- [ ] Formal sign-off from all stakeholders (Treasury, AP Operations, IT, Audit)

#### Rollback Procedure

Rollback after Phase 4 is more complex because the base Payment Method format has been changed:

1. Re-transport the old FBZP configuration (restore old format assignment for base PM) from a saved transport or manually reconfigure
2. This is why Phase 4 should only be executed after Phase 3 has been validated for at least 2 full payment cycles
3. Consider keeping the old format tree in the system (inactive) for 3-6 months post-migration as an emergency fallback

---

### Effort Summary

| Phase | Duration | Person-Days (2 consultants) | Key Activities |
|---|---|---|---|
| Phase 0: Preparation | 3-4 weeks | 12-15 | Config, testing, bank coordination, master data cleanup |
| Phase 1: Pilot | 2-4 weeks | 8-10 | 50-100 vendors, validation, DME comparison |
| Phase 2: Expansion | 2-3 weeks | 8-10 | 50% vendors, monitoring, cross-border validation |
| Phase 3: Full Migration | 1-2 weeks | 6-8 | Remaining vendors, confirm 100% coverage |
| Phase 4: Cleanup | 1-2 weeks | 5-7 | In-place swap, remove supplements, documentation |
| **Total** | **10-15 weeks** | **39-50** | |

> Note: Add 5-10 person-days for project management, stakeholder communication, and buffer for issues. Realistic total: **45-60 person-days.**

---

## Part C — Comparison Table

| Approach | Vendor-Level Granularity | Requires Vendor Master Changes | Requires Custom Code | Consumes PM Characters | Rollback Complexity | Operational Complexity | Recommended |
|---|---|---|---|---|---|---|---|
| **Payment Method Supplements** | Yes | Yes (`LFB1-UZAWE`) | No | No | Low (clear `UZAWE`) | Low (single F110 run) | **Yes** — best fit for phased migration with full granularity and zero PM character consumption |
| **House Bank / Account ID Override** | No (bank-level) | No | No | No | Medium (re-rank banks) | Medium (bank determination changes) | **No** — lacks vendor-level granularity; cannot do per-vendor pilot |
| **BAdI Dynamic Format Switching** | Yes | No (if using Z-table) | Yes (`FI_PAYMEDIUM`) | No | Low (deactivate BAdI) | Medium (code maintenance) | **Partial** — excellent as secondary audit/logging tool; too risky as sole format-routing mechanism in payment critical path |
| **Temporary Duplicate PM** | Yes | Yes (`LFB1-ZWELS`) | No | Yes (10 extra chars) | High (double master data revert) | High (double FBZP config) | **No** — character exhaustion risk + massive config duplication + double master data churn |
| **Parallel F110 Runs** | Yes (via selection) | No | No | No | Low (stop second run) | High (double runs, scheduling, reconciliation) | **No** — doubles operational burden; duplicate payment risk; not sustainable for 10 PMs |
| **Hybrid: Supplements + BAdI Logging** | Yes | Yes (`LFB1-UZAWE`) | Yes (read-only BAdI) | No | Low | Low-Medium | **Yes** — recommended if auditability requirements are strict; BAdI only logs, does not route |

---

## Part D — Risks and Mitigations

### Risk 1: Bank-Side Format Acceptance Readiness

**Description:** The house bank(s) may not be ready to accept the new format version (e.g., pain.001.001.09) at the time of pilot go-live. Banks may have their own testing and certification timelines that do not align with the SAP migration schedule.

**Likelihood:** Medium
**Impact:** High — payments rejected by bank, causing delays and potential late-payment penalties.

**Mitigations:**

1. Engage each house bank 8-12 weeks before Phase 1. Obtain written confirmation of format acceptance and the specific version/schema supported.
2. Use the bank's test file submission channel (if available) to submit sample DME files from QAS before going live in PRD.
3. If the bank is not ready, delay Phase 1 for that specific house bank / company code while proceeding with others.
4. Maintain the old format as fallback (via supplement rollback) until bank confirmation is received.

### Risk 2: DME / IDoc File Validation Gaps

**Description:** The new format may have field-mapping issues that are not caught during DEV/QAS testing: missing mandatory elements (e.g., structured address when bank expects it), incorrect namespace URIs, wrong character encoding, invalid XML structure, or missing UETR.

**Likelihood:** Medium
**Impact:** High — bank rejection or misrouted payments.

**Mitigations:**

1. Perform field-by-field comparison of old vs. new DME output for the same payment in QAS. Document every difference.
2. Validate new-format DME files against the official XML schema (XSD) using an external XML validator before bank submission.
3. For ISO 20022: validate against the ISO 20022 message definition report (MDR) for the target version.
4. Check SAP Note 2665657 (ISO 20022 format notes) and Note 1682833 (SEPA format corrections) for known field-mapping issues.
5. Pilot phase (Phase 1) with only 5-10 vendors specifically tests for these gaps at low volume and low risk.

### Risk 3: Vendor Master Data Inconsistencies

**Description:** The new format (especially ISO 20022) has stricter data requirements than legacy formats: IBAN is mandatory (not bank account number), structured addresses require separate street/city/postal code/country fields (not a single address line), and BIC may be required for cross-border payments. Vendors with incomplete master data will cause format generation errors.

**Likelihood:** High — almost every SAP system has some legacy vendors with incomplete data.
**Impact:** Medium — individual vendor payments fail; format generation errors in `FPY1`.

**Mitigations:**

1. Run a master data quality report in Phase 0: query `LFB1` + `LFBK` (bank details) + `LFA1` (address) for all vendors assigned to the 10 Payment Methods.
2. Flag vendors with: missing IBAN, missing BIC (for cross-border), unstructured address (single-line `STRAS` instead of structured fields), missing country code.
3. Remediate critical gaps before Phase 1. For large populations, prioritize pilot vendors first, then batch-fix remaining vendors before Phase 2.
4. Consider implementing validation in the PMW selection variant or a preprocessing BAdI that rejects vendors with incomplete data rather than generating a malformed DME file.
5. For the November 2026 structured-address mandate: audit `LFA1-STRAS` vs. `LFA1-STRAS` + `ORT01` + `PSTLZ` + `LAND1` and convert to structured format.

### Risk 4: UZAWE Field Already in Use

**Description:** The Payment Method Supplement field (`LFB1-UZAWE`) may already be populated for some or all vendors, used for a different purpose (e.g., differentiating check forms, routing specific payment advices, or supporting a prior format migration that was never cleaned up).

**Likelihood:** Low-Medium (depends on client history).
**Impact:** High — overwriting existing supplement values disrupts existing payment processing logic.

**Mitigations:**

1. **Phase 0 audit:** Run `SE16` → `LFB1` with filter `UZAWE <> ''` to identify all vendors with existing supplement values. Analyze what those values do.
2. **If supplements are in active use:** Choose a different supplement value (e.g., `N1` instead of `01`) that does not conflict. Create the new supplement in `T042D` with that value.
3. **If supplements are legacy (orphaned):** Clean up the orphaned values first, then use `01` as planned.
4. **If supplements are critical and fully consumed:** Fall back to the BAdI-based approach (A.1.3) as the primary mechanism instead of supplements.

### Risk 5: Authorization and Role Impacts

**Description:** Vendor master maintainers may not have authorization to change the `UZAWE` field. F110 operators may need additional authorization to process payments with the new format. Authorization objects `F_LFB1_BUK` (vendor company code data) and `F_REGU_BUK` (payment run authorization) may need adjustment.

**Likelihood:** Medium
**Impact:** Medium — delays in vendor master updates or payment run execution.

**Mitigations:**

1. In Phase 0, verify that the designated migration team (LSMW executor, vendor master maintainers) has authorization for `LFB1-UZAWE` changes in all target company codes.
2. Verify that F110 operators have authorization for the new PMW format execution (authorization object `F_PMTFMT`).
3. Create a temporary authorization role for the migration team if needed, with a defined expiry date.
4. Test authorization in QAS before Phase 1 go-live.

### Risk 6: Payment Run Scheduling Conflicts

**Description:** Phase 1-3 may coincide with high-volume payment periods (month-end, quarter-end, year-end), increasing the risk of issues and reducing the time available for validation and troubleshooting.

**Likelihood:** Medium
**Impact:** Medium — pressure to skip validation steps; higher blast radius if issues occur during peak periods.

**Mitigations:**

1. **Schedule Phase 1 pilot for a low-volume week** (mid-month, not month-end or quarter-end).
2. **Do not expand waves (Phase 2, Phase 3) during month-end close** — freeze for the last 5 business days of each month.
3. **Phase 4 (in-place swap) should be scheduled for the first week after a quarter-end**, when the next peak is farthest away.
4. Maintain a clear "migration freeze calendar" shared with AP Operations.

### Risk 7: ISO 20022 Version Mismatch

**Description:** The target format version in the SAP format tree may not match the version required by the bank or clearing house. For example, SAP may ship pain.001.001.03 (2009 version) while the bank requires pain.001.001.09 (2019 version). Or the SAP system may be on an older support package that does not include the 2019-version format trees.

**Likelihood:** Medium (especially on older ECC systems).
**Impact:** High — bank rejection of the entire DME file.

**Mitigations:**

1. Confirm the exact format version required by each house bank (pain.001.001.03 vs. .09). Document in the migration plan.
2. Check SAP system support package level and available format trees in `DMEE`. For the 2019 versions (pain.001.001.09), SAP Note 2816629 and subsequent notes provide the updated format trees.
3. If the required version is not available, apply the relevant SAP support package or import the format tree via Note attachment.
4. For custom format trees (`Z_*`), verify the XML namespace and version attributes match the target version.
5. Validate the generated XML against the official XSD for the target version before bank submission.

---

## Part E — Migration Checklist

### Phase 0: Preparation

- [ ] Document all 10 Payment Methods per company code: current format, house bank, forms, amounts
- [ ] Export current FBZP configuration (screenshot or `SM30` → `T042Z`, `T042C`, `T042A` entries)
- [ ] Audit `LFB1-UZAWE` for existing supplement usage across all company codes
- [ ] Verify supplement indicator (`T042C-XZAWE`) is active for all PMs in all countries
- [ ] Import / create new PMW format tree in `DMEE` — validate XML schema
- [ ] Verify format tree version matches bank requirements (e.g., pain.001.001.09)
- [ ] Apply relevant SAP Notes (2816629, 2665657, 1682833) if format trees are missing
- [ ] Create PMW selection variants in `OBPM4` for each PM × company code
- [ ] Configure supplement `01` in `FBZP` → PM in Company Code → Supplements for all PMs / company codes
- [ ] Assign new PMW format to supplement `01` in each PM / company code
- [ ] Run vendor master data quality report: IBAN, BIC, structured address
- [ ] Remediate critical master data gaps (missing IBAN, unstructured addresses)
- [ ] Contact each house bank — confirm new format acceptance and version
- [ ] Obtain bank test file submission channel (if available)
- [ ] Verify authorization roles: `UZAWE` maintenance, F110 execution, PMW format authorization
- [ ] Execute end-to-end test in DEV: `F110` with supplement vendors → DME generation → file validation
- [ ] Execute end-to-end test in QAS: same as DEV, with bank test file submission if available
- [ ] Compare old-format and new-format DME output field-by-field — document all differences
- [ ] Resolve all DME comparison discrepancies
- [ ] Transport all configuration to PRD (FBZP, OBPM, OBPM4, DMEE)
- [ ] Prepare pilot vendor selection list (5-10 per PM, meeting criteria in Section A.2.3)
- [ ] Create LSMW project for mass `UZAWE` update (record `XK02` transaction)
- [ ] Test LSMW in QAS with sample vendor list
- [ ] Establish migration freeze calendar (no changes during month-end/quarter-end)
- [ ] Schedule Phase 1 start date (low-volume week)

### Phase 1: Pilot

- [ ] Assign supplement `01` to pilot vendors via `XK02` or LSMW (50-100 vendors total)
- [ ] Verify assignment in `SE16` → `LFB1` → filter `UZAWE = '01'` — count matches pilot list
- [ ] Execute F110 proposal run — review proposal log for pilot vendors
- [ ] Inspect DME preview in `F110` → Environment → Payment Medium
- [ ] Validate new-format DME file in `FDTA`: XML schema, field mapping, totals
- [ ] Execute F110 payment run
- [ ] Monitor payment medium log in `FPY1` — zero errors
- [ ] Transmit new-format DME file to bank
- [ ] Confirm bank acceptance (no rejection within 24h)
- [ ] Reconcile: `REGUH`/`REGUP` totals match bank statement
- [ ] Verify old-format vendors continue to process normally (no regression)
- [ ] Complete at least 2 full F110 cycles with pilot group
- [ ] Document results per vendor: run ID, format used, bank acceptance
- [ ] Obtain sign-off from AP Operations for Phase 2 expansion

### Phase 2: Expansion (50%)

- [ ] Prepare expansion vendor list (~50% of remaining vendors per PM)
- [ ] Execute LSMW in QAS — verify correct `UZAWE` assignment
- [ ] Execute LSMW in PRD
- [ ] Verify in `SE16` → `LFB1` — ~50% of vendor base per PM has `UZAWE = '01'`
- [ ] Execute F110 runs — monitor for increased DME volume in new format
- [ ] Validate DME file record count and totals proportional to vendor coverage
- [ ] Monitor bank processing for larger file volumes
- [ ] If international vendors included: validate cross-border XML elements
- [ ] Monitor F110 runtime — ensure no SLA degradation > 15%
- [ ] Daily reconciliation for 2 weeks: `REGUH`/`REGUP` vs. bank statements
- [ ] Monitor vendor complaints — investigate any format-related issues
- [ ] Complete at least 2 full F110 cycles at 50% coverage
- [ ] Obtain sign-off for Phase 3

### Phase 3: Full Migration (100%)

- [ ] Prepare final vendor list: all remaining vendors without supplement
- [ ] Execute LSMW in PRD — set `UZAWE = '01'` for all remaining vendors
- [ ] Verify 100% coverage: `SE16` → `LFB1` — all PM vendors have `UZAWE = '01'`
- [ ] Execute F110 runs
- [ ] Confirm old-format DME stream is empty (zero files generated in old format)
- [ ] Investigate any remaining old-format output — missed vendors, new vendors
- [ ] Establish process for new vendor creation: no supplement needed (will be handled in Phase 4)
- [ ] Complete at least 2 full F110 cycles at 100% new-format coverage
- [ ] Full reconciliation — zero discrepancies
- [ ] Obtain sign-off from Treasury, AP Operations, and IT for Phase 4

### Phase 4: Cleanup

- [ ] Swap base PM format: `FBZP` → PM in Company Code → change format from old to new for each PM / company code
- [ ] Remove supplement format assignment from `T042D` (or delete supplement `01`)
- [ ] Mass-clear `LFB1-UZAWE` for all vendors via LSMW (set to blank)
- [ ] Verify: `SE16` → `LFB1` → filter `UZAWE = '01'` → count = 0
- [ ] Execute F110 run with base PM format (no supplements) — verify DME output is new format
- [ ] Verify 2 full F110 cycles post-cleanup
- [ ] Deactivate / remove BAdI implementation (if used for logging)
- [ ] Remove supplement-specific PMW selection variants from `OBPM4`
- [ ] Deactivate old format trees in `DMEE` (keep for 3-6 months as emergency fallback)
- [ ] Update vendor master creation procedures: new vendors use base PM (new format by default)
- [ ] Update F110 runbook / operating procedures
- [ ] Update FBZP configuration documentation
- [ ] Archive migration audit trail (per vendor: supplement assignment dates, format change dates, run IDs)
- [ ] Formal sign-off: migration complete — from Treasury, AP Operations, IT, Internal Audit
- [ ] Schedule old format tree deletion: 6 months post-migration

---

## Appendix A — Key SAP Transactions Reference

| Transaction | Purpose | When Used |
|---|---|---|
| `FBZP` | Payment method configuration (all sections) | Phase 0 (config), Phase 4 (swap) |
| `F110` | Automatic payment program execution | All phases |
| `OBPM` | Payment method supplements configuration | Phase 0 (config) |
| `OBPM1` | PMW format assignment | Phase 0 (config) |
| `OBPM4` | PMW selection variants | Phase 0 (config), Phase 4 (cleanup) |
| `DMEE` | DME format tree editor | Phase 0 (validation) |
| `FDTA` | DME file administration / download | All phases (validation) |
| `XK02` / `FK02` | Vendor master change (supplement field) | Phase 1 (individual), Phase 2-3 (small batches) |
| `LSMW` | Mass vendor master update | Phase 1-4 (mass updates) |
| `SM30` → `T042Z` | Direct table maintenance for format assignment | Phase 0 (audit) |
| `SE16` / `SE16N` | Table data browser | All phases (verification) |
| `SE18` / `SE19` | BAdI definition / implementation | Phase 0 (if BAdI logging used) |
| `FPY1`-`FPY3` | Payment Medium Workbench monitor | All phases (monitoring) |
| `SM37` | Job monitoring | All phases (F110 job monitoring) |
| `STMS` | Transport management | Phase 0 (transport config) |
| `SU01` / `PFCG` | User / role maintenance | Phase 0 (authorization) |

## Appendix B — Key Tables Reference

| Table | Content | Key Fields |
|---|---|---|
| `T042C` | Payment methods per country | `XZAWE` (supplement allowed) |
| `T042D` | Payment method supplements | Supplement code, format assignment |
| `T042Z` | Payment methods in company code (format) | PMW format key |
| `TFPM042FZ` | PMW format supplements | `FORMI` (format), `FORMZ` (supplement) |
| `T042A` | Payment methods per company code | Amount limits, house bank |
| `T042E` | Bank determination / ranking | House bank, account ID, ranking |
| `LFB1` | Vendor master — company code segment | `ZWELS` (PM), `UZAWE` (supplement) |
| `LFBK` | Vendor master — bank details | IBAN, BIC, bank key |
| `LFA1` | Vendor master — general data | Address fields (`STRAS`, `ORT01`, `PSTLZ`, `LAND1`) |
| `REGUH` | Payment run header | `RZAWE` (PM), `UZAWE` (supplement), amounts |
| `REGUP` | Payment run items | Line item details |
| `CDPOS` / `CDHDR` | Change documents | Audit trail for `LFB1-UZAWE` changes |

## Appendix C — Relevant SAP Notes

| Note | Title / Purpose |
|---|---|
| 2816629 | ISO 20022 pain.001.001.09 / pain.002.001.10 format trees |
| 2665657 | ISO 20022 format notes and corrections |
| 1682833 | SEPA format corrections and updates |
| 1541877 | PMW format migration guidance |
| 986210 | Payment method supplements — functional documentation |
| 3107250 | `UZAWE` field visibility in BP transaction (S/4HANA) |
| 2175218 | Structured address handling in payment formats |
| 3061862 | `UZAWE` field not updated in `REGUH` after F111 — relevant if using cross-company code payments |

> **Note:** Always check the SAP Support Portal for the latest versions of these notes and any dependent/successor notes before starting the migration.
