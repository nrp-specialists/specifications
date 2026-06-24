# Linked and Hierarchical Records (v2)

How CESNET Invenio repositories (CCMM metadata schema) can relate records to one another — as decided at the meeting of 22 June 2026.

- **Linked records** — already supported; records are related but keep independent lifecycles, access rights, and versioning.
- **Hierarchical (compound) records** — not yet implemented; a single logical record split into many, where the root controls publication, versioning and access for the whole tree.
- **Hybrid approach** — some repositories combine both (e.g. hierarchy for datasets, pid-relation for activity graphs).

📄 **Full specification:** [specification.md](specification.md)
💬 **Discussion:** [hub thread #3](https://github.com/nrp-specialists/specifications/discussions/3)
📋 **Ideation:** [discussion #2](https://github.com/nrp-specialists/specifications/discussions/2) (migrated from [NRP-CZ #15](https://github.com/orgs/NRP-CZ/discussions/15))
