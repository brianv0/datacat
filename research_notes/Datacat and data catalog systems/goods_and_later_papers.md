# Goods (SIGMOD 2016) and the papers that followed

Reading notes, November 2026. Goods was read in full from the published PDF. Unity Catalog, Big Metadata, Production ML Pipelines and the 5W1H catalog paper were read in full by a delegated reader and spot-checked against the text. Other papers in the table were confirmed by title, venue and abstract only. The IEEE Data Engineering Bulletin companion to Goods ("Managing Google's data lake: an overview of the Goods system", 39(3):5-14, 2016) could not be fetched.

## 1. Goods: Organizing Google's Datasets

Halevy, Korn, Noy, Olston, Polyzotis, Roy, Whang. SIGMOD 2016. DOI 10.1145/2882903.2903730.

### The idea

Goods is a *post-hoc* catalog. The alternative, enterprise data management, needs every team to publish into one system. Goods instead lets teams create and use data with any tool, and "works in the background, in a non-intrusive manner, to gather the metadata about datasets and their usage" (section 1). It crawls storage systems and production logs, aggregates what it finds in a central catalog, and builds services on top: search, profile pages, team dashboards, monitoring, provenance views, annotation.

### Scale (section 2, Table 1)

| Measure | Value |
|---|---|
| Datasets in the catalog | 26 billion |
| Paths added per day | 1.6 billion |
| Paths deleted per day | 1.6 billion |
| Storage systems | 6 |
| Dataset formats | more than 20 |

- Only datasets readable by all Google engineers were indexed. The authors expected the count to more than double with restricted data.
- One second per dataset on a thousand machines would take about 300 days, so metadata extraction must be prioritised and differential.
- More than 5% of the catalog is deleted every day and about as much is added. Many deleted datasets are TTL-bound intermediates, but they cannot be ignored because they link long-lived datasets in the provenance graph.

### Challenges named (section 2)

Scale; variety of formats, stores and extraction cost; churn; uncertainty of inferred metadata; computing dataset importance; recovering semantics. "Dataset" itself is hard to define: a Bigtable, its column families and its underlying files can all be datasets, and aliases arise in several ways.

### Metadata (section 3.1, Table 2)

| Group | Metadata |
|---|---|
| Basic | size, format, aliases, last modified time, access control lists |
| Content-based | schema, number of records, data fingerprint, key field, frequent tokens, similar datasets |
| Provenance | reading jobs, writing jobs, downstream datasets, upstream datasets |
| User-supplied | description, annotations |
| Team and project | project description, owner team name |
| Temporal | change history |

- **Schema inference.** Most records are serialized protocol buffers, and files do not name their message type. Goods scans a few records and tests every registered message definition for whether it "could conceivably have generated the bytes". The result is speculative: all candidates are stored with heuristic scores.
- **Content summary.** Frequent tokens by sampling; HyperLogLog cardinality to find candidate keys; per-field checksums and LSH values to find similar or identical datasets and columns.
- **Provenance.** Built from production logs of which jobs read and wrote which datasets, then a transitive closure over the dataset-job graph, with earliest and latest times for each dependency. To bound cost they process only a sample of access events and materialise only a few hops.
- **Semantics.** Comments in protobuf source code (`mpn` is explained by "// Model Product Number"), and matching content against the Knowledge Graph.
- **Extensibility.** Other teams can add their own metadata types through an API; the catalog became "the unifying place for teams to collect their metadata".

### Clusters (section 3.2)

Datasets are grouped by abstracting parts of the path: timestamps, data-centre names, machine names, versions, UUIDs (Table 3). `/dataset/2015-10-10/daily_scan` generalises to `/dataset/<date>/daily_scan`. Composing the dimensions gives a semi-lattice of granularities. Goods creates a catalog entry only for the top-most element of each lattice, which keeps cluster entries few, maps each dataset to exactly one cluster, and keeps clusters stable from day to day.

Clusters serve two purposes: users see one logical dataset instead of thousands of versions and shards, and expensive metadata is computed for a few members and propagated. Propagated metadata is computed on demand, not stored, "to differentiate explicitly between propagated metadata and metadata that we obtained through real analysis". Some clusters have more than ten million members (Figure 3).

### Backend (section 4)

- **Storage.** Bigtable, one row per dataset or cluster, keyed by path. Per-row transactions fit because most processing is per dataset. Batch-only data (raw provenance, the largest column family) lives in separate, heavily compressed column families.
- **Status metadata.** Each row also records, for each module that processed it, a timestamp, success flag and error. Modules coordinate through it: module B skips a row until module A's success entry is present, and re-processes when A does. Bigtable's versioning keeps the history of status entries, which the team used for debugging.
- **Scheduling.** Jobs run independently, in any order, and may be broken or offline. Most finish a pass in under a day. The schema analyser can fall weeks behind after a large influx, so two instances run: one over "important" datasets only (those with user annotations or high provenance centrality), one over everything.
- **Blind writes.** Crawlers write everything they read without first reading the catalog; cheaper than an anti-join, but no-op writes must be avoided where they would re-trigger dependents or block garbage collection.
- **Fault tolerance.** Per-dataset errors go into status metadata and trigger bounded retries. Format libraries that crash or hang are run in a sandboxed process with a watchdog. The catalog is replicated asynchronously to several locations.
- **Garbage collection.** After two incidents where the catalog bloated badly, removal became aggressive and rule-based. Conditions are declarative predicates over metadata and module status, for example: the dataset is gone from storage *and* its latest provenance has been consumed by a successful provenance-linker run. Deletion is two-phase: a tombstone first, actual deletion 24 hours later if the predicate still holds. Other modules ignore tombstoned rows and may not run a single iteration for more than 24 hours. This avoids "dangling rows" (a concurrent module re-creating a deleted row with partial data) without making every write a conditional mutation, which was too expensive.

### Front end (section 5)

- **Profile pages** for each dataset or cluster, editable, with provenance compressed by the same path abstraction when it is too large, cross-links to job and code tools, and generated access snippets in C++, Java and SQL.
- **Search** over a conventional inverted index where each dataset is a document. Tokens are tied to sections, so queries can be qualified: `path:`, `proto:`, `read_by:`, `written_by:`, `downstream_of:`, `upstream_of:`, `kind:`, `owner:`. Paths are indexed as positional tokens so `x/y` matches `a/x/y/b` without indexing every sub-path.
- **Ranking signals:** dataset type (a registered table outranks a file), keyword match by section (path over job names), lineage fan-out (an approximation of PageRank over datasets and jobs, which over-ranks things like the web crawl), and the presence of an owner-written description.
- **Dashboards and monitoring** per team, with alerts on expected properties (shard counts, value distributions) and checks suggested from learned trends.

### Lessons learned (section 6)

1. **Evolve as you go.** Built for discovery; used to audit datasets that use sensitive protocol buffers, to re-find one's own experimental datasets, to understand legacy code through provenance, to bookmark, and to annotate with privacy levels.
2. **Use domain-specific ranking signals.** Provenance distinguishes a master dataset from its denormalised copies; use across team boundaries indicates importance.
3. **Expect and handle unusual datasets.** Start with the simplest ad hoc fix, generalise when needed.
4. **Export data as required.** A key-value store and an inverted index cannot serve path queries over provenance, so the catalog is exported daily as subject-predicate-object triples into a graph system.
5. **Ensure recoverability.** A lost catalog takes weeks to rebuild, and provenance through transient files cannot be recomputed. They keep rolling Bigtable snapshots, a separate snapshot of high-value entries, a replica of the subset that serves profile pages, and they monitor the catalog with Goods itself.

### Future work stated (section 8)

Better ranking and telling production from test data; filling missing metadata from a knowledge graph of datasets, code and teams; content semantics; and combining post-hoc crawling with registration at creation time for freshness. The paper closes on instilling a "data culture" comparable to code discipline.

### What Goods does not address

Access control over the catalog's own metadata (it indexed only world-readable data), dataset versioning as a managed concept, policy on use, and placement.

## 2. Later papers

| Paper | Venue | Notes |
|---|---|---|
| Data Management Challenges in Production Machine Learning (Polyzotis, Roy, Whang, Zinkevich) | SIGMOD 2017 tutorial | Same Google group, moving from datasets to training data |
| Data Integration: After the Teenage Years (Golshan, Halevy, Mihaila, Tan) | PODS 2017 | The only PODS paper found in this line |
| Azure Data Lake Store (Ramakrishnan et al.) | SIGMOD 2017 | Hierarchical namespace independent of physical location |
| Ground: A Data Context Service (Hellerstein et al.) | CIDR 2017 | Metamodel for application, behaviour and change context |
| Aurum: A Data Discovery System (Castro Fernandez et al.) | ICDE 2018 | Graph of column relationships maintained from samples |
| Data Lifecycle Challenges in Production ML: A Survey | SIGMOD Record 2018 | Understanding, validation and cleaning, preparation |
| Google Dataset Search (Brickley, Burgess, Noy) | WWW 2019 | Public successor; owners publish schema.org metadata |
| JOSIE (Zhu, Deng, Nargesian, Miller) | SIGMOD 2019 | Joinable table search |
| Data Lake Management: Challenges and Opportunities (Nargesian et al.) | VLDB 2019 tutorial | Places Goods in the data lake literature |
| Organizing Data Lakes for Navigation (Nargesian et al.) | SIGMOD 2020 | Navigation as an alternative to keyword search |
| Delta Lake; Lakehouse (Armbrust et al.) | VLDB 2020; CIDR 2021 | Table-format line that led to lakehouse catalogs |
| Big Metadata: When Metadata is Big Data (Edara, Pasumansky) | VLDB 2021 | See section 3 |
| Production ML Pipelines: Empirical Analysis (Xin, Miao, Parameswaran, Polyzotis) | SIGMOD 2021 | See section 3; cites Goods |
| Data Lakes: A Survey of Functions and Systems (Hai et al.) | arXiv 2106.09592 | Says Goods is innovative but "heavily tailored to datasets and practices used to produce them in Google" |
| Comprehensive and Comprehensible Data Catalogs (Subramaniam et al.) | arXiv 2103.07532 | See section 3; cites Goods |
| Analyzing and Comparing Lakehouse Storage Systems (Jain et al.) | CIDR 2023 | Metadata handling of Delta, Iceberg, Hudi |
| SANTOS; Table Discovery in Data Lakes tutorial (Miller group) | SIGMOD 2023 | Union search; state of the field |
| Cross Modal Data Discovery, CMDL (Eltabakh et al.) | VLDB 2023 | Discovery across tables and text documents |
| BigLake (Levandoski et al.) | SIGMOD 2024 | Open table formats and object tables for unstructured data |
| Croissant | DEEM workshop at SIGMOD 2024; NeurIPS 2024 | Metadata format for ML datasets on schema.org |
| Unity Catalog (Chandra et al.) | SIGMOD Companion 2025 | See section 3; cites Goods |
| Unified Lineage System: Tracking Data Provenance at Scale | SIGMOD Companion 2025 | Lineage across asset types and granularities; not read, affiliation not confirmed |
| TreeCat: Standalone Catalog Engine for Large Data Systems (Oh, Nilangekar, Deshpande) | arXiv 2503.02956 | Database engine built to be a catalog; opening sections read, venue not confirmed |

No PODS paper on catalogs or data lakes themselves was found. The work after Goods is in SIGMOD, VLDB, CIDR and ICDE.

TreeCat separates *data catalogs* (organisation-level, asynchronous, an "information retrieval system" for assets; Goods, Atlas, DataHub) from *operational catalogs* (in the hot path of engines; HMS, table formats, Unity Catalog), and argues for a standalone, unified operational catalog engine with a hierarchical data model, path query language, versioned storage for time travel, clone and snapshot, and a serializable multi-version optimistic concurrency protocol.

## 3. Digest of four later papers

### Unity Catalog (Databricks, SIGMOD Companion 2025)

- **Problem.** Lakehouse catalogs have "inconsistent governance, narrow interoperability, and lack of support for data discovery" and are table-only.
- **Model.** Metastore (root, one home region), catalog, schema, asset. Leaf assets: tables, views, volumes, models, functions; plus credentials, storage locations and connections. A volume is "a logical storage in a cloud object storage location for organizing files and non-tabular data"; per-file metadata inside a volume is not described. All types sit on a generic entity-relationship model, and a new type is added by a declarative manifest (position in the hierarchy, operations, privileges, lifecycle).
- **Path invariant.** "No two assets in a metastore can have overlapping storage paths", so a path resolves to exactly one asset and one policy. About 7% of tables are also accessed by path.
- **Storage and consistency.** A multi-tenant service over a standard relational database (MySQL in the benchmark). Snapshot-isolated reads and serializable writes per metastore through a persisted metastore version that each write increments conditionally; a mismatch triggers cache reconciliation. "This design avoids reliance on distributed-consensus-based services."
- **Caching.** Write-through multi-version cache keyed by ID, name and path. Working sets are under 100 MB for almost all metastores. Caching gives 3x to 40x lower latency.
- **Scale.** About 9,000 customers, more than 75K workspaces, 100K metastores, more than 100M tables, 550K volumes, 400K models; about 60K requests per second, 98.2% reads.
- **Governance.** SQL-style grants with inheritance to existing and future children. Administration does not imply data access: "a schema owner does not automatically gain SELECT access to tables in the schema". Credential vending returns a token scoped to one asset, valid for tens of minutes; revocation does not cut off holders of live credentials. Row filters and column masks need a trusted engine. Tag-based ABAC was in private preview.
- **Discovery.** Search and lineage are second-tier background services fed by metadata change events, and call a core authorization API to filter results.
- **Federation.** A federated catalog mirrors a foreign one on demand; about 16% of tables are foreign.
- **Open items.** No open standard for a trusted engine; caching absent from the open-source version; revocation lag; federation staleness; multi-table transactions in progress.
- **On others.** HMS has no governance and no transactions. Iceberg REST "does not specify governance APIs beyond temporary credential vending". Goods is cited once with no discussion.

### Big Metadata (Google BigQuery, VLDB 2021)

- **Claim.** "Metadata management at scale is not fundamentally different from data management."
- **Split.** Logical metadata (schema, partitioning, ACLs) is small and centralised. Physical metadata (block locations, row counts, per-block column statistics) grows with columns times blocks; some tables have tens of terabytes of it.
- **CMETA.** A system table per user table where "each row corresponds to a block". Columns hold block locator, creation and deletion timestamps, and one struct per leaf column with counts, min/max, dictionary, bloom filter and S2 covering. Stored in the same columnar format as data.
- **Maintenance.** A change log journals block creation and deletion; background merges produce baselines and deltas. Metadata at any read timestamp is the baseline plus deltas.
- **Query.** The planner uses only logical metadata, then rewrites the query as a semi-join with CMETA, deferring physical metadata until dispatch.
- **Numbers.** More than 60% of queries select under 1% of data. On a 1 PB table with about 3M blocks: 2.5 s against 120 s.

### Production ML Pipelines (Google, SIGMOD 2021)

- **Corpus.** Provenance of 3000 TFX pipelines over four months: about 450,000 models, 7.7M execution nodes, 20M artifact nodes.
- **Model.** "A trace is a directed acyclic graph of execution nodes and artifact nodes." Data-span artifacts carry feature lists, types and summary statistics.
- **Graphlets.** A subgraph anchored on one trainer execution, covering its ancestors, the validation runs on its data, and its descendants; defined as a two-rule Datalog query.
- **Findings.** Training is a minority of compute; analysis and validation about 35%, ingestion about 22%. Consecutive runs share most of their input. About 80% of trained models are never deployed.
- **Open problems.** Tools to query and summarise large traces; snapshot-based access to avoid ingestion copies; cost-based materialisation of intermediates.

### Comprehensive and Comprehensible Data Catalogs (arXiv 2021, revised 2023)

- **Problem.** Catalogs "either make metadata easy for users to store and difficult to retrieve, or they make it easy to retrieve, but difficult to store". Twelve surveyed catalogs all have a catch-all bucket.
- **Model.** 5W1H plus relationships: "Why, When, and How does Who use a Data Asset (located Where) whose contents are described by What?"
- **Guidelines.** One table per profile and one for relationships; append-only history; audit on every read and write; complex values allowed.
- **Evidence.** A 160-person study found the 5W1H partitioning gave more consistent placement of metadata than Google Data Catalog or DataHub.

## 4. Implications for mudcat

From Goods:

1. **Inventory as an ingest path.** Goods' future work is to combine crawling with registration. mudcat registers at write time; reports from site agents should populate the same entities, with inferred values marked by assertion origin and, where ambiguous, kept as scored candidates.
2. **Path abstraction for onboarding.** Goods inferred from paths what mudcat models explicitly (versions, replicas at sites, shards of one artifact). A rule set over the same five dimensions would turn an existing bucket into artifacts, versions and replicas.
3. **Provenance outlives artifacts.** Transient intermediates are 5% of the catalog every day and still carry lineage between long-lived data. Removal of an artifact must not remove its provenance edges, and provenance through purged data cannot be recomputed.
4. **Per-enricher status on each entity.** Timestamp, success and error per enricher per entity gives dependency ordering, retries, freshness windows and debugging without a central scheduler.
5. **Serve lineage at dataset level.** Popular data has tens of thousands of readers. Materialise a few hops and aggregate; keep the raw edges for batch use.
6. **Ranking signals.** Entity kind, lineage fan-out, owner description, use across groups, and inclusion in monitoring are available to mudcat's search without reading data.
7. **Recoverability of the catalog.** Separate protection for the entries that cannot be rebuilt: user-supplied metadata and provenance.
8. **Sandbox format plugins.** Format libraries crash and hang on real data.

From the later papers:

9. **Non-overlapping storage prefixes** per tenant, enforced at creation with a prefix index, so path-based access resolves to one artifact and one policy (Unity Catalog).
10. **A version counter per tenant**, bumped by compare-and-set on every write, gives serializable writes, snapshot reads and cache staleness detection without a consensus service. It fits tenant-leading keys in PostgreSQL (Unity Catalog).
11. **Cache the whole tenant.** Working sets are small and traffic is 98% reads; offer one batched resolve-and-authorize call (Unity Catalog).
12. **Entity types as manifests** over one generic entity, grant and lifecycle core (Unity Catalog).
13. **Administration is not data access**, and grants inherit to future children (Unity Catalog).
14. **State the revocation lag** of vended credentials as a requirement, and note that policy on parts of an artifact needs a trusted reader (Unity Catalog).
15. **Search and lineage as stream consumers** that call back for authorization, which is the design mudcat already has (Unity Catalog).
16. **Split logical from physical metadata.** Names, schema and policy in the transactional store; per-chunk and per-element statistics in columnar tables with only their locators held centrally (Big Metadata). This is the answer to the cost of element-level rows in PostgreSQL.
17. **Validity intervals on physical entries**, built from a change log merged into baselines and deltas, make as-of reads a filter (Big Metadata).
18. **Anchored subgraph queries** for provenance, because raw traces reach thousands of nodes; and record whether an output was ever consumed, since most are not (Production ML Pipelines).
19. **No catch-all metadata bucket.** Namespaced metadata payloads are at risk of becoming one; the 5W1H profiles are a candidate for the built-in namespaces, with "why" as the home of usage policy (Catalogs).
20. **Data catalog and operational catalog are different jobs** with different consistency needs (TreeCat). mudcat intends to be both; the transactional core and the derived search and lineage services should keep that line clear.
