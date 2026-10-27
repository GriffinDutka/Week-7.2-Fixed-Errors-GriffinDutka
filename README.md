# Brainstorm: AI Security Reviewer System

Working notes for designing a large-scale AI security reviewer that pulls from MCP servers, reports, reference tables, and APIs, and produces **repeatable, defensible** review output.

This is a brainstorm, not a spec. Each doc ends with open questions.

## The thesis

The system has three different kinds of material, and most of the design follows from keeping them apart:

| Layer | What it is | Who owns it | How it changes | Trust |
|---|---|---|---|---|
| **Knowledge** | Rules, severity model, rulings (precedent), taxonomy, org context | Humans (security team) | Pull requests, versioned snapshots | Authoritative |
| **Evidence** | The thing under review plus live data from MCP, APIs, and reports | External systems | Every run | Untrusted data, never instructions |
| **Judgment** | The LLM applying knowledge to evidence | The runtime | Pinned model and prompt versions | Must cite or it doesn't count |

**Repeatability comes from shrinking what the LLM decides.** The LLM classifies evidence against a closed checklist and fills in severity *factors*. Code computes severity, code picks which rules apply, code renders the report. The LLM never chooses from an open-ended space when a lookup table could choose for it.

**Preventing drift comes from making the agent unable to change its own knowledge.** The agent can *propose* new rulings into an inbox. Only a human-reviewed PR promotes a proposal into canonical knowledge, and every change is gated by an eval suite.

## Docs

1. [Architecture](docs/01-architecture.md): layers, data flow, and source trust levels
2. [File structure](docs/02-file-structure.md): the repo skeleton, and where severity, rulings, and context live
3. [Knowledge model](docs/03-knowledge-model.md): schemas, sizing (how big or small), and examples
4. [Indexing](docs/04-indexing.md): offline index build and how the running agent retrieves during a review
5. [Review runtime](docs/05-review-runtime.md): the agent loop that produces repeatable output
6. [Drift and evolution](docs/06-drift-and-evolution.md): building on context without rot
7. [Roadmap and open questions](docs/07-roadmap-and-open-questions.md)

## One-paragraph version

A **knowledge repo** holds atomic, ID'd rules (one per file), a severity matrix as data, and rulings as precedent case law. CI validates it against schemas and builds an immutable, content-hashed **snapshot** made of a manifest index, a lexical index, a vector index, and a graph of relationships. A review run pins a snapshot, a model, and a prompt version. It then **routes** the subject to facets, **deterministically selects** a rule pack, **plans** a checklist, **gathers evidence** per check through read-only MCP and API tools, **judges** each check with mandatory citations, **computes** severity in code, **verifies** the output, and writes a **run manifest** so the run can be replayed. A golden eval set gates every change to rules, prompts, or models.
