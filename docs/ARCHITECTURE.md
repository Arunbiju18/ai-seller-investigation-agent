# AI Seller Investigation Agent — Architecture

## 1. Purpose

Investigate an online seller from a user-provided URL. Collect traceable
evidence, compare seller claims with independent sources, and produce a
report that distinguishes supported, contradicted, and unresolved claims.

The system must not guarantee that a seller is safe or label a seller
fraudulent without sufficient evidence.

## 2. Core workflow

1. Accept and validate the submitted URL.
2. Safely inspect relevant public website pages.
3. Extract seller claims and create an investigation checklist.
4. Compare the checklist with an expert-defined checklist before proceeding.
5. Collect evidence using explicitly allowlisted tools.
6. Record source URLs, excerpts, timestamps, and evidence identifiers.
7. Maintain claim-to-evidence relationships and investigate important
   unresolved questions.
8. Calculate risk indicators deterministically from documented rules.
9. Submit the proposed report through a mandatory verification gate.
10. Run deterministic validation, followed by a separate LLM judge.
11. Retry failed verification at most three times, then escalate or return
    an explicitly unverified report.
12. Display the findings, evidence, unknowns, and investigation trace.

## 3. Architecture components

- src/agent: bounded plan-act-observe investigation loop.
- src/tools: separate, allowlisted investigation tools.
- src/evidence: evidence ledger and claim-to-evidence relationships.
- src/verification: checklist validation and report verification gate.
- src/scoring: deterministic scoring and risk dimensions.
- src/storage: SQLite persistence and run recovery.
- tests: automated functional, regression, and security tests.
- tests/fixtures: controlled test cases.
- docs: architecture, decisions, and workshop compliance records.

## 4. Evidence and status rules

Claim status:
- SUPPORTED
- CONTRADICTED
- UNRESOLVED

Check status:
- UNKNOWN
- NOT_APPLICABLE
- NOT_CHECKED
- CONTRADICTED

Missing evidence is not proof of wrongdoing. A failed or skipped check
must never silently count as a clean result.

Weak signals such as SSL presence, shared hosting, registrar choice, and
domain privacy must not independently increase the risk score. Shared
hosting is excluded from the score. Domain age is context, not proof.

A missing web archive snapshot means historical evidence is unavailable;
it does not prove a seller claim false. Identity contradictions require
reliable, relevant evidence.

## 5. Scoring and reporting

Scoring is deterministic. The LLM must never calculate or change the score.

Show verification confidence and evidence coverage separately. Surface
critical contradictions explicitly. Every report finding must cite valid
evidence ledger identifiers and source information.

The report must distinguish findings from unknowns and checks not performed.

## 6. Verification gate

The report submission operation must always invoke the verification gate.
The agent cannot bypass it by choosing not to call a separate verifier.

Layer 1: deterministic checks for schema, completeness, valid evidence
references, source excerpts where required, permitted terminology, and
score/coverage consistency.

Layer 2: a separate read-only LLM judge receives only the proposed report
and evidence ledger. It checks whether the reasoning follows the evidence.

Failed checks provide repair feedback. After at most three failed attempts,
the report must be escalated or clearly marked unverified.

## 7. Agent execution and observability

Use a configurable OpenAI-compatible model client. Keep task-specific
information in the initial user message and general behavior rules in the
system prompt.

Set a bounded investigation budget of approximately 20–30 turns.
Record each turn's model input, proposed tool call, tool result, and outcome.
Provide a step-through mode and save run state to SQLite after every turn.

Reject and log unapproved tool calls.

## 8. Security requirements

- Validate URL schemes and destinations.
- Block loopback, private, link-local, and other prohibited IP destinations.
- Revalidate redirects and resolved addresses to mitigate SSRF.
- Treat webpage text as untrusted data, never as system instructions.
- Do not submit forms or perform purchases.
- Never bypass access controls or CAPTCHA protections.
- Keep credentials out of source control and logs.

## 9. Testing requirements

Test at minimum:
- An established legitimate seller.
- A legitimate new seller with insufficient evidence.
- A seller identity mismatch supported by reliable evidence.
- Mixed and contradictory evidence.
- Prompt injection embedded in webpage text.
- Redirects to private or internal IP addresses.
- Invalid report schemas and evidence references.
- Verification retries and escalation.

Keep development and held-out test cases separate. Report false positives
and false negatives honestly. A small demo dataset does not establish
real-world accuracy.

## 10. Scope boundaries

The first version is a URL-first investigation agent. Do not add a
browser extension, WhatsApp automation, custom graph database, or
unnecessary multi-agent debate before the core workflow works.

## 11. Attribution and transparency

Document which features were built during the hackathon and identify any
reused workshop code. Check the applicable license before reusing code.
Mention Muse Code in the README and demo materials.

## 12. Phase 0 decisions

- Runtime: Python 3.11+ with stdlib SQLite for the planned implementation.
- Models: Muse Spark 1.3 is the intended primary investigator model, pending
  endpoint verification. The reviewer model is independently configurable.
  Muse Spark Open, Muse Wonder, and Muse Glimmer are optional and disabled
  until capability, access, licensing, and cost checks pass. No paid model
  routing without explicit approval. One same-model retry on transient
  errors; no silent fallback. Both model IDs are recorded per run, and
  same-model review is disclosed as a separate step, not independent
  corroboration.
- Verification: 3 total attempts; after the third failure return an explicitly
  unverified result and persist a review-needed entry.
- Turn budget: default 25 investigation turns, hard maximum 30.
- MVP tool limits: 15-second fetch timeout, 1 MB response limit, 2,000-character
  excerpt limit, one tool call per turn.
- Evidence IDs: EV-0001 sequential per run. SQLite lives under data/, excluded
  from Git before any run writes to it.
- Checklist: versioned expert list EX-v1; the plan must cover at least 10 of 12
  items with at least 6 material checks; checked coverage requires 80 percent by
  integer arithmetic with minimum 8 applicable items and at least 6 material;
  below any gate normal submission blocks with missing IDs listed; shortfalls end
  explicitly unverified with a review entry; no silent override.
- Scoring: deterministic risk indicator, never set by the LLM; HIGH requires a
  bar-meeting critical contradiction on reliable independent evidence, from an
  authoritative source for the jurisdiction where registry-based, that actually
  disproves the specific claim; failed or missing lookups never disprove;
  weak signals score zero; missing evidence is never proof of safety or
  wrongdoing.
- Independence: sources sharing one underlying origin count as one source.
- Cost: no paid API calls, no package installation, no application code in
  Phase 0.

Full contracts: [CONTRACTS.md](CONTRACTS.md).

## 13. Contract amendment (v0.3)

- URL-type gate: unsupported, invalid, and unsafe URLs are refused with
  no score, report, or seller risk verdict. Social profiles stay refused
  unless an inspection method is explicitly allowlisted.
- Identity graph: brand, legal entity, trade name, domain, registration,
  and contact nodes with evidence-backed relationship edges, stored as
  relational records in SQLite (not a graph database, per section 10).
  Distinct names are never contradictions by themselves.
- Source-backed reporting: material findings carry a seller-claim versus
  independent basis; fabricated, missing, or invalid references fail
  verification; unavailable sources are never presented as verification.
- Loop and dimensions: the turn loop is documented as
  plan-act-observe-update within fixed budgets; identity, transaction,
  and fulfilment risk dimensions are reporting labels that never move
  the score. Scoring formula, bands, gates, retries, budgets, and
  security controls are unchanged.

Full text: [CONTRACTS.md](CONTRACTS.md) sections 1.9, 1.10, 4.8, 6.1,
and 7.0.
