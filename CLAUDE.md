# CLAUDE.md — WhatsUpNext (root)

Instructions for AI agents working anywhere in this project. Separate
git repositories are nested under this folder for tooling convenience —
`service/`, `app/`, `saas/` (private, not open-sourced). See `SPEC.md`
for the actual product architecture, data model, and API contract, and
`saas/SAAS.md` for SaaS-specific business/hosting internals.

## Working style

- **One step at a time, with explicit go-ahead.** Propose a single
  concrete step, wait for confirmation, then act — don't chain several
  steps together and execute autonomously, even when they seem
  individually reasonable.
- **Show the literal content before acting.** Before running a command or
  writing a file, show the exact command or the full file/test content
  in the message itself — not just a description of intent.
- **Show the actual raw output afterward**, not a paraphrased summary.

## Development practices

- **Strict TDD, across every component**: a failing test precedes any
  implementation code, no exceptions. See each component's own
  `CLAUDE.md` for its specific testing framework.
- **Prefer built-in/idiomatic generators and tooling** for scaffolding
  over hand-writing files from scratch, where the language/framework
  provides them.

## Documentation map

- `README.md` — public pitch. Update if the product's shape changes.
- `SPEC.md` — the technical spec (architecture, data model, API
  contract). **Update in the same step as any change that affects it**
  — a new/changed model, association, endpoint, or architectural
  decision. Don't let implementation drift ahead of the spec.
- `saas/SAAS.md` — SaaS business/hosting internals (private). Same rule:
  update alongside any SaaS-side decision, not after the fact.
- `service/CLAUDE.md`, `app/CLAUDE.md` — per-repo development practices
  specific to that component's tooling.
- This file — project-wide working style and practices.

## Keeping docs in sync

Documentation drift is a known failure mode on this project — `SPEC.md`
already went stale once (missing a whole model after it was added
mid-implementation) before being caught. Treat updating the relevant doc
as part of the change itself, not a follow-up step: if a step adds or
changes something `SPEC.md` describes, update `SPEC.md` before
considering that step done.
