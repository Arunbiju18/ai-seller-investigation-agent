# AI Seller Investigation Agent

An evidence-driven agent that investigates an online seller from a submitted
URL and produces an auditable report to help a buyer make a more informed
decision before paying.

## Project status

Early development. Architecture and workshop compliance documents have been
created. Features are not considered implemented until tested.

## Core principles

- Distinguish verified facts, contradictions, and unresolved claims.
- Link seller claims to traceable evidence.
- Treat missing evidence as unknown, not proof of wrongdoing.
- Do not label a seller fraudulent without sufficient supporting evidence.
- Keep risk scoring deterministic and separate from language-model judgments.
- Show evidence coverage separately from verification confidence.
- Protect against unsafe URLs, private-network access, and webpage prompt injection.
- Report limitations clearly when official verification is unavailable.

## Planned architecture

- `src/agent/` — bounded investigation and plan-act-observe loop
- `src/tools/` — allowlisted investigation tools
- `src/evidence/` — evidence collection and claim relationships
- `src/scoring/` — deterministic risk scoring
- `src/verification/` — report validation and verification gate
- `src/storage/` — SQLite persistence
- `tests/` — automated tests and fixtures
- `docs/` — architecture and workshop compliance documentation

## Model integration

The project will use a configurable OpenAI-compatible client where supported.
Actual provider compatibility, model availability, and tool-calling support
must be verified before implementation.

## Development and evaluation

Development and held-out test cases will remain separate. Tests will cover
legitimate sellers, insufficient evidence, identity mismatches, prompt
injection, and unsafe URL redirects.

A successful demonstration is not proof of real-world fraud-detection accuracy.

## Hackathon development record

New work, reused workshop material, and applicable licenses will be documented
as development progresses. Muse Code will be acknowledged in the README and
demo materials.

## Current documents

- [Architecture](docs/ARCHITECTURE.md)
- [Workshop compliance checklist](docs/WORKSHOP_CHECKLIST.md)
