# A multimodal data catalog for frontier labs and AI-for-science

This is a design position, not a survey. It builds on the earlier report (`Datacat and data catalog systems.md`) and on three later research passes: the latest Rubin Butler design notes, public evidence on frontier-lab data practice, and public evidence on AI-for-science data practice. The supporting notes are in `research_notes/Datacat and data catalog systems/`. Most of the external evidence was seen as search extracts because many hosts were blocked, and no lab publishes its internal training-data systems, so statements about lab practice are inferred from job postings, court filings and talks.

This revision uses established terms from Datacat, iRODS, Rucio and the Butler wherever one exists, and it drops the earlier assumption that a dataset is a table. Much scientific and multimodal data is not tabular: n-dimensional arrays, sensor streams, hierarchical files and single large objects have no rows.

## The position

The catalog's job is to answer five questions about any data object and any model: what is it, where did it come from, where are its replicas, who may use it for what purpose, and what was derived from it. Lance, Iceberg, Zarr, Hugging Face, MLflow, iRODS and Rucio each answer one or two of these. None records usage rights as catalog state, and none joins all five. The system should register existing formats and engines without replacing them, and should hold the state that no other system holds: content identity across datasets, usage policy, the training-set specification, lineage from data to model to evaluation, and replica placement.

## Terminology

| Term used here | Meaning | Equivalent in prior systems |
|---|---|---|
| Collection | A node in the logical namespace that contains data objects and other collections | iRODS collection; Datacat folder |
| Data object | A logical unit of content, independent of where it is stored | iRODS data object; Datacat dataset; Rucio file DID |
| Version | An immutable revision of a data object, identified by content checksum | Datacat dataset version; iRODS has no equivalent |
| Replica | One physical copy of a version on a storage resource, with its own state and checksum | iRODS replica; Datacat location; Rucio replica |
| Storage resource | A place replicas are stored: a bucket and prefix, an HPC file system, a tape system | iRODS resource; Rucio storage element (RSE); Datacat site |
| Dataset | A named, versioned set of data objects treated as a unit | Datacat group; Rucio dataset DID; Butler collection |
| Snapshot | An immutable dataset version, identified by a hash of its contents | Proposed Butler collection snapshot; Iceberg snapshot |
| Metadata | Typed attribute–value–unit entries or nested documents attached to any entity | iRODS AVU; Datacat user metadata |
| View | A rule that resolves a logical name to a specific version and replica | Datacat view (version, site) |
| Replication rule | A declared requirement for where replicas must exist | Rucio replication rule; iRODS resource hierarchy |
| Zone | An independently administered catalog instance that can federate with others | iRODS zone |

Three terms are new because no prior system has them: **source** (the origin of data and the holder of its usage terms), **training-set specification** (called a mixture in SeqIO), and **similarity criterion** (a rule for deciding whether two elements count as the same for evaluation purposes).

## Not all data is tabular

The catalog must describe the internal structure of a data object without assuming rows. Each version carries a **structure descriptor**, extracted by a format plugin at registration. This generalises Datacat's crawler and the Butler's stored formatter name.

| Structure | Examples | Addressable element | Selector |
|---|---|---|---|
| Opaque object | Model weights, a PDF, a proprietary instrument file | The whole object | None |
| Container of members | tar, zip, HDF5 groups, ROOT trees, mmCIF | Member | Member path, byte range |
| Records | Parquet, Lance, JSON Lines, TFRecord | Record | Key or row range |
| N-dimensional array | Zarr, netCDF, OME-Zarr, FITS cubes | Chunk or region | Hyperslab over named dimensions |
| Stream | MCAP and ROS bags, video, audio, waveforms | Message or frame | Channel and time span |
| Graph | Knowledge graphs, meshes, interaction networks | Node, edge or subgraph | Key set |
| Composite | A robot episode (video, sensor streams, action records); a microscopy plate (images plus a design table) | A tuple of the above | Selectors joined on an alignment key such as time or well identifier |

Two consequences follow.

**Elements are enumerated only when the data enumerates them.** Records and container members can be listed in a manifest file. Arrays and streams cannot usefully be listed: a 2 PB reanalysis store has no natural rows, and a training sample is a window cut at load time. For these, the catalog stores the coordinate system (dimensions, coordinate values, chunk grid, channels, time base), and a sample is defined by a selector, not by an identifier in a list.

**Samples belong to the training-set specification, not to the data.** For records, a sample is usually a record. For an array, it is a windowing rule such as "two consecutive time steps over this region at this stride". For a whole-slide image it is a cropping rule; for a stream it is a clip or episode boundary. The same data object yields different samples under different specifications, so sample definition is recorded where it is chosen.

Metadata is not tabular either. It should follow the iRODS attribute–value–unit form with Datacat's typed values (string, number, timestamp), extended with nested documents validated against registered schemas. It attaches at every level: collection, data object, version, replica, and element.

## What the two audiences share, and where they differ

| | Frontier and large multimodal labs | AI-for-science startups |
|---|---|---|
| Scale | 10^4–10^5 datasets, up to ~10^11 elements per dataset, 10^2–10^3 PB | 10^8–10^9 elements, up to tens of PB, far richer metadata per element |
| Dominant structures | Records, streams, container members | Arrays, hierarchical files, composites, records |
| Where meaning lives | In the content and its source (crawl, licence, vendor, generating model) | In the experimental design and the measurement method |
| "The same element" means | Identical or near-identical bytes | Scientifically similar, or the same coordinates or event in time |
| Dominant governance | Copyright, licences with expiry, user opt-out and deletion, legal holds, regulator summaries, exfiltration control | Per-source contracts, patient consent, partner segregation, enclaves that forbid egress, audit trails, models as controlled derivatives |
| Storage resources | Several clouds plus rented GPU capacity | Cloud, rented GPUs and public HPC; raw data is often never copied |
| Lineage | Source → filtered corpus → training set → model → evaluation | The same, plus model → candidate → experiment → new data, repeated each generation |
| Team | Dedicated platform teams building in-house | A few data engineers who buy most of the stack |

The shared core is large, which argues for one system with two configuration profiles.

## Design decisions

### 1. Four planes, with the catalog outside the read path

| Plane | Holds | Scale | Technology |
|---|---|---|---|
| Catalog database | Collections, data objects, versions, replicas, datasets, snapshots, sources, specifications, runs, models, policies, storage resources | 10^4–10^9 rows | PostgreSQL |
| Manifests | Element listings where elements are enumerable, with their metadata, selectors and lineage fields | Up to 10^11 entries | Columnar files in object storage, registered by checksum |
| Derived indexes | Full-text, vector, content-identity and similarity indexes | Rebuildable | Tied to the snapshot and model version that produced them |
| Data | The bytes | — | Object storage, HPC file systems, tape |

A row per file in PostgreSQL, as the earlier report proposed, does not hold for this audience. AlphaFold DB is about 644 million files packed into roughly a million tar archives, and web corpora are hundreds of billions of small documents. The catalog database therefore tracks data objects at the level they are stored (an archive, a shard, an array store), and members or records inside them are listed in manifests.

The catalog must not proxy reads. This is a deliberate departure from iRODS, where all I/O passes through its servers. Meta reported removing a proxy from its read path because storage "remains a major cause of GPU stalls". A training job obtains a resolved manifest and scoped credentials once, then reads from storage directly.

### 2. Snapshots are immutable, hash-identified, and layered

A dataset snapshot lists its member data-object versions, together with any **annotation sets**: independently versioned collections of derived attributes (captions, quality scores, embeddings, safety labels, new assay readouts) keyed by element identifier or selector. Adding an annotation set creates a new snapshot that shares everything else with its parent.

A snapshot is identified by a Merkle hash over its members. This is the Butler's newest proposal applied here. Unchanged parts are shared between snapshots, a snapshot can be verified from its identifier, and membership can be proven later without retaining the content.

Datacat's explicit version numbers stay. The Butler has no version field and expresses versions through collection search order; its own portability note says this blocks incremental workflows.

### 3. Content identity depends on structure

Every version has a content checksum, and several algorithms must be supported because storage resources differ in what they provide. Beyond that, identity is defined per structure:

- **Records and members:** an exact checksum per element, then near-duplicate fingerprints per modality (MinHash for text, perceptual hashes for images and video).
- **Arrays:** a checksum per chunk, plus coordinate identity. The same grid cell in a raw GRIB file and in an analysis-ready Zarr store is the same datum although the bytes differ.
- **Streams:** channel and time span on a declared time base.
- **Domain entities:** a canonical key such as an InChIKey, a sequence cluster at a stated identity threshold, or a structure prototype label.

The catalog maintains a **content-identity index** from fingerprint or key to every place the element occurs. This answers "where does this content appear, in any dataset, in any form". The need is documented: in an August 2026 court filing OpenAI said its tools "can't execute searches across all training datasets and output logs" for specific articles. It is also the hardest component to build at 10^11 elements and should begin with exact checksums and domain keys only.

Elements also carry a logical identifier assigned at the source and preserved through every transformation. Pipelines must cooperate to preserve it; it cannot be reconstructed afterwards.

### 4. Usage policy attaches to sources

A **source** is the unit of origin and the holder of usage terms.

| Source type | Records |
|---|---|
| Crawl | Domain, fetch time, crawler identity, robots and opt-out state at fetch |
| Licence or partner delivery | Contract, permitted uses, term and expiry, deletion obligations |
| Human-data campaign | Vendor, pseudonymous worker, rubric version, quality-assurance outcome |
| User data | Consent tier, retention class |
| Model-generated | Generating model, prompt version, seed, parent elements |
| Instrument run | Instrument, calibration, protocol version, reagent lot, batch, planned versus executed procedure |
| Computation | Method record: level of theory, functional, code version, convergence settings |
| Enclave or federated partner | Access conditions; possibly no readable replica |

Each data object or element references its source. Policy is then evaluated over millions of sources and not over 10^11 elements. Per-element exceptions such as a takedown, an opt-out or a retraction are recorded as an exclusion list, which is an annotation set.

The policy vocabulary should stay small: permitted **purposes** (training, evaluation only, never training, research only, non-commercial), **compartments**, **jurisdiction**, **hazard class**, **retention**, and **obligations** that pass to derived objects (attribution, expiry, destruction at close-out). Obligations pass to models. Science already requires this: an NIH notice of March 2025 treats generative-model weights trained on controlled-access genomic data as a controlled derivative.

Access control follows Datacat and iRODS practice: group-based permissions inherited through the collection hierarchy, with iRODS-style tickets for delegated access. Usage policy is a separate layer, because permission to read is not permission to train.

### 5. The training-set specification is a catalogued object

No public system records the training mixture as catalog state. SeqIO and OpenVLA define mixtures in code and distinguish versions by name suffix. The mixture is nevertheless what a model was trained on.

Three objects are needed:

- **Specification:** declarative. Component snapshots or queries, filters, weights, sample definition (the windowing, cropping or clipping rule for non-record data), sampling order, transformations such as tokenizer version, and seed. Specifications can nest.
- **Pinned specification:** the specification with every component fixed to an exact snapshot hash and every weight to a number, together with the policy decision made at that time and its inputs.
- **Materialisation:** the files written from a pinned specification, in a given representation, on given storage resources.

Policy is enforced when a specification is pinned: never-train data, expired licences, excluded elements and held data are removed, or pinning fails. Because the decision is stored, the question "why was this in the training set" remains answerable later.

Evaluation suites and train/test splits are the same object with a protection class.

### 6. Contamination and leakage share one mechanism

Frontier labs are concerned with evaluation items appearing in training corpora. Science is concerned with test proteins that are 90% identical to training proteins, and with test dates that fall inside the training period. Both reduce to one statement: no training element and test element are related under criterion C above threshold T.

**Similarity criteria** are registered, versioned objects: exact checksum, near-duplicate fingerprint, sequence identity, chemical scaffold, structure prototype, coordinate or time-range overlap, event date before a cutoff. A split or evaluation suite declares which criteria it must satisfy against which pinned specifications, and the catalog verifies this using the content-identity index or a precomputed similarity table. For arrays and streams the check is a range-overlap test on coordinates, which needs no enumeration. Runs N' Poses and PLINDER publish similarity tables by hand today, and OMat24 defines its training set by exclusion against a benchmark.

Protected evaluation sets also have registered canary strings, audited access, and catalog entries for holdout sets held by third parties.

### 7. Replication rules apply to snapshots on storage resources

Rucio's replication rules are the right model, with the unit raised from the file to the snapshot or materialisation. The useful statement is "pinned specification P, in representation R, is complete and verified on storage resource S as of time T". The Butler team's draft diagnosis of its multi-site problems lists the absence of a per-site completeness check.

- **Replica state** follows iRODS: each replica has its own status and checksum, with states for registered, verified, suspect and lost.
- **Storage resources** carry provider, region, jurisdiction, storage class, egress cost, and whether data may leave.
- **Views** generalise Datacat's (version, site) pair: a logical name resolves to the cheapest verified replica for the caller's compute location.
- **Representations** of one snapshot (raw, cloud-optimised, analysis-ready, mirrored) are linked, as in ARCO-ERA5's tiers, and replicas are byte-identical only within a representation.
- **Run location is recorded.** In Getty v. Stability the UK training claim was dropped because training took place elsewhere, so the location of a run is a legal fact.
- **Unreadable data is still catalogued.** UK Biobank and All of Us forbid individual-level egress. The catalog holds metadata, policy and lineage for data with no reachable replica, and a model trained inside the enclave inherits its obligations.
- **Integrity is verified at materialisation** against the checksum recorded at registration, which also defends against corpus poisoning between indexing and download.
- **Zones federate.** Independently administered instances exchange snapshot descriptions and lineage, as iRODS zones do, so a partner or national facility need not share a database.

### 8. Provenance is recorded at registration and stored in two parts

Provenance follows W3C PROV: entities are immutable versions and snapshots, activities are runs, and agents are people, services, instruments and models. Relations are `used` (with a role such as training, validation, test or reference), `wasGeneratedBy` and `wasDerivedFrom`.

The relation records and a searchable summary are held in the catalog database. The full run record (configuration, logs, per-task detail) is stored as a catalogued data object. The Butler team moved provenance out of database tables for this reason: schema changes became easier and the catalog no longer depends on pipeline internals.

Ancestry queries must filter by role. The Butler team observes that unfiltered ancestry is useless because a shared calibration links every raw file to every output; tokenizers and base models have the same effect here.

For non-enumerated data, element-level lineage is the selector recorded in the specification (for example a time range and region), not a list of identifiers.

### 9. Deletion is a tracked procedure, and legal holds override it

Immutability and deletion conflict, so the design must define their interaction. A takedown creates a tombstone. The catalog enumerates every occurrence through the content-identity index and lineage (derived shards, embeddings, caches, replicas), deletes each, and records confirmation per replica. Snapshots containing the element are marked non-materialisable and receive a redacted successor. The original snapshot hash is retained as evidence of what existed.

A **legal hold** overrides deletion and moves data to a segregated retention compartment, with the reason recorded. Both directions occur in practice: one settlement required destruction of specific files and their copies, while a preservation order in another case required retaining everything.

Models trained on tombstoned data are flagged with the affected element count. The response is a recorded human decision.

### 10. Policy enforcement points, not a general rule engine

iRODS showed the value of policy enforcement points around catalog operations, and also the cost of a general rule language executing in the request path. This design keeps a small, fixed set of enforcement points (registration, pinning a specification, materialisation, export, deletion) with declarative policies, and performs everything else asynchronously from a change log.

Bulk registration goes through a batched ingest path. The Butler team measured inserts from hundreds of concurrent workers taking up to ten minutes before adopting a single queue-fed writer.

## Requirements specific to science

- **The method is the instrument.** Computed labels need a structured, comparable method record. Datasets from different methods are merged only under a declared equivalence; Materials Project data mixing two DFT settings is the cautionary case.
- **Assays are versioned.** One study found that 27% of pooled ChEMBL IC50 pairs differ by more than one log unit. Replicates, censored values, failed runs and negative results are data and are retained.
- **Batch structure is metadata.** Controls are catalogued elements, and each element links to its batch and control plate.
- **Reference data joins at query time.** Datacat's foreign metadata, the Butler's dimension records and MetaCat's external filters all join element metadata against reference tables of runs, exposures or calibrations. This remains a core feature.
- **The experimental loop is recorded.** A model produces a candidate with a canonical identity, an experiment tests it, results enter a new snapshot, and the next model trains on it. Because versions are immutable the lineage remains acyclic across generations.
- **Redaction tiers.** RxRx3 anonymises most of its genes; blinded attributes with later unblinding should be supported.
- **Audit.** Append-only audit trails and enforced, logged separation of test data, as a draft EU manufacturing annex requires.

## Requirements specific to frontier labs

- **Agents as clients.** OpenAI built an agent to navigate 70,000 datasets. Discovery must be available through a tool interface that executes with the caller's permissions.
- **Compartments and export control.** Access tiers, two-party approval for bulk export, and audit of anomalous read volume. Applying model-weight security controls to data is my inference.
- **Regulatory reports as queries.** The EU training-content summary asks for the largest scraped domains by size, crawler names and collection periods, which is an aggregation over source attributes of a pinned specification.
- **Trajectories.** Agent transcripts and robot episodes need a standard episode-and-step schema, treated as a composite structure.
- **Human and synthetic data.** Both need typed sources, because synthetic data is a contamination path and human data carries vendor terms.

## Queries the system must answer

1. Which models were trained on anything from source X, and on how many elements?
2. Produce the training-content summary for model M.
3. Is evaluation set E contaminated in model M's pinned specification under criterion C?
4. On which storage resources can specification S be trained today without moving data, and what would staging it elsewhere cost?
5. List every occurrence of this document, image, sequence or array region, in any dataset, representation or storage resource.
6. A licence expires next month: which specifications, models and caches are affected?
7. Which predictions from model generation 3 were tested, with what result, and which results were used to train generation 4?
8. Reproduce the training set of a model from last year, and report what can no longer be included.

## Changes from the earlier report

| Earlier | Now |
|---|---|
| Row per file in PostgreSQL | Row per stored data object; members and records listed in registered manifests; arrays and streams described by coordinates |
| A dataset is a set of files | A dataset is a set of data objects of any structure, with annotation sets |
| Provenance records in PostgreSQL | Relations and summaries in PostgreSQL; full run records as catalogued data objects |
| Replication rules over files | Replication rules over snapshots, with completeness per storage resource |
| Licence propagation as a later feature | Source-level policy and enforcement at pinning are core |
| Build order: catalog, replicas, tables, AI layer, placement | Snapshot registry with specifications and lineage first; identity and policy second; replication third |

The design keeps more of Datacat and iRODS than the previous revision suggested: the logical namespace of collections, data objects with versions and replicas, typed attribute–value–unit metadata, group-based inherited access control, views, a readable query language, and zones. What it adds lies above that layer.

## First users

Frontier labs are recruiting to build this internally. Anthropic's research data platform posting asks for engineers to "develop dataset management, data cataloging, and provenance tooling". This confirms the need and makes them unlikely early adopters of an external system.

AI-for-science startups have the clearest gap and small teams that buy software. Mid-size multimodal labs in video and robotics have adopted open components such as Lance and Ray and lack the layer above them. These are the realistic first users. The frontier-lab requirements remain in the design because regulation and litigation are extending the same obligations to everyone.

A first release that is useful alone would be a snapshot registry over existing Lance, Parquet, Zarr and HDF5 data in object storage, with training-set specifications, pinning, and lineage to training runs imported from MLflow or Weights & Biases, delivered as one service on PostgreSQL.

## Open risks

- **Identity at scale.** A near-duplicate index over 10^11 elements across modalities is a research problem. Begin with exact checksums, coordinates and domain keys.
- **Structure descriptors.** Each format needs a plugin, and array and stream coordinate systems are easy to describe incorrectly. Croissant 1.1 and the CF conventions should be reused where they apply.
- **Pipeline cooperation.** Logical identifiers and source references must be preserved by every transformation. Without integrations for Ray, Spark, Daft and common data loaders, these fields will be empty.
- **Policy scope.** A general policy language would overwhelm the project. Purposes and obligations should remain enumerable until users require more.
- **Ubiquitous inputs.** Tokenizers, base models and calibrations link everything to everything, so ancestry queries need role filters by default.
- **Scope.** The system must not become a storage format, a query engine or a transfer tool.
- **Evidence.** Lab internals are not public. Several gaps are inferred from the absence of public claims, and a competitor may already cover them privately. LaminDB is the closest open-source neighbour on the science side and should be examined before building.
