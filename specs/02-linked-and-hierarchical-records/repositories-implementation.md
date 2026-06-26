## Repository-Specific Architectures

Each repository uses a different combination of the mechanisms described above, based on its domain requirements for record lifecycle, search patterns, and curator workflow.

| Repository | Architecture | Mechanisms | Notes |
|---|---|---|---|
| **BioSim CZ** | Pure hierarchical | `parent_in_hierarchy` | Study = root; experiments = children (3 types, each with own schema). Shared publishing, versioning, and access rights. Simplest match for the hierarchical model. |
| **Czech BioImaging** | Hybrid | `parent_in_hierarchy` (dataset → activity, `isPartOf`) + `pid-relation` (activity graph, `isFollowedBy`) + `related_resource` (multiple parents) | Tree structure rather than graph-with-join; the join variant was evaluated and rejected. File-level metadata may reduce the need for hierarchy in some cases. |
| **Fyzika (Physics)** | Linked records | `related_resource`, collections | Independent lifecycles; root record (DOI, ~10 schemas) + aggregated datasets + individual measurements. Children added daily (e.g., FRAM every night); with >~100 children, the link goes from child upward. |
| **DANTEc** | Hybrid (all mechanisms) | `related_resource` + `pid-relation` + `parent_in_hierarchy` + collections | Fair Digital Object as root (CCMM, DOI, RO-Crate); activities as children (ePIC, shared metadata model). Heterogeneous community (nanofabrication, engineering, logistics) needs maximum flexibility. Pure linked records would lose referential integrity. |
| **Chemical Biology** | TBD | TBD | To be analysed; potentially 10k+ records. |


## Per-Repository Search Patterns

| Repository | Search Pattern | Approach |
|---|---|---|
| **BioSim CZ** | Search experiments by metadata → show only studies (roots) | Composite search index: study record enriched with experiment metadata. Open question: whether standalone experiments should also be indexed individually. |
| **Czech BioImaging** | Search across processing steps → return roots or activities | To be specified in consultation. |
| **DANTEc** | Vertical search (one material, all methods) + horizontal search (one method across materials) | Share use cases, then decide approach. |