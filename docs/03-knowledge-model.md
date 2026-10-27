# 03: Knowledge Model: Severity, Rules, Rulings, Context

## Format recommendation

**Markdown with strict YAML frontmatter**, validated by JSON Schema in CI.

- The **frontmatter** is the machine part: IDs, facets, applicability, factors, links. Indexing and filtering run on it.
- The **body** is the human and LLM part: intent, how to check, examples, edge cases. This is what gets embedded and shown to the checker.

## Rule

```markdown
---
id: LLM-004
title: LLM tool access follows least privilege
version: 3
status: active                     # draft | active | deprecated
owner: ai-security
category: llm
applies_when:                      # deterministic selection: ALL must match
  asset_type: [llm_application, agent]
  has_capability: [tool_use]
not_applicable_when:
  - deployment: [offline_batch]
evidence_required:                 # tells the checker WHAT to fetch (maps to sources/registry.yaml)
  - kind: tool_manifest
  - kind: iam_policy
    for: agent_service_account
default_factors:                   # starting hint for severity, NOT the severity
  impact: high
  exploitability: medium
controls: [OWASP-LLM06, NIST-AC-6]
related_rules: [LLM-001, AUTHZ-003]
effective: 2026-03-01
review_by: 2027-03-01              # forces periodic human re-review (drift defense)
---

## Intent
An agent's tools should not be able to do more than the feature requires...

## How to check
1. Enumerate the tools the agent can call.
2. For each write-capable tool, confirm ...

## Pass looks like
...

## Fail looks like
...

## Known edge cases
- Read-only tools that return secrets count as write-equivalent → see RUL-2026-0017.
```

**Sizing:** target **200 to 800 tokens** for the body. If it's bigger, the rule is really two rules. If it's smaller than about 100, it probably lacks the "how to check" detail the checker needs. A rule should be **one checkable assertion**. If you can't express it as a single pass/fail, split it.

## Severity

Severity is computed. The LLM never emits it directly.

### `factors.yaml`
```yaml
impact:
  values: [low, medium, high, critical]
  guidance:
    critical: "Org-wide compromise, regulated data at scale, safety impact"
    high:     "Single system compromise, restricted data exposure"
    ...
exploitability:
  values: [theoretical, low, medium, high]
  guidance:
    high: "Unauthenticated, network-reachable, public technique"
    ...
exposure:      # often derived from T0 sources rather than the LLM
  values: [internal, partner, internet]
data_class:    # derived from the data classification registry
  values: [public, internal, confidential, restricted]
```

### `matrix.yaml`
```yaml
# base = f(impact, exploitability). Pure lookup, code-evaluated.
base:
  critical: { high: critical, medium: critical, low: high,   theoretical: medium }
  high:     { high: critical, medium: high,     low: medium, theoretical: low }
  medium:   { high: high,     medium: medium,   low: low,    theoretical: low }
  low:      { high: medium,   medium: low,      low: low,    theoretical: info }
```

### `modifiers.yaml`
```yaml
- when: { exposure: internet }
  adjust: +1
- when: { data_class: restricted }
  adjust: +1
  floor: high
- when: { compensating_control: CC-003 }   # e.g. identity-aware proxy
  adjust: -1
  requires_evidence_trust: T1               # can't take this discount on a T3 claim
cap: critical
```

The checker outputs `factors: {impact: high, exploitability: medium, …}`, and each factor comes with a **one-line justification plus an evidence citation**. The engine computes `base → modifiers → precedent override → final`, and the full computation trail goes into the finding. Two runs that agree on factors *must* agree on severity, which is what keeps the output reproducible.

Where possible, **derive factors from T0 data instead of asking the LLM**. Exposure comes from network config. Data class comes from the registry. Every factor you take away from the LLM removes a source of variance.

## Ruling (precedent)

Rulings answer: *"We've seen this exact situation before. What did we decide?"* They are the main mechanism for building on context.

```markdown
---
id: RUL-2026-0041
title: Internal service accounts behind IAP satisfy AUTHN-001
status: active                     # active | superseded | expired
rules: [AUTHN-001]
decision: not_applicable           # pass | fail | not_applicable | severity_override
severity_override: null            # or e.g. { to: low, reason: ... }
fact_pattern:                      # structured: used for deterministic matching
  asset_type: internal_service
  auth_mechanism: service_account
  compensating_controls: [CC-003]
fact_pattern_text: >               # prose: used for semantic matching
  Machine-to-machine service account with no human login,
  reachable only through the identity-aware proxy...
decided_by: [jdoe, security-arch-board]
decided_on: 2026-05-14
source_review: REV-2026-0882       # the review where this came up
expires: 2027-05-14                # precedent has a shelf life
supersedes: null
---

## Rationale
...

## Boundaries
Does NOT apply if the service account has interactive login enabled...
```

**Sizing:** 150 to 500 tokens. The `fact_pattern` (structured) is what makes precedent matching repeatable. Prose similarity alone will match "sort of similar" cases inconsistently.

**Precedence order** when they conflict, from most to least specific:
`ruling (exact fact-pattern match) > ruling (semantic match, human-confirmed) > rule > severity defaults`

A semantically matched ruling that lacks an exact structured match should be **surfaced, not applied**. The checker sees it as a "related precedent" and can cite it, but it can't flip an outcome by itself.

## Context

| Kind | Example | Where | Size guidance |
|---|---|---|---|
| Static org context | Environment definitions, data class definitions, risk appetite | `knowledge/context/` | Small, under 1k tokens per file. Some of it is always in the prompt. |
| Compensating controls | IAP, WAF, DLP, EDR coverage | `knowledge/context/compensating-controls/` | One per file, ID'd like rules. Rulings and modifiers reference them. |
| Reference tables | Approved crypto, vendors, framework map | `knowledge/reference/` | **Never embed.** Expose them as a lookup tool, e.g. `lookup_reference(table, key)`. |
| Live subject context | This app's IAM policy, scanner results, design doc | Evidence plane, cached per run | Chunk per doc 04 |

## Schemas to define first (the contracts)

1. `rule.schema.json`
2. `ruling.schema.json`
3. `finding.schema.json`, sketched below
4. `run-manifest.schema.json`

```yaml
# finding
id: F-<run_id>-007
rule: { id: LLM-004, hash: a1b2c3 }
outcome: fail                         # pass | fail | not_applicable | needs_human
factors:
  impact:         { value: high,   why: "...", evidence: [E-12] }
  exploitability: { value: medium, why: "...", evidence: [E-12, E-15] }
severity:
  base: high
  modifiers_applied: [{ rule: "exposure=internet", adjust: +1 }]
  precedent: null
  final: critical
evidence: [E-12, E-15]                # → EvidenceRecord IDs (source, locator, hash, trust)
precedents_considered: [RUL-2026-0017]
rationale: "..."                      # short, templated
confidence: 0.82                      # from self-consistency, see doc 05
```

## Open questions

- Do you have existing severity definitions, such as a CVSS-like scheme or an internal P0 to P4 scale, that need mapping instead of replacing?
- How many rules do you expect at maturity: 50, 500, or 5,000? That changes whether rule selection can stay purely structured or needs a semantic pre-filter.
- Who is allowed to issue a ruling? One person, or a board? That decides the `decided_by` policy.
