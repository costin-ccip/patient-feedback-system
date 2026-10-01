# Staff Procedure: Feedback Email Stop Requests

**Feature**: `006-feedback-email-opt-out` | **Date**: 2026-10-01 | **Status**: Draft, for Costin
and Liana to review before go-live. Covers spec FR-013, FR-014, FR-015.

Purpose: when someone asks us to stop sending feedback emails by any route other than the
link in the email, record it in CRM the same day if possible and no later than one business
day after we receive it.

## When someone asks to stop

Applies to: a phone call, a message at the front desk, or a reply to a feedback email.

1. Find the person's record in CRM: Patients if they are a patient, Leads if they are a
   prospect. If they are both, set it on both.
2. Check **Feedback Opt-Out**. Set **Feedback Opt-Out Source** to `Reply` if it came as an email
   reply, otherwise `Staff-recorded`. The date fills itself in.
3. If more than one record uses the same email address (for example a family member), set it
   on each record the person is on. Do not set it on someone who did not ask.
4. Reply to a reply, or tell a caller: "We've stopped the feedback emails. If you'd like them
   again, just let us know." Do not mention anything about their feedback or who their
   clinician is.
5. Do not tell the clinician. Opt-out status is not shown to contractors and does not need to
   be passed on.

That is all. Nothing else needs to be sent or recorded. Any feedback request already sent
stays valid until it expires on its own; the person simply gets no new ones.

## If someone wants them back

1. Confirm it is the person themselves asking.
2. Uncheck **Feedback Opt-Out**. The resubscribe date fills itself in.
3. Feedback requests skipped while they were opted out are not sent later. Only future ones
   are.

## Where replies arrive

Replies to a feedback email go to the mailbox the email was sent from (`info@capeclarity.com`).
Whoever reads that mailbox checks it at least once each business day for anything asking to
stop, or asking not to be contacted, and handles it as above. The system does not read
replies itself.

## What the system does for you

- Every feedback email already carries a stop link; most people will use it, and you never
  need to do anything for those.
- If a stop link is used and does not match anyone, Costin gets an alert with no personal
  details. Follow up only if someone has also contacted us.

## Do not

- Do not record an opt-out on a record unless the person (or the person's own message) asked.
- Do not copy an opt-out from a lead to a patient. A new patient is asked again at intake and
  that answer is the one that counts.
- Do not mention opt-out status in any note a contractor can read.
- Do not use this field for marketing email. Marketing unsubscribes are separate and handled
  in Zoho Campaigns; stopping one does not stop the other.
