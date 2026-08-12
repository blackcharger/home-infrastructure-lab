# ADR-0006: Treat AI-generated operations code as untrusted until reviewed

## Status

Accepted — 2026-08-12

## Context

A meaningful share of the automation in this lab was AI-generated and adapted rather than
written from scratch. That is not unusual any more, and treating it as something to hide would
be both dishonest and beside the point. It is how a lot of infrastructure code gets written now.

The problem is that generated operations code has a specific and dangerous property: **it
usually runs.** It is syntactically valid, it uses real flags, it produces plausible logs, and
it completes without error. Every signal a person normally uses to judge whether a script works
comes back positive.

That was tested here, unintentionally. The offsite backup script was generated, adapted, put
into production against irreplaceable data, and ran weekly for months. Clean logs. Objects
landing in storage. Plausible byte counts.

A line-by-line review found two defects that could each have destroyed the data the script
existed to protect:

1. **A failed database dump destroyed the previous good dump.** The dump was written straight
   to its final path, and shell redirection truncates the target *before* the command on the
   left runs. Any failure — unreachable host, stopped container, network blip — left an empty
   file, which the next sync then copied over the only offsite copy. There was no `set -e`, so
   nothing stopped.
2. **Deletions propagated offsite.** A sync with `--delete-during` and no object versioning
   makes the destination match the source, so anything removed locally was removed from the
   backup on the next run.

Neither defect is exotic. One is shell redirection ordering. The other is the documented default
behavior of a flag that was explicitly passed. Both are the kind of thing a reviewer catches
immediately and a passing run never will — and months of successful runs proved nothing, because
both faults only trigger when something *else* fails first.

## Decision

Generated code that touches production data is **untrusted input** until it has been reviewed
line by line, with specific attention to failure paths rather than the happy path.

The review asks, at minimum:

- What happens if each external call fails? Is the failure detected, or does execution continue?
- Are destructive operations ordered so a failure can destroy the thing being protected?
- Does every flag whose default behavior is destructive get checked against its documentation,
  not against what the name suggests?
- Is there a path where the script reports success while having done nothing, or the wrong thing?

Where a fix is applied, it is **verified by inducing the failure**, not by re-reading the code.

## Consequences

- Generated code costs review time. It is still faster than writing from scratch — the review is
  cheaper than the authorship — but "it ran, ship it" is not available.
- The failure paths get read first, which inverts the normal reading order and is where both
  defects in the example were found.
- Fixes require a way to induce the failure. Building that harness is work, and it is the step
  most likely to be skipped. In the backup case it meant pointing the dump at an unreachable
  address and confirming the run aborted with the previous dump intact — a two-minute test that
  is the only reason the fix is known to work rather than believed to.
- This applies with most force to anything that deletes, overwrites, or syncs. Read-only
  automation gets a lighter touch.

## Alternatives considered

**Don't use generated code for production paths.** Defensible, and increasingly impractical.
Rejected: it forgoes a real productivity gain to avoid a problem that review already solves,
and it is not a rule that would survive contact with schedule pressure.

**Rely on tests instead of review.** Good complement, insufficient alone. Both defects here
would have passed any test exercising the normal path, because both only manifest when an
external dependency fails first. Tests would have to be written *for the failure modes* — which
requires having identified them, which is the review.

**Rely on linters.** `shellcheck` is valuable and is now part of the workflow, but neither
defect was a lint error. The redirection was valid shell. The flag was correct usage. Static
analysis catches malformed code, not code that is well-formed and wrong.

**Say nothing about provenance.** Rejected. The audit is the demonstrable skill here, and it
does not depend on who typed the original. Overstating authorship would trade something real
for something unverifiable.

---

*The audit that produced this record, including the induced-failure verification, is in
[aws-backup-pipeline](https://github.com/blackcharger/aws-backup-pipeline/blob/main/docs/audit-2026-07.md).*
