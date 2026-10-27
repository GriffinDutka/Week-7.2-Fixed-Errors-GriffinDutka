# 02: File Structure

Proposed skeleton. Start as a monorepo. `knowledge/` can split into its own repo later if a different team owns it, which is common once a GRC or security-policy team gets involved.

```
ai-security-reviewer/
│
├── knowledge/                         # CANONICAL. Human-owned. Every change via PR + CODEOWNERS.
│   ├── taxonomy/
│   │   ├── categories.yaml            # authn, authz, crypto, data-handling, supply-chain, llm-specific…
│   │   ├── facets.yaml                # controlled vocab: asset_type, data_class, exposure, tech
│   │   └── glossary.yaml              # canonical terms + synonyms (feeds lexical index)
│   │
│   ├── severity/                      # ◀── SEVERITY LIVES HERE
│   │   ├── factors.yaml               # factor definitions + allowed values + guidance per value
│   │   ├── matrix.yaml                # factors → base severity (pure function, code-evaluated)
│   │   ├── modifiers.yaml             # data_class / exposure / compensating-control adjustments
│   │   └── sla.yaml                   # severity → remediation SLA, escalation path
│   │
│   ├── rules/                         # ◀── WHAT TO CHECK. One rule per file.
│   │   ├── authn/
│   │   │   ├── AUTHN-001-mfa-for-admin.md
│   │   │   └── AUTHN-002-session-timeout.md
│   │   ├── data/
│   │   ├── llm/
│   │   │   ├── LLM-001-prompt-injection-boundary.md
│   │   │   └── LLM-004-tool-scope-least-privilege.md
│   │   └── …
│   │
│   ├── rulings/                       # ◀── PRECEDENT ("case law"). One ruling per file. Append-only.
│   │   ├── 2026/
│   │   │   ├── RUL-2026-0041.md       # "Service accounts behind IAP: AUTHN-001 N/A"
│   │   │   └── RUL-2026-0042.md
│   │   └── _superseded/               # moved here, never deleted
│   │
│   ├── context/                       # ◀── ORG CONTEXT (static, slow-changing)
│   │   ├── environments.yaml          # prod/stage/dev definitions, network zones
│   │   ├── data-classes.yaml          # what "restricted" means here, with examples
│   │   ├── compensating-controls/     # org-wide controls that change outcomes (WAF, IAP, DLP…)
│   │   │   └── CC-003-identity-aware-proxy.md
│   │   └── risk-appetite.md           # short; the "spirit" doc the judge prompt may cite
│   │
│   ├── reference/                     # ◀── REFERENCE TABLES (queried, NOT embedded)
│   │   ├── approved-crypto.csv
│   │   ├── approved-vendors.csv
│   │   ├── control-framework-map.csv  # rule_id ↔ SOC2 / ISO27001 / NIST / OWASP LLM Top 10
│   │   └── _schemas/                  # one JSON schema per table
│   │
│   └── playbooks/                     # review type → rule packs + required evidence
│       ├── design-review.yaml
│       ├── vendor-review.yaml
│       ├── llm-feature-review.yaml
│       └── cloud-config-review.yaml
│
├── sources/                           # EVIDENCE CONNECTORS (config, not knowledge)
│   ├── registry.yaml                  # every source: id, kind (mcp|api|file|warehouse), trust level, TTL
│   ├── mcp/                           # MCP server configs + tool allowlists (read-only)
│   └── adapters/                      # code: normalize responses → EvidenceRecord
│
├── schemas/                           # THE CONTRACTS. Everything validates against these.
│   ├── rule.schema.json
│   ├── ruling.schema.json
│   ├── playbook.schema.json
│   ├── evidence-record.schema.json
│   ├── finding.schema.json
│   ├── report.schema.json
│   └── run-manifest.schema.json
│
├── prompts/                           # VERSIONED PROMPTS. Changes are eval-gated like code.
│   ├── router/v3.md
│   ├── checker/v7.md
│   ├── explorer/v2.md
│   └── proposer/v1.md
│
├── indexer/                           # builds snapshots from knowledge/
│   ├── build.py
│   ├── chunkers/                      # per-type: rule, ruling, report, table
│   └── validators/                    # schema, link-integrity, contradiction detection
│
├── runtime/                           # THE REVIEW AGENT
│   ├── router/
│   ├── planner/
│   ├── checker/
│   ├── severity_engine/
│   ├── explorer/
│   ├── verifier/
│   ├── reporter/
│   │   └── templates/
│   └── proposer/
│
├── evals/                             # DRIFT DEFENSE
│   ├── golden/                        # case = subject fixture + frozen evidence + expected findings
│   │   └── case-0001-public-s3-pii/
│   │       ├── subject.yaml
│   │       ├── evidence/              # recorded tool responses (replayable, no live calls)
│   │       └── expected.yaml          # required findings, forbidden findings, severity bounds
│   ├── adversarial/                   # prompt-injection fixtures, misleading docs
│   ├── metrics.py                     # recall, precision, severity agreement, run-to-run stability
│   └── baselines/                     # last accepted scores per snapshot+model+prompt
│
├── inbox/                             # AGENT-PROPOSED, NON-CANONICAL. Triage → PR into knowledge/.
│   ├── candidate-rulings/
│   └── candidate-rules/
│
├── docs/                              # ADRs, onboarding, this brainstorm
│   └── adr/
│
└── (artifact store, not git)
    ├── snapshots/<sha>/               # built indexes + manifest
    └── runs/<run_id>/                 # manifest, evidence cache, findings, report, traces
```

## Where things live, quickly

| Question | Answer |
|---|---|
| Where does **severity** live? | `knowledge/severity/`, as *data* evaluated by `runtime/severity_engine`. Rules carry a `default_factors` hint. Rulings can carry an `override`. The LLM only proposes factor values. |
| Where do **rulings** live? | `knowledge/rulings/YYYY/`, one per file, append-only. Superseding creates a new ruling and moves the old one to `_superseded/` with a back-link. |
| Where does **context** live? | Two places. *Static* org context in `knowledge/context/`, which is versioned and slow. *Live* context about the subject comes from `sources/` at run time and gets cached per run in `runs/<id>/evidence/`. Never mix them. |
| Where do **reference tables** live? | `knowledge/reference/*.csv` with schemas. For tables that are big or live in a warehouse, put only the *pointer* and schema here and query through an adapter. |
| Where does the agent **write**? | Only `runs/<id>/` and `inbox/`. Never `knowledge/`. |

## Why one file per rule and one file per ruling

- **Stable IDs = stable citations.** A finding cites `AUTHN-001@a1b2c3`, the rule ID plus the content hash at snapshot time.
- **Clean diffs and blame.** You can see exactly who changed what, and why, in a PR.
- **Natural chunk boundary.** The retrieval unit equals the authoring unit, so there's no "half a rule" in the context window.
- **Ownership.** CODEOWNERS per folder: `rules/llm/` → AI security team, `severity/` → security leadership.

## Open questions

- Monorepo vs. a separate `knowledge` repo: who approves rule changes, and are they the same people who ship runtime code?
- Markdown with YAML frontmatter, or pure YAML, for rules? Markdown is friendlier to author and gives better retrieval text. YAML is stricter. The recommendation is in doc 03.
- Do you already have a control framework (SOC 2, ISO, NIST 800-53, OWASP LLM Top 10) to map rules to? That decides how `control-framework-map.csv` looks from day one.
