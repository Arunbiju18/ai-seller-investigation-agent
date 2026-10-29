# AI Seller Investigation Agent — Phase 0 Contracts

Status: PROPOSED, version 0.2. Specification only; nothing described here
is implemented. A behavior counts as done only after it exists and its
tests pass.

Scope: record schemas, evidence rules, deterministic scoring, expert
checklist EX-v1, MVP tool contracts, verification semantics, budgets,
provider matrix, and acceptance criteria.

Conventions: MUST and NEVER are normative. SHOULD is recommended.
[Decision] marks an approved default. [Proposed] marks a value this
document asks you to approve by approving this file.

Decisions already approved: Python 3.11+ with stdlib SQLite for the
planned implementation; 3 total verification attempts then an explicitly
unverified result plus a persisted review-needed entry; default 25 turns,
hard maximum 30; fetch timeout 15 seconds; response limit 1 MB; excerpt
limit 2,000 characters; one tool call per turn; evidence IDs EV-0001
sequential per run; SQLite under data/ excluded from Git; no paid API
calls, no package installation, no application code in Phase 0.

## 1. Record schemas

All timestamps are UTC ISO-8601. All IDs are strings. Required means the
record is invalid without the field.

### 1.1 investigation_run

Purpose: one seller investigation from URL submission to final report.

- run_id, required. Unique per run. Format RUN-YYYYMMDD-NNNN.
- submitted_url, required. The exact submitted string.
- created_at, required.
- status, required. One of RUNNING, COMPLETE, UNVERIFIED,
  COVERAGE_GAP, ABORTED.
- config, required. Model IDs, expert_list_version, max_turns,
  budget and limit values in force for the run.
- budget_used, required. Integer turns consumed.
- coverage_pct, optional until checklist stage completes. Integer 0-100.
- score_inputs_ref, optional until scoring completes. Reference to the
  ledger snapshot the score was computed from.
- final_report_id, optional until submission completes.

### 1.2 evidence ledger entry

Purpose: one traceable captured fact with its source.

- evidence_id, required. EV-0001 sequential per run, zero-padded to at
  least 4 digits; beyond EV-9999 continue as EV-10000 with no truncation.
- run_id, required.
- source_url, required. Final URL after redirects.
- source_root, required. Publisher or operator plus data origin, used for
  the independence rule in section 2.
- origin_root, optional. The underlying origin when this entry cites,
  republishes, or derives from another source. Independence in section 2
  is evaluated on origin_root when present, otherwise on source_root.
  Example: a press article repeating a registry carries
  source_root of the press outlet and origin_root of the registry.
- retrieved_at, required.
- excerpt, required. At most 2,000 characters; longer text MUST be cut
  with truncated set true and the cut point recorded.
- truncated, required. True or false.
- reliability, required. One of OFFICIAL, ESTABLISHED_INDEPENDENT,
  SELLER_CONTROLLED, UNVERIFIED_SINGLE.
- relates_to, required, possibly empty. Checklist IDs this entry bears on.
- capture_method, required. Tool name that captured it.

### 1.3 claim

Purpose: one seller or third-party assertion with its resolved status.

- claim_id, required. CL-0001 sequential per run.
- text, required. Verbatim or near-verbatim statement.
- speaker, required. SELLER or INDEPENDENT.
- material, required. True if the finding can move the score or a
  checklist verdict.
- status, required. SUPPORTED, CONTRADICTED, or UNRESOLVED.
- supporting_evidence, required, possibly empty. Evidence IDs.
- refuting_evidence, required, possibly empty. Evidence IDs.
- reliability_note, optional. Required when refuting evidence exists but
  the claim stays UNRESOLVED for reliability reasons.

### 1.4 agent checklist item

Purpose: per-run working state for one expert item.

- expert_id, required. EX-NN from section 5.
- status, required. CHECKED, NOT_CHECKED, or NOT_APPLICABLE.
- na_reason, required when status is NOT_APPLICABLE. Free text
  justification; a bare flag is invalid.
- evidence_ids, required, possibly empty. CHECKED requires at least one.
- note, optional.

### 1.5 expert checklist item

Purpose: versioned project-defined investigation standard.

- expert_id, required. EX-NN, stable across versions once published.
- list_version, required. EX-v1 for this document.
- category, required.
- material, required. True or false.
- question, required. The check to perform.
- evidence_bar, required. What evidence satisfies the check.
- official_source_required, required. True or false.

### 1.6 report

Purpose: the submitted investigation output.

- report_id, required. RPT-RUN-YYYYMMDD-NNNN.
- run_id, required.
- verification_status, required. VERIFIED or UNVERIFIED.
- unverified_reason, required when UNVERIFIED. One of VERIFY_EXHAUSTED,
  COVERAGE_GAP, BUDGET_EXHAUSTED, SECURITY_BLOCK, MODEL_FAILURE.
- findings, required. Each finding carries claim IDs and evidence IDs;
  every cited ID MUST resolve to a stored record.
- unknowns, required, possibly empty.
- not_performed, required, possibly empty.
- na_items, required, possibly empty. Every NOT_APPLICABLE checklist ID
  with its justification, so unjustified exclusions are visible to the
  judge and reviewers.
- contradictions, required, possibly empty. Critical items flagged.
- score, coverage_pct, confidence, required. Computed per section 4.
- limitations, required, possibly empty. Includes unavailable sources.

### 1.7 trace turn

Purpose: one step-through record for replay.

- run_id and turn_no, required. turn_no starts at 1 with no gaps.
- model_input_ref, required. Reference to the stored prompt for the turn.
- proposed_tool, required. Tool name or NONE.
- tool_result_ref, required. Reference to the stored result.
- outcome, required. Short state delta for the turn.
- stored_at, required.

### 1.8 review-queue entry

Purpose: persisted human-review request.

- review_id, required. REV-YYYYMMDD-NNNN.
- run_id, required.
- reason_code, required. VERIFY_EXHAUSTED, COVERAGE_GAP,
  BUDGET_EXHAUSTED, SECURITY_BLOCK, or MODEL_FAILURE.
- summary, required. What failed and which IDs are affected.
- status, required. OPEN or RESOLVED.
- created_at, required.

## 2. Evidence reliability and independence

Tiers, highest first:

- OFFICIAL: government registries, courts, and other authoritative public
  records for the claimed jurisdiction.
- ESTABLISHED_INDEPENDENT: reputable publishers, established review or
  directory platforms with verification practices, and infrastructure
  records from their authoritative operators.
- SELLER_CONTROLLED: the seller site, seller social accounts, and seller
  supplied documents. Never independent of the seller.
- UNVERIFIED_SINGLE: anonymous posts, single unverified claims, and any
  source whose operator or method cannot be established.

Establishing independence, in order: (1) record the operator and the data
collection method in source_root; (2) follow the citation chain to the
underlying origin and record it in origin_root when it differs;
(3) two entries are independent only if their evaluated roots show
separate publishers or operators with separate data collection.

Consequences: a registry plus a press article that only repeats that
registry is ONE source. A registry plus a business directory compiled
from the directory's own survey is TWO sources. Two pages on one domain,
one corporate owner, one syndicated feed, or one wire story republished
is ONE source. Seller-controlled material is never independent of the
seller. Corroboration counts and the confidence HIGH rule use
independent sources only.

## 3. Status values and mapping rules

Claim status: SUPPORTED, CONTRADICTED, UNRESOLVED.
Check status: UNKNOWN, NOT_APPLICABLE, NOT_CHECKED, CONTRADICTED.

- A claim is CONTRADICTED only by evidence meeting the bar in section 4.
  A contradiction from weak or single unverified material leaves the
  claim UNRESOLVED with a reliability_note and scores zero.
- A failed, skipped, blocked, or timed-out check is NOT_CHECKED, never a
  clean or passing result.
- NOT_APPLICABLE requires a written na_reason and excludes the item from
  the coverage denominator. It MUST NOT be used for hard or failed checks.
- Unavailable official-source information is NOT_CHECKED on the check and
  UNKNOWN on any dependent claim. It is NEVER evidence that a business is
  unregistered or that a claim is false.
- Missing, unavailable, or unchecked evidence MUST NEVER be treated as
  proof of safety or of wrongdoing. It affects only coverage, confidence,
  and band gates.

## 4. Deterministic scoring

The score is a risk indicator computed from the ledger, not a probability
of fraud and not proof of safety. The LLM MUST NEVER set, adjust, or
recompute the score. Identical ledger inputs MUST produce identical
outputs.

### 4.1 Dimensions and points

- D1 identity contradiction: +75 when at least one material identity
  claim is CONTRADICTED by reliable independent evidence that actually
  disproves the specific claim as stated: same entity, same identifier,
  number, or date. Registry-based disproof additionally requires an
  authoritative source for the relevant jurisdiction: the official
  registry of the claimed jurisdiction, or, where none is public, no
  registry-based D1 is possible. Reliable means OFFICIAL, or at least
  two independent ESTABLISHED_INDEPENDENT sources agreeing. An
  unsuccessful search, a missing registry result, an unavailable source,
  a timeout, or a paywall MUST NEVER establish that a seller is
  unregistered or that a claim is false; the claim stays UNRESOLVED with
  a note and scores +0. Partial, ambiguous, or near-miss material
  (similar name, different number, different entity) likewise stays
  UNRESOLVED at +0.
- D2 other material contradictions: +10 per contradicted material
  non-identity claim, capped at +25.
- Raw score: the smaller of 100 and D1 + D2. Integer arithmetic only.
- Qualification split: deterministic checks verify D1 structure (claim
  status CONTRADICTED, refuting IDs resolve, tiers qualify, excerpts
  present). The judge verifies D1 substance (the evidence actually
  disproves the specific claim). A D1 claim failing either check fails
  the verification attempt with feedback instead of scoring HIGH.
- Weak, single-unverified, or non-qualifying contradiction material
  contributes +0, leaves the claim UNRESOLVED, and MUST NEVER
  independently trigger HIGH, alone or combined with other weak material.
  The 74 cap without a qualified D1 holds regardless.

### 4.2 Bands, floors, and caps

- LOW 0-24: allowed only when coverage is at least 60 percent, zero
  material claims are CONTRADICTED, and at least one identity claim is
  SUPPORTED by independent reliable evidence. Otherwise the floor is 25.
- UNCERTAIN 25-74: the default band for thin, mixed, or unverified
  evidence.
- HIGH 75-100: allowed only when a D1 critical contradiction meeting the
  section 4.1 bar is present. Otherwise the cap is 74.
- A weak-source or unsupported contradiction MUST NEVER trigger HIGH.

### 4.3 Zero and context-only signals

Score zero always: SSL presence or absence, shared hosting, registrar
choice, domain privacy status. Context notes only, score zero always:
domain age, missing archive snapshots. These appear in the report as
context and MUST NOT move D1, D2, bands, floors, or caps.

### 4.4 Coverage and confidence

- Coverage: checked applicable items divided by applicable items, times
  100. NOT_APPLICABLE items with justification leave the denominator.
  The 80 percent gate passes when checked times 100 is at least 80 times
  applicable, using integer arithmetic with no floats.
- Confidence is reported separately from the score. HIGH requires
  coverage of at least 80 percent, identity resolved as SUPPORTED or
  reliably CONTRADICTED, and material claims resting on at least 2
  independent sources. MEDIUM requires coverage of at least 50 percent.
  Otherwise LOW.

### 4.5 Worked examples

- A, established legitimate: coverage 11 of 12; identity SUPPORTED by an
  official registry record (origin: registry) plus an independent
  business directory entry compiled from the directory's own survey
  (origin: directory): TWO independent origins. A press article repeating
  the registry is stored with origin_root of the registry and does NOT
  count toward the two. Zero contradictions. D1 0, D2 0, raw 0. LOW
  gates pass. Result LOW 0, confidence HIGH.
- B, new thin seller: coverage 5 of 12; no identity support; zero
  contradictions. Normal submission is blocked by the coverage gate. Gap
  report scoring: raw 0, floor applies. Result UNCERTAIN 25, confidence
  LOW, status UNVERIFIED with reason COVERAGE_GAP.
- C, identity mismatch: an official registry record disproves the exact
  claimed registration number for the named entity, plus one
  contradicted material address claim. D1 75, D2 10, raw 85. HIGH bar
  met on substance and structure. Result HIGH 85 with the contradiction
  surfaced explicitly. This is a risk indicator, not a fraud verdict.
- D, weak-source contradiction only: one anonymous post disputes the
  seller name; no reliable source agrees. Claim stays UNRESOLVED with a
  reliability note. D1 0, D2 0. HIGH is unreachable. Result at most
  UNCERTAIN with confidence LOW.
- E, limited scope: 7 applicable items after 5 justified NAs, all 7
  checked (100 percent) with zero contradictions. Applicable 7 is below
  the section 5 minimum of 8, so normal submission blocks on scope, not
  percentage. Result: explicitly unverified coverage-gap report with a
  persisted review entry listing the scope shortfall.

### 4.6 Boundary cases

- Raw 24 with LOW gates failing becomes 25. Raw 20 without a D1 critical
  can never exceed 25 by construction, and the 74 cap holds regardless.
- Coverage exactly at the integer boundary passes: checked times 100
  equal to 80 times applicable is a pass.
- Applicable items below the section 5 minimum block normal submission
  regardless of percentage.
- Zero contradictions with no identity support floors at 25, never LOW.
- Official source unavailable keeps the item NOT_CHECKED in the
  denominator; the claim stays UNRESOLVED at +0.

### 4.7 Scoring test specifications

- T-SCORE-01 to 04: recompute examples A to D by hand; assert exact
  score, band, and confidence.
- T-SCORE-05: weak-source-only contradiction asserts UNRESOLVED, D1 0,
  band below HIGH.
- T-SCORE-06: floor case asserts 25, never LOW.
- T-SCORE-07: cap case with D2 at maximum and no D1 asserts raw score 25
  from D2 alone, the LOW-band floor holding at 25 with LOW disallowed,
  and final score 25 in band UNCERTAIN.
- T-SCORE-08: NA-heavy ledger asserts the denominator excludes justified
  NA items only, and the scope minimum still applies.
- T-SCORE-09: determinism; the same ledger scored twice asserts byte
  identical outputs.
- T-SCORE-10: LLM isolation; varied report prose over a fixed ledger
  asserts an unchanged score.
- T-SCORE-11: each weak and context-only signal asserts zero score
  movement in isolation and in combination.
- T-SCORE-12: near-miss disproof (similar name, different number)
  asserts UNRESOLVED and D1 0.
- T-SCORE-13: registry plus press-repeating-registry asserts ONE
  independent source; confidence HIGH MUST NOT rest on that pair alone.
- T-SCORE-14: unsuccessful registry search with no disproving
  record asserts claim UNRESOLVED, check NOT_CHECKED, D1 0.
- T-SCORE-15: disproof-shaped evidence from a source that is not
  authoritative for the claimed jurisdiction asserts UNRESOLVED, D1 0.

## 5. Expert checklist EX-v1 and coverage gates

Version EX-v1. IDs are stable. 12 items.

- EX-01, identity, material. Question: what identity does the seller
  claim. Bar: seller identity statements plus one independent reliable
  source. Official source not required.
- EX-02, registration, material. Question: is a legal entity or business
  registration verifiable. Bar: official registry record, or an explicit
  documented unavailability. Official source REQUIRED where the claimed
  jurisdiction publishes a registry.
- EX-03, contact, material. Question: are contact details present, and
  are they independently verified. Bar: presence recorded from seller
  property with a ledger entry, PLUS a separate verification state of
  VERIFIED or PRESENT_UNVERIFIED. Presence alone is never verification.
  MVP verification means a match against an independent directory or
  registry listing. Live calls or emails are deferred.
- EX-04, location, material. Question: does stated location match
  independent location evidence. Bar: seller claim plus an independent
  source; official or directory sources preferred but not required.
  Mismatch counts only on reliable evidence.
- EX-05, domain context, non-material. Question: what does registration
  context show. Bar: lookup record. Context only, scores zero.
- EX-06, history context, non-material. Question: does archive history
  exist. Bar: snapshot or explicit unavailable note. Context only.
- EX-07, returns policy, material. Question: is there a returns and
  refund policy. Bar: policy page excerpt or documented absence.
- EX-08, payments, material. Question: which payment methods and buyer
  protections apply. Bar: checkout or policy excerpt; irreversible
  methods noted as context.
- EX-09, reputation, material. Question: what do independent reputation
  signals say. Bar: at least one independent source with a reliability
  note; seller testimonials do not satisfy this item.
- EX-10, offer consistency, material. Question: are offers consistent
  with identity and policy claims. Bar: listing excerpts compared with
  EX-01, EX-07, and EX-08 evidence.
- EX-11, security context, non-material. Question: what are the basic
  connection facts. Bar: HTTPS and DNS notes. Context only, scores zero.
- EX-12, transparency, non-material. Question: are about and terms pages
  present. Bar: pages found or documented absent. Context only.

Material items: EX-01 to EX-04 and EX-07 to EX-10.

Official-source rule: only EX-02 REQUIRES an official source, and only
where the claimed jurisdiction publishes one. An unsuccessful search, a
missing result, or an unavailable, blocked, or paywalled source makes
the item NOT_CHECKED with reason OFFICIAL_SOURCE_UNAVAILABLE; the item
stays in the coverage denominator, the dependent claim stays UNRESOLVED,
and the report lists the limitation. Such an outcome MUST NEVER be
reported as evidence that a business is unregistered or that a claim is
disproved. Only an authoritative source for the relevant jurisdiction
returning evidence that specifically excludes the exact claimed
identifier can support a contradiction, subject to the section 4.1 bar.

Planned-coverage gate (before investigation starts): the agent plan MUST
include at least 10 of the 12 expert items AND at least 6 planned
material checks. Below either threshold the run does not start; the
response lists the missing item IDs and the missing material coverage.
Plan revisions consume turn budget.

Checked-coverage gate (at submission): coverage equals checked
applicable items over applicable items. Checked requires status CHECKED
with at least one linked ledger entry. The gate passes only when both
hold, using integer arithmetic: checked times 100 at least 80 times
applicable, AND applicable at least 8, AND applicable material at least
6. [Proposed] The 8 and 6 minima stop justified NAs from shrinking the
run to too few meaningful checks. Below the gate, normal submission is
BLOCKED and the response lists the missing IDs. There is no silent
override and no MVP override mechanism. When the gate cannot pass by the
budget limit, the run returns an explicitly unverified coverage-gap
result with a persisted review entry carrying the missing IDs and, when
applicable, the scope shortfall.

Gate test specifications:

- T-GATE-01: planned 10 of 12 with 6 material passes; planned 9
  of 12 blocks; planned 10 of 12 with 5 material blocks; every block
  lists missing IDs and missing material coverage.
- T-GATE-02: checked exactly at the integer boundary passes.
- T-GATE-03: applicable 7 with all checked blocks on scope and creates
  a review entry.
- T-GATE-04: applicable material 5 blocks on scope.
- T-GATE-05: below-gate submission lists every missing ID.
- T-GATE-06: no override path exists; below-gate submit returns BLOCKED
  and never a verified report.
- T-GATE-07: coverage gate still failing at budget exhaustion asserts
  normal submission stays BLOCKED, the final result is explicitly
  UNVERIFIED with reason COVERAGE_GAP, a review entry is persisted
  carrying the missing checklist IDs, and the result can never be marked
  VERIFIED.

## 6. Model registry and task routing

Roles: INVESTIGATOR (plan-act-observe turns) and REVIEWER (read-only
verification judge). The run config pins one model per role; mid-run
changes are forbidden. Every trace turn records model_id, latency_ms,
and input/output token counts when reported.

Registry statuses: INTENDED, CONFIGURED, OPTIONAL_DISABLED.

- muse-spark-1.3: INTENDED primary for INVESTIGATOR. [Decision]
  Endpoint behavior is NOT_TESTED; verification from section 10 is
  required before integration.
- REVIEWER model: independently configurable. Default when unconfigured
  is the INVESTIGATOR model. It MUST run tool-less on report plus ledger
  only. Structured output SHOULD be used; unparseable judge output fails
  the verification attempt with feedback JUDGE_UNPARSEABLE.
- muse-spark-open, muse-wonder, muse-glimmer: OPTIONAL_DISABLED.
  Enablement per model requires all four checks recorded: (1) capability,
  role-required features verified on the intended route; (2) access,
  endpoint or weights available to the project; (3) licensing, official
  license text reviewed with obligations recorded; (4) cost, rates
  confirmed and use explicitly approved. No paid model routing without
  explicit approval.

Routing policy: investigation turns go to the INVESTIGATOR model and
judge calls go to the REVIEWER model. No other LLM tasks exist in the
MVP. The routing table lives in the run config.

Same-model disclosure: using one model ID for both INVESTIGATOR and
REVIEWER provides a separate verification step (an isolated, tool-less
re-check of the report against the ledger) but NOT independent model
corroboration. The run config and the report MUST record both model IDs
and MUST disclose when they are identical. Per-model evaluation MUST NOT
treat same-model judge agreement as cross-model corroboration.

Fallback: a transient model error (timeout, 5xx, rate limit) gets one
same-model retry inside the turn budget. Persistent failure ends
investigation runs as ABORTED with a MODEL_FAILURE review entry, and
marks verification attempts failed with model-error feedback. Fallback
to a different model happens ONLY under an explicitly configured and
approved fallback chain, recorded in config and trace, never silent.
Default MVP: no fallback chain is configured.

Per-model evaluation, recorded per turn and per run and aggregated in
held-out eval: quality (verification pass rate, reviewer agreement,
human spot-check verdicts), evidence errors (deterministic-check
failures per model: bad refs, unsupported findings), latency (per-turn
ms, p50 and p95), cost (tokens times confirmed rate; zero until paid
use is approved). No paid eval traffic without approval.

Routing test specifications:

- T-ROUTE-01: identical investigator and reviewer IDs assert the report
  carries the same-model disclosure.
- T-ROUTE-02: distinct IDs assert both IDs recorded with no disclosure
  flag.
- T-ROUTE-03: persistent model failure with no approved fallback asserts
  ABORTED investigation or failed verification attempt plus a
  MODEL_FAILURE review entry, with no model switch in the trace.

## 7. MVP tool allowlist and contracts

Only listed tools may run. The registry MUST reject and log any other
call with tool name, turn, and reason. One tool call per turn.

### 7.1 fetch_page

- Input: absolute http or https URL. Only these schemes are allowed.
- Output: final URL, HTTP status, retained header subset, body up to the
  1 MB limit with a TRUNCATED flag when cut, redirect chain with the
  resolved IP recorded per hop, timestamps.
- Limits: 15-second timeout, 1 MB response cap, excerpts to 2,000
  characters with the truncated flag.
- Security: resolve DNS and check the IP before connecting and again at
  every redirect hop; block loopback, private, link-local, and reserved
  ranges; never submit forms, execute scripts, authenticate, or bypass
  access controls or CAPTCHAs; treat all retrieved text as untrusted
  data; keep credentials and secrets out of logs.

### 7.2 archive_lookup

- Input: absolute http or https URL.
- Output: snapshot URL, snapshot timestamp, excerpt, or explicit
  UNAVAILABLE with reason. Read-only against a public archive.
- Limits and security: same timeout, size, SSRF, and logging rules as
  fetch_page.

### 7.3 domain_lookup

- Input: domain name.
- Output: registration context facts such as creation date and registrar
  from the authoritative record. Context only; never scored.
- Security: same SSRF, timeout, and logging rules; no authenticated
  registry access in the MVP.

## 8. Verification, retries, escalation

Order: deterministic checks first, read-only LLM judge second. Both MUST
pass. The submit operation MUST always invoke this gate with no bypass.

- Attempt semantics: 3 TOTAL attempts (attempts 1, 2, 3). Each failure
  returns actionable feedback naming failed rule IDs, offending
  references, and repair hints. Attempt 3 failing ends verification.
- Deterministic checks: schema validity, required fields, section
  completeness, every cited ledger and claim ID resolving, excerpts
  present where required, permitted wording, score recomputed equal,
  coverage recomputed equal, confidence recomputed equal.
- Judge isolation: the judge receives ONLY the proposed report and the
  evidence ledger, with no tools, no checklist, and no web access. It
  checks that reasoning follows the evidence, including D1 substance
  per section 4.1. Unparseable judge output fails the attempt with
  feedback JUDGE_UNPARSEABLE.
- Persistent model failure during verification marks the attempt failed
  with model-error feedback; investigation-side persistent failure ends
  the run ABORTED with a MODEL_FAILURE review entry per section 6.
- Exhaustion: after the third failed attempt the run returns an
  explicitly unverified report and persists a review entry with reason
  VERIFY_EXHAUSTED.
- Unverified output MUST carry verification_status UNVERIFIED, the
  reason code, partial findings labeled as unverified, unknowns,
  limitations, and the verification failure summary.
- Coverage-gap path: section 5 blocking at the budget limit produces the
  same unverified shape with reason COVERAGE_GAP and the missing IDs.

## 9. Budgets and limits

- Investigation turns: default 25, hard maximum 30. [Decision]
- Tool calls: one per turn. [Decision]
- Fetch timeout: 15 seconds per request, no silent retry beyond one
  transient retry inside the same turn budget. [Decision]
- Response limit: 1 MB per response; reading stops at the cap with
  TRUNCATED true. [Decision]
- Excerpt limit: 2,000 characters per ledger entry with the truncated
  flag. [Decision]
- Trace and persistence: every turn stored before the next begins.

## 10. Provider compatibility matrix

No live compatibility is claimed. No paid call, login, or key was used.
Statuses: CONFIRMED (inspected official source), UNCONFIRMED
(third-party reports only), NOT_TESTED (no check run on our route).

Inspected evidence:

- E1: Google Cloud enterprise agent platform doc for meta/muse-spark-1.3
  at https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/partner-models/meta/muse-spark-1-3
  states a reasoning model for agentic workflows with built-in tool
  calling and MCP support, 1M-token context, model ID
  meta/muse-spark-1.3 in Preview, with Function calling, Structured
  output, and Reasoning listed as Supported in preview and Batch
  predictions not supported. This describes Google Cloud serving, not
  our configured endpoint.
- E2: OpenAI structured outputs guide at
  https://developers.openai.com/api/docs/guides/structured-outputs
  describes JSON Schema enforcement with strict mode.
- E3: OpenAI help center function calling article at
  https://help.openai.com/en/articles/8555517-function-calling-in-the-openai-api
  describes function calling, strict true argument enforcement, and JSON
  mode via response_format json_object with prerequisites.

Muse Spark 1.3:

- Existence and model ID: CONFIRMED via E1. Availability on our route:
  NOT_TESTED.
- Tool calling in principle: CONFIRMED via E1. On our endpoint:
  NOT_TESTED.
- Structured output in principle: CONFIRMED via E1. On our endpoint:
  NOT_TESTED.
- 1M context as documented property: CONFIRMED via E1. Effective limit
  on our route: NOT_TESTED.
- Chat Completions request shape and auth: UNCONFIRMED; third-party
  pages claim an OpenAI-compatible Meta Model API, but no official Meta
  API doc was inspected.
- Cost and retention: UNCONFIRMED and NOT_TESTED.

Muse Spark Open:

- Identity, weights, license, and serving routes: UNCONFIRMED;
  third-party reports only, no official doc inspected.
- Every capability on any route we might use: NOT_TESTED.
- License and attribution obligations MUST be verified against the
  official license text before any use or distribution.

Muse Wonder and Muse Glimmer:

- Identity, capabilities, access, license, and cost are UNCONFIRMED (no
  official doc inspected) and NOT_TESTED on any route. Both stay
  OPTIONAL_DISABLED per section 6. Research stays at free doc level
  unless you approve more.

Generic OpenAI-compatible client:

- Pattern reference (chat completions, tools, strict schemas, JSON
  mode): CONFIRMED via E2 and E3 as the pattern, not as our endpoint.
- Every capability against the actual configured endpoint: NOT_TESTED.

No-cost verification method per cell, in order: read the official
provider or platform doc; inspect the hackathon-provided endpoint base
URL and model list from configuration; build and schema-validate request
shapes offline with no network use; run a local mock round-trip. A live
probe requires your explicit approval and has not been run.

## 11. Phase 0 acceptance checklist

- Contracts reviewed and approved as the exact text in this file.
- Worked examples A to E recomputed by hand with matching results.
- EX-v1 frozen with stable IDs and the section 5 gate rules.
- Tool contracts and security boundaries stated for all three tools.
- Verification order, 3-attempt counting, judge isolation, and
  unverified output shape exact.
- Budgets set to the section 9 values.
- Every provider matrix cell honestly labeled; no live claim made.
- Only docs/CONTRACTS.md created, ARCHITECTURE appended, README linked;
  no other file touched; no code, packages, paid calls, or Git changes.
- Model registry, routing, fallback, and per-model eval recorded with
  no paid routing.
- Independence rule applied; example A uses two genuinely independent
  origins.
- D1 specificity bar with structure/substance split and failing tests.
- Coverage gates with integer rules, 8/6 minima, scope block, and gate
  tests.

## 12. Open items and deferred decisions

- data/ exclusion from Git: REQUIRED before any run writes, but the
  .gitignore edit is deferred because Git settings are out of scope for
  this change. Needs a later approved edit.
- Live provider probe: not run; needs explicit approval per turn.
- Numeric caps in section 9 are starting values; adjust only by a later
  approved doc change.
- Applicable minimum 8 and material minimum 6 are [Proposed]; approving
  this file approves them.
- Reviewer default (same-as-investigator unless configured) and the
  no-fallback-chain default are [Proposed].
- No MVP override exists for the coverage gates; any future explicit
  exception path needs a new approved change.

## 13. Changelog

- 0.2 PROPOSED: clarify T-SCORE-07 into raw, floor, and final asserts;
  add T-GATE-07 for budget-exhaustion coverage-gap end state. No formula,
  weight, threshold, or routing changes.
- 0.1 PROPOSED: initial Phase 0 contracts for review. Nothing implemented.
