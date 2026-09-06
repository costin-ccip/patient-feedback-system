# Data Retention: 24-Hour Raw Forms Purge — Investigation & Status

**Date**: 2026-09-06

**Applies to**: All milestones (M0, M1, and future) that collect responses via
a public Zoho Form written back to `Milestone_Instances`. Referenced from
`m0-plan.md` and `m1-plan.md`'s Constitution Check sections rather than
duplicated there.

## Requirement

The constitution's Compliance & Data Handling Requirements section requires
raw Zoho Forms entries to be purged "on a short, defined retention window"
once copied into CRM — the constitution itself does not fix that window at
24 hours; its own Governance TODOs mark the exact window as still open. 24
hours has been the working target in practice (Costin, 2026-09-06) and is a
reasonable candidate, but is not yet a ratified constitutional value. Either
way, no purge mechanism has ever been implemented (discovered as a gap during
the M0 backfill, see `m0-plan.md`).

## Investigation (2026-09-06)

Explored a generic, once-built fix (a purge step added to every milestone's
write-back function) before implementing anything. Two blockers found:

1. **No documented delete-entry API for Zoho Forms.** Zoho Creator (a
   different Zoho product) exposes a REST API for deleting records; standalone
   Zoho Forms — which is what M0 uses and M1 will use — does not appear to
   expose an equivalent. A Deluge `invokeurl` call to hard-delete a submitted
   entry isn't something we could build against a documented, supported
   endpoint.
2. **The native alternative doesn't fit either.** Zoho Forms has a built-in
   "Auto-Trash Form Submissions" setting (Settings → Submissions & Storage →
   Auto-Trash) that can auto-trash entries after a configurable number of
   days. But: (a) it requires a Premium or Zoho One Enterprise plan, and the
   Zoho Forms account in use (liana.preudhomme@capeclarity.com) is currently
   on the **Free** plan, so it's unavailable as-is; (b) even if upgraded,
   auto-trashed entries stay recoverable in trash for 5 additional days before
   permanent deletion, so actual purge time would be (configured interval) + 5
   days — not a clean fit for the working 24-hour target, even though the
   constitution's own wording ("a short, defined retention window") doesn't
   literally lock in that number.

## Decision (Costin, 2026-09-06)

Log this as a known limitation and revisit later, rather than block M0/M1 on
it. Options on the table when we do revisit:

1. **Upgrade the Zoho Forms subscription to Premium**, enable Auto-Trash, and
   explicitly document the resulting N+5-day purge window as an accepted
   divergence from the 24-hour working target (would need Costin's sign-off
   on that divergence, and on formally setting the constitution's still-open
   retention-window TODO to whatever number is actually chosen).
2. **Manual periodic deletion** by Costin (or whoever holds the standing
   PII-access relationship) from the Zoho Forms UI — a documented manual
   control, similar in spirit to M0's documented manual-trigger fallback.
3. **Further research** — check Zoho Forms' Control Panel/API section directly
   or contact Zoho support for an undocumented or admin-tier deletion
   endpoint before concluding automation isn't possible at all.

No option has been chosen yet. This remains an open go-live blocker alongside
the still-open `TODO(BAA_SCHEDULE)` (Principle V) — both should be resolved
before any milestone handles real (non-test) patient data.
