# Linked and Hierarchical Records — Specification

**Status:** [OPEN]
**Version:** 2.0
**Date:** 2026-06-24
**Discussion:** [hub thread #3](https://github.com/nrp-specialists/specifications/discussions/3) — DM suggests to start a new discussion focused primarily on commenting the specification
**Ideation:** [NRP-CZ #15](https://github.com/orgs/NRP-CZ/discussions/15)

## Summary

Two mechanisms for CESNET Invenio repositories (CCMM metadata schema) to relate records to one another:

- **Linked records** — already supported. Records are related but independent: each can be created, published, versioned, and access-controlled separately, potentially by different authors or institutions. Linking can be implemented via the CCMM `related_resource` field, the `pid-relation` field type, and query-based collections.

- **Hierarchical (compound) records** — not yet implemented. A single logical record split into many for performance or manageability: the root record controls publication, versioning, and access for the entire tree. Each child links upward to its single parent via system metadata.

## Terminology

- **root record** — has no parent
- **parent record** — has at least one child
- **child record** — has a parent

## Linked Records — Existing Mechanisms

Linked records are supported today and require no new development (except the same-model pid-relation fix in Section 2.2.3). Repository teams configure these mechanisms in their metadata schemas.

### Related Resource (`related_resource`)

The CCMM `related_resource` field links a record to any internal or external resource via a URL, IRI, or DOI, qualified by a relation type from the CCMM controlled vocabulary (including `isPartOf`, `hasPart`, `isDerivedFrom`, and other standard relation types).

#### Data Model

```yaml
record:
  related_resource:
    - iri: <PID or URL of the related resource>
      resource_relation_type: is_part_of | has_part | is_derived_from | ...
```

#### Behavior

- Resources sit in a single flat list — there is no way to group items into contextual sets (e.g., pairing a *sample* with its *instrument*).
- No extra metadata can be attached to a relation beyond the CCMM field itself.
- Export to CCMM 1.1.0 is automatic — the same mechanism is used for both internal and external resources.

#### Limitations

- **No referential integrity.** Invenio does not enforce that the target of a `related_resource` link exists or remains valid. However, tombstone records typically keep links resolvable after a target is removed.
- Related resource can be referenced both from the parent record or from the child records. When stored on the parent level, **max. recommended number** of referenced resources = **100**.
- **Flat structure only.** Items cannot be composed into tuples or structured groups within a single `related_resource` list.

#### Responsibility

| Aspect | Owner |
|---|---|
| Platform support | Built into Invenio / CCMM — no new work |
| Schema configuration | Repository teams |

### PID Relation (`pid-relation`)

An internal link to another record within the same repository, resolved via the internal Invenio PID. Through the `keys` parameter, selected field values from the referenced record are **copied** into the linking record's OpenSearch index at index time. Values are pinned to the referenced version (`@v` attribute) and do not propagate back to the source or transitively through multiple hops.

#### Data Model

`pid-relation` can sit anywhere in the metadata structure, including inside nested lists. It is not limited to a single flat namespace like `related_resource`.

```yaml
record:
  instrument:
    type: pid-relation
    id: <internal PID of the instrument record>
    keys:                         # fields to copy from the target record
      - name
      - category
      - "@v"                      # version pin (optional)
```

#### Behavior

- Values are copied once at index time into the parent record's OpenSearch document. There is no SQL join at query time — the copied values are the searchable representation.
- Copies are **one hop only**: copying stops at the directly referenced record; transitive chains are not resolved.
- Version pinning via `@v` freezes the copied values at a specific version of the target record.

#### Limitations

- **Same-repository only.** Cross-repository linking must use `related_resource` instead.
- **No automatic CCMM export mapping.** Unlike `related_resource`, pid-relation links have no single obvious representation in CCMM 1.1.0.
- **Currently cannot link records of the same model.** This is a known bug affecting self-referencing schemas — see Section 2.2.3.

#### Planned Fix: Same-Model Linking

| Component | Owner | Timeline |
|---|---|---|
| Fix pid-relation self-reference limitation | CESNET Invenio | End of August 2026 (CESNET Invenio 14) |

After this fix, a record will be able to contain a `pid-relation` field whose target is a record of the same metadata model.

#### Responsibility

| Aspect | Owner |
|---|---|
| Platform support (including same-model fix) | CESNET |
| Field placement and `keys` configuration in metadata schemas | Repository teams |

### Record Collections

A query-defined dynamic group of records displayed on a dedicated page. Collections are a built-in Invenio feature.

#### Behavior

- A collection is defined by an OpenSearch query; membership is dynamic and changes as records are created, updated, or deleted.
- Each collection has a name but no own metadata, PID, DOI, version, or curator workflow.

#### Limitations

- No immutability — collections are not suitable for citing as a persistent data resource.
- No curation lifecycle — collection membership is purely query-driven with no approval step.
- No versioning — the collection's contents at a point in time cannot be frozen or archived.

#### Responsibility

| Aspect | Owner |
|---|---|
| Platform support | Built into Invenio — no new work |
| Query definition and collection configuration | Repository teams |

### File-level Metadata

Invenio supports describing individual files attached to a record with key–value metadata, see InvenioRM docs on [Files](https://inveniordm.docs.cern.ch/reference/metadata/#files). The metadata are indexed into OpenSearch and are searchable.

#### Behavior

- Metadata is attached to individual files within a record, not to the record itself.
- Metadata are indexed into OpenSearch alongside the record, making them discoverable in search queries.

#### Limitations

- **Flat key–value only.** The upload component does not support hierarchical metadata keys, vocabulary-bound values, or structured nested fields.
- **Default max 100 files per record.** Exceeding this requires configuration changes.
- **No PID per individual file** (DOI applies only to the whole record); ARK would allow a per-file PID.

#### Potential Role

File-level metadata may substitute for sub-records in simple cases where the only reason to split a record is to attach structured metadata to individual components. 

#### Responsibility

| Aspect | Owner |
|---|---|
| Platform support | Built into Invenio — no new work |
| File-metadata schema definition | Repository teams |
| API-based uploads with structured metadata | Repository teams (background processor or API client) |

## Hierarchical (Compound) Records — Planned New Feature

Hierarchical records are **not yet implemented** in CESNET Invenio. This section describes the planned design, which will be refined through individual consultations with each repository.

### Motivation

Several NRP domain repositories need hierarchical records to:

- **Share a publishing workflow** — a single publish action for an entire tree of records, rather than publishing each child individually.
- **Share access control** — access rights defined once on the root and inherited by all children.
- **Maintain referential integrity** — the parent–child link is kept in the system-level metadata, enforced by the platform, not user-editable opaque strings in user metadata.
- **Reduce curator burden** — without hierarchy, a curator must manually approve and manage each child record independently (painful at scale, e.g., DANTEc's 100+ activities per dataset).

### Data Model

```yaml
root_record:
  id: <PID>                          # DOI; the concept DOI remains stable across versions
  metadata: <schema>
  lifecycle: published               # a single publish action for the entire tree

child_record:
  id: <PID>                          # may use an alternative PID (Handle / ePIC / ARK)
  parent_in_hierarchy: <parent PID>  # reverse link; system metadata, set only at creation
  metadata: <schema>                 # each level may have its own schema
```

Key properties of the data model:

- **Parent link is system metadata** — stored outside user-editable metadata fields, set only at record creation. Later changes to the parent link require a formal request (not self-service).
- **One child = one primary parent** — multiple parents are not supported in the hierarchical implementation. Secondary relationships (beyond the primary parent) must use `related_resource`.
- **Link direction: child → parent** — the link is stored on the child, not on the parent. This avoids parent-record bloat when a root has many children (the parent holds no list of child IDs).
- **Each level may have its own metadata schema**.

### Lifecycle Behavior

#### Publishing

- **Root publishes the whole tree.** A single publish action on the root record transitions all children to `published` as well.
- **Children cannot be published or retracted independently.** The child's lifecycle is derived from the root.

#### Access Control

- **Access rights are inherited strictly from the root.** All children share the same access policy as the root record.
- Whether exceptions to this rule are needed for specific repositories (DANTEc, Czech BioImaging) is an open question (see Section 7).

#### Search

- Search queries return individual child records, not roots.
- The UI is responsible for reconstructing the tree from search results: querying for children of a given root, displaying the parent context on a child record, and navigating the hierarchy.
- See Section 3.5 for search limitations and workarounds.

### Versioning

Changing any child's files, thus creating a new version of a child record, creates a **new version of the entire tree**: a new root version with a new PID, plus new versions and identifiers for all children. The old identifiers remain discoverable. This behaviour has no workaround as of now, and probably will never have. The implications are obvious a single data edit to one child regenerates PIDs for the entire tree — for trees with many children or frequent updates, this may an issue.

**Important:** changes to a record's *metadata* **do not create** a new *version* of the modified record, just a new **revision**. Revisions of records at all levels of the tree can be made as needed, they do not trigger any change to the root record or to the whole tree. With a new revision of a record, its PID remains the same, the changes are stored in the database, they cannot be accessed through API, only the sequence number of the revision is available in the record's metadata.

### Search Architecture

#### Core Limitation

OpenSearch cannot perform SQL-style JOINs or cross-record aggregations. A query such as "find all roots where at least 5 children used the same method" is **not possible** with a generic search approach — it would require storing child-level aggregated information in the root's index document.

#### Search Returns Children, Not Roots

By default, a search query matches child records based on their own metadata. The user sees individual children in search results. To show only roots or to group children under roots, the UI must:

1. Execute a search against child metadata to find matching children.
2. Collect the parent IDs of the matched children.
3. Query for those parent records and display them grouped.

This is a two-query pattern with no pagination guarantee across both queries.

#### Workaround: Composite Search Index

Child metadata fields needed for search and pagination can be **copied** into the root record's OpenSearch index via the `keys` mechanism of `pid-relation`. This creates a composite index document where the root carries aggregated child data, enabling:

- **Root-level search** — query root records by child metadata values.
- **Root-level pagination** — paginate over roots, not children.
- **Root-level facets and aggregations** — filter by child attributes at the root level.

The trade-off is a doubled index size (child data exists both in the child's own index and copied into the root's index).

### Download and bulk operations
- **No server-side generation of a composite ZIP, never from the GUI/browser** (performance; cannot be parallelised).
- Bulk download via the **command-line client (`nrp-cmd`)** – the user searches for records by relation (root ID / Related Resource) and downloads them in batch.
- A possible **shortcut in the CLI client** per specific repository; this limitation needs to be **documented** so the groups account for it from the start.
- **Bulk actions over linked records** (bulk operations over a set of linked/child records) mentioned as another possible path – to be specified.

### Implementation Plan

| Component | Owner | Timeline | Notes |
|---|---|---|---|
| Self-reference pid-relation fix | CESNET | Aug 2026 | Prerequisite for some use cases. |
| `parent_in_hierarchy` system field (backend) | CESNET | Q4 2026 | Core Invenio feature; creates the child→parent link as non-user-editable system metadata. |
| Children-tab UI | CESNET | Q4 2026 | UI tab on a parent record listing its children, with navigation to each child. Part of core. |
| Root publish cascades to children | CESNET | Q4 2026 | Publish action on root transitions all children. |
| Access-right inheritance from root | CESNET | Q4 2026 | Children inherit the root's access policy. |
| Composite search index tooling | CESNET | TBD | Candidate for core; maps root + children into a single OpenSearch index document. |
| Alternative PIDs (Handle / ePIC / ARK) | Repository teams / community | TBD | Communities develop or adapt PID providers. |
| Domain-specific GUI customizations | Repository teams | TBD | Custom UI beyond the core children tab. |
| File-level metadata schemas | Repository teams | TBD | Define per-record-type file-metadata structure. |
| Curation and notification workflows | Repository teams (+ CESNET advisory) | TBD | Depends on automation-vs-manual decisions per repository. |

### CESNET Support Model

CESNET will:

- **Advise** on the choice of approach (linked vs. hierarchical vs. hybrid) for a given use case.
- **Provide high-level guidance** on implementation within CESNET Invenio.
- **Accept parts into Invenio core** if the implementation is general enough and useful across repositories.
- **Require a consultation** before a repository begins hierarchical-record implementation, to determine whether the work belongs in the library component or the Invenio core repository component.

## Persistent Identifiers

| PID type | Status | Effort | Owner | Notes |
|---|---|---|---|---|
| **DOI** | Built-in, out of the box | Zero | CESNET Invenio | Default for root records. Concept DOI remains stable across versions. |
| **Internal Invenio PID** | Built-in | — | CESNET Invenio | Random (parallelization-safe). No naming convention; sequential IDs possible via PID provider but bottleneck at DB lock. |
| **ARK** | Implemented in Central/West Africa consortium | Medium | Community | No external server needed; repository itself serves as resolver. Supports hierarchical semantics and per-file PIDs. Multiple institutions must share one NAAN prefix. |
| **Handle** | No Invenio support in current version | High | Repository teams / community | Requires a Handle server. Maintained Python library exists but no Invenio bridge. Old B2Share implementation (~6 years old) is incompatible with current Invenio. |
| **ePIC** | No Invenio support in current version | High | Repository teams / community | Based on Handle infrastructure. B2SHARE v3 / B2INST v3 used pyhandle v1 on a different Invenio version; not reusable without work. Price/governance through NTK with Göttingen. |
| **Internal community identifiers** (BioSimCZ) | Customizing the Invenio PID provider | Medium | Rapository teams | A URL with a prefix (`MD_0001`) resolves to the landing page; the address bar keeps the internal Invenio ID. |
| **External accession ID** (BioSimCZ) | Via secondary ID or custom PID provider | Medium–high | Repository teams | BioSim CZ will receive IDs from a European catalog service. |

## Migration Path

- **Linked → hierarchical** conversion is technically possible with migration work. You **cannot switch "on the fly"** from pure Related Resource to a strict hierarchy managed by Invenio – a conversion is required.
- During the interim period before hierarchy is available, repositories can use linked records. The curator manually approves and manages each child individually — painful at scale, but functional.
- Once hierarchical records are implemented, repositories can bulk-convert their existing linked-record trees into the hierarchical model.

## Open Questions

- [ ] How does a composite index approcach achieved through `pid-relation` go along with hierarchical records?
- [ ] Can child records in a hierarchy have independent access rights? DANTEc and Czech BioImaging may need exceptions to the strict inheritance rule.
- [ ] Is ePIC minting via pyhandle viable in current Invenio, or is a new bridge needed from scratch?
- [ ] Should bulk download of children be supported server-side, or is client-side via `nrp-cmd` CLI sufficient?
- [ ] What is the DOI versioning strategy for dynamic datasets (snapshots / canonical DOI / date in citation)?
- [ ] Should "hierarchical records" be renamed to "compound records" (main-record / sub-record / record)?

### Curation and Notification in LINKED records
- **Notifying the curator/creator of a change to a child** that is involved in multiple relations – warn that the child is used *N×* and let them decide whether to create a new version.
- The question is **how far to automate the versioning and curation** – automation only makes sense where the behaviour is always the same (otherwise an override is needed and database consistency cannot be guaranteed).
- **Re-framing the whole problem as "tagging + graph structure"** (David) – a new angle in the closing discussion.
  - **Version tagging** – marking versions with tags so it is clear what each version contains

## Changelog

| Version | Date | Summary |
|---|---|---|
| v2.0 | 2026-06-24 | Rewritten as a technical specification. Restructured mechanisms into detailed subsections with data models, behavior, limitations, and responsibility splits (CESNET vs. repository teams). Moved meeting-minutes content (decisions, action items) into the implementation plan and open questions. Added per-mechanism technical depth. |
| v1.0 | 2026-06-24 | Initial structured specification derived from the meeting of 22 June 2026. |
