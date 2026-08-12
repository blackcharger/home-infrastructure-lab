# ADR-0004: Encrypt offsite backups client-side, before upload

## Status

Accepted

## Context

Tier 3 of the backup strategy sends data offsite to object storage. That data includes a
personal photo library and database dumps — the irreplaceable category, the reason the backup
exists at all.

Cloud providers offer server-side encryption, and it is genuinely useful: it protects data at
rest on their disks and it is one checkbox. But under server-side encryption the provider
receives plaintext, performs the encryption, and holds the keys. The threat model it covers is
"someone steals a disk from the datacenter." It does not cover a compromise of the storage
account itself, and it requires trusting the provider's handling of both the data and the keys.

The question is which side of the network boundary encryption happens on.

## Decision

Encrypt **on the storage host, before anything is transmitted.** The backup job writes through
an `rclone` `crypt` remote layered over the object-storage remote. Object contents and object
names are both encrypted locally; only ciphertext crosses the network.

The provider never receives plaintext and never holds a decryption key.

## Consequences

- A compromise of the storage account yields ciphertext. Credentials for that account are the
  most exposed part of the system — they sit on an internet-connected host in a config file —
  and this decision is what makes their theft survivable.
- Filenames are obscured too. A directory listing does not reveal what is stored, which matters
  more than it first appears: object keys leak structure, dates, and subject matter.
- Deduplication and server-side features that require readable content are unavailable. Not
  used here, so not a real cost.
- **The decision relocates the risk rather than removing it.** The encryption key now lives on
  the machine being backed up. If that host is lost without an off-box copy of the key, the
  offsite archive is permanently undecryptable — a backup that survives the fire and cannot be
  read. Key custody is therefore not a footnote to this decision, it *is* the decision's main
  cost. Mitigated by holding the key in a password manager plus an offline copy.
- Verification is harder. You cannot glance at the bucket and confirm the right things are
  there — the names are ciphertext. This is what makes a periodic restore drill mandatory
  rather than optional.

## Alternatives considered

**Server-side encryption (SSE-S3 / SSE-KMS).** One checkbox, no key custody burden, and
integrates with cloud-native tooling. Rejected for this data: the provider sees plaintext, and
"the provider is trustworthy and uncompromised" is an assumption rather than a control. SSE-KMS
remains the right answer for workloads whose tooling cannot see through client-side
encryption — a database operator using a cloud SDK directly, for example, cannot write through
a `crypt` remote, and that path legitimately uses server-side encryption with a managed key.
Different tool, different threat model, decided separately.

**Encrypting archives with GPG before upload.** Equivalent protection, and the key-custody
problem is identical. Rejected on operational grounds: it does not compose with an incremental
sync, so it means full re-uploads or hand-rolled chunking. `crypt` gives the same property while
preserving incremental behavior.

**No encryption, relying on bucket access controls.** Rejected. Access control is one
misconfiguration away from failing, and the data does not tolerate that.

---

*The pipeline implementing this decision, and an audit of it, are in
[aws-backup-pipeline](https://github.com/blackcharger/aws-backup-pipeline).*
