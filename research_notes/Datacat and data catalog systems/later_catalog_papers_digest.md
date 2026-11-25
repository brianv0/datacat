# Digest of four later catalog papers

Reading notes, November 2026. A delegated reader read the four papers in full and wrote this digest. The main session checked the headline numbers and quotes against the paper texts. A shorter form of this digest is in `goods_and_later_papers.md`, section 3.

All four files were read in full. "§" is the paper's own section number, quotes are verbatim, and "(inference)" marks my reading, not a claim in the paper.

## 1. Unity Catalog (Chandra et al., Databricks; SIGMOD-Companion '25, Berlin, 13 pp.)

**Problem (§1).** Lakehouse catalogs have "inconsistent governance, narrow interoperability, and lack of support for data discovery" and are table-only. Stated requirements: consistent governance, universality, high performance.

**Data model (§3.2, §4.2)**
- Namespace: metastore (root, one home region) → catalog → schema → asset, named `catalog.schema.asset`.
- Leaf assets: tables, views, volumes, models, functions. The metastore also holds configuration securables: credentials, storage locations, connections.
- Volume: "a logical storage in a cloud object storage location for organizing files and non-tabular data". Per-file metadata inside a volume: not stated.
- Models: a RegisteredModel asset with multiple versions, child of a schema, added by implementing MLflow's RestStore and ArtifactRepository (§4.2.3).
- All types sit on a "generic entity-relationship data model" (§4.2.2): lookup by name or ID, parent-child listing, grants, a provisioning/cleanup state machine, and an extension point for lookup by path.
- New types are added via "a declarative manifest to UC's asset types registry" (hierarchy position, operations, privileges, authorization rules, lifecycle).
- Managed and external assets differ in who allocates paths; both obey "no two assets in a metastore can have overlapping storage paths" (§4.2.1). Soft delete, cascade to children, background garbage collection.

**Architecture and storage (§4.5, §5)**
- Multi-tenant Java/Scala service over a "standard relational database"; it "supports using various OLTP databases". The benchmark used MySQL (db.m5.24xlarge). Table and key design: not stated.
- Consistency: snapshot-isolated reads and serializable writes per metastore. A metastore version is persisted in the DB; a write increments it conditional on the node's in-memory version, and a mismatch triggers cache reconciliation (evict all, or selectively via change events). "This design avoids reliance on distributed-consensus-based services".
- Metastores are sharded across nodes (best-effort, non-exclusive ownership).
- Mutable metadata: write-through multi-version cache keyed by ID, name and path (hash-maps, versioned lists, URL-tries for path overlap). Credentials and immutable data: TTL caches. One batched API call per query.
- Scale (§1, §6.1): ~9,000 customers, >75K workspaces, 100K metastores, 200K catalogs, 4M schemas, >100M tables, 550K volumes, 400K models. Largest catalogs hold ≥500K tables.
- Load: ~60K requests/s, 98.2% reads. Working set is <100 MB for almost all metastores and <~10 MB for 90%. 90% of containers are re-accessed within 10 s, leaf assets within 100 s.
- Caching gives 3×–40× lower latency; without it throughput tops out below 10K requests/s. End-to-end performance is statistically equal to HMS in local mode (§6.2).

**Governance (§3.3, §4.3)**
- SQL-style grants: SELECT also needs USE SCHEMA and USE CATALOG. Every securable has an owner; MANAGE delegates.
- Privileges inherit to "all existing and future securables". Admin rights do not confer data rights: "a schema owner does not automatically gain SELECT access to tables in the schema".
- Credential vending: clients hold no storage credentials. UC returns a token "downscoped to only have access to the data asset in question", valid "tens of minutes"; "revoking privileges does not immediately block clients with active credentials".
- Path-based access: UC resolves the path to its unique asset and applies the same privileges. ~7% of tables are also accessed by path (§6.2).
- Row filters and column masks are enforced only by "trusted" engines isolated from user code; untrusted engines delegate to a data filtering service.
- ABAC over tags on securables or columns: private preview.
- Audit covers API requests, lifecycle changes and access decisions (§4.2.1).
- Lineage: engines submit it through lineage APIs (§1). The lineage data model: not stated.
- Federation (§4.2.4): a federated catalog mirrors a foreign one on demand, with mirroring done by the engine. UI-only clients "may see stale metadata until it is mirrored by some engine". 26 foreign table types; ~16% of tables are foreign.
- Open access (§1): Delta Sharing, UniForm, and an Iceberg REST Catalog interface.
- Discovery (§4.4): search and lineage are "second-tier" background services fed by metadata change events; they call a core authorization API to filter results.

**Comparisons (§2)**
- HMS "does not have governance (it relies on cloud storage policies for access control)", has no transactions, and is tables-only.
- Iceberg REST "does not specify governance APIs beyond temporary credential vending"; Polaris adds a governance API but is Iceberg-only.
- Glue, Fabric/Purview/OneLake, BigLake/Dataplex and Horizon are "not open" and table-only.
- Goods is cited once as applied metadata work, with no discussion. DataHub and Amundsen are not mentioned; Collibra and Alation are named as API consumers.

**Limits and open items (no dedicated section)**
- No open "trusted engine" standard (§4).
- Caching and some features are absent from the open-source version (§4).
- Revocation lag (§4.3.1).
- Federation staleness (§4.2.4).
- ABAC unreleased (§3.3).
- Multi-table transactions are "ongoing work" via catalog-owned Delta tables (§6.3).

## 2. Big Metadata (Edara and Pasumansky, Google; PVLDB 14(12):3083–3095, 2021)

**Problem (§1).** Systems either co-locate block metadata with data (must open each block) or centralise a small summary (limits scale). BigQuery instead treats metadata as data: "metadata management at scale is not fundamentally different from data management" (§7).

**Data model (§4)**
- Logical metadata (schema, partitioning/clustering spec, row/column ACLs) is small and centralised. Physical metadata (block locations, row counts, MVCC, per-block column statistics) has cardinality O(C×N); "some of our tables have tens of terabytes of metadata".
- CMETA is a system table per user table: "each row corresponds to a block in the table" (§4.2).
- Columns: `_block_locator`, `creation_timestamp`, `deletion_timestamp`, `block_total_bytes`, plus one struct per leaf column.
- Each struct holds total_rows, total_nulls, total_bytes, min/max, hash bucket/modulus, dictionary, bloom filter, and S2 covering for geography.
- CMETA is stored in the same columnar format as data (Capacitor) and clustered on max_value of the partitioning/clustering columns.

**Maintenance (§4.3)**
- A metadata change log in replicated storage journals block creation and deletion with commit timestamps.
- Background LSM-style merges produce columnar baselines and deltas.
- "At any given read timestamp, the table's metadata can be constructed by reading the baseline available at that timestamp and any deltas".
- Streaming rows not yet compacted keep metadata in ingestion-server memory.

**Querying (§5)**
- The planner uses only logical metadata, then rewrites the query as a semi-join with CMETA on `_block_locator`: "we defer the reading of physical metadata for the tables until the actual dispatch of partitions".
- Filters are converted to "falsifiable expressions" over column statistics (§5.2–5.3).
- The CMETA scan is interleaved with the data scan (§5.7).
- CMETA's own block locators and min-max values are small and live in the centralised metadata (§5.8). A materialized view holds current-version rows.

**Numbers**
- >60% of queries select <1% of data (§4).
- 1 PB table with ~3M blocks: 2.5 s vs 120 s; 30,000× less resource for the most selective query, 6,000× for low selectivity (§6).
- 10 TB table: 5–10× faster.
- Scanning the PB table's metadata took 500 ms (§7).

**Other**
- Access control and multi-tenancy: not stated beyond ACLs being logical metadata.
- Comparisons (§2): HMS has "varying scalability limits" on partitions per query. Delta Lake's Parquet-compacted log is "similar" in columnar layout. DB2 BLU synopsis tables must fit in memory. Goods: not cited.
- Future work (§7): CMETA-driven range-partitioned joins; in-memory caching of hot CMETA columns with vectorized processing.

## 3. Production ML Pipelines (Xin, Miao, Parameswaran, Polyzotis; SIGMOD '21; arXiv 2103.16007v1)

**Problem (§1).** No quantitative evidence existed on production ML pipelines. The paper analyses provenance of 3000 TFX pipelines at Google.

**Provenance model (§2.1–2.2)**
- "a trace is a directed acyclic graph of execution nodes and artifact nodes", with edges for input/output.
- MLMD records Execution, Artifact and (unused here) Context nodes, with start/completion and creation timestamps.
- Data-span artifacts carry feature lists, types and summary statistics.
- MLMD "automatically records the end-to-end provenance using a graph-based model". Its storage backend: not stated.

**Graphlets (§4.1, App. A)**
- A graphlet is anchored on one Trainer execution, so that "each subgraph represents a single (logical) end-to-end pipeline run related to an individual model".
- It includes (a) all ancestor executions and their artifacts, (b) analysis/validation executions on those data spans, and (c) descendants not on paths to other Trainers.
- It is defined as a two-rule Datalog query with a stop predicate. Warm-start edges are cut.

**Numbers**
- Corpus: 3000 pipelines, ~450,000 models, four months, 7.7M execution nodes, 20M artifact nodes; up to 6,953 nodes per trace (§2.2, §3.1).
- Mean lifespan 36.6 days; mean 7.2 trainer runs/day (median 1.0).
- Trainer runs: 64% DNN.
- Compute: training is under a third (the abstract says 20%); analysis/validation ~35%; ingestion ~22%, partly from "hermetic" copies (§3.3).
- Consecutive graphlets share input spans with mean Jaccard 0.647 (§4.2.1).
- "approximately 80% of all graphlets are unpushed", about 3 between pushes (§4.3). The code is unchanged 84.5% of the time.
- A Random Forest on graphlet features removes 50% of wasted compute with no loss of freshness (§5.3.2).

**Other**
- Similarity is computed from span statistics alone, because the raw data is not in the corpus and "might also be siloed for privacy reasons" (§4.2.2).
- Goods is cited once, with Ground, as provenance for data lakes; no discussion.
- Access control and metadata storage: not stated.

**Open problems named**
- "Tools to efficiently query or summarize these complex traces can become indispensable" (§3.1).
- Large-K top-K vocabulary queries (§3.2).
- Incremental view maintenance for data preparation (§4.2.1).
- Cost-based materialisation of intermediate artifacts (§3.3).
- Snapshot-based source access to avoid ingestion copies (§3.3).
- Telemetry on whether unpushed models are transitively useful (§4.3.2).

## 4. Comprehensive and Comprehensible Data Catalogs (Subramaniam, Ma, Li, Mohanty, Castro Fernandez; arXiv 2103.07532v3, 1 Feb 2023; no venue printed)

**Problem (abstract).** Catalogs "either make metadata easy for users to store and difficult to retrieve, or they make it easy to retrieve, but difficult to store". Scope: "This paper does not propose a full-fledged data catalog."

**Model (§2.1, §3)**
- A metadata item (MI) is a key:value pair about an asset. A mental model (MM) is a partitioning of MIs.
- 5W1H+R mnemonic: "Why, When, and How does Who use a Data Asset (located Where) whose contents are described by What?"
- Who: creators, accessors, roles, privileges.
- What: anything derived by reading the data, or schema semantics.
- When: lifecycle times.
- Why: purpose and intended or prohibited use.
- Where: location, access path, format.
- How: processes, scripts and queries that produced or read it.
- +R: MIs about more than one asset (similarity, provenance, versions), typed by which profile they relate.

**Evidence**
- 155 questions from the literature were reduced to 27 and classified as Discovery, Governance, Integration or Compliance (§2.2, Table 1).
- Twelve catalogs were surveyed (Table 2: Ground, Atlas, Amundsen, DataHub, Marquez and others). All have explicit partitions plus a catch-all, which makes it "easy to store MIs but difficult to find them later" (§2.3).
- User study (§4): 160 Prolific participants, between-subjects, against Google Data Catalog and DataHub.
- Mean entropy: 1.698 vs 2.212 vs 1.903. "None" share: 0.199 vs 0.329 vs 0.298.
- It lost on Q25 (format/repository), because Where-profile lacks concrete terms (§4.3.2).

**Implementation guidelines (§3.4)**
- G1: "there should be a table for each 5W1H profile and relationships".
- G2: preserve history; "operations on the catalog are append-only".
- G3: audit metadata on every set/get (who, version, time); event streaming is suggested.
- G4: complex values, e.g. JSON.

**Other**
- Architecture, scale, access control: not stated (explicit non-goals).
- Goods is cited for the definition of discovery and as a source of questions.
- Ontologies (DCAT) are positioned as key vocabularies inside a partition, not as the MM (§5.1).
- Limits: no UI work; question wording may echo partition names (§4.3.2).

## Implications for a new multimodal catalog

1. **Path uniqueness (UC §4.2.1, §4.3.1, §5).** Enforce non-overlapping storage prefixes per tenant at create time and keep a prefix index (UC uses URL-tries). Path-based requests then resolve to exactly one asset and one policy. (Inference: replicas can be several locations of one asset as long as each location maps to one asset.)
2. **Tenant version counter (UC §4.5).** One persisted version per tenant, bumped by compare-and-set on every write, gives per-tenant serializable writes, snapshot reads and cache-staleness detection without a consensus service. (Inference: it fits tenant-leading keys in PostgreSQL.)
3. **Cache the whole tenant (UC §4.5, Figs 4–5).** Working sets are ≤100 MB and 98% of traffic is reads. Use a write-through multi-version cache indexed by ID, name and path, and offer one batched resolve-and-authorize call.
4. **Asset types as manifests (UC §4.2.2–4.2.3).** Keep one generic entity/grant/lifecycle core and declare per-type operations, privileges and lifecycle. (Inference: dataset → immutable snapshot can mirror RegisteredModel → model version.)
5. **Separate admin from data access (UC §3.3).** Container owners manage grants but get no implicit read. Inherit grants to future children.
6. **Vend short-lived, asset-scoped credentials (UC §4.3.1, §4.3.2).** Document the revocation lag of tens of minutes. (Inference: sub-asset usage-policy rules on files or arrays need a trusted reader.)
7. **Change stream as the core/discovery contract (UC §4.4, §4.5).** Run search and lineage as background consumers, expose an authorization API for result filtering, and reuse the stream for selective cache invalidation.
8. **Split logical from physical metadata (Big Metadata §4.1, §5.8).** Keep schema, policy and names in the OLTP store. Put O(columns × blocks) statistics in columnar system tables, and store only their locators and min-max centrally. (Inference: this applies to array chunks and archive members.)
9. **Snapshots from validity intervals (Big Metadata §4.2–4.3, §5.8).** Give physical entries creation and deletion timestamps, build them from a change log merged into baselines plus deltas, and keep a materialized "current" view. As-of reads become a filter.
10. **Provenance as an execution/artifact DAG (ProdML §2.1–2.2, §4.1, App. A).** Use bipartite nodes with timestamps and a grouping context, and ship anchored subgraph queries because raw traces reach thousands of nodes. (Inference: the two Datalog rules map to recursive SQL.)
11. **Statistics on provenance nodes (ProdML §2.2, §4.2.2, §4.3).** Per-artifact summary statistics allow similarity and drift checks without data access. Record downstream "pushed/consumed" events, since ~80% of produced models were never deployed. (Inference: this is useful for retention.)
12. **Descriptive metadata schema (Catalogs §2.3, §3.4).** Use explicit Who/What/When/Why/Where/How tables plus typed relationships, no catch-all bag, and append-only values with who/version/time. (Inference: the Why profile, covering purpose and prohibited uses, is the natural home for usage policy.)
