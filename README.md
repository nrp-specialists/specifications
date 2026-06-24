# NRP Specifications

Canonical specifications for the **NRP repository platform** (CESNET Invenio, CCMM metadata schema), maintained by the NRP repository system specialists.

This repository is the **single source of truth** for each specification. Each spec is versioned here as Markdown; open questions and feedback are discussed in the **[Discussions](https://github.com/nrp-specialists/specifications/discussions)** tab, with one hub thread per specification acting as a signpost back to the canonical document.

## How it works

| | Where | Purpose |
|---|---|---|
| 📄 **Specification** | `specs/<nn-slug>/` in this repo | The authoritative, versioned text. Changes go through pull requests. |
| 💬 **Discussion** | [Discussions](https://github.com/nrp-specialists/specifications/discussions) tab | One pinned hub thread per spec. Asynchronous feedback, open questions, decisions. The thread links back to the spec; the spec links to the thread. |

Keeping the two separate means the spec stays clean and citable, while the conversation around it stays open and threaded — instead of a single long document where text and comments are interleaved.

## Specifications

| # | Specification | Status | Document | Discussion |
|---|---|---|---|---|
| 01 | Linked and Hierarchical Records | Proposed | [specs/01-linked-and-hierarchical-records](specs/01-linked-and-hierarchical-records/) | [hub thread](https://github.com/nrp-specialists/specifications/discussions/1) |

## Contributing

- **Propose a change to a spec** — open a pull request against the relevant file in `specs/`.
- **Raise an open question or give feedback** — comment in the spec's hub thread in [Discussions](https://github.com/nrp-specialists/specifications/discussions).
- **Propose a new spec** — open a discussion describing the topic; once there is rough consensus, it is added as `specs/<nn-slug>/`.

## Status values

- **Proposed** — under discussion; structure and content may still change.
- **Accepted** — agreed; changes are tracked as new versions.
- **Implemented** — delivered in the platform.
- **Superseded** — replaced by a later spec (linked from the header).

## Contact

NRP repository system specialists — nrp.repozitare@eosc.cz
