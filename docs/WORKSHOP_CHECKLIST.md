# Meta Workshop Compliance Checklist

Purpose: track implementation and testing of all 12 required workshop patterns.
Do not mark an item complete until its behavior has been implemented and tested.

## Priority 1 — First six patterns

- [ ] 1. Mandatory verification gate inside report submission.
      The submit_report operation always invokes verification.

- [ ] 2. Deterministic checks first.
      Validate report schema, completeness, evidence ledger references,
      permitted wording, and score/coverage consistency.

- [ ] 3. Separate LLM judge second.
      A read-only judge receives only the report and evidence ledger.

- [ ] 4. Retry at most three times, then escalate.
      Verification feedback returns to the agent; repeated failure produces
      an explicitly unverified result or human escalation.

- [ ] 5. Verify the investigation checklist before work.
      Compare the agent checklist with an expert-defined checklist.
      Initial target: at least 80% coverage.

- [ ] 6. Step-through trace console.
      Show each turn's model input, selected tool, tool result, and outcome.

## Remaining six workshop patterns

- [ ] 7. Checklist and evidence records.
      Maintain unresolved questions and a traceable evidence ledger.

- [ ] 8. Plan-act-observe loop with a turn budget.
      Use a bounded investigation loop, targeting approximately 20–30 turns.

- [ ] 9. Allowlisted tools only.
      Reject and log any unapproved tool call.

- [ ] 10. Save state every turn.
       Persist investigation state in SQLite so runs can resume.

- [ ] 11. Generic system prompt; task in the first user message.
       Keep behavior instructions separate from the submitted seller URL.

- [ ] 12. Plan first with Muse Code; configurable OpenAI-compatible client.
       Request and review a plan before implementation. Support model
       switching through configuration where the provider permits it.

## Project-specific additions

- [ ] Deterministic risk scoring; the LLM cannot change the score.
- [ ] Claim-to-evidence relationships and explicit status values.
- [ ] Verification confidence shown separately from evidence coverage.
- [ ] Test legitimate, new/uncertain, and identity-mismatch seller cases.
- [ ] Test prompt injection and private-IP redirect protection.
- [ ] Keep development and held-out test cases separate.
- [ ] Document new hackathon work and identify reused code and licenses.
- [ ] Mention Muse Code in the README and demo materials.

## Status policy

These checkboxes measure implementation and test completion.
Creating this checklist does not mean any requirement is implemented.
