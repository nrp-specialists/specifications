# Linked and Hierarchical Records — Specification

**Status:** [OPEN]
**Version:** 1.0
**Date:** 2026-06-24
**Discussion:** [hub thread #3](https://github.com/nrp-specialists/specifications/discussions/3)
**Ideation:** [discussion #2](https://github.com/nrp-specialists/specifications/discussions/2) (migrated from [NRP-CZ #15](https://github.com/orgs/NRP-CZ/discussions/15))
**Meeting:** 22 June 2026
**Responsible:** Miroslav Šimek (CESNET), Illyria Brejchová & Daniel Mikšík (repository system specialists)

## Summary

Two ways for CESNET Invenio repositories to relate records to one another:

- **Linked records** — already supported. Records are related but independent: each can be created, published, versioned and access-controlled separately, potentially by different authors or institutions. Implemented via the CCMM `related_resource` field, the `pid-relation` field type, and query-based collections.

- **Hierarchical (compound) records** — not yet implemented. A single logical record split into many for performance or manageability: the root record controls publication, versioning and access for the whole tree. The link goes from each child upward to its single parent.

The meeting confirmed that 3 out of 5 domain repositories need hierarchical records.

## Terminology

- **root record** — has no parent; controls lifecycle, access rights, and versioning for the entire tree.
- **parent record** — has at least one child.
- **child record** — has a parent; links upward.

## Use Cases (per repository)

| Repository | Approach | Rationale |
|---|---|---|
| **BioSim CZ** | pure hierarchical | Study = root; experiments = children (3 types, own schemas). Shared publishing, versioning, and access rights. Simplest match for the hierarchical model. |
| **Czech BioImaging** | hybrid | Hierarchy for dataset → activity (`isPartOf` / `wasGeneratedBy`); pid-relation for the activity graph (`isFollowedBy`). Multiple parents via Related Resource. Tree structure instead of graph-with-join (join variant rejected). |
| **Fyzika (Physics)** | linked records | Independent lifecycles; root record (DOI, ~10 schemas) + aggregated datasets + individual measurements. Children added daily (e.g. FRAM every night). With >~100 children, link goes from child upward. |
| **DANTEc** | hybrid (all mechanisms) | Fair Digital Object as root (CCMM, DOI, RO-Crate); activities as children (ePIC, shared metadata model). Heterogeneous community (nanofabrication, engineering, logistics) needs maximum flexibility. Pure linked records rejected (loss of referential integrity). |
| **Chemical Biology** | TBD | To be analysed (potentially 10k+ records; Marek Moos left the meeting early). |

## Existing Mechanisms

### Related Resource (CCMM)

CCMM's `related_resource` field links a record to any internal or external resource via URL/IRI/DOI with a relation type from the CCMM controlled vocabulary (includes `isPartOf`/`hasPart`).

- **Advantages:** Standard CCMM 1.1.0 export; same mechanism for internal and external resources.
- **Limitations:** Invenio enforces no referential integrity; all resources sit in a single flat list — no way to group items into contextual sets (e.g. pairing a sample with its instrument); no extra metadata beyond the CCMM field.

### PID Relation

An internal link to a record in the same repository (via internal Invenio PID). Through the `keys` parameter, field values from the referenced record are **copied** into the linking record (for indexing/search — no SQL join is possible in OpenSearch). Values freeze at the referenced version (`@v` attribute) and do not propagate back or transitively.

- **Advantages:** Supports controlled vocabularies; can sit anywhere in metadata including inside structured lists.
- **Limitations:** Works only within the same repository; no automatic CCMM export mapping; copying is not transitive (one hop only); currently **cannot link records of the same model** — fix planned.

### Record Collections

A query-defined dynamic group of records with a dedicated page. Built into Invenio.

- **Advantages:** Almost zero setup.
- **Limitations:** No metadata beyond a name; no PID or DOI; no versions; no curator workflow.

### File-Level Metadata

Invenio supports tagging individual files with key-value metadata (indexed into OpenSearch → searchable).

- **Advantages:** Works out of the box; may replace sub-records in simple cases (Vláďa / Czech BioImaging is reconsidering whether hierarchy is needed at all thanks to this).
- **Limitations:** Upload component supports only flat key-value (not hierarchical, not vocabulary items); schema must be defined outside the record YAML; all files share the same metadata model regardless of record type; default max 100 files per record.

## Planned: Hierarchical (Compound) Records

### Data model

```yaml
root_record:
  id: <PID>                          # DOI; stable concept DOI spans all versions
  metadata: <schema>
  lifecycle: published               # single publish action for the whole tree

child_record:
  id: <PID>                          # may use alternative PID (Handle / ePIC / ARK)
  parent_in_hierarchy: <parent PID>  # reverse link; system metadata, set only at creation
  metadata: <schema>                 # each level may have its own schema
```

### Behavior

- **Root publishes the whole tree;** children cannot be published or retracted independently.
- **Access rights inherited strictly from root.**
- **Parent link is system metadata** (not user-editable) — settable only at record creation; later changes require a formal request.
- **One child = one primary parent** — multiple parents not supported in the implementation (use Related Resource for secondary relationships).
- **Link direction: child → parent** (not parent → children list) — avoids parent-record bloat with many children.

### Versioning cascade

Changing any child (files or metadata) creates a **new version of the entire tree**: a new root version with a new PID, plus new identifiers for all children. The old identifiers remain discoverable. Mirek describes this as a significant problem with no workaround yet.

Version tagging is being considered: marking versions with tags so it is clear what each version contains.

### Curation

When a child that is part of multiple relationships changes, the curator/creator should be notified ("this child is used N×") and decide whether to create a new version. The open question is **how far to automate**: automation only makes sense where behaviour is always the same.

### Search limitations

- OpenSearch cannot do SQL joins or aggregations across child records. Searching "root where at least 5 children used the same method" is **not possible** generically.
- Search returns individual children, not roots by default — the UI must reconstruct the tree.
- **Workaround:** copy selected child metadata into the root record via `keys` in pid-relation, enabling root-level search and pagination.

### CESNET's support model

CESNET will:

- Advise on the choice of approach for a given use case.
- Provide high-level guidance on implementation.
- Accept parts into Invenio core if general enough and useful across repositories.
- Require a **consultation before implementation begins** (library vs. repository component).

## Persistent Identifiers

| PID type | Status | Effort estimate | Notes |
|---|---|---|---|
| **DOI** | built-in, out of the box | zero | Default for root records. |
| **Internal Invenio ID** | random (parallelization-safe) | — | No naming convention; sequential IDs possible via PID provider but bottleneck at DB lock. |
| **ARN** | implemented in Central/West Africa consortium | medium | No external server needed; repository itself serves as resolver; supports hierarchical semantics and per-file PIDs; multiple institutions must share one NAAN prefix. |
| **Handle** | no Invenio support in current version | high | Requires a Handle server; maintained Python library exists but no Invenio bridge. Old B2Share implementation (6 years old) is incompatible. |
| **ePIC** | no Invenio support | high | Based on Handle infrastructure; B2SHARE v3 / B2INST v3 used pyhandle v1 but on a different Invenio version; not reusable without work. Price/governance through NTK with Göttingen. |
| **External accession ID** | via secondary ID or custom PID provider | medium–high | BioSim CZ will receive IDs from a European catalog service. |

## Search Requirements (per repository)

| Repository | Search pattern | Approach |
|---|---|---|
| **BioSim CZ** | search by experiment metadata → show only studies (roots) | Build a composite search index (study + experiment metadata); the doubled index size trade-off is accepted. Open question: whether to also index standalone experiments. |
| **Czech BioImaging** | search across processing steps → return roots or activities | To be specified in consultation. |
| **DANTEc** | vertical (one material, all methods) + horizontal (one method across materials) | Share use cases → then decide. |

## Key Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | **Hierarchical records will be implemented in Invenio core** | 3/5 repos need them; specification will be refined after individual consultations. Implementation target: Q4 2026. |
| 2 | **PID relation self-reference fix** | CESNET will fix by end of August 2026 (CESNET Invenio 14). After fix, same-model linking will be possible. |
| 3 | **BioSim = pure hierarchical** | Confirmed at the meeting. |
| 4 | **Czech BioImaging = hybrid** | Hierarchy for dataset structure, pid-relation for activity graph. Multiple parents via Related Resource. |
| 5 | **Fyzika = linked records** | Independent lifecycles needed for dynamic datasets. |
| 6 | **DANTEc = hybrid (all)** | Maximum flexibility for heterogeneous community. |
| 7 | **Bulk download: client-side only** | `nrp-cmd` CLI; no server-side ZIP generation. No GUI support. |
| 8 | **Composite search index for BioSim** | Study enriched with experiment metadata for pagination. |
| 9 | **Transition strategy: "for now, linked records"** | Start with linked records; convert to hierarchical when implemented. Conversion requires work — cannot switch "on the fly" from Related Resource to strict hierarchy. Until then, manual curation (painful at scale). |
| 10 | **Verification step before implementation** | *Are hierarchical records truly necessary?* The added complexity (versioning, download, search) must be justified by use cases. The argument FOR hierarchy is curator sustainability and record integrity. |

## Alternate Viewpoint: Tag + Link, Don't Graph

David (CESNET) raised an alternative framing: perhaps the problem isn't graphs of records but **tagged records with relations**. Start with plain linked records + curator discipline → when use cases crystallize, convert to strict hierarchy (technically possible with conversion).

**Counterpoints:**

- DANTEc: 100+ children per dataset; curator can't manually approve each.
- BioSim CZ: wants shared publishing + versioning from day one.
- Systemic vs. user metadata: parent must live in Invenio's system layer (not user-editable), so plain Related Resource in user metadata can't drive lifecycle.

- **Linked → hierarchical possible** with conversion work (Mirek confirmed).
- During the interim: curator manually approves each child (painful at scale).

## Migration Path

- **Linked → hierarchical** is possible with conversion work.
- In the interim: curator manually approves each child individually.
- Once hierarchy is implemented, bulk-convert the community's records.

## Risk / Concern Register

| Issue | Severity | Mitigation |
|---|---|---|
| Versioning creates new PIDs for all children on any change | 🔴 high | Needs deeper design; Mirek has no workaround yet. |
| File tagging (key-value) could replace hierarchy for some cases | 🟡 medium | Reduces complexity for simple use cases; Vláďa already reconsidering. |
| ePIC / Handle has zero Invenio support | 🟡 medium | Community must write provider; not a CESNET priority. |
| DANTEc heterogeneity (unknown data shapes) | 🟡 medium | Start with hybrid; add mechanisms as use cases emerge. |
| Lazy initialization of self-referencing pid-relation | 🟢 low | Performance penalty on lookup; acceptable for DANTEc's ~100 activities. |
| Upload component can't handle hierarchical file metadata in GUI | 🟢 low | Workaround: upload via API; extract metadata via background processor. |

## Action Items

| Task | Who | Deadline |
|---|---|---|
| Fix PID Relation (self-reference / lazy init) + vocabulary-item indexing bug | CESNET Invenio | end of August 2026 (CESNET Invenio 14) |
| Individual in-depth consultations with repositories — analysis of search use cases, verification of hierarchy need | Illy, Daniel, Mirek + repo teams | summer 2026 |
| Coordinate consultation dates | repository system specialists | June 2026 |
| Specification of hierarchical records based on consultations | repository system specialists + CESNET | by September 2026 |
| Core support for hierarchical records (back-end + children-tab UI) | CESNET | targeted end of 2026 |
| Go through Anastasiia's metadata table; write search use cases (vertical/horizontal); then DANTEc meeting | Marek, Mirek, Anastasiia, Illy, Daniel | by end of week |
| Share BioSim specification and UI mockups (GitHub) | repository system specialists | after meeting |
| Decide DOI versioning strategy for dynamic datasets (snapshots / canonical DOI / date in citation) | Michal L. (Physics), Mirek | at consultation |
| Provide contacts/git for ARK implementation and PID provider source code (ePIC) | Mirek | on request |
| Tool for composite search index (study + experiments) — candidate for core | Mirek + Vladimír H. | by agreement |

## Open Questions

- [ ] Should "hierarchical records" be renamed to "compound records" (main-record / sub-record / record)?
- [ ] Can child records have independent access rights (DANTEc / Czech BioImaging may need exceptions)?
- [ ] How to handle root-change propagation to children without overwhelming curators?
- [ ] How far should versioning and curation be automated vs. manual?
- [ ] Is ePIC minting via pyhandle viable in current Invenio, or is a new bridge needed from scratch?
- [ ] Are hierarchical records truly necessary, or could file-level metadata tagging + linked records suffice for some use cases?
- [ ] For BioSim: should standalone experiments still be indexed individually, or only via the composite study record?
- [ ] Must a parent and its children share the same metadata schema?

## Changelog

| Version | Date | Summary |
|---|---|---|
| v1.0 | 2026-06-24 | Initial structured specification derived from the meeting of 22 June 2026 (per-repository decisions, existing mechanisms, planned hierarchy, PID landscape, search implications, risk register, action items). Incorporates the ideation document from NRP-CZ Discussion #15 (v1.2, June 10, 2026). |
