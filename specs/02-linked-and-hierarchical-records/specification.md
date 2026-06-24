# Linked and Hierarchical Records — Specification

**Status:** [OPEN]
**Version:** 2.0
**Date:** 2026-06-24
**Discussion:** [hub thread #3](https://github.com/nrp-specialists/specifications/discussions/3)
**Ideation:** [discussion #2](https://github.com/nrp-specialists/specifications/discussions/2) (migrated from [NRP-CZ #15](https://github.com/orgs/NRP-CZ/discussions/15))

## Summary

Two mechanisms for CESNET Invenio repositories (CCMM metadata schema) to relate records to one another:

- **Linked records** — already supported. Records are related but independent: each can be created, published, versioned, and access-controlled separately, potentially by different authors or institutions. Implemented via the CCMM `related_resource` field, the `pid-relation` field type, and query-based collections.

- **Hierarchical (compound) records** — not yet implemented. A single logical record split into many for performance or manageability: the root record controls publication, versioning, and access for the entire tree. Each child links upward to its single parent via system metadata.

Three of the five NRP domain repositories require hierarchical records; the remaining two will use linked records.

## Terminology

- **root record** — has no parent; controls lifecycle, access rights, and versioning for the entire tree.
- **parent record** — has at least one child.
- **child record** — has a parent; links upward to it.

## Linked Records — Existing Mechanisms

Linked records are supported today and require no new platform development (except the same-model pid-relation fix in Section 2.2.3). Repository teams configure these mechanisms in their metadata schemas.

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

- No persistent identity — collections cannot be cited.
- No curation lifecycle — collection membership is purely query-driven with no approval step.
- No versioning — the collection's contents at a point in time cannot be frozen or archived.

#### Responsibility

| Aspect | Owner |
|---|---|
| Platform support | Built into Invenio — no new work |
| Query definition and collection configuration | Repository teams |

### File-Level Metadata

Invenio supports tagging individual files attached to a record with key–value metadata. These tags are indexed into OpenSearch and are searchable.

#### Behavior

- Metadata is attached to individual files within a record, not to the record itself.
- Tags are indexed into OpenSearch alongside the record, making them discoverable in search queries.

#### Limitations

- **Flat key–value only.** The upload component does not support hierarchical metadata keys, vocabulary-bound values, or structured nested fields.
- **Schema is defined outside the record YAML.** Each record type shares the same file-metadata schema regardless of the record-level metadata model.
- **Default max 100 files per record.** Exceeding this requires configuration changes.

#### Potential Role

File-level metadata may substitute for sub-records in simple cases where the only reason to split a record is to attach structured metadata to individual components. Czech BioImaging is evaluating whether file-level metadata can replace planned child records for their dataset→image hierarchy.

#### Responsibility

| Aspect | Owner |
|---|---|
| Platform support | Built into Invenio — no new work |
| File-metadata schema definition | Repository teams |
| API-based uploads with structured metadata | Repository teams (background processor or API client) |

## Hierarchical (Compound) Records — Planned New Feature

Hierarchical records are **not yet implemented** in CESNET Invenio. This section describes the planned design, which will be refined through individual consultations with each repository.

### Motivation

Three of the five NRP domain repositories need hierarchical records to:

- **Share a publishing workflow** — a single publish action for an entire tree of records, rather than publishing each child individually.
- **Share access control** — access rights defined once on the root and inherited by all children.
- **Maintain referential integrity** — the parent–child link is system-level metadata, enforced by the platform, not user-editable opaque strings in user metadata.
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

- **Parent link is system metadata** — stored outside user-editable metadata fields, settable only at record creation. Later changes to the parent link require a formal request (not self-service).
- **One child = one primary parent** — multiple parents are not supported in the hierarchical implementation. Secondary relationships (beyond the primary parent) must use `related_resource`.
- **Link direction: child → parent** — the link is stored on the child, not on the parent. This avoids parent-record bloat when a root has many children (the parent holds no list of child IDs).
- **Each level may have its own metadata schema** — a study (root) and its experiments (children) can use different CCMM schemas.

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

Changing any child's files or metadata creates a **new version of the entire tree**: a new root version with a new PID, plus new identifiers for all children. The old identifiers remain discoverable.

This is a significant problem with no workaround yet. The implications:

- **PID explosion** — a single metadata edit to one child regenerates PIDs for the entire tree. For trees with many children or frequent updates, this may become unmanageable.
- **Citation stability** — individual children cannot be cited stably across versions unless reference is made to the root's concept DOI.
- **Granularity trade-off** — splitting a record into many children improves manageability but magnifies the versioning problem.

Version tagging is under consideration: marking root versions with descriptive tags so that it is clear what each version of the tree contains (e.g., "added measurement 2026-06", "corrected instrument metadata").

### Curation and Notification

When a child record that participates in multiple relationships (e.g., a child linked via hierarchy to a root and via `related_resource` to another record) is modified, the curator or creator should be notified: "this child is used N×." The curator then decides whether to create a new version of the hierarchy.

The open question is **how far to automate**: full automation is only viable where desired behavior is always the same (e.g., always create a new version on any metadata change). For cases requiring curator judgment, a notification-only approach is more appropriate.

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

#### Per-Repository Search Patterns

| Repository | Search Pattern | Approach |
|---|---|---|
| **BioSim CZ** | Search experiments by metadata → show only studies (roots) | Composite search index: study record enriched with experiment metadata. Open question: whether standalone experiments should also be indexed individually. |
| **Czech BioImaging** | Search across processing steps → return roots or activities | To be specified in consultation. |
| **DANTEc** | Vertical search (one material, all methods) + horizontal search (one method across materials) | Share use cases, then decide approach. |

### Implementation Plan

| Component | Owner | Timeline | Notes |
|---|---|---|---|
| `parent_in_hierarchy` system field (backend) | CESNET | Q4 2026 | Core Invenio feature; creates the child→parent link as non-user-editable system metadata. |
| Children-tab UI | CESNET | Q4 2026 | UI tab on a parent record listing its children, with navigation to each child. Part of core. |
| Root publish cascades to children | CESNET | Q4 2026 | Publish action on root transitions all children. |
| Access-right inheritance from root | CESNET | Q4 2026 | Children inherit the root's access policy. |
| Composite search index tooling | CESNET | TBD | Candidate for core; maps root + children into a single OpenSearch index document. |
| Self-reference pid-relation fix | CESNET | Aug 2026 | Prerequisite for some hierarchical use cases. |
| Repository-specific metadata schemas | Repository teams | During consultation | Each repo defines schemas for root and child records. |
| Alternative PIDs (Handle / ePIC / ARK) | Repository teams / community | TBD | Not a CESNET priority; community must write or adapt PID providers. |
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
| **External accession ID** | Via secondary ID or custom PID provider | Medium–high | Repository teams | BioSim CZ will receive IDs from a European catalog service. |

## Repository-Specific Architectures

Each repository uses a different combination of the mechanisms described above, based on its domain requirements for record lifecycle, search patterns, and curator workflow.

| Repository | Architecture | Mechanisms | Notes |
|---|---|---|---|
| **BioSim CZ** | Pure hierarchical | `parent_in_hierarchy` | Study = root; experiments = children (3 types, each with own schema). Shared publishing, versioning, and access rights. Simplest match for the hierarchical model. |
| **Czech BioImaging** | Hybrid | `parent_in_hierarchy` (dataset → activity, `isPartOf`) + `pid-relation` (activity graph, `isFollowedBy`) + `related_resource` (multiple parents) | Tree structure rather than graph-with-join; the join variant was evaluated and rejected. File-level metadata may reduce the need for hierarchy in some cases. |
| **Fyzika (Physics)** | Linked records | `related_resource`, collections | Independent lifecycles; root record (DOI, ~10 schemas) + aggregated datasets + individual measurements. Children added daily (e.g., FRAM every night); with >~100 children, the link goes from child upward. |
| **DANTEc** | Hybrid (all mechanisms) | `related_resource` + `pid-relation` + `parent_in_hierarchy` + collections | Fair Digital Object as root (CCMM, DOI, RO-Crate); activities as children (ePIC, shared metadata model). Heterogeneous community (nanofabrication, engineering, logistics) needs maximum flexibility. Pure linked records would lose referential integrity. |
| **Chemical Biology** | TBD | TBD | To be analysed; potentially 10k+ records. |

## Migration Path

- **Linked → hierarchical** conversion is technically possible with migration work.
- During the interim period before hierarchy is available, repositories should use linked records (`related_resource`). The curator manually approves and manages each child individually — painful at scale, but functional.
- Once hierarchical records are implemented, repositories can bulk-convert their existing linked-record trees into the hierarchical model. The conversion script must translate user-metadata `related_resource` links into system-level `parent_in_hierarchy` links.

## Risk Register

| Issue | Severity | Mitigation |
|---|---|---|
| Versioning creates new PIDs for all children on any change | 🔴 high | Needs deeper design; currently no workaround. Version tagging may help with traceability but does not solve the PID-explosion problem. |
| File-level metadata could replace hierarchy for some cases, reducing the urgency of implementation | 🟡 medium | Reduces complexity for simple use cases. Czech BioImaging already reconsidering whether hierarchy is needed. |
| ePIC / Handle has zero Invenio support | 🟡 medium | Community must write or adapt PID providers; not a CESNET priority. May delay child-record minting with non-DOI PIDs. |
| DANTEc heterogeneity — unknown data shapes in a single repository | 🟡 medium | Start with hybrid approach; add mechanisms as concrete use cases emerge from the community. |
| Lazy initialization of self-referencing pid-relation | 🟢 low | Performance penalty on first lookup; acceptable for DANTEc's ~100 activities. Fix planned (Aug 2026). |
| Upload component cannot handle hierarchical file metadata in GUI | 🟢 low | Workaround: upload via API; extract metadata via background processor. |

## Open Questions

- [ ] Should "hierarchical records" be renamed to "compound records" (main-record / sub-record / record)?
- [ ] Can child records have independent access rights? DANTEc and Czech BioImaging may need exceptions to the strict inheritance rule.
- [ ] How to handle root-change propagation to children without overwhelming curators?
- [ ] How far should versioning and curation be automated vs. manual? Full automation is only viable where behavior is always the same.
- [ ] Is ePIC minting via pyhandle viable in current Invenio, or is a new bridge needed from scratch?
- [ ] Are hierarchical records truly necessary, or could file-level metadata tagging + linked records suffice for some use cases?
- [ ] For BioSim: should standalone experiments still be indexed individually, or only via the composite study record?
- [ ] Must a parent and its children share the same metadata schema?
- [ ] Should bulk download of children be supported server-side, or is client-side via `nrp-cmd` CLI sufficient?
- [ ] What is the DOI versioning strategy for dynamic datasets (snapshots / canonical DOI / date in citation)?

## Changelog

| Version | Date | Summary |
|---|---|---|
| v2.0 | 2026-06-24 | Rewritten as a technical specification. Restructured mechanisms into detailed subsections with data models, behavior, limitations, and responsibility splits (CESNET vs. repository teams). Moved meeting-minutes content (decisions, action items) into the implementation plan and open questions. Added per-mechanism technical depth. |
| v1.0 | 2026-06-24 | Initial structured specification derived from the meeting of 22 June 2026. |
