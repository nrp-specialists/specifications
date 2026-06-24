# Specification 01 — Linked and Hierarchical Records

> **Status:** Draft &nbsp;•&nbsp; **Version:** 1.2 &nbsp;•&nbsp; **Last updated:** 2026-06-24
>
> **Editor:** Miroslav Šimek (CESNET)
> **Contributors:** Illyria Brejchová (Masaryk University), Daniel Mikšík (Masaryk University)
> **Discussion:** see the hub thread in [Discussions](../../../../discussions)
> **Origin:** consolidated from [NRP-CZ Discussion #15](https://github.com/orgs/NRP-CZ/discussions/15)

This document explains how CESNET Invenio repositories based on the CCMM metadata schema can support linked and hierarchical records.

The creation of this document was motivated by the preliminary use cases of five domain specific repositories described in the document "[Hierarchické záznamy v0.4](https://researchinfracz.sharepoint.com/:w:/s/NRP/IQAnMyAOFxv2SI0wiHgVBzDVAaCU0cXgTxxMY1nBtKlkMtI?e=2B6hry)" and analyzed in [Hierarchical Record Requirements Grouping](https://gist.github.com/illyriab/85e4f41b012ce131abfaf17035e60763).

**Linked records** are already supported in the core Invenio framework and in CESNET Invenio extensions. They are suitable when records are related to each other but remain independent. Linked records may be used to represent hierarchical relationships but are not limited to them. Each record can be created and published separately, potentially by different authors or institutions, and have distinct access rights and versioning.

**Hierarchical records** are not yet implemented in CESNET Invenio repositories. We use this term for records that are related in a hierarchy but are not independent units that can be created and published separately. Publishing the root record would publish the entire hierarchy. The depth of the hierarchy is not limited. Access control for the whole hierarchy would be determined strictly by the access control of the root record. Conceptually, such records can be viewed as a single record with large metadata and potentially many files that are split into multiple records for performance or manageability reasons.

The following terms will be used to discuss records relative to their position within a hierarchy:

- root record (has no parent)
- parent record (has a child)
- child record (has a parent)

## Table of contents

- [Linked records](#linked-records)
  - [The related_resource field](#the-related_resource-field)
  - [The pid-relation field type](#the-pid-relation-field-type)
  - [Record collections](#record-collections)
- [Hierarchical records](#hierarchical-records)
- [Related topics](#related-topics)
  - [Visualization of record graphs and hierarchies](#visualization-of-record-graphs-and-hierarchies)
  - [Alternative PIDs](#alternative-pids)
  - [Metadata comparison among sibling records](#metadata-comparison-among-sibling-records)
- [How to decide which linking strategy to use](#how-to-decide-which-linking-strategy-to-use)
  - [Implementation advice per use case](#implementation-advice-per-use-case)

## Linked records

CESNET Invenio repositories can link one record to another, either within the same repository or to an external resource.

This differs from relational databases. Repositories are decentralized, so referential integrity cannot be enforced. If a referenced record is removed, the linking record is not notified and cannot prevent that removal.

In practice, this is usually manageable. Well-behaved repositories leave a tombstone record behind when a referenced record is removed, so the link often remains resolvable even after the original record is gone.

CESNET Invenio repositories currently support several linking patterns:

- Potentially external records linked through the CCMM `related_resource` field.
- Records in the same repository linked through the `pid-relation` data type.
- Query-based collections of records (for example, all measurements created with instrument ABC).

---

### The related_resource field

The [CCMM related_resource](https://techlib.github.io/CCMM/en/#RelatedResource) field is the standard way to link a record to another resource, especially an external one. It is similar to DataCite's relatedItem field.

Example of external related resource:

```xml
<related_resource>
    <iri>https://opendata.chmi.cz/air_quality/now/data/</iri>
    <title>Kvalita ovzduší – aktuální hodinové údaje</title>
    <resource_url>https://opendata.chmi.cz/air_quality/</resource_url>
    <resource_type>
        <iri>http://purl.org/coar/resource_type/FF4C-28RK</iri>
        <label xml:lang="en">observational data</label>
    </resource_type>
    <resource_relation_type>
        <iri>https://vocabs.ccmm.cz/registry/codelist/RelationType/IsDerivedFrom</iri>
        <label xml:lang="en">is derived from</label>
        <label xml:lang="cs">je odvozen od (čeho)</label>
    </resource_relation_type>
</related_resource>
```

Example of internal related resource:

```xml
<related_resource>
    <iri>https://doi.org/10.83100/8wej-ry35</iri>
    <title>DELPHI</title>
    <resource_url>https://doi.org/10.83100/8wej-ry35</resource_url>
    <resource_type>
        <iri>http://purl.org/coar/resource_type/RMP5-3GQ6</iri>
        <label xml:lang="en">collection</label>
    </resource_type>
    <resource_relation_type>
        <iri>https://vocabs.ccmm.cz/registry/codelist/RelationType/IsPartOf</iri>
        <label xml:lang="en">is part of</label>
        <label xml:lang="cs">je součástí</label>
    </resource_relation_type>
</related_resource>
```

Although the resource_relation_type in the example above comes from the vocabs.ccmm.cz registry, the vocabulary is extensible. [The values for the resource relation type vocabulary defined by CCMM can be viewed here](https://vocabs.ccmm.cz/RelationType/en/page/?uri=https://vocabs.ccmm.cz/registry/codelist/RelationType/). Terms "is part of" and "has part" for representing hierarchies are already defined. Your repository may define additional relation types that are not covered by that registry. To export records as CCMM 1.1.0-compliant records, those local terms must be mapped to the appropriate official term in vocabs.ccmm.cz.

CESNET Invenio provides an endpoint that can create this metadata automatically from a DataCite DOI or a Handle identifier. Because a Handle identifier does not itself contain all required metadata, the implementation tries to retrieve them directly from the repository that stores the related record. This may fail, in which case the user is prompted to provide the missing metadata manually.

#### Advantages and limitations

Advantages:

- Standard way to link to resources
- Same implementation supports linking to internal and external resources
- Works directly with CCMM 1.1.0 export

Limitations:

- In the current CESNET Invenio implementation, additional metadata cannot be attached beyond what related_resource already supports
- All related resources are stored in a single, flat `related_resource` field, and there is no mechanism to group these items into separate contextual sets. For example, if the record were to link to several resources of two different types (e.g. *instrument* and *sample*), they all sit side by side as one undifferentiated list:

```json
"related_resources": [
{sampleA},
{instrumentA},
{sampleB},
{instrumentB},
]
```

From this it is not possible to reliably tell which items belong together — i.e. that `sampleA` pairs with `instrumentA` and `sampleB` with `instrumentB`. There is no way to associate them as linked groups, such as:
```json
"related_resources": [
[{sampleA}, {instrumentA}],
[{sampleB}, {instrumentB}]
]
```

---

### The pid-relation field type

The [`pid-relation` data type](https://nrp-cz.github.io/docs/customize/model_backend/model_reference#pid-relation) is used to link records within the same repository. It is not part of the CCMM standard, but it is supported by CESNET Invenio. In Invenio the `pid-relation` data type is often used to reference terms from external controlled vocabularies but is general enough to link to any resource, including internal records.

To create a `pid-relation` link, add a definition to metadata.yaml. In this example, a new field called `instrument` is created to link a dataset record to the record of the instrument that measured the data.

```yaml
instrument:
  type: pid-relation
  model: "models.instruments.instruments_model"
  label:
    en: Instrument
  help:
    en: Reference to the instrument that was used to collect the data.
  keys:
    - metadata.name
    - metadata.serial_number
```

In some cases, you may want to use a vocabulary instead of the generic pid-relation type. This is useful when the linked record is a controlled vocabulary term that can be defined only by repository administrators, not by depositors. A controlled vocabulary helps keep linked records consistent and aligned with repository standards.

In that case, the configuration would look like this:

```yaml
instrument:
  type: vocabulary
  vocabulary-type: "instruments"
  label:
    en: Instrument
  help:
    en: Reference to the instrument that was used to collect the data.
```

When creating a record, use the PID value of the instrument record in the instrument field:

```json
{
  "instrument": {
    "id": "<PID value of the instrument record>"
  }
}
```

The name and serial_number fields are then populated automatically from the linked record. When the record is retrieved, the field looks like this:

```json
{
  "instrument": {
    "id": "<PID value of the instrument record>",
    "name": "<name of the instrument>",
    "serial_number": "<serial number of the instrument>",
    "@v": "<version identifier of the instrument record>"
  }
}
```

#### Advantages and limitations

Advantages:

- Supports controlled vocabularies
- Allows linked records to be placed anywhere in the metadata, including inside structured lists (for example, multiple measurements within one dataset, each with its own instrument)

Limitations:

- Linked records must be stored in the same repository
- Export to CCMM is not automatically supported, because there is no single obvious mapping for the metadata. Depending on the use case, some parts might map to `related_resource` and others to `provenance`.

---

### Record collections

A collection is a group of related records defined by a query. If a record matches the query, it automatically becomes part of the collection. Invenio then displays a separate page for each collection and lists all matching records there. For more detail see the [InvenioRDM documentation of the feature](https://inveniordm.docs.cern.ch/operate/customize/collections/).

#### Advantages and limitations

Advantages:

- Requires almost no setup: define the query and let Invenio do the rest

Limitations:

- Collections cannot contain their own metadata beyond the collection name
- A PID such as a DOI cannot be generated automatically for collections
- Records are not added to collections manually or through a curator workflow; they are included automatically based on the query
- Collections do not have versions

## Hierarchical records

As stated above, we view a hierarchical record conceptually as a single record that is split into multiple records for performance or manageability reasons. This means:

- There is a single publish action for the entire hierarchy. Child records cannot be published or retracted independently.
- If a file needs to be updated, the entire hierarchy must be republished to keep the PIDs consistent, and a new version of the root record will be created and assigned a new PID (such as a DOI). Just like all records in Invenio, the root record will have a stable concept DOI referring to all versions of the record. Changing the metadata of a child record does not require republishing the entire hierarchy.
- Access rights are managed on the root record.

To represent the hierarchical relationship, we do not use links from the parent record to the child records, because there may be too many such links, and the parent record would become too large. Instead, the relation is reversed: each child record stores its relation to the parent record.

In the technical implementation, each child record has a parent-in-hierarchy pid-relation field that points to its immediate parent record. This field ensures that:

- permissions defined on the parent record are always used
- when the parent is published, the child record is published as well
- when the parent is deleted, the child record is deleted as well

By design, a child record can only have one parent record: we do not support the use case where a child record has multiple parent records. The reason is primarily administrative. With multiple parents, it would be unclear how to handle permissions. For example, one parent might mark the record as open while another marks it as restricted. In the UI, the landing page of a parent record would show a list of child records, including pagination and sorting options, because the number of child records may be very large.

In the editor, there would be a Child records tab. It would show a paginated list of child records linked to the parent record, together with a Create &lt;something&gt; button that would allow the user to create a new child record within the hierarchy.

**Advantages and limitations**

Advantages:

- Easier for users to understand: there is a single curation process for the main record and its child records

Limitations:

- Not suitable for generic graphs (e.g. child records can have multiple parents) or for cases where records in the hierarchy need different access control settings
- Export to CCMM is not automatically supported, because there is no single obvious mapping for the metadata. Depending on the use case, some parts might map to related_resource and others to provenance.

## Related topics

### Visualization of record graphs and hierarchies

This cannot be implemented in a fully generic way because the requirements vary too widely. In one repository, there may be only a few child records that can be visualized as a directed acyclic graph. In another repository, a parent record may have thousands of child records, which cannot be visualized in the same way.

CESNET Invenio provides an API for retrieving the metadata of child records. A specific repository can then use this API, for example from React components, to build its own visualization of the record hierarchy.

---

### Alternative PIDs

A common use case is to assign DOI for the root records but use an alternative PID for child records. DOI can be unnecessarily robust for child records especially if they can't fullfil DOI metadata requirements and if it is not expected for child records to be cited independently or be available long term. It is also important to factor in the potential direct cost of very large amounts of DOI even though it would be mitigated by the financial support from the National DOI Centre at the National Library of Technology.

The only external PID CESNET Invenio currently supports is DOI.

**Internal "PID" of Invenio**

Invenio itself assigns an internal "PID" that does not have a naming convention. To make it persistent even during migrations, one would have to write a PID resolver. CESNET Invenio did something like this for NMA, where the contents of the PID field (DOI, handle) was used and if left empty, a new one is minted (and used in the URL to access the record). However, it will not be persistent in case the domain changes.

**Handle**

In principle, there is no problem, but it requires a Handle server, and it must be agreed who will run it, whether each repository has its own or if there will be a central NRP instance. However, running a handle server is not that complicated. There is a maintained python library, so it might not be that difficult to connect it to Invenio.

**ePIC**

ePIC is based on handle, so the handle infrastructure is reusable. There are several ePIC providers, but they do not have public prices. At the moment, it may be more affordable than DOI but is not guaranteed to always be cheaper. CESNET Invenio does not plan to maintain an ePIC plugin at the moment. InvenioRDM instances "B2SHARE v3" and "B2INST v3" use the [pyhandle v1 library](https://pypi.org/project/pyhandle/) to mint ePIC identifiers. However, they are built on a different version of Invenio that mints PIDs differently so the implementation is not reusable without work.

**ARK**

No additional external components are needed to technically support ARK in Invenio, the repository itself would serve as a resolver. Other advantages are the URL naming convention, registration in the registry, the ability to capture hierarchical relationships (it would be used especially for records of descendants that fall under the parent record with a DOI) and the fact that it would be possible to use ARK to create identifiers for files (if the repository team did not want to create a record for each photo just to assign it a PID). The biggest disadvantage is for repository instances serving multiple institutions - each institution has to assign its own NAAN (prefix). Either it would be necessary to implement a mechanism for deciding which NAAN to use for which record or only the NAAN of the institution operating the repository could be used. It is more like a stronger URL than a full PID, but it could be very functional for granular objects inside the repository, although not as future proof as something based on handle.

---

### Metadata comparison among sibling records

Sibling record metadata will be accessible via API for both linked and hierarchical records and can be displayed in a community-defined component. For example, it would be possible to display a diff of the json with record metadata for any two chosen siblings.

## How to decide which linking strategy to use

- Consider using controlled vocabularies. They are easy for users to understand and easy to map to related resources.
- Sometimes a single record is sufficient if the number of child records is small. The main limitation is that the record has only a single file table, so each part cannot have its own files.
- Use a hierarchy if your use case is essentially a large record split into smaller parts for performance or manageability reasons.
- Otherwise, use linked records. You may need to design a specialized user interface that reflects the use case, for example by replacing the default related-records tab with a custom implementation.
- Do not forget collections, they are a good way to group related records together (for example, yearly experiments).

### Implementation advice per use case

The CESNET Invenio Team analyzed the preliminary requirements of each repository and identified two main approaches. For each use case described by one of the repositories a specific implementation approach is suggested. [The analysis can be accessed here](https://gist.github.com/illyriab/85e4f41b012ce131abfaf17035e60763).

It is possible to discuss your specific use case with the NRP repository system specialists (nrp.repozitare@eosc.cz) or comment on the hub thread for this specification in [Discussions](../../../../discussions).

---

## Open questions

These points were raised in the original NRP-CZ discussion and remain open for this spec. Track them in the hub thread in [Discussions](../../../../discussions).

- **Terminology** — should "hierarchical records" be renamed to "compound records", with main-record / sub-record / record? (raised by xulman)
- **Schema constraints** — to what extent must a parent and its child records share the same metadata schema? They are not required to be identical, but there are technical requirements at publication time. (mesemus)
- **ePIC / pyHandle** — confirm which pyhandle versions are usable and whether any part of the B2SHARE/B2INST approach can be reused. (illyriab)

## Changelog

- **1.2** (2026-06-10) — version migrated from NRP-CZ Discussion #15; ePIC section updated with pyHandle library details.
