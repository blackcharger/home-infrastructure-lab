# ADR-0005: Choose storage class by recovery objective, not by data size

## Status

Accepted

## Context

Offsite backup data falls into categories with very different cost profiles. The bulk media
library is large and rarely retrieved. Database dumps are small. Service configuration archives
are tiny.

The instinctive rule is *big and cold goes in the cheapest tier.* Deep-archive classes cost
roughly a twentieth of standard storage, so for a photo library the saving is real money.

That instinct produced a defect. The database dump was written into a subdirectory of the media
tree, so it inherited the media sync's deep-archive storage class. Nothing about that was
visible in configuration — it followed from a directory layout.

Deep-archive retrieval takes **12 or more hours**. So the recovery path had become: database is
lost → the dump exists, offsite, encrypted, exactly as designed → wait half a day before you can
begin restoring.

Nobody would agree to that recovery objective if asked. Nobody was asked. It was inherited from
where a file happened to live.

## Decision

Select storage class by **how quickly the data must be recoverable**, and make that selection
explicit per backup path rather than letting it follow directory structure.

- **Bulk media → deep archive.** Retrieval measured in hours is acceptable for a total-loss
  scenario, which is the only scenario in which it is retrieved.
- **Database dumps → infrequent-access.** Must be recoverable in minutes. Small enough that the
  price difference is immaterial.
- **Service configs → standard.** Small, and fetched during exactly the kind of incident where
  waiting is worst.

Each sync path names its storage class explicitly, and dumps are excluded from the media path
so they cannot inherit the wrong one again.

## Consequences

- Real recovery time for the database dropped from 12+ hours to a download and a restore.
- Cost impact is negligible, because the data that had to move is small. This was never a
  cost-versus-speed tradeoff — it was a defect wearing the costume of one.
- Backup paths must be defined by recovery objective rather than convenience. A file's location
  on disk is no longer allowed to determine its storage class.
- Deep-archive classes bill a **180-day minimum per object**. Anything deleted or superseded
  earlier is still charged for the remainder. This shapes retention and lifecycle policy:
  expiring noncurrent versions sooner than 180 days saves nothing and discards recoverable
  copies already paid for.

## Alternatives considered

**Everything in deep archive.** Cheapest, and the reason the defect existed. Rejected: it
applies a 12-hour retrieval to data whose entire purpose is to shorten an outage.

**Everything in standard.** Simple and fast. Rejected on cost — the media library dominates the
volume, and standard storage for it is roughly twenty times the price for retrieval speed that
scenario never needs.

**Intelligent-tiering across the board.** Automatic movement based on access patterns, no manual
classification. Genuinely attractive, and reconsidering it is reasonable. Rejected for now
because backup data has no meaningful access pattern to learn from — it is written and never
read until an emergency, so the automation would optimize on a signal that does not exist, and
it adds per-object monitoring charges across a large object count.

---

## The generalization

The mistake is worth naming because it is not specific to storage classes: **a default inherited
from structure is still a decision, it is just one nobody made.** The dump was in deep archive
because of where the file sat, and that held until someone asked what the recovery time actually
was. Recovery objectives should be stated, then implemented — not read back out of the
configuration afterward.
