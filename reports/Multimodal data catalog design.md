# A multimodal data catalog for frontier labs and AI-for-science

This is a design position, not a survey. It builds on the earlier report (`Datacat and data catalog systems.md`) and on three later research passes: the latest Rubin Butler design notes, public evidence on frontier-lab data practice, and public evidence on AI-for-science data practice. The supporting notes are in `research_notes/Datacat and data catalog systems/`. Most of the external evidence was seen as search extracts because many hosts were blocked, and no lab publishes its internal training-data systems, so claims about lab practice are inference from job postings, court filings and talks.

## The position in one paragraph

The catalog should not be a file catalog with AI features added. Its job is to answer five questions about any byte and any model: what is this, where did it come from, where is it now, who may use it for what, and what was built from it. Lance, Iceberg, Zarr, Hugging Face, MLflow and Rucio each answer one of these well. Nothing answers the fourth, and nothing joins them. The system should therefore register existing formats and engines as physical forms, and own the four things nobody else holds as state: **content identity across datasets, legal and policy state, the training recipe, and the lineage from data to model to evaluation to (in science) the experiment that tested the model's prediction.** Replica placement is the fifth, and it matters because GPUs are rarely where the data is.

## What the two audiences have in common, and where they differ

| | Frontier and large multimodal labs | AI-for-science startups |
|---|---|---|
| Scale | 10^4–10^5 datasets, up to ~10^11 rows per table, 10^2–10^3 PB | 10^8–10^9 rows, up to tens of PB, far more columns per row |
| Where meaning lives | In the content and its source (crawl, licence, vendor, generator model) | In the experimental design and the measurement method |
| "Same sample" means | Same or near-same bytes | Scientifically similar (sequence identity, scaffold, structure prototype) or same event in time |
| Dominant governance | Copyright, licences with expiry, user opt-out and deletion, legal holds, regulator summaries, exfiltration control | Per-source contracts, patient consent, partner segregation, enclaves that forbid egress, audit trails, models as controlled derivatives |
| Where data sits | Several clouds plus rented GPU capacity | Cloud, rented GPUs and public HPC; the raw tier is often never copied |
| Lineage shape | Source → filtered corpus → mixture → model → evaluation | The same, plus model → candidate → experiment → new data, repeating each generation |
| Team | Dedicated platform teams, mostly building in-house | A handful of data engineers who buy most of the stack |

The shared core is large: immutable content-addressed bytes packed into shards, samples addressed by pointers into those shards, dataset snapshots, replicas across sites, derived indexes, and a record of what each training run consumed. The differences are in metadata vocabulary and policy, which argues for one core with two profiles, not two products.

## Eight design decisions

### 1. The catalog tracks artifacts; samples live in manifests

The earlier report kept a row per file in Postgres. That does not survive this audience. Web corpora are "hundreds of billions of little files", AlphaFold DB is about 644 million files inside a million tar shards, and robotics and video datasets pack many episodes or clips into each file. The revised split is:

| Plane | Holds | Scale | Technology |
|---|---|---|---|
| Control | Datasets, snapshots, sources, recipes, runs, models, policies, locations, replica state per artifact | 10^4–10^9 rows | Postgres |
| Manifest | Sample tables: scalar columns, pointer columns, lineage columns, fingerprints | up to 10^11 rows | Columnar files (Lance, Parquet, Iceberg) in object storage, registered by digest |
| Index | Text, vector, content-identity and similarity indexes | derived | Rebuildable, tied to the snapshot and model version that produced them |
| Data | The bytes | — | Buckets, HPC file systems; the catalog is never in the read path |

An **artifact** is a shard, a file, a Zarr store or a table snapshot. A **pointer cell** addresses a byte range, a row range, a tar or zip member, an HDF5 dataset, a Zarr chunk, a frame or time span, or a (topic, time range) in a sensor log. The Butler's move to zip aggregation with member addressing, LeRobot v3 and MCAP all converged on this independently.

The catalog staying out of the read path is not optional. Meta reported removing a proxy from its read path because storage "remains a major cause of GPU stalls". A training job should get a resolved manifest and scoped credentials once, then read directly.

### 2. A dataset snapshot is a set of column groups over a stable row space

Training examples keep acquiring columns: captions, quality scores, embeddings, safety labels, legal flags, new assay readouts. Rewriting the dataset for each is what makes versioning expensive. A snapshot should be a list of independently versioned **column groups** joined on sample ID, each with its own producer and provenance. Adding an annotation layer creates a new snapshot that shares every other group with its parent.

A snapshot is identified by a Merkle hash over its column-group digests. This is the Butler's newest proposal (hash-identified immutable collection snapshots) applied one level down, and it gives three properties at once: unchanged parts are shared, a snapshot can be verified from its ID, and membership can be proven later without holding the content.

### 3. Sample identity has two parts, and one of them is pluggable

Every sample gets a **logical ID** assigned at its source and carried as a column through every transform. Pipelines must cooperate here; it cannot be inferred afterwards.

Every sample also gets **fingerprints**. Exact content hashes come first. Near-duplicate fingerprints are per modality: MinHash for text, perceptual hashes for images and video, embedding neighbours where nothing cheaper works. For science, the fingerprint is a domain key: an InChIKey, a sequence cluster at a stated identity threshold, a structure prototype label.

The catalog owns a **global identity index** from fingerprint to (snapshot, row). This is the piece that answers "where does this content appear, anywhere, in any form". Its absence is documented: in an August 2026 court filing OpenAI said its tools "can't execute searches across all training datasets and output logs" for specific articles. It is also the hardest piece to build at 10^11 rows, and should start exact-only.

### 4. Legal and policy state attaches to sources, with per-sample exceptions

A **source** is the unit of origin, and it is where rights live:

| Source type | Carries |
|---|---|
| Crawl | Domain, crawl time, crawler identity, robots and opt-out state at fetch |
| Licence or partner delivery | Contract, permitted uses, term and expiry, deletion obligations |
| Human-data campaign | Vendor, pseudonymous worker, rubric version, QA outcome |
| User data | Consent tier, retention class |
| Model-generated | Generator model, prompt version, seed, parent samples |
| Instrument run | Instrument, calibration, protocol version, reagent lot, batch, plan versus what actually ran |
| Computation | Method record: level of theory, functional, code version, convergence settings |
| Enclave or federated partner | Access conditions; possibly no readable replica at all |

Every sample row carries a source ID. Policy is then evaluated over a few million sources, not a hundred billion samples. Per-sample exceptions (a takedown, an opt-out, a retraction) are a deny-set layer, which is just another column group.

Policy itself should stay small: a set of **purposes** (train, evaluate only, never train, research only, non-commercial), **compartments**, **jurisdiction**, **hazard class**, **retention**, and **obligations** that flow to derived things (attribution, expiry, destroy on close-out). Obligations propagate to models. That is already required in science: an NIH notice of March 2025 treats generative-model weights trained on controlled-access genomic data as a controlled derivative.

### 5. The recipe is a first-class object, and resolving it is the policy gate

No public system models the training mixture as catalog state. SeqIO and OpenVLA define mixtures in code, with versions distinguished by name suffix, and Lance tags pin data but not weights or sampling. Yet the mixture is what a model was actually trained on.

Three objects, in order:

- **Recipe:** declarative. Components (snapshot references or queries), filters, weights, sampling and curriculum, transforms such as tokenizer version, seed. Recipes nest.
- **Resolution:** the recipe frozen. Exact snapshot digests, numeric weights, and the policy decision made at that moment, with its inputs.
- **Materialization:** the shards actually written, in which representation, at which locations.

Policy is enforced at resolution: never-train data, expired licences, opted-out samples and held data are excluded or the resolution fails. Because the decision is recorded, "why was this in the training set" has an answer a year later.

Evaluation suites and train/test splits are the same object with a protection class. That makes a split verifiable, which leads to the next decision.

### 6. Contamination and leakage are one mechanism

Frontier labs worry about evaluation items leaking into training corpora. Science worries about test proteins that are 90% identical to training proteins, or test dates that fall inside the training period. Both are the same statement: *no pair of (training sample, test sample) is related by relation R above threshold T.*

So the catalog should have **leakage relations** as registered, versioned objects: exact hash, near-duplicate fingerprint, sequence identity, scaffold, structure prototype, event date before a cutoff. A split or evaluation suite declares which relations it must satisfy against which resolutions, and the catalog checks it using the identity index or a precomputed similarity table. Runs N' Poses and PLINDER already ship such tables by hand; OMat24 defines its training set by exclusion against a benchmark.

Protected evaluation sets also get registered canaries, audited access, and entries for holdouts held by third parties.

### 7. Placement is about complete snapshots at sites, not files

The useful statement is "resolution R, in representation P, is complete and verified at site S as of time T". The Butler's own draft diagnosis lists the lack of a per-site completeness check among its multi-site failures.

- **Locations** carry provider, region, jurisdiction, storage class, egress cost, and whether data may leave at all.
- **Placement intents** say where a resolution must be and by when; pluggable movers do the copying.
- **Resolution by locality** picks the cheapest verified copy for the caller's compute.
- **Run location is recorded.** In Getty v. Stability the UK training claim was dropped because training happened elsewhere, so where a run executed is a legal fact.
- **Unreadable data still gets an entry.** UK Biobank and All of Us forbid individual-level egress; the catalog holds metadata, policy and lineage for data it can never fetch, and a model trained inside the enclave still inherits its obligations.
- **Integrity is checked at materialization** against the hash recorded at index time, which is also the defence against corpus poisoning between indexing and download.

### 8. Deletion is a process with receipts, and holds override it

Immutability and deletion conflict, and the design has to say how. A takedown creates a tombstone. The catalog enumerates every copy through the identity index and lineage (derived shards, embeddings, caches, replicas), deletes each, and records a receipt per replica. Snapshots containing the sample are marked non-materializable and get a redacted successor; the old snapshot's hash remains as evidence of what existed without the content.

A **legal hold** overrides deletion and routes data to a segregated retention compartment, with the reason recorded. Both directions are real: one settlement required destroying specific files and their copies, while a preservation order in another case required keeping everything.

Models trained on tombstoned data are flagged with the affected sample count. What to do about them is a recorded human decision, not an automatic action.

## What is specific to science

- **The method is the instrument.** Computed labels need a structured, comparable method record. Datasets from different methods merge only under a declared equivalence; Materials Project data mixing two DFT settings is the cautionary case.
- **Assays are versioned.** One study found 27% of pooled ChEMBL IC50 pairs differ by more than a log unit. Replicates, censored values, failed runs and negative results are data and must be retained.
- **Batch structure is metadata.** Controls are rows, and each sample links to its batch and control plate.
- **The loop closes.** A model produces a candidate with a canonical identity; an experiment tests it; results enter a new snapshot; the next model trains on it. Because versions are immutable, this is still an acyclic graph across generations. Nothing found in the research links these steps in an open standard.
- **Redaction tiers.** RxRx3 anonymises most of its genes; a catalog should support blinded columns with later unblinding.
- **Audit.** Append-only audit trails and enforced, logged separation of test data, as a draft EU manufacturing annex requires.

## What is specific to frontier labs

- **Agents as users.** OpenAI built an agent to navigate 70,000 datasets. Discovery must work through a tool interface that runs with the caller's permissions.
- **Compartments and export control.** Access tiers, two-party approval for bulk export, and read-volume anomaly audit. Applying weight-security controls to data is my inference.
- **Regulator reports as queries.** The EU training-content summary asks for the top scraped domains by size, crawler names and collection periods. That is a group-by over source columns of a resolution.
- **Trajectories.** Agent transcripts and robot episodes need a canonical episode-and-step schema.
- **Human and synthetic data.** Both need typed provenance, because synthetic data is a contamination path and human data carries vendor terms.

## Acceptance queries

A design like this is easiest to judge by what it must answer. Each should be one query:

1. Which models were trained on anything from source X, and how many samples each?
2. Produce the training-content summary for model M.
3. Is evaluation set E contaminated in model M's training resolution, under relation R?
4. Where can I train recipe R today without moving data, and what would it cost to stage it at cluster C?
5. Show every copy of this document, image or sequence, in any dataset, in any form, at any site.
6. This licence expires next month: which recipes, models and caches are affected?
7. Which predictions from model generation 3 were tested, with what result, and which results trained generation 4?
8. Reproduce the exact training set of a model from last year, and tell me what could no longer be included.

## What changes from the earlier report

| Earlier | Now |
|---|---|
| Row per file in Postgres | Row per artifact; members and samples in registered manifests |
| Folder hierarchy as the organising idea | Datasets, sources and recipes as the organising ideas; paths are aliases |
| Provenance records in Postgres | Edge skeleton and summaries in Postgres; full run records as catalogued files (the Butler's conclusion) |
| Replication rules over files | Completeness of a resolution at a site |
| Licence propagation as a later feature | Source-level policy and the resolution gate are core |
| Build order: catalog, replicas, tables, AI layer, placement | Snapshot registry with recipes and lineage first; identity and policy second; placement third |

Honestly stated, this is no longer a successor to Datacat. It inherits three of Datacat's ideas: logical datasets separate from physical replicas, resolving the right copy for where the caller is, and a readable filter language.

## Who to build for first

Frontier labs are hiring to build this themselves. Anthropic's research data platform posting asks for people to "develop dataset management, data cataloging, and provenance tooling". That confirms the need and makes them unlikely first adopters of an outside system.

AI-for-science startups have the clearest gap and small teams that buy. Mid-size multimodal labs (video, robotics) have already adopted open components such as Lance and Ray and lack the layer above them. These two are the realistic first users. The frontier-lab requirements still belong in the design, because regulation and litigation are pushing the same obligations onto everyone.

A first version that is useful alone: a snapshot registry over existing Lance, Parquet and Zarr data in buckets, recipes with resolution, and lineage to training runs imported from MLflow or Weights & Biases, shipped as one service on Postgres.

## Open risks

- **Identity at scale.** A near-duplicate index over 10^11 samples across modalities is a research-grade problem. Start with exact hashes and domain keys.
- **Pipeline cooperation.** Stable sample IDs and source IDs must be carried by every transform. Without integrations for Ray, Spark, Daft and the common loaders, the columns will be empty.
- **Policy creep.** A general policy language will sink the project. Keep purposes and obligations enumerable until real users force more.
- **Ubiquitous inputs.** Tokenizers, base models and calibrations link everything to everything. Ancestry queries need role filters by default.
- **Scope.** The system must not become a storage format, a query engine or a transfer tool.
- **Evidence.** Lab internals are not public. Several gaps are inferred from the absence of claims, and a competitor may already cover them privately. LaminDB is the closest open-source neighbour on the science side and should be examined before building.
