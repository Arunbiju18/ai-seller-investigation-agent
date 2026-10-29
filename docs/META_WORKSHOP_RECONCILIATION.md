# Meta Workshop Requirements Reconciliation

## Purpose

Cross-check the two Meta agent workshops against this project's design.
This document records requirements; it does not claim implementation.

## Agents 101: Agent loop

- [ ] Use a plan-act-observe agent loop.
- [ ] Ask Muse Code for a plan before generating application code.
- [ ] Keep the system prompt generic; submit the seller URL in the first user message.
- [ ] Maintain an investigation checklist and detailed evidence records.
- [ ] Keep individual tools in separate modules.
- [ ] Enforce a bounded turn budget, targeting 20-30 turns and never exceeding
      the configured hard limit.
- [ ] Provide step-through tracing of model input, selected tool, tool result,
      and resulting state.
- [ ] Persist state after every turn in SQLite.
- [ ] Support resuming an interrupted run and replaying a recorded demonstration.
- [ ] Use a configurable LLMClient for model switching where provider
      compatibility permits it.

## Agents 102: Verification

- [ ] Make verification mandatory inside submit_report.
- [ ] Run deterministic validation before the LLM judge.
- [ ] Validate report schema, required fields, evidence references, source
      excerpts where required, permitted wording, and score consistency.
- [ ] Give the read-only LLM judge only the report and evidence ledger.
- [ ] Return verification failures as actionable feedback to the agent.
- [ ] Allow no more than three failed verification attempts before escalation.
- [ ] Return an explicitly unverified result when verification does not pass.
- [ ] Compare the investigation checklist against an expert-defined checklist
      before investigation; use 80% as the initial workshop target, not proof
      of accuracy or seller safety.
- [ ] Reject and log unapproved tool calls.
- [ ] Include a human sign-off step for high-stakes output.
- [ ] Evaluate whether a separate read-only reviewer session is useful.

## Seller-investigation requirements added by our project

- [ ] Keep risk scoring deterministic; the LLM cannot directly set the score.
- [ ] Distinguish supported, contradicted, and unresolved claims.
- [ ] Treat missing evidence as unknown, not proof of wrongdoing.
- [ ] Separate evidence coverage from verification confidence.
- [ ] Apply SSRF, redirect, and private-network protections.
- [ ] Treat webpage instructions as untrusted input.
- [ ] Test legitimate sellers, insufficient evidence, identity mismatch,
      prompt injection, and unsafe URL redirects.
- [ ] Keep development tests separate from held-out evaluation cases.
- [ ] Document reused workshop code and licenses, and identify new hackathon work.

## Acceptance rule

A checkbox is complete only after the behavior exists and its tests pass.
Documentation alone is not implementation.

Provider compatibility and model/tool-calling capabilities must be verified
rather than assumed.
