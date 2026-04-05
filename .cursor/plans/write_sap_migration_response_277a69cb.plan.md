---
name: Write SAP Migration Response
overview: Write a comprehensive SAP Payment Method format migration strategy document (~150-200 pages-worth of content) into `responses/sap-payment-method-migration-response.md`, structured across five parts (A through E) as specified in the prompt, grounded by domain knowledge in `CLAUDE.md` and supplemented with web research on ISO 20022 deadlines and SAP OSS notes.
todos:
  - id: write-preamble
    content: Write executive summary and ISO 20022 context preamble
    status: completed
  - id: write-part-a
    content: "Write Part A: evaluate all 6 mechanisms, recommend supplements as primary, detail FBZP config, runtime trace, vendor assignment, expansion, rollback"
    status: completed
  - id: write-part-b
    content: "Write Part B: 5 migration phases (0-4) with config actions, validation criteria, rollback, effort estimates"
    status: completed
  - id: write-part-c
    content: "Write Part C: comparison table of all approaches"
    status: completed
  - id: write-part-d
    content: "Write Part D: 7 risks with mitigations"
    status: completed
  - id: write-part-e
    content: "Write Part E: phase-by-phase markdown checkbox checklist"
    status: completed
  - id: review-quality
    content: Review full document for completeness, accuracy, SAP terminology, and formatting
    status: completed
isProject: false
---

# SAP Payment Method Format Migration — Response Document

## Target File

[responses/sap-payment-method-migration-response.md](responses/sap-payment-method-migration-response.md) (currently empty)  
  
Further files could also be created if needed (e.g. some additional important and useful information, steps, approaches etc.)

## Structure

The document will follow the exact five-part structure demanded by the prompt, plus a preamble and appendix:

### Preamble

- Executive summary of the recommended strategy (Payment Method Supplements as primary lever, with BAdI as secondary/optional enhancement)
- Key ISO 20022 deadlines (Nov 2025 SWIFT MT sunset, Nov 2026 SEPA structured address mandate)

### Part A — Recommended Approach

- **Evaluation of all 6 mechanisms** with verdict per mechanism:
  - Payment Method Supplements (RECOMMENDED PRIMARY) -- vendor-level granularity, no char exhaustion, standard SAP, reversible
  - House Bank / Account ID override (NOT recommended as primary) -- bank-level, not vendor-level
  - BAdI FI_PAYMEDIUM (RECOMMENDED SECONDARY / OPTIONAL) -- full programmatic control but custom code risk
  - Temporary duplicate Payment Method (NOT recommended) -- char exhaustion, double maintenance
  - Parallel F110 runs (NOT recommended) -- operationally complex, changes user workflow
  - Hybrid: Supplements + lightweight BAdI logging (mentioned as best-of-both)
- **Detailed walkthrough of the supplement approach:**
  - FBZP config steps: `FBZP` -> Payment Methods in Company Code -> create supplement `01` in `T042D` -> assign new PMW format (e.g., `SEPA_CT`) to supplement
  - Runtime trace: `F110` -> `SAPFPAYM` -> reads `REGUH` (Payment Method + Supplement) -> looks up `T042D`/`T042Z` -> supplement `01` resolves to new format -> generates DME via PMW
  - Vendor assignment: `XK02`/`FK02` field `LFB1-UZAWE` set to `01` for pilot vendors; mass update via LSMW or BDC recording
  - Gradual expansion: 5-10 vendors -> 50% -> 100% by waves of LSMW runs
  - Rollback: blank out `UZAWE` in vendor master -> instant fallback to base Payment Method format

### Part B — Migration Phases (0 through 4)

Five phases with per-phase:

- Config actions (transactions, tables, fields)
- Validation / acceptance criteria
- Rollback procedure
- Effort estimate (person-days for 2-consultant team)

Estimated total effort: ~45-60 person-days across all phases for 10 PMs x multiple company codes.

### Part C — Comparison Table

Markdown table with 7 columns comparing all 6 approaches on:

- Vendor-level granularity
- Vendor master changes required
- Custom code required
- PM character consumption
- Rollback complexity
- Operational complexity
- Recommendation verdict

### Part D — Risks and Mitigations

7 risks:

1. Bank-side format acceptance readiness
2. DME/IDoc validation gaps (field mapping, structured address)
3. Vendor master data quality (missing IBAN, unstructured addresses)
4. UZAWE field already in use for other purposes
5. Authorization / role impacts (supplement field access)
6. F110 scheduling conflicts during cutover
7. ISO 20022 version mismatch (pain.001.001.03 vs .09)

### Part E — Checklist

Markdown checkbox list organized by phase (Phase 0 through Phase 4), ready for copy-paste into a project tracker.

## Key Technical References Used

- `CLAUDE.md` domain knowledge (FBZP architecture, T042* tables, format resolution, BAdIs)
- ISO 20022 migration deadlines (Nov 2025 / Nov 2026)
- SAP transactions: `FBZP`, `OBPM`, `OBPM1`, `OBPM4`, `DMEE`, `FDTA`, `F110`, `XK02`, `FPY1`-`FPY3`
- Tables: `T042D`, `T042Z`, `T042C`, `LFB1`, `REGUH`, `REGUP`, `TFPM042FZ`
- SAP Notes: 1541877 (PMW migration), 986210 (supplements), 3107250 (UZAWE visibility in BP)

