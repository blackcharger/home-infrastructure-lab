# Architecture Decision Records

Short records of decisions made in this lab, why they were made, and what was rejected.

The point is not the decisions themselves — it's that the reasoning survives. Six months later
a configuration that looks like an oversight is usually a tradeoff nobody wrote down. These
files are the difference between "why is it built this way?" and "read ADR-0002."

## Format

Each record carries **Status**, **Context**, **Decision**, **Consequences**, and **Alternatives
considered**. Accepted records are not edited afterward — if a decision changes, a new ADR
supersedes the old one and both stay. The history of the reasoning is the useful part.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-external-quorum-witness.md) | External quorum witness instead of a third cluster node | Accepted |
| [0002](0002-file-server-outside-cluster.md) | File server deliberately excluded from the cluster | Accepted |
| [0003](0003-nvr-vm-removed-from-ha.md) | NVR VM removed from HA management after a cross-node reset fault | Accepted |
| [0004](0004-client-side-encryption.md) | Encrypt offsite backups client-side, before upload | Accepted |
| [0005](0005-storage-class-by-recovery-objective.md) | Choose storage class by recovery objective, not by data size | Accepted |
| [0006](0006-ai-generated-code-is-reviewed.md) | Treat AI-generated operations code as untrusted until reviewed | Accepted |

## A note on scope

These cover the lab as built. Decisions for the in-progress GitOps migration (Terraform, k3s,
Argo CD) are not recorded here yet, because they are not made yet. Writing an ADR for something
unbuilt would defeat the purpose.
