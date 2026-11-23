# Amundsen vs DataHub as data-science / enterprise metadata catalogs (state as of 2026-11-23)

Research access notes (read first):
- `docs.datahub.com`, `archive.docs.datahub.com` and `www.amundsen.io` were not reachable from this session (network allowlist). DataHub and Amundsen documentation was therefore read from the markdown sources in the GitHub repositories (`github.com` / `raw.githubusercontent.com`), which are the origin of the docs sites. Where a fact comes only from a web-search snippet of a page I could not open, it is marked "(search snippet, not opened)".
- The GitHub API tools were restricted to another repository, so commit/release data comes from the public GitHub HTML pages.
- No Reddit r/dataengineering or Hacker News threads surfaced in searches; community sentiment is under-sourced (see Gaps).
- Vendor comparison blogs (Atlan, decube, OvalEdge, etc.) are used only where they agree with primary sources, and are labelled.

---

## 1. Amundsen: architecture, data model, lineage, ranking, and project health

### Takeaway
Amundsen is a simple, pull-based, three-microservice discovery catalog (Flask/React frontend, metadata service over Neo4j/Atlas/Neptune, search service over Elasticsearch, plus the `databuilder` batch ETL library) whose distinguishing idea was usage-ranked search. It is now dead as an upstream project: the GitHub repository was archived (read-only) in September 2026 "due to inactivity", after only one commit in 2025 and no meaningful feature work since late 2024.

### Cited Findings

Architecture
- Frontend: "Flask-based web app which representation layer is built with React with Redux, Bootstrap, Webpack, and Babel"; it calls the search and metadata services. — [amundsen docs/architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md)
- Search service: a REST proxy that "leverages Elasticsearch's search functionality (or Apache Atlas's search API, if that's the backend you picked)"; the architecture doc still says "Currently only table resources are indexed and searchable"; the index is built by the databuilder Elasticsearch publisher. — [amundsen docs/architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md)
- Metadata service: "currently uses a Neo4j proxy to interact with Neo4j graph db and serves frontend service's metadata"; metadata "is represented as a graph model". — [amundsen docs/architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md)
- Databuilder: an ingestion library; "At Lyft, we build the metadata once a day using an Airflow DAG". Ingestion is batch/pull; no streaming path is documented. — [amundsen docs/architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md)
- Backends per README: Neo4j or Apache Atlas as the metadata service persistent layer; AWS Neptune via the `amundsengremlin` library; relational databases via `amundsenrds` ORM models. — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- The project is composed of three microservices (frontend, search, metadata) plus the databuilder and common libraries, hosted by LF AI & Data. — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)

Data model, connectors, ranking
- README description: a data discovery and metadata engine that indexes data resources (tables, dashboards, and more) and ranks search using usage patterns (highly queried tables rank earlier). Resource types named: tables, dashboards, ML features, people (from HR systems). — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- Table connectors listed: Athena, EventBridge, Glue, Redshift, Cassandra, Druid, Hive, CSV, dbt, Delta Lake, Elasticsearch, BigQuery, DB2, Kafka Schema Registry, SQL Server, MySQL, Oracle, PostgreSQL, Presto, Trino, Vertica, Snowflake, and anything with a dbapi/SQLAlchemy interface. Dashboard connectors: Superset, Mode, Redash, Tableau, Databricks SQL. Column stats via Pandas Profiling. Orchestration: Airflow. — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- Late commits show the kind of features still being added in 2024: "table owner categories" (Sep 2024), "Enhance search logging" (Dec 2024), neo4j driver upgrade to latest 4.x (Apr 2024). — [amundsen commit history](https://github.com/amundsen-io/amundsen/commits/main/)

Project health 2024–2026
- GitHub banner: "This repository was archived by the owner on Sep 29, 2026. It is now read-only." README notice: "Due to inactivity, this project was archived in September 2026. The contents will remain available for historical purposes." — [amundsen-io/amundsen](https://github.com/amundsen-io/amundsen), [README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- Repo counters at archive: ~4.8k stars, 958 forks, 2,713 commits on main. — [amundsen-io/amundsen](https://github.com/amundsen-io/amundsen)
- Commit history tail: last commit 2026-09-10 "Adding notice to README about archiving repo (#2366)"; 2026-08-20 "Removed links in README to amundsen.io (#2365)"; the only 2025 commit is 2025-04-02 "moving Marcos to emeritus maintainer (#2289)"; last functional commits are Dec 2024 (search logging, tab overflow fix). 2024 activity is dominated by dependabot bumps plus a handful of fixes from ~5 human authors. — [amundsen commit history](https://github.com/amundsen-io/amundsen/commits/main/)
- Vendor sources (Atlan and others; biased, but consistent with the commit log) state the last release was 14 Aug 2024 with no releases in 2025. — [Atlan: amundsen vs datahub](https://atlan.com/amundsen-vs-datahub) (search snippet, not opened)
- One comparison blog gives the archive date as 10 Sep 2026 (the date of the README notice commit) rather than the GitHub banner date of 29 Sep 2026. — [decube comparison](https://www.decube.io/post/open-source-data-catalog-comparison) (search snippet); contradicted on the exact day by [GitHub banner](https://github.com/amundsen-io/amundsen)
- Users: the README lists 37 organizations as official users, including Lyft, ING, Square, Snap, Instacart, Asana, Brex, Chan Zuckerberg Initiative, Edmunds, Workday, REA Group. This list is historical; it was not pruned before archiving. — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- New York MTA published an article on building its internal data catalog (surfaced in an Amundsen search; I did not open it to confirm which software it uses or its date). — [MTA article](https://www.mta.info/article/how-we-built-mtas-internal-data-catalog) (search snippet, not opened)

### Inferences
- Amundsen went from "maintenance mode" (2023–2024: dependency bumps, small fixes, mostly from Lyft-affiliated maintainers) to "dormant" (2025: one housekeeping commit) to "archived" (Sep 2026). Removing the amundsen.io links from the README in Aug 2026 suggests the project website was being retired too (consistent with `www.amundsen.io` not resolving from this session, though that may equally be the session's network allowlist).
- For a new design, Amundsen is useful only as a reference for (a) how little is needed for a usable discovery catalog (graph store + search index + nightly batch ETL), and (b) usage-based ranking as a first-class relevance signal. It should not be considered as a base to build on.
- The architecture doc itself was stale for years ("only table resources are indexed" while the README advertises dashboards, features and people) — a symptom of low maintenance well before archiving.
- Amundsen has no change-log/event stream, no versioned metadata, no typed extension mechanism beyond free-form "programmatic descriptions", tags and badges, and no access-control model of its own; these gaps are why it lost to DataHub/OpenMetadata once governance became the buying criterion. (Structural inference from the architecture docs; the absence of features is not itself stated in a source.)

### Gaps
- Could not open amundsen.io docs, so the detailed entity model (Table/Column/Schema/Cluster/Database nodes, Badge, Tag, User READ/OWNER/FOLLOW edges, Dashboard hierarchy, Feature) and the search ranking formula (Elasticsearch `function_score` on `total_usage`) are from prior knowledge and are NOT verified against a source in this session.
- Lineage: from prior knowledge Amundsen added table- and column-lineage display around 2021 (frontend feature flags, backed by Neo4j edges written by databuilder, e.g. from dbt/OpenLineage-style extractors), and it was always shallow compared with DataHub. Not verified here.
- Exact final release versions of each sub-package (frontend/metadata/search/databuilder are versioned separately on PyPI) were not retrieved; the GitHub "Releases" section rendered empty.
- Which listed adopters still run Amundsen in 2026 is unknown; Lyft's own current status is not documented in anything I found.

---

## 2. DataHub: architecture, metadata model, APIs, ingestion, governance features

### Takeaway
DataHub is a schema-first, stream-oriented metadata platform: a Java metadata service (GMS) persists immutable, versioned "aspects" keyed by entity URN in a relational DB, publishes every change to Kafka, and derives a search index (Elasticsearch/OpenSearch) and a graph index from that log; a Python ingestion framework (140+ connectors) pushes or pulls metadata in. It is by far the most feature-complete and most actively developed open-source catalog, at the cost of a heavy, multi-system deployment.

### Cited Findings

Architecture and design principles
- Self-description: a "3rd generation data catalog" with a "model-first philosophy, with a focus on unlocking interoperability between disparate tools & systems". Three stated principles: schema-first modelling in a "serialization agnostic language"; "stream-oriented" metadata infrastructure so changes are reflected within seconds and consumers can subscribe; federated metadata serving, where multiple metadata services "communicate with the central search index and graph using Kafka". Claims proven scale of "10M+ assets and O(1B) relationships". — [docs/architecture/architecture.md](https://github.com/datahub-project/datahub/blob/master/docs/architecture/architecture.md)
- APIs: REST and GraphQL plus "an AVRO-based API over Kafka to communicate metadata changes across your organization". — [docs/architecture/architecture.md](https://github.com/datahub-project/datahub/blob/master/docs/architecture/architecture.md)
- The metadata store "consists of a Spring Java Service hosting a set of Rest.li API endpoints" along with MySQL, Elasticsearch and Kafka for primary storage and indexing; it exposes ingest, get-by-primary-key, search, and relationship-fetch APIs. The GraphQL API is the strongly typed entity-oriented API used by the React UI. — [docs/components.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/components.md)
- Kubernetes deployment: "DataHub consists of 4 main components: GMS, MAE Consumer (optional), MCE Consumer (optional), and Frontend"; dependencies are Kafka, a local DB (MySQL, Postgres or MariaDB), a search index (Elasticsearch), and a graph index (Neo4j or Elasticsearch). The prerequisites chart example also deploys Schema Registry and Zookeeper pods. Neo4j is optional (switch `graph_service_impl`). A `datahub-system-update` job handles SQL and index setup. — [docs/deploy/kubernetes.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/deploy/kubernetes.md)
- v1.9.0 release notes mention GMS/MAE/MCE startup behaviour and the `DataHubUpgradeHistory_v1` Kafka marker topic, confirming the MAE/MCE consumer naming is still current in Nov 2026. — [DataHub releases](https://github.com/datahub-project/datahub/releases)

Entity–aspect model
- "DataHub takes a schema-first approach to modeling metadata", using Pegasus (PDL) schemas with custom annotations that drive storage, serving, indexing and ingestion. — [docs/modeling/metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Entity = type + unique identifier (URN) + a set of aspects. Aspects are "the smallest atomic unit of write in DataHub"; aspects of the same entity are updated independently. Common shared aspects: ownership, globalTags, glossaryTerms, institutionalMemory, status (soft delete), subTypes. — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Keys and URNs: a key aspect holds the identifying fields; all key fields must be STRING or ENUM and REQUIRED; keys serialize to URNs of the form `urn:li:<entity-name>:key` or `urn:li:<entity-name>:(k1,k2,...)`. "Urns provide a friendly handle by which Entities can be queried without requiring a fully materialized struct." — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Relationships are not stored separately: they are foreign-key (URN) fields inside aspects annotated with `@Relationship`, from which the graph index is derived; "Relationships permit edges to be traversed bi-directionally." — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- "All Data ingested to DataHub's Metadata Service is validated against a PDL schema, with each @Aspect corresponding to a single schema." — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Entity registry: since January 2022 Snapshot models are deprecated for adding entities; entities are declared in `entity-registry.yml`, validated by GMS at startup, making new entities/aspects a configuration change. — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Core entities named: Data Platform, Dataset, Chart, Dashboard, Data Job, Data Flow. "Tables, Views, Streams, Document Collections, and Files are all modeled as 'Datasets' on DataHub." — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Versioned aspects: "all versioned aspects are stored inside a relational database that can be backed up and restored." Timeseries aspects (profiles, usage, operations) carry `timestampMillis` and "are NOT persisted inside the relational store" — they live only in the search index and Kafka, which complicates disaster recovery; they support range queries and aggregations (LATEST, SUM, CARDINALITY). — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Version numbering: each aspect version is immutable; version 0 is always the latest and older versions count up from 1. The stated reasons: Rest.li has no cheap "get max key", and computing max+1 under less-than-serializable isolation risks overwriting a version; with the v0 scheme "Repeatable Reads" (InnoDB default) makes a lost update fail rather than corrupt history. — [docs/advanced/aspect-versioning.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/advanced/aspect-versioning.md)
- Conditional writes: `If-Version-Match` and `If-Modified-Since` exist and from v1.9.0 also apply to generic JSON patches. — [DataHub releases (v1.9.0)](https://github.com/datahub-project/datahub/releases)

Ingestion
- The ingestion framework is "a modular, extensible Python library" that extracts from sources, transforms into the metadata model and writes "via either Kafka or using the Metadata Store Rest APIs directly"; supports schema extraction, profiling and usage extraction; driven by a YAML recipe and `datahub ingest`. — [docs/components.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/components.md)
- Connector count: "140+ source connectors" in the current Core-vs-Cloud table (older docs said 70+). — [managed-datahub-overview.md](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md)
- Kafka sink limitation: GMS's async (Kafka) path accepts only UPSERT/UPDATE/CREATE/CREATE_ENTITY (plus PATCH); DELETE and RESTATE are REST-only. — [metadata-ingestion/sink_docs/datahub.md](https://github.com/datahub-project/datahub/blob/master/metadata-ingestion/sink_docs/datahub.md) (search snippet)
- Async APIs "perform only basic schema validation when receiving MCP requests" (1.0-era note). — [docs.datahub.com releases](https://docs.datahub.com/docs/releases) (search snippet, not opened)
- Rollback semantics changed in v1.9.0: the UI Rollback button and `datahub ingest rollback --nuke` no longer delete entities written to by later runs or users. — [DataHub releases (v1.9.0)](https://github.com/datahub-project/datahub/releases)
- `datahub recipe probe` (v1.9.0) previews what a recipe would ingest. — [DataHub releases (v1.9.0)](https://github.com/datahub-project/datahub/releases)
- Actions framework consumes Kafka; from v1.6.0 offsets are committed asynchronously by default (higher throughput, possible redelivery). — [v1.6.0 release](https://github.com/datahub-project/datahub/releases/tag/v1.6.0)

Lineage, governance, ML, structured properties (availability in OSS "Core")
- Column-level lineage and impact analysis, business glossary, ownership management, incident management, quality/health status on asset profiles, metrics and semantic models, context documents, logical-models UI, GraphQL + MCP retrieval, full API/SDK: all in Core. — [managed-datahub-overview.md](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md)
- Structured properties: reworked in 1.0; full-page create/edit, orderable allowed values (v1.8.0); views can filter on structured properties (v1.8.0); stricter GMS validation and orphaned assignments dropped by default (v1.6.0); hard delete requires prior soft delete (v1.8.0). — [v1.8.0 notes](https://github.com/datahub-project/datahub/releases?page=2), [v1.6.0](https://github.com/datahub-project/datahub/releases/tag/v1.6.0)
- Data products can nest under a parent and are filterable in global search (v1.8.0). Domain-scoped policies are evaluated against proposed domains on new entities (v1.8.0). — [v1.8.0 notes](https://github.com/datahub-project/datahub/releases?page=2)
- ML entities: 1.0 made "AI/ML assets" first class: unified search across models, model groups and data assets, versioning, model statistics, end-to-end lineage; MLflow ingestion of model group versions, experiments and runs. Incidents can be raised on ML models, ML features, ML feature tables and schema fields (v1.8.0). — [v1.0.0 release](https://github.com/datahub-project/datahub/releases/tag/v1.0.0), [v1.8.0 notes](https://github.com/datahub-project/datahub/releases?page=2)
- Platform instances: a first-class concept; changing platform instance or domain is authorization-checked against old and new values (v1.9.0); Iceberg warehouses map to a Data Platform Instance. — [DataHub releases (v1.9.0)](https://github.com/datahub-project/datahub/releases), [docs/iceberg-catalog.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/iceberg-catalog.md)
- Access policies: privileges are fine-grained and steadily tightened through 2026 (timeseries reads require View Dataset Profile/Usage/Operations; query SQL requires View Entity Queries; non-expiring access tokens disabled by default; role/group membership changes require specific privileges). — [v1.8.0 notes](https://github.com/datahub-project/datahub/releases?page=2)

Iceberg REST catalog
- DataHub exposes an Iceberg REST catalog endpoint (`/iceberg/`) usable from Spark SQL and PyIceberg; "this feature is currently in open Beta". Warehouse -> Data Platform Instance (holds storage credentials); namespace -> container; table/view -> dataset whose URN is a UUID that "persists across renames". Credential vending via `X-Iceberg-Access-Delegation=vended-credentials` (STS sessions, default 3600 s). Limitations: "AWS S3 only support", "Multi-table transactional /commit is not yet supported", purge not implemented. — [docs/iceberg-catalog.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/iceberg-catalog.md)

AI / agent features
- "Agent Context Kit": guides, SDKs and an MCP server for agents to use business definitions, context documents, ownership, lineage and sample queries. — [Agent Context Kit docs](https://docs.datahub.com/docs/dev-guides/agent-context/agent-context) (search snippet, not opened)
- Open-source MCP server (`mcp-server-datahub` on PyPI) can run locally over stdio or as a shared HTTP service; has `search_documents` and `grep_documents` tools and a `SEMANTIC_SEARCH_ENABLED` flag (default false). `datahub-agent-context` on PyPI provides LangChain and Google ADK tool builders (versions 1.4.x–1.6.0.x). — [mcp-server-datahub PyPI](https://pypi.org/project/mcp-server-datahub/), [datahub-agent-context PyPI](https://pypi.org/project/datahub-agent-context/) (search snippets)
- Semantic search in Core: uses Elasticsearch 8.18+ with Vertex AI and local Ollama embedding providers (v1.6.0); a `semanticText` aspect provides curated embedding text (v1.6.0.3); unstructured document ingestion with chunking and embedding is in Core; a Google Drive source ingests Google Docs as Document entities (v1.9.0, alpha). — [v1.6.0](https://github.com/datahub-project/datahub/releases/tag/v1.6.0), [releases p2](https://github.com/datahub-project/datahub/releases?page=2), [releases](https://github.com/datahub-project/datahub/releases), [managed-datahub-overview.md](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md)

Project health
- GitHub: ~13.0k stars, 3.8k forks, 17,273 commits, Apache-2.0. README claims 750+ contributors, 16,000+ community members, "3,000+ organizations worldwide", named users Netflix, Visa, Etsy, Slack, Apple, FIS, Miro, and "10M+ assets in production". — [datahub-project/datahub](https://github.com/datahub-project/datahub)
- Release velocity Nov 2026: v1.9.0 on 16 Nov, v1.9.1 on 19 Nov, with five release candidates in the preceding three weeks and parallel security back-ports to the v1.6.0.x line. — [DataHub releases](https://github.com/datahub-project/datahub/releases)

### Inferences
- The essential architecture is "relational DB as source of truth for versioned aspects + Kafka change log + derived search and graph indices". Everything else (GraphQL, UI, policies, Iceberg REST, MCP) is layered on GMS. The search and graph indices are in principle rebuildable from the SQL store (the `RestoreIndices` job is referenced in v1.8.0 notes), but timeseries aspects are not, which is a real DR asymmetry.
- The DataHub "Dataset" is an intentionally overloaded abstraction (table, view, stream, file, document collection). This keeps the model small but pushes distinctions into `subTypes` and platform-specific properties.
- Relationships being derived from annotated fields inside aspects means an edge always has exactly one owning aspect/entity; this simplifies write atomicity but makes edge properties and many-to-many relationships with their own lifecycle awkward.
- The rate of authorization-related breaking changes in v1.6.0.3, v1.8.0 and v1.9.0 suggests the original permission model was coarse (e.g. "Edit Entity on a user" implied admin-like powers; entity GET exposed timeseries data) and has needed systematic hardening after 1.0.

### Gaps
- Kafka topic names (`MetadataChangeProposal_v1`, `FailedMetadataChangeProposal_v1`, `MetadataChangeLog_Versioned_v1`, `MetadataChangeLog_Timeseries_v1`, `PlatformEvent_v1`, `DataHubUsageEvent_v1`, `DataHubUpgradeHistory_v1`) and the MCE->MCP / MAE->MCL renaming history are from prior knowledge; only `DataHubUpgradeHistory_v1` was confirmed in a source this session.
- The SQL table layout (`metadata_aspect_v2` with urn, aspect, version, metadata JSON, systemmetadata, createdon, createdby) and Cassandra as an alternative primary store are from prior knowledge, not verified here.
- Not verified in this session (docs site unreachable): details of data contracts, assertions model, stateful ingestion (checkpointing and stale-entity soft-deletion), OpenAPI v3 entity endpoints, custom-entity/no-code model extension workflow (custom model plugins loaded into the entity registry), and whether a Kafka-less deployment mode exists in 2026. From prior knowledge: assertions and data contracts exist as entities in Core, but the monitoring engine that evaluates them is Cloud-only — consistent with the Core-vs-Cloud table in section 3.
- Shirshanka Das's "DataHub: Popular metadata architectures explained" (LinkedIn Engineering, Dec 2020) was not re-read; its three-generation taxonomy (monolith pull; service with push API; log-oriented/event-sourced) is reflected in the "3rd generation" language of the architecture doc.

---

## 3. DataHub 1.0 (2025) and subsequent releases through Nov 2026; open source vs DataHub Cloud

### Takeaway
1.0 (March 2025) was mostly a maturity statement — new V2 UI, first-class AI/ML assets, reworked structured properties, beta Iceberg REST catalog — and the 2025–2026 line (1.1 to 1.9.1) has added semantic search/documents, metrics and semantic models, agent/MCP tooling, a platform refresh (Spring Boot 4, Java 25, Play 3, ES 7 dropped, OpenSearch 3), and a long series of security/authorization hardening changes. The commercial split has widened: observability monitors, workflows/approvals, propagation, usage-aware ranking, AI documentation, hosted MCP and fine-grained access control are Cloud-only.

### Cited Findings

Release timeline (version, date, headline)
- v1.0.0: GitHub release page shows 17 March [2025]; the docs releases page lists 2025-03-14; a press release dated 31 Jan 2025 announced it. Highlights: new UI (platform-based navigation, reworked lineage view with column-level detail, integrated data-quality signals), AI/ML assets as first class, Iceberg REST Catalog (beta), CLI additions (container command, ingest MCPs from S3). New UI opt-in via `THEME_V2_DEFAULT=true`. — [v1.0.0 release](https://github.com/datahub-project/datahub/releases/tag/v1.0.0); date variants from [docs releases page](https://docs.datahub.com/docs/releases) and [BigDATAwire](https://www.bigdatawire.com/this-just-in/datahub-launches-fully-enterprise-ready-release-datahub-1-0) (search snippets)
- v1.4.0: 2026-02-06. New Java SDK V2 (fluent builder), Python SDK Tag entity, parametrized assertion runs, full Pydantic v2 migration; v1.4.0.3 fixed configuration for document cataloging and semantic search. — [docs releases page](https://docs.datahub.com/docs/releases) (search snippet, not opened)
- v1.5.0: 2026-03-24. V1 UI "officially sunset"; all development targets V2 UI. — [docs releases page](https://archive.docs.datahub.com/docs/1.5.0/releases) (search snippet, not opened)
- v1.6.0: published 21 May [2026]. V1 UI code removed; frontend on Play 3 / Apache Pekko; GMS on Spring Boot 4 / Spring Framework 7 / Spring Kafka 4; Docker images on Java 25; semantic search on Elasticsearch 8.18+ with Vertex AI and Ollama embeddings; SQLAlchemy profiler replaces Great Expectations as default; optional zero-downtime Elasticsearch upgrade path via Helm; new sources (Airbyte, Aerospike, Flink, dlt, Matillion, Fabric Data Factory, Omni, Pinecone, StarRocks, Informatica Cloud); search filter `value` field removed in favour of `values`; many connector URN shape changes (Athena, Fivetran, stored procedures, Vertex AI). Upgrade may involve reindexing and "an aspect schema version sweep on large catalogs". — [v1.6.0 release](https://github.com/datahub-project/datahub/releases/tag/v1.6.0)
- v1.6.0.x is being maintained as a patch line in parallel with newer minors (v1.6.0.3 on 25 Sep, v1.6.0.4 on 30 Oct, v1.6.0.5 on 12 Nov 2026, all security/authorization back-ports; v1.6.0.3 raised the Python floor to 3.11). The releases page labels it an LTS line in the fetched summary; the v1.6.0 release note itself does not use the term. — [releases](https://github.com/datahub-project/datahub/releases), [releases p2](https://github.com/datahub-project/datahub/releases?page=2)
- v1.7.0: 2026-08-04. — [docs releases page](https://docs.datahub.com/docs/releases) (search snippet only; content not retrieved)
- v1.8.0: 22 Oct 2026. OpenSearch 3.x support; Metrics and Semantic Models as upstreams with column-level lineage (off by default); dark mode; nested data products; new sources (Informix, Monte Carlo, SQLMesh); Search V3 document IDs become SHA-256 of the URN (existing V3 environments need RestoreIndices with clean, or a reindex); non-expiring tokens disabled; Great Expectations SQL profiler removed entirely; connector support tiers Alpha/Beta/GA. — [releases p2](https://github.com/datahub-project/datahub/releases?page=2)
- v1.9.0: 16 Nov 2026. Metrics on by default; Elasticsearch 7.x dropped (ES 8.x or OpenSearch 2.x/3.x required); Search V3 scoring/query-field changes; timeseries indices refresh every 60 s by default; system client credentials only sent to the configured GMS URL; Hazelcast deserialization allow-list; Airflow plugin dataset URNs changed to match S3/GCS/Azure Blob sources (old lineage stays on old URNs); SQLAlchemy 2.0 in ingestion. Requires CLI/SDK 1.7.0.19 and Helm chart 1.1.10. — [releases](https://github.com/datahub-project/datahub/releases)
- v1.9.1: 19 Nov 2026 (latest). Fix for OpenSearch 2.x semantic search returning nothing with multi-OR filters; OpenJDK 25.0.5.1 CVE refresh. — [releases](https://github.com/datahub-project/datahub/releases)
- DataHub Cloud has its own version line (v2.0.0 … v2.4.0 release notes exist). — [managed release notes v2.4.0](https://docs.datahub.com/docs/managed-datahub/release-notes/v_2_4_0) (search result title only)

Core vs Cloud (current master docs)
- In Core: 140+ connectors and unified search; dbt metric / semantic model ingestion; unstructured document ingestion, chunking and embedding; GraphQL and MCP retrieval; skills library; full API and SDK; native agent surfaces; column-level lineage and impact analysis; context documents (incl. import from GitHub); logical models UI; quality/health status display; incident management; ownership; business glossary; multi-language UI; community support. — [managed-datahub-overview.md](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md)
- Cloud-only: cross-platform query-history mining; AI documentation / context document generation; automated evals; change proposals and collaborative review; hosted and scoped MCP servers; "Ask DataHub" AI agent; agent registry; service catalog; usage-aware search ranking; lineage-based propagation and logical-model propagation; personalized home and asset views; notifications (email/Slack/Teams); AI anomaly detection; freshness/volume/schema/column/custom-SQL monitors; data health dashboard; assertion notifications; in-VPC validation; pipeline circuit breakers; bi-directional metadata sync; compliance forms and workflow engine; metadata tests; access-request workflows; fine-grained access control; PrivateLink, IP restrictions, in-VPC remote ingestion agent; 99.5% SLA. — [managed-datahub-overview.md](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md)
- README framing: OSS is self-hosted Apache 2.0 with "full access to the metadata graph and integrations"; Cloud is managed and includes data observability and the "full Context Platform". — [datahub-project/datahub](https://github.com/datahub-project/datahub)

### Inferences
- The table says "Fine-grained access control" is Cloud-only while Core clearly has a policy engine with many privileges (section 2). The Cloud feature is therefore likely a finer tier (e.g. aspect/field-level or attribute-based restrictions); the docs row alone does not say. Treat Core as having metadata-level RBAC/policies but not everything a regulated enterprise wants.
- "Usage-aware search ranking" being Cloud-only is notable: the single feature Amundsen was known for is paywalled in DataHub.
- The vendor has repositioned from "data catalog" to "context platform for AI agents" in 2025–2026 (table section names: Context Ingestion / Intelligence / Hub / Activation). The open-source project benefits (MCP server, semantic search, documents) but roadmap priority is clearly agent-facing features rather than, e.g., storage/replica modelling.
- 2026 has been a high-churn year for self-hosters: UI V1 removed, ES 7 dropped, Spring Boot 4, Search V3 document-ID change requiring reindex, GE profiler removed, repeated connector URN changes, many authz breaking changes. A separately maintained 1.6.0.x patch line appears to be the vendor's answer for conservative operators.

### Gaps
- Release notes for v1.1.0, v1.2.0, v1.3.0 (2025) and v1.7.0 (Aug 2026) were not retrieved; their dates other than v1.7.0 are unknown here. v1.3 existed by the time of GitHub issue #16041 (see section 4).
- The year on several GitHub release dates is inferred (page shows day/month only); the sequence and the explicit "2026" on v1.8.0 make 2026 near-certain for v1.6.0–v1.9.1.
- The Acryl Data -> "DataHub" company rebrand date and funding details were not sourced this session (prior knowledge: 2025, alongside a Series B). Unverified.
- Whether the 1.6.0.x line is formally designated "LTS" with a published support window is unconfirmed.

---

## 4. Scale and operational complexity

### Takeaway
Amundsen's footprint is small (three stateless Python services + Neo4j + Elasticsearch + an Airflow-scheduled ETL) but everything is rebuilt by batch and nothing is maintained upstream any more. DataHub's minimum production footprint is GMS + frontend + SQL DB + Kafka (+ schema registry, historically Zookeeper) + Elasticsearch/OpenSearch + system-update job, optionally separate MAE/MCE consumers, Neo4j and an Actions pod; the best-documented pain is search-index reindexing during upgrades, followed by breaking changes in connectors' URN shapes.

### Cited Findings
- DataHub quickstart needs Docker with 8 GB RAM and Python 3.10+; Minikube needs "more than 7GB of RAM … to run DataHub and its dependencies". — [datahub-project/datahub README](https://github.com/datahub-project/datahub), [docs/deploy/kubernetes.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/deploy/kubernetes.md)
- Moving parts: 4 DataHub components (GMS, optional standalone MAE and MCE consumers, frontend) plus Kafka, SQL DB, Elasticsearch, graph index (Neo4j or Elasticsearch), with Schema Registry and Zookeeper in the prerequisites chart example; dependencies can be managed services. — [docs/deploy/kubernetes.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/deploy/kubernetes.md)
- Reindex timeouts: on v1.3, system-update failed during reindex of a large Elasticsearch index because the operation timed out after 30 seconds, failing the whole upgrade. — [GitHub issue #16041](https://github.com/datahub-project/datahub/issues/16041) (search snippet)
- Reindex never converging: an upgrade reindex job "loops forever" on large, busy indices even with consumers stopped and `allowDocCountMismatch=true`, because the source keeps growing past the snapshot. — [GitHub issue #18949](https://github.com/datahub-project/datahub/issues/18949) (search snippet)
- The upgrade guide warns that deployments on Elasticsearch 8.x may trigger a full mappings reindex on every system-update run, with a workaround of disabling the reindex flag after the first successful update. — [Updating DataHub](https://docs.datahub.com/docs/how/updating-datahub) (search snippet, not opened)
- A Cloud release note (v2.2.x) describes a fix where stall detection could start a second reindex while the first was running, and the failure path could delete a fully copied destination index. — [managed release notes v2.2.0](https://docs.datahub.com/docs/managed-datahub/release-notes/v_2_2_0) (search snippet)
- Vendor support guidance: back up before upgrading because upgrades can migrate or reindex data; reindexing "can take significant time for large catalogs (10K+ entities)" and should run in maintenance windows. — [DataHub support: upgrading and rolling back](https://support.datahub.com/hc/en-us/articles/41912155382299-Upgrading-and-Rolling-Back-a-Self-Hosted-DataHub-Deployment), [DataHub support: index corruption](https://support.datahub.com/hc/en-us/articles/41912079306779-Elasticsearch-Index-Corruption-or-Mapping-Issues) (search snippets)
- v1.9.0 includes "a reindex race fix"; v1.6.0 added an optional zero-downtime Elasticsearch upgrade path through Helm; v1.8.0 changed Search V3 document IDs so that a fill without clean "leaves duplicates". — [releases](https://github.com/datahub-project/datahub/releases), [v1.6.0](https://github.com/datahub-project/datahub/releases/tag/v1.6.0), [releases p2](https://github.com/datahub-project/datahub/releases?page=2)
- Timeseries aspects are stored only in the search index and Kafka, not the relational store, complicating backup/restore. — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Ingestion fragility as visible in release notes: connector upgrades repeatedly change URNs so that existing lineage or metadata is orphaned ("old lineage stays on old datasets" for Airflow in v1.9.0; Snowflake `preserve_column_case` "changes every column's schemaField URN"; Sigma, Athena, Fivetran, Kafka Connect, Vertex AI URN changes in v1.6.0/v1.8.0). — [releases](https://github.com/datahub-project/datahub/releases), [releases p2](https://github.com/datahub-project/datahub/releases?page=2), [v1.6.0](https://github.com/datahub-project/datahub/releases/tag/v1.6.0)
- Security patch load: each 2026 release carries dozens of dependency CVE bumps across Java, Python and Node (Jackson, Netty, Spring, pac4j, aiohttp, etc.), and images bundle large Python dependency sets (spaCy model, `unstructured`, sqlglot). — [releases](https://github.com/datahub-project/datahub/releases), [releases p2](https://github.com/datahub-project/datahub/releases?page=2)
- An independent academic comparison rates DataHub's infrastructure overhead as "Very High … (Kafka, DBs)" and notes excessive resource consumption during setup. — [arXiv 2608.04724, GitOps-driven annotation catalog](https://arxiv.org/pdf/2608.04724) (search snippet; single paper from an unrelated domain, weak evidence)
- Comparison blogs agree that DataHub and Apache Atlas have the heaviest deployment requirements among open-source catalogs. — [Atlan open-source catalog guide](https://atlan.com/open-source-data-catalog-tools/) (vendor, search snippet)
- Amundsen at Lyft rebuilt metadata once per day via an Airflow DAG. — [amundsen docs/architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md)

### Inferences
- The support article's threshold of "10K+ entities" for "large catalogs" is strikingly low next to the "10M+ assets" marketing claim; it suggests that reindex time becomes operationally visible very early, and that multi-million-entity deployments need careful ES sizing and maintenance windows. A scientific file catalog with 10^8–10^9 file entries would be one to two orders of magnitude beyond the claimed proven scale if each file were an entity.
- The recurring class of bugs is "derived index out of sync with / being rebuilt from the source of truth while writes continue". Any design that derives search/graph projections from a log needs a first-class, tested, online re-projection mechanism (dual-write to a new index + atomic alias swap + bounded catch-up) from day one.
- Identity embedded in URNs (platform, name, env, casing, instance) is a long-term operational liability: any connector normalization change is a re-keying of the graph. DataHub's own Iceberg catalog sidestepped this with UUID URNs that survive renames — an implicit admission.
- Amundsen has roughly 5 stateful/stateless parts and DataHub roughly 8–11; but Amundsen's simplicity is bought by having no change stream, no history, and full batch re-extraction.

### Gaps
- No independent, quantitative benchmarks (ingest throughput, query latency, index size per entity) for DataHub at 10M+ entities were found.
- No Reddit/HN operator testimony was retrieved; the pain-point list is drawn from GitHub issues, the vendor's own upgrade/support docs and release notes, which under-represent "we gave up" stories.
- Whether DataHub in 2026 supports running without Kafka (e.g. an embedded/in-process log) or without a separate schema registry by default was not confirmed (prior knowledge: an internal GMS-hosted schema registry and KRaft-based Kafka removed the Confluent Schema Registry and Zookeeper requirements in 2023–2024; unverified here).
- Typical Neo4j/Elasticsearch sizing for Amundsen at scale was not sourced.

---

## 5. Fit for non-tabular scientific data (files, collections, replicas, versions, typed user metadata, HDF5/FITS/ROOT)

### Takeaway
Neither system models scientific data management concepts natively. DataHub can represent a file or a folder-of-files as a "Dataset" and can carry typed user metadata through structured properties and custom aspects, but has no notion of a logical file with multiple physical replicas, replica state, checksums/sizes as first-class indexed fields, dataset-of-files containment at 10^8+ scale, or HDF5/FITS/ROOT introspection. Amundsen has essentially nothing beyond tables. I found no evidence of a production scientific-research deployment of either for file cataloguing.

### Cited Findings
- DataHub: "Tables, Views, Streams, Document Collections, and Files are all modeled as 'Datasets'". — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- DataHub S3 / data-lake source: "It allows mapping an individual file or a folder of files to a dataset in DataHub"; schema inference requires `s3:GetObject` (reads object content). — [metadata-ingestion/docs/sources/s3/s3_pre.md](https://raw.githubusercontent.com/datahub-project/datahub/master/metadata-ingestion/docs/sources/s3/s3_pre.md)
- DataHub URN keys must be composed of required STRING/ENUM fields; identity is therefore a string tuple (for datasets: platform, name, environment), not a content hash or opaque ID. — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- DataHub platform instances distinguish multiple deployments of the same platform; in the Iceberg catalog a warehouse (storage root + credentials) is a platform instance. — [docs/iceberg-catalog.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/iceberg-catalog.md)
- Typed user metadata: structured properties (typed, with allowed values, validated by GMS, filterable in views) and custom aspects registered through the entity registry. — [releases p2 (v1.8.0)](https://github.com/datahub-project/datahub/releases?page=2), [v1.6.0](https://github.com/datahub-project/datahub/releases/tag/v1.6.0), [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Versioning: DataHub versions each aspect (metadata history) and since 1.0 has versioning for ML models/model groups. — [aspect-versioning.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/advanced/aspect-versioning.md), [v1.0.0 release](https://github.com/datahub-project/datahub/releases/tag/v1.0.0)
- DataHub's Iceberg REST catalog is S3-only in beta. — [docs/iceberg-catalog.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/iceberg-catalog.md)
- Amundsen indexes tables, dashboards, ML features and people; its connector list is SQL engines, Hive/Glue, Delta Lake, Kafka Schema Registry and BI tools; no file or object-store resource type is listed. — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- Amundsen adopters include the Chan Zuckerberg Initiative (a science funder; the listing does not say what data it catalogued). — [amundsen README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md)
- Name collision warning: "DataHub: A self-hosted, schema-driven catalogue for simulation and experimental data" (Zenodo; github.com/MatPro-IFE/DataHub) is an unrelated project (S3-compatible object store + PostgreSQL + Neo4j knowledge graph, reads VTK/NumPy/HDF5/XDMF/Exodus II). It is not LinkedIn/Acryl DataHub. — [Zenodo record 23378345](https://zenodo.org/records/23378345) (search snippet)
- A targeted search for DataHub (LinkedIn) deployments with HDF5/FITS/ROOT or "research data" returned no relevant deployment; results were scientific-catalog projects unrelated to DataHub (e.g. INL "Datum", which extracts metadata from HDF5, GeoTIFF, NetCDF, Parquet, Iceberg). — [INL Datum](https://inlsoftware.inl.gov/product/datum) (search snippet)

### Inferences
Mapping of scientific-catalog requirements onto DataHub's model (my assessment, built on the cited model facts):

| Requirement | DataHub | Amundsen |
|---|---|---|
| Single file as catalog entry | Possible (Dataset per file), but each entity costs several SQL aspect rows + a search doc + graph edges + Kafka messages; impractical at 10^8–10^9 files | Not modelled |
| File collection / dataset of files | Folder-as-dataset via path specs, or Container hierarchy; no first-class, queryable membership list with per-file size/checksum | Not modelled |
| Physical replicas across sites | No concept. Workarounds: one Dataset per (platform instance, path) linked by a "sibling"-style or custom relationship, or a custom aspect listing locations. No replica state, no replica-level operations | Not modelled |
| Data versions | Aspect history is metadata versioning, not data versioning; version-set style linking exists for ML assets since 1.0; Iceberg snapshots via the table format | Not modelled |
| Arbitrary typed user metadata | Good: structured properties (typed, validated, searchable/filterable) and custom aspects | Weak: free-form descriptions, tags, badges |
| Range/numeric queries over user metadata (e.g. run number between X and Y) | Limited to what the Elasticsearch mapping of structured properties supports; not a relational predicate engine | No |
| HDF5/FITS/ROOT introspection | None found; schema inference targets tabular/semi-structured formats | None |
| Lineage file -> file / job -> file | Yes in principle (dataset and DataJob/DataProcessInstance lineage, OpenLineage ingestion), but at dataset granularity | Shallow table lineage |

- The DataHub pieces that transfer well to a scientific catalog are generic: aspect-per-facet storage, typed structured properties with a registry, change log, URN-addressed relationships, policy model, and search projection. The pieces that do not transfer are the entity vocabulary (table-centric Dataset/SchemaField) and the per-entity cost model (assumes 10^5–10^7 entities, not 10^9 files).
- For a system serving both worlds, the logical-file/physical-replica split should be a core concept (like the Iceberg mapping: warehouse/storage-site as an instance, tables with stable UUID identity), not something encoded into URNs. Encoding site/path into identity (as DataHub dataset URNs do with platform+name+env) makes replicas look like unrelated datasets.

### Gaps
- No verified scientific/research-facility deployment of LinkedIn/Acryl DataHub or Amundsen for file-level cataloguing was found. Absence of evidence in a few searches is not proof; pharma/biotech companies likely use DataHub for tabular/ML assets, but I found no citable source this session.
- Supported file formats for the S3/ABS/GCS sources (prior knowledge: CSV, TSV, JSON/JSONL, Parquet, Avro, Delta) and the `path_spec` `{table}`/partition semantics were not verified; HDF5/FITS/ROOT support is, to my knowledge, absent.
- Whether DataHub's "sibling" relationship or dataset "version set" entities can be used generally for replicas/versions of arbitrary datasets in 1.9 was not verified.
- Largest known per-deployment entity counts beyond the "10M+" README claim are unknown.

---

## 6. Anchoring against alternatives: OpenMetadata, Apache Atlas, Unity Catalog OSS, Apache Gravitino, Marquez/OpenLineage

### Takeaway
As of late 2026 the open-source "catalog of catalogs" space has consolidated to DataHub and OpenMetadata, with Amundsen archived, Atlas legacy/heavy, and Marquez apparently stagnating while the OpenLineage specification lives on inside other tools. The growth area is "operational" catalogs that engines actually read/write through (Unity Catalog OSS, Apache Gravitino, Iceberg REST catalogs such as Polaris) — and Gravitino is the only one with a first-class non-tabular "fileset" concept, which makes it the most relevant comparator for scientific file collections. Most evidence in this section is from secondary sources and search snippets and should be re-verified.

### Cited Findings
- DataHub and OpenMetadata are consistently described as the two leading open-source metadata platforms in 2026. — [Atlan open-source catalog guide](https://atlan.com/open-source-data-catalog-tools/), [datalakehousehub: Metadata Platforms in 2026](https://datalakehousehub.com/blog/metadata-platforms-in-2026/) (vendor/blog, search snippets)
- Star counts conflict between secondary sources: Atlan says OpenMetadata "now has the larger GitHub following"; fastero lists ~10,500 stars for DataHub vs ~5,500 for OpenMetadata (mid-2026). The DataHub repo itself shows 13.0k stars in Nov 2026, so the fastero DataHub figure is stale or wrong, and neither OpenMetadata figure is verified. — [Atlan](https://atlan.com/open-source-data-catalog-tools/); contradicted by [fastero](https://fastero.com/blog/datahub-vs-openmetadata-open-source-data-catalogs); DataHub actual from [GitHub](https://github.com/datahub-project/datahub)
- OpenMetadata: reported 2.0.0 on 24 Aug 2026 and 2.0.7 on 27 Oct 2026; another source lists 2.0.2 as of September. — [revisit.tech: Data catalogs in 2026](https://revisit.tech/blog/data-catalogs-in-2026-revisit-openmetadata-datahub-and-apache-atlas-before-you-add-another-metadata-platform/), [toolsfordata](https://toolsfordata.com/lists/open-source-data-governance-tools/) (search snippets, unverified against GitHub)
- Apache Gravitino: 1.2.0 released 13 Mar 2026 (Table Maintenance Service, ClickHouse catalog, end-to-end UDF management, redesigned web UI); 1.2.1 patch 12 May 2026; reported to have graduated to an Apache Top-Level Project in June 2025. — [Gravitino blog](https://gravitino.apache.org/blog/) and secondary sources (search snippets)
- Unity Catalog OSS: 0.7.0 reported released 14 Oct 2026; Apache-2.0, LF AI & Data project started by Databricks; "a table and AI asset catalog that engines use to read and write data"; described as less mature outside Databricks. — [datastackindex](https://datastackindex.com/data-observability/compare/openmetadata-vs-unity-catalog/), [marmotdata](https://marmotdata.io/resources/open-source-data-catalog/) (search snippets)
- Apache Atlas: rated strong on lineage/governance in Hadoop ecosystems but heavy to deploy. — [Atlan](https://atlan.com/open-source-data-catalog-tools/) (search snippet)
- Marquez: described as lean and niche (OpenLineage reference implementation). Sources conflict on activity: one says no release since March 2025 and no commits since April 2026; the most consistent evidence in the search summary points to v0.50.0 (Oct 2024) as the last release. — [revisit.tech](https://revisit.tech/blog/data-catalogs-in-2026-revisit-openmetadata-datahub-and-apache-atlas-before-you-add-another-metadata-platform/) and others (search snippets; conflicting, unverified)
- DataHub itself now implements an Iceberg REST catalog (beta) and ingests from Unity Catalog, Glue, Dataplex, etc., i.e. it both overlaps with and federates the operational catalogs. — [docs/iceberg-catalog.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/iceberg-catalog.md), [v1.6.0](https://github.com/datahub-project/datahub/releases/tag/v1.6.0)

### Inferences
Anchored positioning (my synthesis; architecture characterisations of the alternatives are from prior knowledge unless cited above):

| System | Kind | Architecture in one line | Momentum late 2026 |
|---|---|---|---|
| DataHub | Passive/active metadata platform | SQL aspects + Kafka log + ES/OpenSearch + optional Neo4j; Java + Python | Strong, vendor-driven, pivoting to "context for AI agents" |
| OpenMetadata | Passive metadata platform | Single Java service + MySQL/Postgres + ES/OpenSearch; JSON-Schema-defined entities; no Kafka required; ingestion via Airflow or its own runner | Strong; 2.0 in 2026 (unverified details); the main "simpler to run" alternative |
| Amundsen | Discovery catalog | Flask services + Neo4j + ES + batch ETL | Archived Sep 2026 |
| Apache Atlas | Governance catalog (Hadoop era) | JanusGraph on HBase + Solr + Kafka hooks; type system with classifications propagating along lineage | Legacy; alive mainly inside Cloudera and as the base Atlan forked |
| Unity Catalog OSS | Operational catalog | REST server, 3-level namespace, tables + volumes + models + functions, credential vending | Slow pre-1.0 cadence (0.7.0); credibility depends on Databricks |
| Apache Gravitino | Federated operational "catalog of catalogs" | Java server federating Hive/Iceberg/JDBC/Kafka catalogs; first-class Fileset and Model catalogs; Iceberg REST service; GVFS virtual filesystem | Rising; ASF TLP, 1.x releases in 2026 |
| Marquez / OpenLineage | Lineage store / lineage event spec | Java service + Postgres storing OpenLineage run events | Marquez stagnant (conflicting reports); OpenLineage spec widely consumed (DataHub, OpenMetadata, Airflow, Spark, Flink) |

- Two distinct product categories are converging: (a) passive metadata graphs (DataHub, OpenMetadata) that describe assets, and (b) operational catalogs (Unity, Gravitino, Polaris/Iceberg REST) that engines resolve names and credentials through. A scientific data catalog with replica management is closer to (b) — it must be authoritative for "where are the bytes" — while needing (a)'s discovery, lineage and annotation features. DataHub's Iceberg REST endpoint shows category (a) reaching into (b).
- Gravitino's Fileset (a named, governed handle over a directory of non-tabular files, with multiple storage locations in recent versions, per prior knowledge) and Unity Catalog's Volumes are the nearest mainstream analogues to scientific file collections; both are directory-granular rather than per-file with replica state, so neither replaces a Rucio-style replica catalog.
- OpenMetadata is the cautionary counter-example to DataHub's Kafka-centric design: it reached comparable feature breadth with a request/response service and a relational store, emitting change events as an output (webhooks/event tables) rather than using a broker as the write path.

### Gaps
- None of the alternative projects' repositories were opened directly; versions, dates and star counts above are from secondary sources and, for Marquez and OpenMetadata stars, mutually inconsistent. Re-verify against GitHub before publishing.
- Gravitino fileset details (multiple locations per fileset, credential vending, GVFS) are from prior knowledge, not verified this session.
- Apache Polaris, Nessie, Lakekeeper and OpenLineage release status were not researched (out of the stated scope but relevant to the operational-catalog trend).
- No data on OpenMetadata's project health (contributors, release cadence) was verified.

---

## 7. Head-to-head summary and lessons for a new design

### Takeaway
Copy from DataHub: URN-addressed entities composed of independently written, schema-validated, versioned aspects in a relational source of truth; a typed registry for user-defined properties; a change log that every secondary index and integration consumes; and relationships derived from typed references. Do not copy: a broker in the mandatory write path, identity encoded into mutable naming strings, timeseries data living only in the search index, non-online reindexing, and a table-centric entity vocabulary. From Amundsen, keep only the lesson that a small deployable core plus usage-ranked search won early adoption — and that a catalog without a sustaining organisation and an extensible model does not survive.

### Cited Findings

Head-to-head (each row's facts are cited in sections 1–4)

| Dimension | Amundsen | DataHub |
|---|---|---|
| Status | Archived 29 Sep 2026; last functional commits Dec 2024 — [repo](https://github.com/amundsen-io/amundsen) | v1.9.1 on 19 Nov 2026; multiple releases per month — [releases](https://github.com/datahub-project/datahub/releases) |
| Community | 4.8k stars, 958 forks, 2,713 commits — [repo](https://github.com/amundsen-io/amundsen) | 13.0k stars, 3.8k forks, 17,273 commits, 750+ contributors claimed — [repo](https://github.com/datahub-project/datahub) |
| Source of truth | Graph DB (Neo4j / Atlas / Neptune) — [README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md) | Relational DB holding versioned aspects — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md) |
| Ingestion | Pull only, batch (daily Airflow DAG at Lyft) — [architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md) | Push and pull, via REST or Kafka; 140+ connectors — [components.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/components.md), [overview](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md) |
| Model extensibility | Fixed resource types — [README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md) | Entity registry (YAML) + PDL aspects; structured properties — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md) |
| Metadata history | None documented | Every aspect version immutable, v0 = latest — [aspect-versioning.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/advanced/aspect-versioning.md) |
| Search ranking | Usage-based ranking is the headline feature — [README](https://raw.githubusercontent.com/amundsen-io/amundsen/main/README.md) | Usage-aware ranking is Cloud-only; Core has Search V3 scoring + optional semantic search — [overview](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md), [releases](https://github.com/datahub-project/datahub/releases) |
| Lineage | Not in architecture doc (shallow, see Gaps §1) | Table, column and job-level lineage in Core — [overview](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md) |
| Governance | Owners, tags/badges (see Gaps §1) | Policies, domains, data products, glossary, incidents in Core; workflows/forms/tests Cloud-only — [overview](https://github.com/datahub-project/datahub/blob/master/docs/managed-datahub/managed-datahub-overview.md) |
| APIs | REST microservices, OpenAPI for search — [architecture.md](https://github.com/amundsen-io/amundsen/blob/main/docs/architecture.md) | GraphQL, Rest.li, OpenAPI, Kafka (Avro), MCP — [architecture.md](https://github.com/datahub-project/datahub/blob/master/docs/architecture/architecture.md), [releases p2](https://github.com/datahub-project/datahub/releases?page=2) |
| Ops footprint | 3 services + Neo4j + ES + ETL scheduler | GMS + frontend + SQL + Kafka + ES/OpenSearch (+ consumers, Neo4j, actions) — [kubernetes.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/deploy/kubernetes.md) |

Publicly documented DataHub design tensions
- Sequential version numbers were judged unworkable ("major challenges in both rest.li modeling & transaction isolation"), leading to the v0-is-latest scheme with read-modify-write under repeatable-read isolation. — [aspect-versioning.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/advanced/aspect-versioning.md)
- Snapshot-based modelling (one big union per entity) was abandoned in Jan 2022 in favour of per-aspect proposals and a YAML entity registry. — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- Timeseries aspects bypass the relational store, with the docs conceding this complicates recovery. — [metadata-model.md](https://github.com/datahub-project/datahub/blob/master/docs/modeling/metadata-model.md)
- The Kafka write path cannot express deletes and does only basic validation, so the "stream-first" API is not a complete write API. — [sink docs](https://github.com/datahub-project/datahub/blob/master/metadata-ingestion/sink_docs/datahub.md), [docs releases page](https://docs.datahub.com/docs/releases) (search snippets)
- Conditional writes (`If-Version-Match`) were retrofitted and only extended to JSON patches in v1.9.0 (Nov 2026). — [releases](https://github.com/datahub-project/datahub/releases)
- Rollback of ingestion runs could delete entities later touched by other writers until v1.9.0. — [releases](https://github.com/datahub-project/datahub/releases)
- Reindexing of derived search indices is a recurring upgrade failure mode (timeouts, non-convergence, duplicate docs, accidental index deletion). — [issue #16041](https://github.com/datahub-project/datahub/issues/16041), [issue #18949](https://github.com/datahub-project/datahub/issues/18949), [releases p2](https://github.com/datahub-project/datahub/releases?page=2)
- The Iceberg catalog integration chose UUID dataset URNs that persist across renames, unlike name-derived URNs elsewhere. — [docs/iceberg-catalog.md](https://raw.githubusercontent.com/datahub-project/datahub/master/docs/iceberg-catalog.md)

### Inferences

Worth copying
1. Aspect as the unit of write, validation, versioning and authorization. It lets independent producers (crawler, user, pipeline, policy engine) own disjoint facets of the same entity without read-modify-write conflicts on a monolithic record, and gives per-facet history for free. For a science + data-science catalog: `fileProperties`, `replicas`, `provenance`, `userMetadata`, `schema`, `ownership` as separate aspects.
2. Relational source of truth, everything else derived. DataHub's most durable decision is that versioned aspects sit in MySQL/Postgres and search/graph are projections. Keep that; make the projections truly rebuildable online.
3. A change log with a stable, schema'd envelope (entity, aspect, old value, new value, system metadata such as run ID and actor). It powers search indexing, graph edges, actions/automation, audit, federation and downstream sync. A transactional outbox table in the primary DB, tailed to consumers (and optionally bridged to Kafka/NATS), gives the same property without making a broker mandatory.
4. Schema-first model with a registry, plus a lightweight runtime path for user-defined typed properties (DataHub needed both: PDL aspects for platform developers and structured properties for users). For science use, the typed, validated, queryable user-metadata path is the primary one, not an add-on.
5. Stable entity references used uniformly across APIs, lineage and policies (the URN idea), and relationships declared as typed references inside aspects so that graph edges are derived, not separately maintained.
6. Ingestion as a library with recipes, connector support tiers, a dry-run/probe, per-run IDs in system metadata, and stateful stale-entity handling — but with safe rollback semantics from the start.
7. Soft delete (`status` aspect) and conditional writes (version match) as baseline API features.
8. An agent-facing surface (MCP server, document/semantic search) is now table stakes for 2026 catalogs; the Core implementation is small because it sits on the existing search and graph APIs.

Worth avoiding / doing differently
1. Do not put a message broker in the mandatory write path or the minimum deployment. The async Kafka API ended up weaker (no deletes, weaker validation) and Kafka + schema registry are a large share of the ops burden and of the "very high infrastructure overhead" criticism. Synchronous transactional write + outbox first; broker optional.
2. Do not derive identity from mutable or connector-dependent naming (platform, path, casing, instance, env). Use opaque stable IDs with names/paths as indexed, changeable attributes and alias lookup; DataHub's repeated URN-change breaking notes and its UUID choice for Iceberg tables are the evidence. This is essential when the same logical file has many physical paths.
3. Do not keep any class of metadata only in the search index. Store timeseries/statistics in the primary store (partitioned tables) or a real time-series store with backup semantics.
4. Design online re-projection (new index, dual write, verify, alias swap) and schema-migration sweeps as a core, continuously tested subsystem; DataHub's worst operator stories are here.
5. Do not overload one "Dataset" entity for everything when file-level scale differs by orders of magnitude. Separate cheap, high-cardinality records (files, replicas: narrow relational tables, 10^9 rows) from rich, low-cardinality catalog entities (datasets/collections, tables, models: aspect model, search-indexed, 10^5–10^7). Index the rich tier in search; query the cheap tier relationally.
6. Avoid three overlapping API surfaces (Rest.li, OpenAPI, GraphQL) with separately enforced authorization — several 2026 breaking changes exist only to bring them back into agreement. One canonical API with authorization enforced beneath it.
7. Be deliberate about the open-core boundary if there is one: DataHub's Cloud-only list (usage-aware ranking, propagation, monitors, approvals, fine-grained access control) is exactly what a self-hosting research organisation would expect built in.
8. Keep the dependency surface small: the per-release CVE churn across Java, Python and Node stacks is itself an operational cost.

Net recommendation for the new system: take DataHub's data-model ideas (entity + aspects + typed property registry + change log + derived indices) and OpenMetadata/Amundsen's deployment simplicity (one service + one relational DB + one search index), and add what none of them have: a first-class logical-file / physical-replica / storage-site model with stable IDs, collection membership at 10^9 scale, and data (not just metadata) versioning. Gravitino's fileset and Unity's volumes are the nearest industry concepts to interoperate with; OpenLineage is the lineage wire format to accept; an Iceberg REST facade is the likely way to expose tabular assets to engines.

### Gaps
- No first-party DataHub retrospective ("what we would do differently") was found; the regrets listed are inferred from design docs, breaking-change notes and issues rather than quoted admissions. Maintainer conference talks and the DataHub blog may contain explicit statements and were not searched exhaustively.
- Independent practitioner criticism (Reddit, HN, engineering blogs of adopters who migrated away) was not retrieved; the one independent data point is the arXiv comparison noting very high infrastructure overhead.
- LinkedIn's internal DataHub diverged from the open-source project years ago (prior knowledge); whether LinkedIn still runs something recognisably DataHub in 2026 was not researched.
- All recommendations about scale tiers (10^9 files) are design judgement; no benchmark of DataHub at that cardinality exists in the sources found.
