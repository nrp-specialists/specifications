# Feature: Linked and Hierarchical Records in CESNET Invenio

**Status:** Proposed
**Version:** 1.3 (updated 2026-06-24)
**Discussion:** [nrp-specialists/specifications #1](https://github.com/nrp-specialists/specifications/discussions/1) · originated from [NRP-CZ/discussions/15](https://github.com/orgs/NRP-CZ/discussions/15)
**Responsible:** Miroslav Šimek, CESNET

## Summary

Two ways for CESNET Invenio repositories (CCMM metadata schema) to relate records to one another:

- **Linked records** — already supported. Records are related but independent: each can be created, published, versioned and access-controlled separately, potentially by different authors or institutions. Implemented via the CCMM `related_resource` field, the `pid-relation` field type, and query-based collections.
- **Hierarchical (compound) records** — not yet implemented. A single logical record split into many for performance or manageability: the root record controls publication, versioning and access for the whole tree, and each child points to a single parent.

## Use Cases

Derived from the preliminary requirements of five domain repositories ([analysis](https://gist.github.com/illyriab/85e4f41b012ce131abfaf17035e60763)):

- **BioSim CZ** — hierarchical: Study (root) → Experiments (children).
- **Czech BioImaging** — hybrid: dataset → activity hierarchy (`isPartOf`) plus PID relations for the activity graph (`isFollowedBy`).
- **Fyzika** — linked: generic root + aggregated datasets + measurements; many children, so each child links up to its parent.
- **DANTEc** — hybrid: Fair Digital Object (root, DOI) → activities (children, shared model); needs both vertical and horizontal search.
- **Chemická biologie** — to be analysed (potentially 10k+ records).

## Specification

### Terminology

- **root record** — has no parent
- **parent record** — has a child
- **child record** — has a parent

### Data Model

**Linked records** — relate records without giving up their independence:

```yaml
record:
  id: <PID>
  related_resource:            # CCMM, internal or external resource
    - iri: <PID / URL of the related record>
      resource_relation_type: is_part_of | has_part | is_derived_from | ...
  instrument:                  # alternatively, a typed in-repository link
    type: pid-relation         # (or a controlled vocabulary)
    id: <PID of the instrument record>
```

Plus **collections**: query-defined groups of records (no own metadata beyond a name, no PID, no versions).

**Hierarchical (compound) records** — the relation is stored on the child, not the parent (a parent may have very many children):

```yaml
root_record:
  id: <PID>                          # DOI; stable concept DOI spans all versions
  metadata: <schema>
  lifecycle: published               # single publish action for the whole tree

child_record:
  id: <PID>                          # may use an alternative PID (Handle / ePIC / ARK)
  parent_in_hierarchy: <parent PID>  # reverse link; system metadata, set only at creation
  metadata: <schema>                 # each level may have its own schema
  cannot_publish_independently: true
```

### Behavior

**Linked records**

- Referential integrity is not enforced (repositories are decentralized); tombstone records usually keep links resolvable after a target is removed.
- `related_resource` is a single flat list — items cannot be grouped into contextual sets (e.g. pairing a *sample* with its *instrument*).
- `pid-relation` links must stay within the same repository but can sit anywhere in the metadata, including inside structured lists.
- Export to CCMM is automatic for `related_resource`; `pid-relation` and collections have no single obvious CCMM mapping.

**Hierarchical (compound) records**

- Root publication → the entire hierarchy is published; children cannot be published or retracted independently.
- Updating a file → republish the tree → new root version with a new PID (the concept DOI stays stable); editing only a child's *metadata* does not require republishing.
- Access is inherited strictly from the root; the parent/root link is system metadata, settable only at creation.
- A child has exactly one parent (multi-parent cases use `related_resource` instead).
- Search returns children; the UI propagates the root's link/context. Each record has a single file table (no per-part files unless split).

## Implementation Phases

| Phase | Scope | Timeline | Owner |
|-------|-------|----------|-------|
| 1 | Linked-records API fixes (same-model links lazy init; vocabulary-item indexing bug) | by 08/2026 | CESNET |
| 2 | Hierarchical/compound backend: `parent-in-hierarchy` relation, Child-records tab, search within a child | Q4 2026 | CESNET |
| 3 | Compound search-index record (map root + children into one index document to fix pagination) | TBD | CESNET (maybe core) |
| — | Alternative PIDs (Handle / ePIC / ARK), domain-specific GUI, file-level metadata | TBD | user communities |

## Open Questions

- [ ] Rename "hierarchical" → "compound", with *main-record / sub-record / record*? (@xulman)
- [ ] Must a parent and its children share the same metadata schema? Not identical, but publish-time constraints exist. (@mesemus)
- [ ] ePIC: which `pyhandle` versions are usable, and is any of the B2SHARE/B2INST approach reusable? (@illyriab)
- [ ] Can child records have independent access rules? (DANTEc / Czech BioImaging may need exceptions)
- [ ] How to handle root-change propagation to children without overwhelming curators?
- [ ] Bulk download of children — client-side shortcut only, or any server-side support?

## Changelog

- **v1.3** (2026-06-24): Restructured into this template (Summary, Use Cases, Data Model, Behavior, Implementation Phases); migrated into the `nrp-specialists/specifications` repo.
- **v1.2** (2026-06-10): Document as published in NRP-CZ Discussion #15 — linked vs hierarchical records, related topics (incl. ePIC/pyHandle update), decision guide.
