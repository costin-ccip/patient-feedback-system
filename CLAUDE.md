# Working in this repo

This repo tracks the patient feedback system for Cape Clarity, built with GitHub
Spec Kit (see `.specify/memory/constitution.md` and `README.md`). It covers six
milestones (M0-M5); requirements for all of them live in one spec:
`specs/001-feedback-collection-pipeline/spec.md`.

## Milestone implementation notes

Each milestone that has been built also gets its own implementation-notes file,
e.g. `specs/001-feedback-collection-pipeline/m0-implementation-notes.md` for M0.
Unlike spec.md (requirements, all milestones), these files are per-milestone,
as-built documentation: exact component names/IDs, verbatim code and formulas,
data contracts, known gotchas and workarounds, and access constraints that shaped
the build.

**Convention: if you (human or AI) change how a milestone actually works —
a Zoho Flow function, a CRM field, an Analytics formula, a dashboard panel, a
data contract — update that milestone's implementation-notes file in the same
session, before moving on.** Treat it the same as updating a code comment next to
code you just changed: not a separate task to remember later, part of finishing
the change. If a milestone doesn't have a notes file yet and you're building it,
create one.

## Process note

The constitution's Rollout Workflow section requires new milestone/dashboard work
to go through specify -> plan -> tasks rather than being configured ad hoc
directly in Zoho Flow. M0 was built ad hoc, without plan.md/tasks.md, before this
convention file existed — see the "Process note" section at the end of
`m0-implementation-notes.md`. Future milestones should follow the full spec-kit
workflow where practical.

## Access constraints

Do not open the CRM's Leads or Patients modules in the browser without explicit
permission from Costin — use MCP tool calls (getRecords/createRecords/etc.) for
any CRM module that doesn't carry PII instead.


## Branch reconciliation (2026-09-06)

A separate session had already built this repo's real history on GitHub
(what is now `main`: constitution v1.2.1, a materially different spec.md —
richer Milestones table, Clinical Safety Flag Rules / User Story 4, stricter
zero-contractor-dashboard-access framing, Principle VI on testimonial/
marketing use). A later session (this one, in an earlier part of the same
conversation) instead initialized a fresh, disconnected local repo from
scratch — constitution v1.0.1, a differently-worded and less complete spec —
and built the M0 backfill (`m0-plan.md`/`m0-tasks.md`) and all of the M1
planning (`m1-plan.md`/`m1-research.md`/`m1-data-model.md`) against that
divergent, non-authoritative version, never knowing the real one existed on
GitHub until Costin asked for a staleness check before the first push.

Costin confirmed the GitHub branch is authoritative. The two histories share
no common commit, so reconciliation was: keep the GitHub history as `main`,
preserve the disconnected local work under the branch
`main-orphaned-v1-nonauthoritative` (not deleted, for audit purposes only —
not for reuse), and re-apply the M0/M1 planning documents as new commits on
top of the authoritative `main`, corrected wherever they conflicted with or
mis-cited the real constitution/spec. See the "Constitution Check" sections
of `m0-plan.md` and `m1-plan.md` for exactly what was corrected, and
`m1-data-model.md` / `m0-implementation-notes.md` for the `5A` dead-CRM-config
correction.

**Lesson for future sessions**: before doing any spec-kit work in a repo that
has a remote configured, run `git fetch` and diff local history against the
remote's actual branches before trusting what's in the local working
directory or assuming a local git init reflects what's already on GitHub.
