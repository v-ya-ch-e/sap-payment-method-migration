# Papa Prompts

A collection of expert-level prompts paired with domain context files, designed to extract high-quality, actionable responses from LLMs on complex enterprise topics.

## Project Structure

```
├── CLAUDE.md          # Domain knowledge context (fed to the LLM alongside the prompt)
├── prompts/           # Curated prompts
│   ├── sap-payment-method-migration.md
│   └── RU_sap-payment-method-migration.md       # Russian translation
└── responses/         # Generated responses
    ├── sap-payment-method-migration-response.md
    └── RU_sap-payment-method-migration-response.md  # Russian translation
```

## How It Works

Each topic follows a two-part pattern:

1. **Context file (`CLAUDE.md`)** — structured domain knowledge that grounds the LLM so it reasons with accurate terminology, table names, transaction codes, and architectural constraints instead of hallucinating them.
2. **Prompt (`prompts/`)** — a detailed, role-based prompt that defines the situation, constraints, and expected output structure.
3. **Response (`responses/`)** — the resulting LLM output, kept as a reference artifact.

## Current Topics

### SAP Payment Method Format Migration

Covers phased migration of payment formats (e.g., legacy RFFO\* → ISO 20022 pain.001) across multiple company codes in a production SAP ECC/S/4HANA environment.

- **Context:** `CLAUDE.md` — FBZP configuration, Payment Method Supplements, PMW format resolution, key tables (`T042*`, `REGUH`/`REGUP`), BAdIs, and risk mitigation patterns.
- **Prompt:** `prompts/sap-payment-method-migration.md` — asks for approach evaluation, phased timeline, comparison table, risk analysis, and a ready-to-use checklist.
- **Response:** `responses/sap-payment-method-migration-response.md` — full migration strategy recommending Payment Method Supplements as the primary lever, with a 12–16 week phased rollout plan.
- **Russian translations:** `prompts/RU_sap-payment-method-migration.md` and `responses/RU_sap-payment-method-migration-response.md`.

## Usage

1. Copy the contents of `CLAUDE.md` and paste it into your LLM conversation as grounding context.
2. Follow with the prompt from `prompts/`.
3. Optionally customize the variables called out in the prompt's "How to Use" section (old/new format names, number of company codes, ECC vs. S/4HANA, country-specific constraints).
4. Compare the LLM output against the reference in `responses/` or use it as a starting point for your own engagement.
