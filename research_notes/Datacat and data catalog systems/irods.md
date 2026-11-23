# iRODS (Integrated Rule-Oriented Data System) — technical evaluation, state as of Nov 2026

> **Method caveat (read first).** In this session direct page fetches to `irods.org`, `docs.irods.org` and `ncbi.nlm.nih.gov` were blocked by the network allowlist; only web search worked. Findings marked **[search-verified]** come from search-result extracts of the cited page during this session (2026-11-23). Findings marked **[doc-recall]** are long-standing, stable facts about iRODS that I am confident of from prior knowledge of the cited documentation, but whose page I could not re-open today; treat exact wording/version details there as needing a quick re-check. Nothing is invented; where I had no source, the item is under Gaps.

## Architecture: catalog, servers, zones, resources, plugins, clients

### Takeaway
iRODS is a classic 2000s-lineage "data grid": one relational catalog (iCAT) per zone, fronted by C++ servers speaking a bespoke TCP protocol, that virtualises heterogeneous storage behind a logical namespace. Everything (storage, auth, DB, network, rule engines, API) is a plugin, and the single RDBMS per zone is both the source of truth and the scaling ceiling.

### Cited Findings
- A zone has one catalog (iCAT) held in a relational DB; servers have a `catalog_service_role` of either `provider` (holds the DB connection, coordinates the zone) or `consumer` (talks to a provider, typically offers storage). **[search-verified]** — [iRODS docs glossary](https://docs.irods.org/4.2.0/system_overview/glossary/), [Installation docs](https://docs.irods.org/4.2.1/getting_started/installation/)
- Catalog database must be PostgreSQL, MySQL (MariaDB) or Oracle. **[search-verified via a secondary description]** — [PLOS ONE / PMC12795369](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12795369/); installation matrix at [docs.irods.org](https://docs.irods.org/5.0.2/getting_started/installation/) **[doc-recall]**
- Composable resources form a tree: "coordinating" resources (replication, compound, passthru, random, round-robin, load-balanced, deferred) are branch nodes carrying decision logic; "storage" resources (unixfilesystem, S3, etc.) are leaves. **[search-verified]** for the two-kind split — [iRODS docs glossary](https://docs.irods.org/4.2.0/system_overview/glossary/); full list at [Composable resources docs](https://docs.irods.org/5.0.2/plugins/composable_resources/) **[doc-recall]**
- Federation = two independent zones (each with own catalog) interoperating. Admins must exchange `catalog_provider_hosts`, `negotiation_key`, `zone_key`, `zone_name`, `zone_port`, define the remote zone in their own iCAT, and define each remote user locally (`user#zone`) before any ACL can be granted. **[search-verified]** — [Federation docs](https://docs.irods.org/4.2.8/system_overview/federation/)
- A published comparison characterises iRODS as "operating at a smaller scale than Rucio, typically targeted to single data centres". **[search-verified, single source]** — [JENA Computing Initiative WP2 report, arXiv:2503.09213](https://arxiv.org/pdf/2503.09213)
- iRODS 5.0.0 (20 May 2025) redesigned the server process model: proper signal handling, service-manager (systemd) status reporting, no automatic reload of `server_config.json`; the control plane, `irods-grid` and `irodsctl` were removed; all non-zone-wide config consolidated into `server_config.json`; the server no longer depends on the service account's `irods_environment.json`. **[search-verified]** — [iRODS 5.0.0 release post](https://irods.org/2025/05/irods-5-0-0-is-released/), [5.0.2 release notes](https://docs.irods.org/5.0.2/release_notes/)
- Deprecated things removed in 5.0: `imeta qu`, `imeta` interactive mode, DataObjLock/Unlock APIs, RBUDP transport, `msiSendMail`, and the ability to disable Strict ACLs. **[search-verified]** — [Dev update April 2025](https://irods.org/2025/04/irods-development-update-april-2025/), [UGM 2025 Technology Update slides](https://irods.org/uploads/2025/Draughn-iRODS-UGM2025_Technology_Update-slides.pdf)
- 5.0 tracks access time (atime) per replica, applied asynchronously in the background, queryable via GenQuery. **[search-verified]** — [iRODS 5.0.0 release post](https://irods.org/2025/05/irods-5-0-0-is-released/)
- 5.1.0 adds a new `irods` authentication scheme (to address limits of the legacy `native` scheme) and server-side protection of certain paths (override is temporary, removed in iRODS 6). **[search-verified]** — [iRODS 5.1.0 release post](https://irods.org/2026/10/irods-5-1-0-is-released/)
- Default checksum hash scheme stays MD5 in iRODS 5; moves to SHA-256 in iRODS 6. **[search-verified]** — [Dev update April 2026](https://irods.org/2026/04/irods-development-update-april-2026/)
- Client/access-layer ecosystem (maintained by the Consortium): iCommands (CLI), python-irodsclient (PRC), irods4j (Java, 0.7.0 in 2026; successor to Jargon), HTTP API (C++ REST-ish server; gained physical/logical quota endpoints in 2026; a Python wrapper reached 0.1.0 in March 2026), S3 API (presents iRODS as S3 endpoint; 0.6.0 June 2026, still pre-1.0, "basic support for Windows-based S3 GUIs"), NFSRODS (NFSv4.1 server), Metalnx (web UI), plus a CSI driver for Kubernetes. **[search-verified]** — [Dev update Sept 2026](https://irods.org/2026/09/irods-development-update-september-2026/), [Dev update March 2026](https://irods.org/2026/03/irods-development-update-march-2026/), [UGM 2026 S3 API slides](https://slides.com/alanking/ugm2026-irods-s3-api), [TRiRODS page](https://irods.org/trirods/)
- Wire protocol is a custom binary/packed-XML RPC over TCP, default port 1247, with server-to-server redirection and parallel-transfer high ports (default 20000–20199). **[doc-recall]** — [iRODS protocol docs](https://docs.irods.org/5.0.2/system_overview/process_model/)
- Plugin interfaces: resource, authentication, database, network, API, microservice, rule engine. **[doc-recall]** — [Plugins overview](https://docs.irods.org/5.0.2/plugins/plugin_overview/)
- Kubernetes: the official `irods/irods_k8s` repo is scoped to the Consortium's own test-request infrastructure, not a general production Helm chart. **[search-verified]** — [irods/irods_k8s](https://github.com/irods/irods_k8s)

### Inferences
- The architecture is "smart server, thin storage": all I/O is brokered by iRODS servers (clients are redirected to the resource server owning the replica). Data written to storage out-of-band is invisible until registered/scanned. This is the opposite of lakehouse catalogs, where engines hit object storage directly with vended credentials.
- No HA built in: catalog HA is delegated to the DBMS (Postgres replication/Patroni etc.) plus multiple providers behind a load balancer. Zone = unit of consistency; cross-zone is loosely coupled (no shared namespace, no global search).
- The bespoke protocol is a real adoption tax: firewalls (1247 + port range), custom client libraries per language, and the HTTP/S3 APIs are late (2023+) gateways still pre-1.0.
- 5.0 was largely a debt-paydown release (process model, config, removals), not a feature release — a sign of a mature but slow-moving codebase.

### Gaps
- Could not re-verify current exact supported DB versions/OS matrix for 5.1 (search indicates EL9/EL10, Ubuntu 22/24 (26.04 in progress), Debian 12/13; EL8/Ubuntu 20/Debian 11 dropped in 5.0 — [UGM 2025 build slides](https://irods.org/uploads/2025/Kitsinger-iRODS-iRODS_Build_and_Packaging_2025_Update-slides.pdf)).
- No source found for a community-supported production Helm chart/operator; appears not to exist officially.

## Data model: collections, data objects, replicas, resources, users/groups, AVUs, ACLs, tickets

### Takeaway
The model is a POSIX-like logical tree (collections → data objects) decoupled from physical replicas on resources, with free-form AVU triples attachable to four entity types and a per-object ACL system. It is file-centric: there is no first-class "dataset", "table", "version", or "schema" entity.

### Cited Findings
- Logical namespace `/zone/home/user/...`; a data object has 1..n replicas, each on a resource with its own physical path, status (good/stale/intermediate/locked), checksum, size, timestamps. Catalog tables: `R_COLL_MAIN`, `R_DATA_MAIN` (one row per replica), `R_RESC_MAIN`, `R_USER_MAIN`, `R_META_MAIN` (unique AVU triples), `R_OBJT_METAMAP` (object↔AVU links), `R_OBJT_ACCESS`. **[doc-recall]** — [iRODS docs: data objects](https://docs.irods.org/5.0.2/system_overview/data_objects/), [catalog schema in source](https://github.com/irods/irods/blob/main/plugins/database/src/icatSysTables.sql.pp)
- AVUs = attribute, value, optional unit; all three are strings; attachable to data objects, collections, resources, users (groups are users). AVU rows are shared across objects: detaching leaves orphan rows in `R_META_MAIN` which must be purged with `iadmin rum`; needed "only once unused rows start slowing the database", and notably slower on MySQL. **[search-verified]** — [iadmin docs](https://docs.irods.org/4.2.5/icommands/administrator/)
- No AVU composition (nested/structured values) and no native way to require that an object's AVUs conform to a template; authors state this "may hinder the implementation of the FAIR principles". **[search-verified]** — [van Schayck et al., Maastricht, JSON2AVU, UGM 2019](https://irods.org/uploads/2019/vanSchayck-Maastricht-JSON2AVU-paper.pdf)
- Permissions: classic levels null/read/write/own on data objects and collections with optional collection inheritance; 4.3.0 added ten granular levels (read_metadata, read_object, create_metadata, modify_metadata, delete_metadata, create_object, modify_object, delete_object, own). **[doc-recall]** — [4.3.0 release notes](https://docs.irods.org/4.3.0/release_notes/)
- Strict ACLs can no longer be disabled as of 5.0. **[search-verified]** — [UGM 2025 Technology Update slides](https://irods.org/uploads/2025/Draughn-iRODS-UGM2025_Technology_Update-slides.pdf)
- Tickets: bearer tokens granting read or write on a specific object/collection, with optional expiry, use-count/byte limits and host/user/group restrictions. **[doc-recall]** — [iticket docs](https://docs.irods.org/5.0.2/icommands/user/#iticket)
- GenQuery2 in 5.0 honours group permissions (GenQuery1 behaviour differed), and 5.1.0 fixed a group-permission expansion bug. **[search-verified]** — [5.0.2 release notes](https://docs.irods.org/5.0.2/release_notes/), [5.1.0 release post](https://irods.org/2026/10/irods-5-1-0-is-released/)
- Sanger reports, in production on 4.2.7, replica failures leaving data objects with single, dirty (stale) or triple replicas that must be detected and repaired. **[search-verified]** — [UGM 2024 programme/abstracts](https://irods.org/ugm2024/)
- KU Leuven "A Year of iRODS" lists replica management "not yet robust enough" among lessons learned. **[search-verified]** — [Barcena, KU Leuven, UGM 2021](https://irods.org/uploads/2021/Barcena-KULeuven-A_Year_of_iRODS_Lessons_Learned-slides.pdf)
- Yoda's 2025 talk mentions building iRODS "data integrity metrics" to find issues such as missing checksums. **[search-verified]** — [UGM 2025](https://irods.org/ugm2025/)

### Inferences
- The strong idea worth copying: **logical object ≠ physical replica**, with replica state tracked in the catalog and placement delegated to a resource tree. This is exactly what lakehouse catalogs lack and what "real science" needs.
- Weak points to avoid: (a) one row per replica in the same table as the logical object (R_DATA_MAIN) — makes object-level updates multi-row and has historically produced replica-state inconsistencies; (b) no immutable versions/snapshots — overwrite is in place; (c) no dataset-level entity, so "a dataset" is by convention a collection; (d) ACL rows per object × per user/group → large `R_OBJT_ACCESS` and expensive permission-joined queries.
- Checksums are optional and MD5 by default until iRODS 6, so integrity is a policy the site must add — multiple large sites built their own integrity audits.

### Gaps
- No authoritative published statement of practical limits (max AVUs per object, objects per collection) — a Google-group thread exists ([iRODS-Chat](https://groups.google.com/g/Irod-Chat/c/6Ef7WP59uHo)) but content not retrieved.

## Policy / rule engine: languages, PEPs, delayed rules, capabilities, policy composition — how well does it work?

### Takeaway
The rule engine is iRODS's signature feature and its biggest operational liability: enormously flexible event hooks on every server operation, but policies are site-specific code with a steep learning curve, weak tooling, and historically fragile async execution. The packaged "capabilities" help, but the long-promised declarative "policy composition" only reached a 0.1.0 release in April 2026.

### Cited Findings
- Each server runs a rule engine triggered by events (policy enforcement points); rule engine plugin framework supports the native iRODS Rule Language and Python (plus C++ plugins). **[search-verified]** for the event model — [PMC12795369](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12795369/); [Python REP repo](https://github.com/irods/irods_rule_engine_plugin_python)
- Dynamic PEPs fire `pre`/`post`/`except`/`finally` around every plugin operation (API, resource, database, auth, network, microservice) — hundreds of hook points. **[doc-recall]** — [Dynamic PEP docs](https://docs.irods.org/5.0.2/plugins/dynamic_policy_enforcement_points/)
- Delayed rules are queued in the catalog and executed by a single delay server per zone; the Consortium had to fix the queue being blocked by long-running rules and queue processing slowing as the queue grows, and to add delay-server migration between hosts. **[search-verified]** — [Russell & Draughn, iRODS Delay Server Migration, UGM 2022](https://irods.org/uploads/2022/Russell-Draughn-iRODS-iRODS_Delay_Server_Migration-paper.pdf)
- 5.1.0 release notes still list fixes for memory leaks in the native Rule Language. **[search-verified]** — [5.1.0 release post](https://irods.org/2026/10/irods-5-1-0-is-released/)
- Packaged capabilities and their 2026 status (all from Consortium's own updates): storage tiering (released for 5.1.x; built from simpler policies: replication, verification, retention, violating-object discovery); indexing to Elasticsearch/OpenSearch (released with 4.3.5.0 March 2026, no later release noted by Sept 2026); audit via AMQP (5.0.2; long-lived connections and multi-endpoint support still in progress; typically paired with Logstash/Elasticsearch dashboards); logical quotas (native REP "nearing completion" April 2026, counting bug fixed Sept 2026, being integrated into the server); automated ingest (Python/Celery-based scanner syncing filesystem or S3 bucket into the catalog; CI added 2026); physical quotas still being improved Jan 2026. **[search-verified]** — [Dev update Nov 2026](https://irods.org/2026/11/irods-development-update-november-2026/), [Sept 2026](https://irods.org/2026/09/irods-development-update-september-2026/), [April 2026](https://irods.org/2026/04/irods-development-update-april-2026/), [March 2026](https://irods.org/2026/03/irods-development-update-march-2026/), [Jan 2026](https://irods.org/2026/01/irods-development-update-january-2026/), [TRiRODS](https://irods.org/trirods/)
- Policy Composition rule engine plugin: described in March 2026 as "long-awaited... very close"; first release 0.1.0 in April 2026; last activity listed as April 2026 in the August update. (The concept was presented at UGMs from ~2019–2020.) **[search-verified]** — [Dev update March 2026](https://irods.org/2026/03/irods-development-update-march-2026/), [April 2026](https://irods.org/2026/04/irods-development-update-april-2026/), [Aug 2026](https://irods.org/2026/08/irods-development-update-august-2026/)
- KU Leuven: "first step is easy (demo setup)", then steep learning curve, documentation gaps, and difficulty working out how to write rules for domain-specific use cases; conclusion still positive ("flexible and powerful"). **[search-verified]** — [Barcena, UGM 2021](https://irods.org/uploads/2021/Barcena-KULeuven-A_Year_of_iRODS_Lessons_Learned-slides.pdf)
- Early academic criticism: the complexity of the iRODS rule language makes it unsuitable for workflow designers; suggests a workflow engine layered on top instead. **[search-verified; ~2008–2010, dated]** — [Rule-based curation and preservation of data (ResearchGate)](https://www.researchgate.net/publication/220285257_Rule-based_curation_and_preservation_of_data_A_data_grid_approach_using_iRODS)

### Inferences
- In practice sites fall into two camps: (1) minimal rules + external orchestration (Sanger's baton/partisan tooling, cron jobs), or (2) a large bespoke ruleset that becomes a product in itself (Yoda's ruleset, KU Leuven/ManGO, CyVerse). Either way the policy layer is the upgrade-blocker: PEP signatures and microservice behaviour shift between minor versions.
- Synchronous PEPs run in the request path: a slow or buggy rule degrades every client operation; there is no sandboxing, versioning, dry-run or unit-test harness for policy.
- Events are *pull-your-own*: the audit plugin emits PEP-level AMQP messages (very verbose, low-level), not a curated, stable, object-level change-event stream with ordering/replay guarantees. A new design should make a durable event log (outbox/CDC) a first-class API and keep policy *outside* the request path where possible.
- Design lesson: declarative, data-driven policy ("replicate N copies to class X; tier after 30d; verify every 90d") — which is what Rucio's replication rules do and what iRODS policy composition aspired to — beats a Turing-complete hook language for 90% of needs. iRODS took ~6 years to ship a 0.1.0 of the declarative version.

### Gaps
- No independent (non-Consortium) production evaluations of the indexing, tiering or policy-composition capabilities found; real-world robustness is unverified. KU Leuven noted using OpenSearch/Elasticsearch separately for search, which is weak evidence that sites wire their own.
- Could not confirm whether "5.1.1" refers to the server or only to the storage-tiering plugin version (search extract ambiguous).

## Search and query: GenQuery / GenQuery2, specific queries, AVU limits, metadata templates

### Takeaway
Query is the weakest part of iRODS as a *catalog*: a restricted SQL-ish projection language over the iCAT with server-chosen joins, untyped string AVUs in an EAV schema, no full-text or faceting natively, and no schema enforcement. GenQuery2 (default-capable in 5.x) fixes syntax, not the underlying EAV model; schema/template efforts remain outside the core after ~9 years.

### Cited Findings
- GenQuery2: syntax mirrors standard SQL where possible — CAST, GROUP BY, aggregates, ORDER BY, `FETCH FIRST N ROWS ONLY`/LIMIT, `=, !=, <, <=, >, >=, LIKE, BETWEEN, IN, IS [NOT] NULL`, AND/OR/NOT with parentheses, case-insensitive keywords. It deliberately does not expose FROM: the server decides the joins from the requested columns so the permission model can be enforced. **[search-verified]** — [Draughn, GenQuery2 paper, UGM 2023](https://irods.org/uploads/2023/Draughn-iRODS-GenQuery2_A_more_standardized_powerful_parser_for_the_iRODS_namespace-paper.pdf), [irods/irods#6168](https://github.com/irods/irods/issues/6168)
- In 5.0 GenQuery2 gives the user control of DISTINCT and honours group permissions; this is NOT backward compatible with 4.3.4 and earlier. 5.1.0 adds columns exposing when metadata was attached. python-irodsclient docs still described GenQuery2 as "experimental... subject to change". **[search-verified]** — [5.0.0 release post](https://irods.org/2025/05/irods-5-0-0-is-released/), [5.1.0 release post](https://irods.org/2026/10/irods-5-1-0-is-released/), [python-irodsclient on PyPI](https://pypi.org/project/python-irodsclient/)
- Specific queries: admin-registered raw SQL statements run by alias — the escape hatch when GenQuery cannot express something; they bypass the permission-enforcing join logic and are DB-dialect-specific. **[doc-recall]** — [iquest / specific query docs](https://docs.irods.org/5.0.2/icommands/user/#iquest)
- Historical scale validation of AVU search: after adding five DB indexes and fixing leaks (iRODS 2.4), `imeta` queries were tested at ~50 million data objects / 300 million AVUs. This is the largest *documented* AVU-query test I could find and is >10 years old. **[search-verified]** — [Legacy release notes](https://docs.irods.org/4.2.2/legacy_release_notes/)
- Community report: because of the flat (EAV) schema the DB planner can pick different plans for identical searches, e.g. 2 minutes vs 15 minutes. **[search-verified; informal, undated]** — [iRODS-Chat thread](https://groups.google.com/g/Irod-Chat/c/6Ef7WP59uHo)
- GitHub issue: in a zone with >110,000 users, listing `/<zone>/home` takes over 5 minutes (permission-joined collection listing). **[search-verified]** — [irods/irods#7045](https://github.com/irods/irods/issues/7045)
- Metadata templates: 2017 whitepaper proposed a JSON-schema-based template with a Jargon reference implementation; the Metadata Templates Working Group (2019→) concluded iRODS should *not* define or manage templates itself — the server's role is at most validation — and that templates be namespaced via attribute-name prefixes. KU Leuven's ManGO implements this convention (schema identifier as AVU-name prefix). **[search-verified]** — [2017 Metadata Templates whitepaper](https://irods.org/uploads/2017/20170315-MetadataTemplates-Whitepaper.pdf), [WG "Building Blocks and Lessons Learned", UGM 2024](https://irods.org/uploads/2024/Russell-iRODS-iRODS_Metadata_Templates_Working_Group_Building_Blocks_and_Lessons_Learned-slides.pdf), [Montes, KU Leuven, UGM 2023](https://irods.org/uploads/2023/Montes-KULeuven-Towards_rich_and_standardized_metadata_in_iRODS-paper.pdf)
- Maastricht proposed JSON Schema templates plus a JSON↔AVU bidirectional encoding (using the unit field to encode nesting) with optional server-side validation policy. **[search-verified]** — [JSON2AVU paper](https://irods.org/uploads/2019/vanSchayck-Maastricht-JSON2AVU-paper.pdf)
- In 2026 a University of Groningen paper proposed "iRODS Labels" as a complementary mechanism to AVUs, needing new DB tables and changes to GenQuery/GenQuery2 column mapping and SQL generation — i.e. users are still proposing core data-model changes to work around AVU limits. Proposal only; not known to be merged. **[search-verified]** — [Tsyganov, Groningen, UGM 2026 paper](https://irods.org/uploads/2026/Tsyganov-Groningen-iRODS_Labels_A_Proposal_for_a_Complementary_Labeling_Mechanism_in_iRODS-paper.pdf)
- Full-text / faceted search is delivered only by exporting to Elasticsearch/OpenSearch through the indexing capability. **[search-verified]** — [TRiRODS](https://irods.org/trirods/), [Dev update March 2026](https://irods.org/2026/03/irods-development-update-march-2026/)

### Inferences
- AVU pathologies to design against: values are strings (numeric/date comparisons need CAST and defeat indexes); the "unit" field is abused as a namespace/structure carrier; each additional metadata predicate adds a self-join pair on `R_META_MAIN`/`R_OBJT_METAMAP`, so multi-attribute queries degrade combinatorially; shared-AVU rows make updates copy-on-write and leave garbage (`rum`).
- Permission enforcement inside the query (join to `R_OBJT_ACCESS` + group expansion) is correct but is the dominant cost for non-admin queries on large zones (#7045).
- The WG outcome — "templates are a client/convention concern" — means there is no interoperable schema layer across iRODS sites; every portal (Yoda, ManGO, Metalnx templates, CyVerse DE) invented its own. A new catalog should make typed, versioned, namespaced schemas a core feature with server-side validation, and store metadata as typed columns/JSONB with proper indexes rather than global EAV.
- Search-via-sidecar (ES/OpenSearch) creates a second, eventually-consistent, separately-secured index; ACL trimming in the search index is the hard part and is left to the site.

### Gaps
- No current (2023+) public benchmark of GenQuery/GenQuery2 metadata queries at 10⁸–10⁹ objects found.
- Fate of the "iRODS Labels" proposal (accepted/merged?) unknown.

## Scale and performance: largest deployments, bottlenecks, pain points

### Takeaway
Proven envelope: tens of PB and on the order of 10⁸ data objects per zone, with a handful of sites beyond that; the relational catalog (and everything being synchronous through it) is the bottleneck. Marketing claims of "billions of files / hundreds of PB" are not backed by a named, verifiable deployment in what I could find.

### Cited Findings
- Wellcome Sanger: ~14 PB total iRODS capacity across two data centres (Jan 2018); "over a hundred iRODS servers, mostly storage, around 50PB (most replicated so half usable)" (UGM 2022); "20 petabytes of genomic data" managed, production still on iRODS 4.2.7 (UGM 2024 abstract); iRODS FAQ cites ">24 PB, replicated to nearly 48 PB" (undated). **[search-verified]** — [Sanger HPC blog 2018](https://hpc-news.sanger.ac.uk/2018/01/15/more-and-more-pb), [Constable, Sanger, UGM 2022 slides](https://irods.org/uploads/2022/Constable-Sanger-Data_the_final_frontier_These_are_the_voyages_of_the_Informatics_Digital_Solutions_team_at_Sanger-slides.pdf), [UGM 2024](https://irods.org/ugm2024/), [iRODS FAQ](https://irods.org/faq)
- Other FAQ figures (old, undated): CC-IN2P3 80M files; iPlant/CyVerse >110M files, ~1 PB, >20k users. **[search-verified]** — [iRODS FAQ](https://irods.org/faq)
- CyVerse-scale user counts: a zone with >110,000 users where listing `/zone/home` takes >5 min. **[search-verified]** — [irods/irods#7045](https://github.com/irods/irods/issues/7045)
- Yoda (Utrecht + other Dutch universities, on iRODS): >10,000 users / >3 PB (2023); >13,000 users / >4 PB (2025). **[search-verified]** — [UGM 2023](https://irods.org/ugm2023/), [UGM 2025](https://irods.org/ugm2025/), [Utrecht University news](https://www.uu.nl/en/news/five-other-dutch-universities-start-using-yoda-utrecht-universitys-research-data-management-service)
- Upgrade cost scales with catalog size: the 4.2→4.3-era move to ID-based resource hierarchies requires a one-time full-table update of all data objects — 13 minutes for 10 M objects in the lab, estimated 2–3 hours per 100 M. **[search-verified]** — [4.3.5 release notes](https://docs.irods.org/4.3.5/release_notes/)
- Old performance study: with a ~13 M-file catalog, transfer time for small files rose ~8× because catalog writes get slower as the DB grows. **[search-verified; ~2010, old versions]** — [Managing large datasets with iRODS — a performance analysis](https://www.researchgate.net/publication/224209303_Managing_large_datasets_with_iRODS-A_performance_analysis)
- A Jan-2026 PLOS ONE paper (IT4Innovations / VSB Ostrava) evaluates an iRODS-based setup for distributed computational workflows and its performance. **[search-verified existence only; body not retrievable]** — [PMC12795369](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12795369/), [VSB repository](https://dspace.vsb.cz/items/f8604803-6860-43f1-ad1b-359e553e1712)
- DZone claims organisations have scaled iRODS to "hundreds of petabytes across multiple data centers" without naming any. **[search-verified; unreliable/unattributed]** — [DZone](https://dzone.com/articles/irods-an-open-source-approach-to-data-management-i)
- S3 as backing storage: early PoC (Bayer, 2016) reported catalog↔bucket consistency issues and rename timeouts because logical paths were used as object keys; the S3 resource plugin has since been rewritten to stream directly to S3 (plugin version tracks server, 5.1.0 in Aug 2026). **[search-verified]** — [Bayer, iRODS and Objectstorage, UGM 2016](https://irods.org/uploads/2016/06/Bayer_iRODS-and-Objectstorage_UGM2016.pdf), [UGM 2020](https://irods.org/ugm2020/), [Dev update Aug 2026](https://irods.org/2026/08/irods-development-update-august-2026/)

### Inferences
- Pain-point pattern consistent across sources: (1) many small files — every create is several synchronous DB transactions plus an agent process per connection (process-per-connection model), so ingest rates are bounded by DB round-trips, not storage; sites tar/bundle or bulk-register; (2) catalog DB is a single writer node; vertical scaling + PgBouncer + careful indexing is the only path, no sharding; (3) large collections and permission-joined listings; (4) upgrades: Sanger was still on 4.2.7 (released 2019/2020) in 2024, i.e. the flagship user lagged ~2 major-minor series — strong evidence that upgrades are costly; the Consortium shipped 4.3.5 (133 issues closed) in March 2026 specifically "for deployments which cannot upgrade to iRODS 5 yet".
- For comparison, Rucio (ATLAS) manages >1 billion files / exabyte scale across 130+ sites with a similar single-RDBMS design but a far narrower, purpose-built schema and no per-file ACLs or EAV metadata in the hot path — suggesting iRODS's ceiling comes from schema generality and synchronous policy, not from "RDBMS catalog" per se.

### Gaps
- No verified current object count for any top deployment (Sanger, CyVerse, SURF, KU Leuven, CC-IN2P3). "Billions of objects" is asserted in Consortium material ([Yoda 2018 slides quoting it](https://irods.org/uploads/2018/Smeele-UtrechtUni-Yoda-slides.pdf)) but I found no named zone with ≥10⁹ data objects.
- Whether Sanger has since upgraded to 4.3/5.x is unknown.
- PLOS ONE 2026 paper's actual numbers not retrieved.

## Current status: releases, cadence, governance, license, community, users

### Takeaway
iRODS is alive, professionally maintained and still gaining members (IBM, Scripps, NCHC in 2026), but it is a small-team project with a slow cadence (roughly one feature release per year, two maintained series), concentrated in life-science/academic HPC and European research-data-management services.

### Cited Findings
- Release timeline: 4.3.0 (June 2022 **[doc-recall]**) … 4.3.5 (March 2026, closes 133 issues, declared the final 4.3 release); 5.0.0 (20 May 2025); 5.0.1 (27 May 2025, fixes infinite redirect loop on hostname misconfiguration); 5.0.2 (Oct 2025; adds EL10 and Debian 13, fixes for irsync, GenQuery2, unattended install, physical quotas); 5.1.0 (announced Oct 2026 after "more than a year" of development; was "still creeping towards a release" in Aug 2026). **[search-verified]** — [5.0.0](https://irods.org/2025/05/irods-5-0-0-is-released/), [5.0.1](https://irods.org/2025/05/irods-5-0-1-is-released), [5.0.2 announcement](https://groups.google.com/g/iROD-Chat/c/XFoZJhNIPF0), [4.3.5 announcement](https://groups.google.com/g/iROD-Chat/c/VCmSfIPQWoU), [Dev update March 2026](https://irods.org/2026/03/irods-development-update-march-2026/), [Dev update Aug 2026](https://irods.org/2026/08/irods-development-update-august-2026/), [5.1.0](https://irods.org/2026/10/irods-5-1-0-is-released/)
- Versioning from 5.0: major releases are 5.0, 6.0, 7.0…; UGM 2026 Consortium Update lists "groundwork for iRODS 6 and 7". **[search-verified]** — [UGM 2025 Technology Update slides](https://irods.org/uploads/2025/Draughn-iRODS-UGM2025_Technology_Update-slides.pdf), [UGM 2026 Consortium Update](https://slides.com/irods/ugm2026-consortium-update)
- Governance: iRODS Consortium, administered by RENCI (UNC Chapel Hill), founded 2013 by RENCI, DICE and DDN; membership-led via monthly Technology Working Group and Planning Committee meetings, with an executive board over budget/bylaws; membership tiers $12k–$90k/year; 4 new members between UGM 2025 and UGM 2026; recent joiners NCHC Taiwan (Jan 2026), IBM (June 2026), Scripps Research (Sept 2026). **[search-verified]** — [About the Consortium](https://irods.org/about), [UGM 2026 Consortium Update slides](https://irods.org/uploads/2026/Russell-iRODS-UGM2026_Consortium_Update-slides.pdf), [IBM joins](https://irods.org/2026/06/ibm-joins-the-irods-consortium), [Scripps joins](https://www.storagenewsletter.com/2026/09/03/scripps-research-joins-the-irods-consortium/), [NCHC joins](https://www.storagenewsletter.com/2026/01/28/national-center-for-hpc-joins-the-irods-consortium/), [Invest in Open Infrastructure profile](https://infrafinder.investinopen.org/solutions/irods)
- License: BSD 3-Clause. **[doc-recall]** — [irods/irods LICENSE](https://github.com/irods/irods/blob/main/LICENSE)
- UGM 2026 was held 29 June – 2 July 2026 in Barcelona; Consortium Update highlights integrations/partnerships with Globus, OMERO, RSpace and Dataverse. **[search-verified]** — [UGM 2026](https://irods.org/ugm2026/), [Consortium Update](https://slides.com/irods/ugm2026-consortium-update)
- Notable users with evidence in this session: Wellcome Sanger (genomics, 20–50 PB), CyVerse/iPlant, CC-IN2P3, Utrecht University Yoda (+ five other Dutch universities, also offered by SURF as "SURF Yoda"), KU Leuven/VSC (ManGO), Maastricht (DataHub), University of Groningen, Bayer (2016 talk), TACC (member since 2018), IT4Innovations. **[search-verified]** — [iRODS FAQ](https://irods.org/faq), [SURF Yoda at EUR](https://www.eur.nl/en/research/research-services/research-data-management/surf-yoda), [KU Leuven UGM 2021](https://irods.org/uploads/2021/Barcena-KULeuven-A_Year_of_iRODS_Lessons_Learned-slides.pdf), [Bayer UGM 2016](https://irods.org/uploads/2016/06/Bayer_iRODS-and-Objectstorage_UGM2016.pdf), [TACC joins](https://irods.org/2018/09/tacc-joins-the-irods-consortium)

### Inferences
- Bus factor: core development is a single small team at RENCI funded by membership fees; no evidence of substantial outside core contributors. Roadmap is steered by paying members, which favours stability for incumbents over modern-catalog features.
- Cadence 2022–2026: ~5 patch releases on 4.3 over 4 years, 5.0 in 2025, 5.1 in late 2026 — i.e. ~12–17 months between feature releases. Many ecosystem pieces remain pre-1.0 after years (S3 API 0.6.0, policy composition 0.1.0, HTTP API Python wrapper 0.1.0).
- Adoption is sticky rather than growing fast: users are institutions with long-lived archives; I found no notable industry "data platform" team adopting iRODS for analytics/ML.

### Gaps
- Full current member list and total member count not retrieved; NIEHS and Bayer current membership unverified in this session (both are historically associated with iRODS).
- No hard numbers on community size (contributors, GitHub stars, mailing-list traffic) retrieved.
- UGM 2026 Technology Update content (iRODS 6 roadmap specifics) not retrieved.

## What iRODS lacks relative to modern data catalogs

### Takeaway
iRODS is a file-virtualisation and policy system with a metadata side-table, not a data catalog in the DataHub/Unity/OpenMetadata/Iceberg-REST sense: it has no notion of tables, schemas, columns, versions, lineage graphs, or a discovery product, and its event and API surfaces are low-level.

### Cited Findings
- No structured/typed metadata or template enforcement in core (see AVU findings). — [JSON2AVU paper](https://irods.org/uploads/2019/vanSchayck-Maastricht-JSON2AVU-paper.pdf), [Metadata Templates WG 2024](https://irods.org/uploads/2024/Russell-iRODS-iRODS_Metadata_Templates_Working_Group_Building_Blocks_and_Lessons_Learned-slides.pdf)
- Lineage: a UGM 2020 talk states that when files move between storage systems "there is no lineage captured or maintained" (framed as a gap to address); no later source shows a lineage feature in core. **[search-verified]** — [UGM 2020](https://irods.org/ugm2020/)
- Discovery/search requires an external Elasticsearch/OpenSearch fed by the indexing plugin; the default web UI is Metalnx; domain portals (Yoda, ManGO, CyVerse DE) are separate site-built products. **[search-verified]** — [TRiRODS](https://irods.org/trirods/), [Yoda](https://publication.yoda.uu.nl/), [KU Leuven 2023 paper](https://irods.org/uploads/2023/Montes-KULeuven-Towards_rich_and_standardized_metadata_in_iRODS-paper.pdf)
- Event stream is the audit plugin's AMQP PEP messages (multi-endpoint and long-lived connections still under development in 2026). **[search-verified]** — [Dev update Nov 2026](https://irods.org/2026/11/irods-development-update-november-2026/)
- Cloud-native: no official production Helm chart; S3 is a storage backend behind iRODS servers; the S3-compatible front end is 0.6.0. **[search-verified]** — [irods_k8s](https://github.com/irods/irods_k8s), [UGM 2026 S3 API slides](https://slides.com/alanking/ugm2026-irods-s3-api)
- Publishing/repository functions (DOIs, landing pages) come from integrations (Dataverse, Yoda's DataCite flow), not from core. **[search-verified]** — [UGM 2026 Consortium Update](https://slides.com/irods/ugm2026-consortium-update), [Yoda publication platform](https://publication.yoda.uu.nl/)

### Inferences
Feature-gap table (my synthesis; "absent" = not found in any iRODS documentation or UGM material reviewed):

| Modern-catalog capability | iRODS status |
|---|---|
| Dataset/table as first-class entity | Absent — collections by convention |
| Schema / column-level metadata, types, stats | Absent |
| Table formats (Iceberg/Delta/Hudi), Iceberg REST catalog | Absent; files are opaque blobs |
| Immutable versions, snapshots, time travel, branches | Absent (in-place overwrite; replicas ≠ versions) |
| Lineage graph (job→dataset, column-level), OpenLineage | Absent; only audit trail of operations |
| Typed, validated metadata schemas / glossary / tags / ownership / domains | Absent in core; AVU conventions per site |
| Full-text + faceted discovery UI | External (OpenSearch + site portal) |
| Stable change-event stream (Kafka/CDC), webhooks | Partial — verbose AMQP audit of PEPs |
| REST/GraphQL/OpenAPI-first API, SDK generation | Partial — HTTP API (recent), bespoke protocol primary |
| Credential vending / direct-to-object-store access for engines (Spark, Trino, DuckDB) | Absent — all I/O proxied through iRODS servers (or S3 API gateway) |
| OIDC/OAuth-native auth, fine-grained tokens | Partial — PAM/OIDC plugins, tickets; `native` scheme being replaced in 5.1 |
| ABAC / row- & column-level policies, data masking | Absent — per-object ACLs only |
| Data quality / contracts / profiling | Absent |
| Kubernetes operator, stateless horizontal scale-out | Absent officially |
| Persistent identifiers (DOI/Handle) | Via site rules/integrations |

- Conversely, what modern DS catalogs lack and iRODS has (worth carrying into the new design): replica-aware catalog with per-replica state and checksums; placement/tiering/retention policy over heterogeneous storage incl. tape/HSM (compound resources); POSIX-like logical namespace independent of storage; per-object ACLs and bearer tickets; federation between administrative domains; registration of data in place; server-side enforcement for all access paths.

### Gaps
- I did not find a published head-to-head "iRODS vs DataHub/OpenMetadata/Unity Catalog" comparison; the table above is inference from documentation, not a cited comparison.

## Published criticisms, comparisons (Rucio, dCache, Globus, DataHub-type) and lessons learned

### Takeaway
Independent comparative literature is thin and mostly positions the tools as complementary: iRODS = site/institution-scale policy and metadata; Rucio = declarative, experiment-scale multi-site replica management; dCache = storage system; Globus = transfer/sharing SaaS. The most useful criticism comes from adopters' own UGM experience reports.

### Cited Findings
- JENA (European nuclear/particle/astro computing) WP2 report describes iRODS as operating at smaller scale than Rucio and typically targeted to single data centres; also notes Rucio's complexity "still represents a significant obstacle to smaller teams of scientists". **[search-verified]** — [arXiv:2503.09213](https://arxiv.org/pdf/2503.09213)
- Rucio: manages all ATLAS data across the WLCG, >130 data centres; declarative replication rules over Rucio Storage Elements (POSIX/S3/grid); Apache-2.0. **[search-verified]** — [Rucio – Scientific data management, arXiv:1902.09857](https://arxiv.org/pdf/1902.09857)
- A June 2026 blog guide (iRODS vs Rucio vs DataLad) concludes each "takes a fundamentally different approach" and the choice depends on domain, infrastructure and collaboration model. **[search-verified; low-authority blog]** — [pistack.xyz guide](https://www.pistack.xyz/posts/2026-06-09-self-hosted-scientific-data-management-irods-rucio-datalad-guide/)
- iRODS itself now positions Globus as a partner/integration (alongside OMERO, RSpace, Dataverse) rather than a competitor. **[search-verified]** — [UGM 2026 Consortium Update](https://slides.com/irods/ugm2026-consortium-update)
- KU Leuven lessons (1 year in production): easy demo → hard production; steep learning curve; documentation gaps; rules for domain use cases are hard; replica management not robust enough; issues with atomic metadata operations and a MySQL-related conflict; no big performance differences observed; overall verdict positive. **[search-verified]** — [Barcena, UGM 2021](https://irods.org/uploads/2021/Barcena-KULeuven-A_Year_of_iRODS_Lessons_Learned-slides.pdf)
- Sanger (2024): operational burden of detecting/repairing inconsistent replica states at 20 PB while pinned to an old release (4.2.7). **[search-verified]** — [UGM 2024](https://irods.org/ugm2024/)
- Sanger's original 2011 paper remains the canonical positive case study (iRODS chosen for metadata + replication + rules for sequencing data). **[search-verified existence]** — [Chiang et al., BMC Bioinformatics 2011](https://pmc.ncbi.nlm.nih.gov/articles/PMC3228552)
- No documented case of a major site publicly migrating *away* from iRODS was found. **[search-verified absence in one extended search]** — (search performed 2026-11-23; see e.g. results around [DZone overview](https://dzone.com/articles/irods-an-open-source-approach-to-data-management-i))

### Inferences
**Strengths (honest list)**
1. Logical/physical separation with replica tracking across POSIX, S3, tape/HSM — still rare in catalogs.
2. Server-side policy enforcement at every operation; capable of real preservation/compliance workflows.
3. Per-object ACLs, tickets, federation between autonomous zones.
4. Open source (BSD-3), 20+ year lineage (SRB→iRODS), stable funding consortium, on-prem friendly, vendor-neutral.
5. Mature for large-file genomic/imaging archives at 10s of PB.
6. Pluggable everything; multiple access protocols now (native, HTTP, S3, NFS, WebDAV via Davrods, CSI).

**Weaknesses**
1. Catalog = single RDBMS with a generic schema; write-heavy/small-file workloads and permission-joined queries hit the wall around 10⁸–10⁹ rows.
2. AVU metadata: untyped, flat, schema-less, EAV-slow; templates never landed in core.
3. Rule engine: powerful but bespoke, hard to test, in the request path; declarative policy arrived as 0.1.0 only in 2026.
4. Operational complexity and costly upgrades (flagship sites lag by years); replica-state inconsistencies require site tooling.
5. Bespoke protocol and client stack; modern APIs are young; weak UI — every serious site builds its own portal.
6. Not cloud-native; I/O proxied through servers; no direct engine integration.
7. No dataset/table/version/lineage/schema concepts → poor fit for data-science catalogs.
8. Small core team; slow cadence; ecosystem components long-term pre-1.0.

**Design lessons for a new catalog serving both science and data science**
- Keep: logical entity ↔ N physical replicas with state, checksum, storage class; declarative replication/retention/tiering rules (Rucio-style) evaluated by async reconcilers; in-place registration; federation via stable global identifiers.
- Change: make *dataset* (file collection, table, model) the first-class versioned entity, files/replicas beneath it; typed schema-validated metadata (JSON Schema/JSONB or columnar), not EAV; lineage (OpenLineage-compatible) and durable ordered event log as core primitives; policy out of the synchronous path; HTTP/OpenAPI-first with credential vending for direct storage access; search index as a managed, ACL-aware projection; horizontal scalability of the catalog (partition by namespace/dataset) and avoid one-row-per-file-per-ACL designs; upgrades must be online and schema migrations incremental.

### Gaps
- No iRODS-vs-dCache or iRODS-vs-Globus peer-reviewed comparison found in this session; positioning above is inference plus Consortium's own partnership framing.
- No iRODS-vs-DataHub/OpenMetadata/Amundsen comparison found; appears not to exist in the literature (the communities barely overlap).
- HN / blog criticism threads were not surfaced by search; anecdotal operator complaints (beyond UGM talks and GitHub issues) are under-sampled here.
- Because `irods.org` fetches were blocked, UGM 2024–2026 proceedings were read only via search extracts; a follow-up pass with `irods.org`, `docs.irods.org`, `slides.com` and `ncbi.nlm.nih.gov` allow-listed would firm up numbers (object counts, member list, 6.0 roadmap).
