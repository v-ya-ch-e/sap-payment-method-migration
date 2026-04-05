# Prompt: SAP Payment Method — Phased Format Migration Strategy

> **Usage:** Feed this prompt (along with `CLAUDE.md` as context) to an LLM to get a detailed, actionable migration plan for switching SAP payment formats in production.

---

## The Prompt

```text
You are a senior SAP FI/AP consultant with 15+ years of experience in payment processing,
FBZP configuration, Payment Medium Workbench (PMW), and large-scale production migrations.

## Situation

A client has the following SAP landscape:

- Multiple company codes (each with its own FBZP configuration).
- 10 different Payment Methods actively used per company code (e.g., T = bank transfer,
  C = check, U = urgent transfer, etc.).
- Each Payment Method currently uses an OLD payment format (either a classic RFFO* program
  or an older PMW format tree).
- The goal is to migrate ALL Payment Methods across ALL company codes to a NEW payment
  format (e.g., ISO 20022 pain.001/pain.002, CGI XML, or a new custom PMW format tree).
- The system is a productive SAP ECC or S/4HANA environment with daily payment runs via F110.
- Hundreds of vendors are assigned to each Payment Method.

## Constraints

1. **Character exhaustion risk:** Payment Method codes are single-character (A-Z, 0-9).
   Creating a duplicate Payment Method for every existing one (10 old + 10 new = 20)
   may exhaust the available characters, especially if some are already taken.
   Additionally, duplicating all configuration (FBZP settings, bank determination,
   house banks, forms, authorizations) per Payment Method is a massive maintenance burden.

2. **Big-bang risk:** Simply replacing the format in an existing Payment Method
   (in-place swap) immediately affects every vendor and every payment run using that
   method. If the new format has issues (bank rejection, field mapping errors, missing
   data), the entire payment process is disrupted with no easy rollback.

3. **Production safety:** The client requires:
   - Phased rollout (start with a small pilot group, expand gradually).
   - Ability to rollback to the old format at any point without reconfiguration.
   - Minimal disruption to daily F110 operations.
   - No changes to F110 run parameters, selection variants, or user workflows
     (if possible).

4. **Auditability:** Every migration step must be traceable. The client must be able
   to demonstrate which vendors used which format on which date.

## Your Task

Provide a **detailed, step-by-step migration strategy** that addresses the following:

### Part A — Recommended Approach

1. Evaluate the following migration mechanisms and recommend the BEST primary approach
   (or a combination). For each, explain why it does or does not fit:
   - **Payment Method Supplements (Zahlwegergänzung)** — using the 2-char supplement
     field (UZAWE / T042D) to route pilot vendors to the new format while keeping the
     base Payment Method on the old format.
   - **House Bank / Account ID level format override** — creating a parallel account ID
     with the new format and adjusting bank determination ranking.
   - **BAdI-based dynamic format switching** — implementing FI_PAYMEDIUM or similar BAdI
     to select format at runtime based on a pilot vendor list / Z-table / date range.
   - **Temporary duplicate Payment Method** — creating one temporary Payment Method code
     per original, used only during migration, then decommissioned.
   - **Parallel F110 runs** — separate payment runs for old-format and new-format vendors.
   - **Any other standard SAP mechanism** I may have overlooked.

2. For your recommended approach, provide:
   - The exact FBZP configuration steps (which sections to change, which fields).
   - How the format resolution works at runtime (trace the logic from F110 → SAPFPAYM
     → format selection).
   - How vendor assignment to the pilot group is managed.
   - How to expand the pilot gradually (10% → 50% → 100%).
   - How rollback works at each phase.

### Part B — Migration Phases

Provide a phased migration timeline with concrete steps:

- **Phase 0: Preparation** — what to configure, test in DEV/QAS, validate with banks.
- **Phase 1: Pilot (5-10 vendors per Payment Method)** — go-live with a small group,
  validation criteria, DME file comparison (old vs. new format).
- **Phase 2: Expansion (50%)** — scale to half the vendor population, monitoring plan.
- **Phase 3: Full migration (100%)** — switch remaining vendors, final format swap.
- **Phase 4: Cleanup** — remove temporary configuration, update documentation.

For each phase, specify:
- Exact configuration actions (transactions, tables, fields).
- Validation / acceptance criteria before proceeding to the next phase.
- Rollback procedure if issues are found.
- Estimated effort (person-days) for a team of 2 SAP consultants.

### Part C — Comparison Table

Provide a comparison table of all approaches with these columns:
- Approach name
- Vendor-level granularity (yes/no)
- Requires vendor master changes (yes/no)
- Requires custom code (yes/no)
- Consumes Payment Method characters (yes/no)
- Rollback complexity (low/medium/high)
- Operational complexity (low/medium/high)
- Recommended for this scenario (yes/no + brief reason)

### Part D — Risks and Mitigations

List the top 5-7 risks specific to this migration and how to mitigate each.
Include:
- Bank-side format acceptance readiness
- Idoc / DME file validation gaps
- Master data inconsistencies
- Authorization / role impacts
- Payment run scheduling conflicts

### Part E — Checklist

Provide a ready-to-use checklist (in markdown checkbox format) that the team can
follow during each phase of the migration.

## Output Format

- Use clear headings and subheadings.
- Include specific SAP transaction codes, table names, and field names.
- Where relevant, show configuration examples (e.g., "In FBZP → Payment Methods in
  Company Code → Company 1000 → Payment Method T → Supplement 01 → set PMW format
  to SEPA_CT").
- Use tables for comparisons.
- Use numbered steps for procedures.
- Write in English. Use SAP standard terminology.
```

---

## How to Use This Prompt

1. **Provide context first:** Paste the contents of `CLAUDE.md` (or attach it) before this prompt so the LLM has the domain knowledge grounding.

2. **Customize the variables:** Before sending, replace or specify:
   - The exact **old format** name (e.g., `RFFOD__T`, a specific PMW format key).
   - The exact **new format** name (e.g., `SEPA_CT`, `CGI_XML_V3`, your custom Z-format).
   - The **number of company codes** involved.
   - Whether you are on **ECC** or **S/4HANA** (some BADIs differ).
   - Any **country-specific** constraints (e.g., German SEPA, US ACH, Swiss SIC).

3. **Iterate:** After the first response, follow up with:
   - "Show me the exact FBZP screenshots / field-by-field configuration for Phase 1."
   - "Generate the LSMW/BDC recording steps for mass-updating the supplement field in vendor master."
   - "What SAP OSS notes are relevant for PMW format migration?"
   - "How would this differ if we use the Payment Method Supplement approach vs. BAdI approach — show both side by side."

4. **Validate with SAP documentation:** Cross-reference the LLM output with:
   - SAP Help Portal: [Payment Medium Workbench](https://help.sap.com)
   - SAP Note 1541877 (PMW format migration guidance)
   - SAP Note 986210 (Payment method supplements)
