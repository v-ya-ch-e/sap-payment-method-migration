---
name: SAP Payment Method Migration
overview: Analyze the SAP Payment Method format migration challenge, create a CLAUDE.md knowledge file, and craft a detailed LLM prompt that will produce a comprehensive phased migration strategy for switching payment formats in production without disruption.
todos:
  - id: claude-md
    content: Create CLAUDE.md with SAP Payment Method architecture, format assignment mechanics, migration levers (Supplements, House Bank overrides, BADIs, PMW), risk patterns, and key transactions/tables
    status: completed
  - id: prompt-file
    content: Create prompts/sap-payment-method-migration.md with a detailed structured prompt requesting phased migration strategy with rollback, concrete config steps, and pros/cons
    status: completed
isProject: false
---

# SAP Payment Method Format Migration — Prompt Engineering

## Context

The user manages an SAP FI/AP landscape where each company code has ~10 Payment Methods configured (via `FBZP`). They need to migrate from an old payment format (e.g., classic RFFO* programs or older PMW formats) to a new format (e.g., CGI XML, ISO 20022 pain.xxx, or a custom PMW format). The core dilemma:

- **Duplicate Payment Methods** — safe but unsustainable (single-char codes exhaust quickly, doubles maintenance burden)
- **In-place format swap** — dangerous, big-bang, no phased rollback
- **Need** — a controlled, phased, reversible production migration strategy

## Deliverables

### 1. CLAUDE.md — SAP Payment Method Migration Knowledge Base

A reference file containing all critical SAP domain knowledge an LLM needs to produce a high-quality answer:

- **SAP Payment Method architecture**: `FBZP` structure (country-level, company-code-level, Payment Method Supplements), single-char limitation
- **Format assignment points**: where the format program / PMW format variant is configured and how it's resolved at runtime
- **Key migration levers in standard SAP**:
  - Payment Method Supplements (Zahlwegerganzung) — sub-variants within a single Payment Method code
  - House Bank / Account ID level overrides
  - PMW format variants and selection logic
  - F110 payment run parameters (test vs. productive, variant management)
  - BADIs: `SAMPLE_PROCESS_00001820`, `FI_PAYMEDIUM`, user-exits in RFFOUS_T / similar
  - Vendor master / payment method assignment per vendor
- **Risk mitigation patterns**: test runs, parallel runs, reconciliation, rollback procedures
- **SAP best practices for format migration** from OSS notes and documentation

### 2. LLM Prompt — Expert SAP Migration Strategy

A structured, detailed prompt that:

- Sets the SAP expert persona and context
- Describes the exact scenario (10 Payment Methods, multi-company, format migration)
- Lists the constraints (character limit, no big-bang, production safety)
- Requests a specific deliverable: phased migration plan with rollback capability
- Asks for concrete FBZP / PMW configuration steps, not just theory
- Requests pros/cons comparison of approaches

## File Structure

```
papa-prompts/
  CLAUDE.md              -- SAP domain knowledge for LLM context
  prompts/
    sap-payment-method-migration.md  -- The crafted prompt
```

## Implementation Steps

1. Create `CLAUDE.md` with SAP Payment Method domain knowledge, migration levers, key transactions, BADIs, and constraints
2. Create `prompts/sap-payment-method-migration.md` with a structured expert prompt that leverages the CLAUDE.md context to elicit a detailed phased migration strategy

