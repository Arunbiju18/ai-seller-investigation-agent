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
