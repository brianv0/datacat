# Rucio (scientific data management system, CERN/ATLAS origin) — technical evaluation, state as of 2026-11-23

Research-method caveat (applies to all sections): in this session direct fetches of rucio.cern.ch, indico.cern.ch, arxiv.org, epj-conferences.org and ep-news-wp.web.cern.ch were blocked by the network allowlist; only github.com pages could be fetched in full. Everything else rests on search-result extracts of those pages (primary sources, but seen as snippets, not full text). Items labelled "(snippet)" were not read in full context. Items labelled "(paper, not re-read)" are well-known content of the 2019 Rucio paper that I cite from the paper's URL without having re-fetched it in this session; treat exact wording/numbers there as needing a spot-check.

## 1. Data model: DIDs, hierarchy, RSEs, replicas, accounts, path mapping

### Takeaway
Rucio's model is a global, experiment-wide namespace of immutable identifiers (`scope:name`) arranged as files -> datasets -> containers, mapped onto abstract storage endpoints (RSEs) by a replica table. It is a *file/replica* model: there is no notion of table, column, schema, version-of-a-dataset, or partition; the smallest unit is an opaque file with size + checksum.

### Cited Findings
- Rucio describes itself as "a software framework that provides functionality to organize, manage, and access large volumes of scientific data using customisable policies", originally developed for ATLAS. — [rucio/rucio README](https://github.com/rucio/rucio)
- Data Identifiers (DIDs) are `scope:name` tuples; three DID types: files, datasets (collections of files), containers (collections of datasets/containers). A DID name is unique within a scope and is never reused, even after deletion. Files can belong to several datasets; hierarchy is a DAG, not a tree. (paper, not re-read) — [Barisits et al. 2019, arXiv:1902.09857](https://arxiv.org/pdf/1902.09857)
- Collections carry status flags: open/closed (closed = content frozen), monotonic (content can only be added), complete (all files have replicas); DIDs have availability states (available/lost/deleted) and may be suppressed/obsolete. (paper, not re-read) — [Barisits et al. 2019](https://arxiv.org/pdf/1902.09857)
- RSEs (Rucio Storage Elements) are logical abstractions of storage with protocols, attributes (key/value tags such as tier, country, type=tape) and distances; RSE expressions are a small set-algebra language over those attributes (union `|`, intersection `&`, complement `\`) used to target rules. (paper, not re-read) — [Barisits et al. 2019](https://arxiv.org/pdf/1902.09857)
- Path mapping: on a *deterministic* RSE the physical path is computed from the DID alone; on a *non-deterministic* RSE extra metadata or an explicit path must be supplied. `lfn2pfn` algorithms apply only to deterministic RSEs; separate "non-deterministic PFN" algorithms exist; the algorithm is chosen by the RSE attribute `lfn2pfn_algorithm`, falling back to `lfn2pfn_algorithm_default` in the `[policy]` config. (snippet) — [Policy package algorithms](https://rucio.cern.ch/documentation/operator/policy_packages/policy-package-algorithms), [RSE configuration](https://rucio.cern.ch/documentation/operator/rse_configuration)
- The `deterministic` flag of an RSE is effectively immutable; Rucio refuses to change it on an RSE that already holds replicas. (snippet, third-party admin guide) — [ORNL NDIP Rucio admin guide](https://ndip.ornl.gov/docs/admin_guide/services/rucio/)
- Scope naming is used to carry semantics by convention: in ATLAS the "project" field of the dataset nomenclature doubles as the Rucio scope (e.g. `mc20_13TeV`, `data18_13TeV`). (snippet) — [ATLAS nomenclature (rucio-mcp docs)](https://rucio-mcp.readthedocs.io/en/latest/atlas-nomenclature/)
- Hierarchical (directory-like) data does not fit the flat scope:name + dataset/container model well: SKAO's 2024 assessment lists "hierarchical data remains a challenge", with several workarounds discussed, each with disadvantages. (snippet) — [SKAO DDM assessment, Rucio Workshop 2024-09-30](https://indico.cern.ch/event/1343110/contributions/6089547/attachments/2937876/5160662/JSC%20-%20DDM%20Assessment%20-%20Rucio%20Workshop%2030-09-24%20(1).pdf)
- DUNE needed physical paths derived from metadata: its policy package generates physical file names by querying MetaCat, and files must exist in MetaCat before being declared to Rucio. (snippet) — [DUNE CHEP 2024 slides](https://indico.cern.ch/event/1338689/contributions/6010878/attachments/2952884/5191237/CHEP_2024_DUNE_Rucio.pdf), [Fermilab slides 2021 (Timm)](https://lss.fnal.gov/archive/2021/slides/fermilab-slides-21-111-scd.pdf)
- A new `is_opendata` column for DIDs appears in 42.0.0rc1 (2026-11-20), and an "Open Data EOS integration" feature landed in 41.0.0; i.e. open-data publication state is being modelled as yet another fixed DID column. — [rucio/rucio releases](https://github.com/rucio/rucio/releases), [41.0.0 release](https://github.com/rucio/rucio/releases/tag/41.0.0)

### Inferences
- Design ideas worth copying: (a) permanent, never-reused identifiers; (b) the separation of logical name from physical location, with a pure function for path derivation where possible (deterministic RSEs remove a whole class of catalog/storage drift and make the replica table small); (c) storage described by attributes and addressed by expressions rather than by name.
- Design weaknesses for a general catalog: identity is name-based with semantics packed into naming conventions; no first-class versioning (immutability + never-reuse pushes versioning into names); no directory/hierarchical namespace; no sub-file or structured (table/column) entity; files in a dataset are a flat set.
- Deterministic-vs-non-deterministic being frozen per RSE, and physical-name generation living in per-community Python "policy packages", means site-layout decisions are hard to revisit later.

### Gaps
- I could not re-read the 2019 paper or the current "concepts" docs pages this session, so details such as archive (zip constituent) DIDs, exact DID state names and the `lifetime` semantics on DIDs are stated from the paper/docs as I know them, not from a verified 2026 page. Archive support (files registered as constituents of a ZIP archive DID, resolved at download time) exists per the docs but I have no fetched source for its current state.
- No verified 2026 statement on maximum practical collection sizes (files per dataset) or name-length limits.

## 2. Declarative replication: rules, locks, quotas, subscriptions, deletion, consistency, recovery, transfer tools, tape

### Takeaway
The replication-rule engine is Rucio's real core and its strongest idea: users state the desired end state ("N copies of this DID on RSEs matching this expression until this lifetime"), and a set of asynchronous daemons converge storage toward it, with locks protecting replicas, tombstones enabling lazy cache-like deletion, and a recovery chain for bad/lost replicas. The cost is an eventually-consistent, many-daemon system whose behaviour is hard to reason about and whose failure modes (stuck rules, stuck locks, dark data) need operator attention.

### Cited Findings
- A replication rule = (account, DID, RSE expression, copies, optional lifetime, grouping, weight). Rules on collections apply to current and *future* content. Each satisfied (file, RSE) pair gets a replica lock; replicas with no locks become eligible for deletion. Account quotas are charged per rule per RSE. (paper, not re-read) — [Barisits et al. 2019](https://arxiv.org/pdf/1902.09857)
- Judge daemons manage the rule lifecycle: injector, evaluator, repairer, cleaner. The evaluator "will keep track of new data added to datasets that are already replicated to trigger the necessary transfer requests"; the cleaner removes expired rules; the repairer handles stuck rules. (snippet) — [Rucio daemons docs](https://rucio.cern.ch/documentation/started/main_components/daemons)
- Subscriptions (transmogrifier daemon) are standing data-placement policies: they match metadata of newly created DIDs and create rules automatically. (snippet) — [Rucio daemons docs](https://rucio.cern.ch/documentation/started/main_components/daemons)
- Transfers: conveyor daemons — preparer, throttler (queues transfers under limits; requires the preparer), submitter, poller (polls the transfer tool), receiver, stager (tape recalls), finisher. Transfer tools are pluggable (FTS3 is the main one; Globus is supported). (snippet) — [Rucio daemons docs](https://rucio.cern.ch/documentation/started/main_components/daemons), [Transfers overview](https://rucio.cern.ch/documentation/operator_transfers/transfers-overview)
- Globus support is maintained: "Globus SDK 3+ support" is listed in 42.0.0rc1 (2026-11-20). — [rucio/rucio releases](https://github.com/rucio/rucio/releases)
- Deletion: when a rule's lifetime expires the judge-cleaner sets a tombstone on the now-unlocked replicas; the reaper deletes replicas that are unlocked and tombstoned. Greedy mode deletes immediately; non-greedy mode deletes only when space is needed (so unlocked replicas act as a cache). The undertaker removes expired DIDs (not replicas). (snippet) — [Deletion overview](https://rucio.cern.ch/documentation/started/concepts/deletion_overview), [Rucio daemons docs](https://rucio.cern.ch/documentation/started/main_components/daemons)
- Bad/lost replica chain: replicas can be declared bad (`rucio-admin replicas declare-bad`); the necromancer re-transfers from another copy if one exists, otherwise marks the file lost; minos sorts bad PFNs into temporarily vs permanently unavailable; the suspicious-replica-recoverer promotes repeatedly failing ("suspicious") replicas that exist elsewhere to bad; the dark reaper removes quarantined dark replicas. (snippet) — [rucio-necromancer](https://rucio.cern.ch/documentation/bin/rucio-necromancer), [Suspicious replica recoverer](https://rucio.cern.ch/documentation/operator/suspicious-replica-recoverer), [Replica workflow](https://rucio.cern.ch/documentation/started/concepts/replica_workflow)
- Consistency checking relies on comparing storage-side dumps against catalog dumps to find dark data (on storage, not in catalog) and lost files (in catalog, not on storage); this is an offline, dump-based auditor process, not transactional. (paper, not re-read) — [Barisits et al. 2019](https://arxiv.org/pdf/1902.09857)
- Even at ATLAS the number of "unavailable" files "hovers at ~2 M" (includes scheduled-but-not-yet-transferred files), and ATLAS reported ~200k failed transfers/day against 1M+ successful in 2023 — i.e. the reconciliation loop is permanently busy. (snippet) — [ATLAS DDM automation, TIM 2025](https://indico.cern.ch/event/1472836/contributions/6325665/attachments/3000923/5288207/DDM%20Automation%20TIM%202025.pdf), [ATLAS Data Flows & Rucio, Oct 2023](https://indico.cern.ch/event/1252369/contributions/5515384/attachments/2734461/4754446/ATLAS%20Data%20Flows%20&%20Rucio.pdf)
- "Tests for stuck locks" were still being added in 41.3.0 (Sept 2026), and the ability to declare dark replicas from the CLI was removed in 41.0.0. — [rucio/rucio releases](https://github.com/rucio/rucio/releases), [41.0.0 release](https://github.com/rucio/rucio/releases/tag/41.0.0)
- QoS: the client "QoS Policy functionality" is being removed in 42.0.0rc1 — the dedicated QoS-policy feature did not survive; QoS/tape is expressed through RSE attributes and expressions instead. — [rucio/rucio releases](https://github.com/rucio/rucio/releases)
- Legacy protocols were removed in 41.0.0 (2026-07): RFIO, SRM and GridFTP RSE protocol implementations. — [41.0.0 release](https://github.com/rucio/rucio/releases/tag/41.0.0)
- Rucio "is not designed to act as a backup service or a software management system" (2020 UK evaluation). (snippet) — [Rucio/DIRAC evaluation, Feb 2020](https://indico.cern.ch/event/876274/contributions/3692660/attachments/1986191/3309617/RucioDiRAC3Evaluation20200212.pdf)
- CMS finds rule/subscription targeting limited by where metadata lives: "our metadata does not live in rucio, which makes filtering rules hard to create". (snippet) — [CMS Rucio community report 2024](https://indico.cern.ch/event/1343110/contributions/6093893/attachments/2937877/5160663/CMSRucio%20Community%20Report.pdf)

### Inferences
- The rule/lock/tombstone triple is a clean, reusable abstraction: ownership of a replica is the set of rules that lock it; "delete" is "remove my rule"; storage GC is separated from user intent. A new catalog should adopt the declarative-placement pattern but can make it optional (most data-science users have one bucket).
- Policy targeting is only as good as the metadata the rule engine can see. Because the rich metadata lives in external catalogs (DBS, AMI, MetaCat, Butler), subscriptions can only key on names/scopes and the few built-in columns — an argument for keeping placement policy and metadata in one queryable system.
- Consistency is achieved by periodic reconciliation rather than by construction; with object stores offering inventories/events, a new design could do this incrementally.
- Rebalancing (BB8) and dynamic placement (C3PO) exist as daemons per the 2019 paper, but I found no 2025-26 evidence about how widely they are used outside ATLAS.

### Gaps
- No fetched 2026 documentation on tape specifics (staging throttles, tape-colocation hints / archive metadata passed to FTS, multi-hop transfers). An Indico entry "Archive metadata discussions - ATLAS/RUCIO" exists ([link](https://indico.cern.ch/event/1265912/)) but content not read.
- Rubin reports an "order of magnitude more files going to tape than HEP" as a scalability concern (abstract only; outcomes not seen) — [Rucio workshop 2025 contribution](https://indico.cern.ch/event/1545309/contributions/6717962).
- No verified numbers for rule-evaluation latency or reaper deletion rates.

## 3. Architecture: REST server, daemons, database, messaging, auth, multi-VO, WebUI, clients, deployment

### Takeaway
Classic three-tier design: stateless Python REST servers and ~20-30 Python daemons, all coordinating exclusively through one central relational database (Oracle at ATLAS/CMS, PostgreSQL at most others), emitting events to a message broker. Horizontally scalable at the service tier, vertically bound at the database tier; operationally a substantial distributed system with many moving parts and, in 2026, a notable run of security advisories.

### Cited Findings
- Daemons are stateless workers that shard work among themselves (heartbeats + hash partitioning) and coordinate via the DB; this design dates from the original ATLAS implementation ("thread-safe and use sharding techniques to reduce concurrency and distribute the work"). (snippet) — [ATLAS DDM Rucio CHEP13](https://indico.cern.ch/event/214784/contributions/1512770/attachments/341109/475997/ATLAS_DDM_RUCIO_CHEP13.pdf)
- Hermes delivers event messages via STOMP to a message broker (ActiveMQ) and via SMTP as e-mail. (snippet) — [Rucio daemons docs](https://rucio.cern.ch/documentation/started/main_components/daemons)
- Kafka is not the native path: Rubin *modified* Hermes to publish Butler information to Kafka, choosing "Kafka instead of ActiveMQ to replicate messages between sites, even though this would mean modifying the standard Hermes client". (snippet) — [Rubin USDF Rucio-Butler integration docs](https://df-ops.lsst.io/usdf-applications/drp/rucio-butler-integration/info.html), [DMTN-198](https://dmtn-198.lsst.io/v/DM-35905)
- Databases: Rucio is tested against SQLite, MySQL/MariaDB, PostgreSQL and Oracle (via SQLAlchemy). 42.0.0rc1 adds MySQL 8.4 and PostgreSQL 16 test images; 41.4.0 (Oct 2026) upstreamed "ATLAS query hints for Oracle" — production performance at scale depends on hand-tuned, vendor-specific hints. (snippet + fetched) — [Database operations docs](https://rucio.cern.ch/documentation/operator/database), [rucio/rucio releases](https://github.com/rucio/rucio/releases)
- PostgreSQL operation is less well documented than Oracle: a BNL (Belle II) operator talk says the two "differ quite a bit in their operations", that configuration/operational knowledge for high-performance Rucio on PostgreSQL "is not as well documented", and reports "quite busy DB disks, ~10s KIOPS, ~100s MB/s" on PostgreSQL 12. (snippet) — [BNL Rucio DB operation, Rucio Workshop 2024](https://indico.cern.ch/event/1343110/contributions/6065565/attachments/2939550/5163940/BNL%20Rucio%20DB%20operation.pdf)
- Analytics are not served from the catalog DB: ATLAS "shovels the majority of its Oracle database contents to HDFS daily via cron-triggered scripts". (snippet) — [ATLAS DDM automation, TIM 2025](https://indico.cern.ch/event/1472836/contributions/6325665/attachments/3000923/5288207/DDM%20Automation%20TIM%202025.pdf)
- Authentication: userpass, X.509, SSH, OIDC tokens. 41.0.0 removed SAML authentication and the OIDC "auto" login flow, and now requires `rsa-sha2-256` SSH signatures; 40.0.0 removed the OAuth manager daemon from the repository. — [41.0.0 release](https://github.com/rucio/rucio/releases/tag/41.0.0), [40.0.0 release](https://github.com/rucio/rucio/releases/tag/40.0.0)
- Token transition is slow and still in progress: the May 2025 WLCG overview planned "first production use of Rucio tokens in ATLAS in 2025-Q4"; the Rucio SIG Tokens page gives an end date in line with the WLCG token transition (snippet shows both March 2026 and 2028 — ambiguous). A Nov 2024 CILogon test with Rucio+FTS3 needed three OIDC clients and had open issues on token audience. (snippet) — [WLCG Token Transition Overview 2025-05-06](https://indico.cern.ch/event/1484669/contributions/6500523/attachments/3061295/5414301/Token_Transition_Overview-250506-v11.pdf), [Rucio SIG Tokens](https://rucio.cern.ch/documentation/sig_tokens), [Rucio auth token test, 2024](https://indico.cern.ch/event/1482711/contributions/6247618/attachments/2981331/5251299/rucio_auth_token.pdf)
- Multi-VO: a multi-VO mode exists (used by STFC's multi-tenant service), but token flows and multi-VO FTS interplay are awkward — WLCG DOMA notes say some token-delegation approaches "may not be always acceptable for multi-VO FTS instances". (snippet) — [STFC Rucio Data Management Service](https://www.scd.stfc.ac.uk/Pages/SCD-STFC-Rucio-Data-Management-Service.aspx), [WLCG DOMA BDT meeting notes](https://indico.cern.ch/event/1289800/contributions/5422343/note/)
- Community-specific behaviour (permissions, schema/naming validation, lfn2pfn, surl construction) lives in Python "policy packages"; the long-term goal is to remove all community policies from upstream Rucio (issue #4798). 41.0.0 fixed policy-package schema overrides and replaced the Belle II policy package. (snippet + fetched) — [Rucio 36 roadmap notes](https://indico.cern.ch/event/1431526/contributions/6022681/note/), [41.0.0 release](https://github.com/rucio/rucio/releases/tag/41.0.0)
- New WebUI (rucio/webui, "v2.0"): Next.js 16 App Router, React 19, TypeScript 5.7, TailwindCSS/Radix, NextAuth v5, custom NDJSON streaming SDK with web workers, hexagonal architecture, per-deployment `FEATURE_*` toggles; auth via userpass, OIDC, X.509. The repo has 5 stars / 37 forks; README roadmap lists streaming optimisation, design-system maturity, "UI expansion", "new server endpoints" as still in progress. — [rucio/webui](https://github.com/rucio/webui)
- Clients: Python client API and CLI; a new CLI (`rucio did ...`, `rucio rse ...`, `rucio rule ...`) has been progressively replacing the old one through 40-41, and 42.0.0rc1 deprecates the old CLI. A JupyterLab extension and Helm charts are versioned with the server's major. — [40.0.0 release](https://github.com/rucio/rucio/releases/tag/40.0.0), [rucio/rucio releases](https://github.com/rucio/rucio/releases), [Release policy](https://github.com/rucio/documentation/blob/main/docs/started/releasepolicy.md)
- Deployment: Kubernetes + Helm charts is the reference deployment (CMS pioneered a Kubernetes/Helm deployment). (snippet) — [Rucio @ JLab slides 2020](https://indico.jlab.org/event/356/contributions/5690/attachments/4824/5987/Rucio__JLab.pdf)
- Security posture 2026: 41.5.0 / 38.6.3 (2026-10-16) fixed four advisories — RCE in the reaper daemon via replica PFN, command injection in ROOT GUID extraction during upload, command injection via file names during archive extraction in the download client, and missing authorization on Open Data catalog endpoints. 41.2.2 (2026-09-07) fixed a Painless script injection in the Elasticsearch metadata plugin (GHSA-89hf-hh2w-258f). — [rucio/rucio releases](https://github.com/rucio/rucio/releases)

### Inferences
- The "DB is the bus" architecture is simple and robust for workers, but makes the relational DB the single scaling and availability bottleneck, pushes analytics out to a separate Hadoop copy, and ties peak performance to Oracle-specific tuning that smaller PostgreSQL sites do not inherit.
- The 2026 advisories (shell/command injection from file names and PFNs, unauthenticated endpoints, script injection in a metadata plugin) indicate a codebase grown in a trusted-collaboration setting; a multi-tenant catalog exposed to untrusted users needs a stronger security model from day one.
- Authorization is coarse: account-based, with permission logic implemented as Python functions in policy packages rather than as declarative, per-object ACLs/RBAC.
- Event stream via STOMP/ActiveMQ is a good hook for integration (Rubin's Butler ingest depends on it), but modern consumers want Kafka/CloudEvents; Rubin had to patch Hermes.

### Gaps
- Whether upstream Hermes gained native Kafka support by 2026 — not verified.
- Whether ATLAS actually reached production token use on the planned 2025-Q4 date — not confirmed.
- No verified count of daemons or of DB tables in release 41.
- Exact multi-VO maturity in 2026 (an older meeting note said only single-VO instances were officially supported) — not resolved.

## 4. Metadata: built-in columns vs JSON plugins vs external backends; Rucio as a metadata catalog

### Takeaway
Rucio is a strong replica/placement manager and a weak metadata catalog, by design and by the testimony of its own users. Every large experiment pairs it with a separate, purpose-built metadata/provenance system; Rucio's own generic-metadata features (JSON column, external Mongo/Postgres/Elasticsearch plugins, filter engine) exist but are schemaless, lightly used in production, limited in query power, and still being redesigned in 2025-26.

### Cited Findings
- Rucio "was initially designed to support a fixed set of metadata for all the DIDs"; this column metadata is "very suitable to perform data accounting", and because of its limited use-case, generic JSON key-value storage and external backends (such as MetaCat) were added later. (snippet, Belle II paper) — [OSTI 2448340](https://www.osti.gov/pages/servlets/purl/2448340)
- Plugin architecture: `did_column_meta` (fixed columns of the DIDs table), `json_meta` (JSON blob per DID in the main DB), plus external-backend plugins for MongoDB, PostgreSQL (JSONB) and Elasticsearch. (snippet) — [DID metadata operator docs](https://rucio.cern.ch/documentation/operator/did_meta), [Rucio SIG Metadata report 2024-03-25](https://rucio.cern.ch/documentation/files/Rucio_SIG_Metadata_report_2024-03-25.pdf)
- Maturity (ePIC/EIC 2025 assessment): column and Postgres JSON/JSONB plugins rated production-ready; the Elasticsearch backend is "experimental in Rucio, with no production deployments yet". (snippet) — [ePIC Rucio metadata slides](https://indico.bnl.gov/event/29623/contributions/112971/attachments/65477/112429/rucio_eic_metadata.pdf)
- Query capability: up to Rucio 1.27, `list-dids` filtering supported only equality and AND. From 1.28 (2022) a "filter engine" parses filter strings and translates them to SQLAlchemy, PostgreSQL and MongoDB queries; documented operators are `=, >, >=, <, <=` (plus wildcards on names and AND/OR groups). Filter input is a nested dict of key/value pairs. (snippet) — [DID metadata operator docs](https://rucio.cern.ch/documentation/operator/did_meta), [Rucio SIG Metadata report 2024](https://rucio.cern.ch/documentation/files/Rucio_SIG_Metadata_report_2024-03-25.pdf)
- A filter-engine bug was still being fixed in 41.0.0 (July 2026): recursive `list_dids` dropped FILE DIDs and overwrote `did_type` filters (#8402). — [41.0.0 release](https://github.com/rucio/rucio/releases/tag/41.0.0)
- Metadata search "does not scale indefinitely" (2020 UK evaluation finding). (snippet) — [Rucio/DIRAC evaluation, Feb 2020](https://indico.cern.ch/event/876274/contributions/3692660/attachments/1986191/3309617/RucioDiRAC3Evaluation20200212.pdf)
- Direction of travel: the 2024 roadmap planned to "move all metadata to a JSON approach and deprecate the did_columns plugin"; the Nov 2025 community workshop had a session on metadata evolution "from fixed metadata columns and key:value attributes, to a schema-governed catalog". I.e. as of late 2025 schema governance was a proposal, not a shipped feature. (snippet) — [Development roadmap discussion 2024](https://indico.cern.ch/event/1343110/sessions/566779/attachments/2940421/5165695/Development%20roadmap%20discussion.pdf), [Rucio Workshop 2025 contributions](https://indico.cern.ch/event/1545309/contributions/)
- What experiments pair it with:
  - ATLAS: AMI is "the dataset metadata catalogue"; AMI and Rucio "are complementary tools with different purposes"; AMI holds metadata for O(10^6) datasets and O(10^9) files. (snippet) — [AMI overview](https://indico.ijclab.in2p3.fr/event/9726/attachments/21684/30700/ami-overview.pdf), [ATLAS software docs: AMI](https://atlassoftwaredocs.web.cern.ch/analysis-software/AnalysisSWTutorial/find_data_ami)
  - CMS: "will continue to use DBS, a custom physics metadata database, in conjunction with Rucio". (snippet) — [Transitioning CMS to Rucio (CHEP 2019)](https://www.epj-conferences.org/articles/epjconf/pdf/2020/21/epjconf_chep2020_04033.pdf)
  - DUNE: Rucio is only the replica manager; file metadata and provenance (parentage) live in MetaCat, with a separate Data Dispatcher for delivery. MetaCat's data model was described as "very similar to the Rucio model", yet was built as a separate service. (snippet) — [DUNE Rucio GDB Oct 2020](https://indico.cern.ch/event/813752/contributions/4051564/attachments/2122693/3573056/DUNE_Rucio_GDB_2020_Oct_Wenlong.pdf)
  - Rubin/LSST: the Data Butler is the science-facing registry (data IDs, dataset types, collections, provenance); Rucio moves files between facilities; `rucio-register` registers Butler datasets in Rucio, and `ctrl_ingestd` at each data facility listens to Kafka messages from (modified) Hermes and ingests arrived files into the local Butler; MirrorMaker2 replicates topics to UK/FR sites. (snippet) — [Rubin USDF Rucio-Butler integration](https://df-ops.lsst.io/usdf-applications/drp/rucio-butler-integration/info.html), [Rubin CHEP 2024 multi-site processing](https://indico.cern.ch/event/1338689/contributions/6011017/attachments/2953535/5192481/CHEP2024_Preparation%20of%20Multi-Site%20Data%20Processing%20at%20the%20Vera%20C.%20Rubin%20Observatory.pdf)
- Side effect of the split: DUNE's setup "is harder to test because it needs a local MetaCat instance and multiple Docker containers"; CMS cannot easily write rules/subscriptions against metadata that lives in DBS. (snippet) — [DUNE CHEP 2024 slides](https://indico.cern.ch/event/1338689/contributions/6010878/attachments/2952884/5191237/CHEP_2024_DUNE_Rucio.pdf), [CMS Rucio community report 2024](https://indico.cern.ch/event/1343110/contributions/6093893/attachments/2937877/5160663/CMSRucio%20Community%20Report.pdf)

### Inferences
- The universal "Rucio + separate metadata catalog" pattern (AMI, DBS, MetaCat, Butler; SAM before MetaCat at Fermilab) is the single most important finding for a new design: the community has repeatedly had to build the second system, then build glue (naming conventions, registration daemons, Kafka bridges, policy packages that call the metadata service) and then live with two sources of truth and cross-system consistency problems.
- What Rucio's metadata lacks relative to those companions: typed/validated schemas, provenance/parentage, dataset-level semantic queries (runs, conditions, processing versions), a real query language (MetaCat has MQL; Butler has data-ID expressions), and user-facing discovery.
- JSON-blob metadata in the transactional DB competes with the replica workload; that is why external backends exist — but then consistency between catalog and metadata store becomes the plugin's problem.
- A new catalog that unifies replica management and rich, schema-governed metadata + lineage in one logical service addresses a demonstrated gap rather than a hypothetical one.

### Gaps
- I could not open the Rucio SIG Metadata report (2024) or the 2025 "schema-governed catalog" talk in full; the exact proposed design and whether any of it shipped in 39-42 is unverified. The 40.0.0 and 41.0.0 release notes list no metadata features and no schema changes, which suggests it had not shipped by July 2026.
- Current operator set for JSON-key filters (vs column filters) not confirmed.
- Which communities run json_meta or external plugins in production at scale — no reliable list found.

## 5. Scale and performance

### Takeaway
Rucio is proven at a scale almost nothing else in open-source data management reaches: two independent >1 EB instances, ~1 billion files, 120+ sites, >1M transfers/day. That proof is specifically for file-replica bookkeeping on Oracle with a dedicated expert team; evidence for metadata-query performance and for many-small-files workloads is much weaker.

### Cited Findings
- ATLAS instance: "more than 1 exabyte of data, stored in a billion files, distributed over 120 data centres globally"; "more than 4 exabytes of data access and transfer per year". (snippet) — [rucio.cern.ch](https://rucio.cern.ch)
- Both ATLAS and CMS each manage more than one exabyte with Rucio (CERN EP news, ~2025). (snippet) — [Rucio crosses the exabyte horizon](https://ep-news-wp.web.cern.ch/rucio-crosses-exabyte-horizon/)
- ATLAS 2023: 1M+ successful transfers/day, ~200k failed/day, 2+ PB/day between storages, ~1.5 EB/year transferred. (snippet; most recent per-day figures found are from 2023) — [ATLAS Data Flows & Rucio, Oct 2023](https://indico.cern.ch/event/1252369/contributions/5515384/attachments/2734461/4754446/ATLAS%20Data%20Flows%20&%20Rucio.pdf), [Rucio @ CERN Storage Day 2024](https://indico.cern.ch/event/1353101/contributions/5805588/attachments/2819681/4923541/Rucio%20@%20CERN%20Storage%20day.pdf)
- CMS: ~200 PB and ~1 PB/day at migration time (2020); by CHEP 2024 "over 200 PB of disk and 500 PB of tape", >1 PB/day, average ~19 GB/s. (snippet) — [Transitioning CMS to Rucio](https://www.epj-conferences.org/articles/epjconf/pdf/2020/21/epjconf_chep2020_04033.pdf), [CMS CHEP 2024 contribution](https://indico.cern.ch/event/1338689/contributions/6010859), [CMS Rucio community report 2024](https://indico.cern.ch/event/1343110/contributions/6093893/attachments/2937877/5160663/CMSRucio%20Community%20Report.pdf)
- Conflict to note: the CMS "500 PB tape + 200 PB disk" (2024) vs ">1 EB" (2025 news item) are different dates/definitions; I could not reconcile them from full text.
- Belle II's previous bespoke DDM supported up to ~150k transfers/day before migrating to Rucio. (snippet) — [Rucio @ JLab 2020](https://indico.jlab.org/event/356/contributions/5690/attachments/4824/5987/Rucio__JLab.pdf)
- Rubin is actively evaluating scalability "especially with many small files" and an order of magnitude more files to tape than HEP (2025 workshop abstract). (snippet) — [Rucio Workshop 2025, Rubin](https://indico.cern.ch/event/1545309/contributions/6717962)
- DB-tier symptoms: Oracle query hints upstreamed from ATLAS (41.4.0); heavy IOPS on Belle II's PostgreSQL; analytics offloaded to HDFS. — [rucio/rucio releases](https://github.com/rucio/rucio/releases), [BNL Rucio DB operation](https://indico.cern.ch/event/1343110/contributions/6065565/attachments/2939550/5163940/BNL%20Rucio%20DB%20operation.pdf), [ATLAS DDM automation 2025](https://indico.cern.ch/event/1472836/contributions/6325665/attachments/3000923/5288207/DDM%20Automation%20TIM%202025.pdf)

### Inferences
- The cost model is per file (replica rows, locks, requests), so HEP-style large files (GB) scale well and astronomy/ML-style billions of small files stress it; Rubin's concern is consistent with this. A new catalog should treat "collection of many small objects" (manifest/archive/table) as a first-class entity rather than one row per object per replica.
- Row-per-lock-per-replica bookkeeping multiplies with number of rules; this is a structural bottleneck independent of DB vendor.

### Gaps
- No 2026 figures for file counts, rules, locks, DB size, or API request rates; no published upper-bound benchmarks on PostgreSQL. HL-LHC (2030+) scaling projections not found in this session.
- The CHEP 2024 Rucio paper (epjconf_chep2025_01068) could not be opened.

## 6. Adoption beyond LHC, adaptations, complaints, operational burden

### Takeaway
Adoption is broad across HEP, astroparticle and astronomy (CMS, Belle II, DUNE, IGWN/LIGO-Virgo, XENON, SKA, CTAO, Rubin, ESCAPE, ePIC/EIC, plus multi-tenant services such as STFC's), but nearly always as the replica/transfer layer under an experiment-specific catalog, and nearly always with a dedicated ops team. Recurring complaints: complexity for small teams, painful major upgrades, WLCG-centric assumptions (X.509, FTS, grid storage), poor fit for hierarchical data and small files.

### Cited Findings
- Two CHEP papers document cross-experiment experience: "Rucio beyond ATLAS: experiences from Belle II, CMS, DUNE, EISCAT3D, LIGO/VIRGO, SKA, XENON" (CHEP 2019), which compares "common successes, pitfalls, and oddities", and "Experience with Rucio in the wider HEP community" (CHEP 2021: ATLAS, Belle II, CMS, ESCAPE, IGWN, LDMX, Folding@Home, STFC). (abstracts only) — [OSTI 1764079](https://www.osti.gov/biblio/1764079), [OSTI 1839557](https://www.osti.gov/biblio/1839557)
- Motivation for adoption: growing data rates make experiment-specific DDM systems "very difficult, if not prohibitive" even for small experiments. (snippet) — [OSTI 1764079](https://www.osti.gov/biblio/1764079)
- Complexity for small teams: the 2025 JENA Computing WP2 report states Rucio's "complexity remains a significant obstacle for smaller scientific teams with limited time and computing expertise". (snippet) — [JENA WP2 report, arXiv:2503.09213](https://arxiv.org/pdf/2503.09213)
- SKA/ESCAPE deployment experience (2022): community help was valuable, but "upgrading major versions is not a particularly user friendly experience"; the team's deployment log ran to "100+ pages". (snippet) — [ESCAPE DIOS workshop, deployment experience, Mar 2022](https://indico.in2p3.fr/event/26307/contributions/105835/attachments/69440/98054/ESCAPE%20DIOS%20Workshop,%20Deployment%20Experience,%20March%202022.pdf)
- SKAO 2024 DDM assessment: Rucio's decentralised model gives fault tolerance on site outage but "a higher operational burden at each site"; hierarchical data remains a challenge; yet "relatively low operational overhead" is listed among advantages in its summary (the deck is internally mixed). SKA proceeded: SRCNet v0.1 ran Rucio test campaigns across realistic science workflows (2025). (snippet) — [SKAO DDM assessment 2024](https://indico.cern.ch/event/1343110/contributions/6089547/attachments/2937876/5160662/JSC%20-%20DDM%20Assessment%20-%20Rucio%20Workshop%2030-09-24%20(1).pdf), [Rucio Workshop 2025 session](https://indico.cern.ch/event/1545309/sessions/629796)
- Belle II: computing model is DIRAC-based, so Rucio was integrated beneath DIRAC (a Rucio file-catalog plugin for DIRAC); Belle II/DIRAC-specific code inside Rucio core was still being cleaned up in 40.0.0 (April 2026). Uses PostgreSQL at BNL, operated by non-DBAs. (snippet + fetched) — [OSTI 1839557](https://www.osti.gov/biblio/1839557), [40.0.0 release](https://github.com/rucio/rucio/releases/tag/40.0.0), [BNL Rucio DB operation](https://indico.cern.ch/event/1343110/contributions/6065565/attachments/2939550/5163940/BNL%20Rucio%20DB%20operation.pdf)
- DUNE: replaced only the replica-manager part of the monolithic SAM; challenges included metadata-dependent non-deterministic paths and the ordering dependency on MetaCat. (snippet) — [Fermilab slides 2021](https://lss.fnal.gov/archive/2021/slides/fermilab-slides-21-111-scd.pdf), [DUNE GDB 2020](https://indico.cern.ch/event/813752/contributions/4051564/attachments/2122693/3573056/DUNE_Rucio_GDB_2020_Oct_Wenlong.pdf)
- Rubin: had to patch Hermes for Kafka and build `rucio-register` + `ctrl_ingestd` to keep Butler registries in sync across USDF/UKDF/FrDF. (snippet) — [Rubin Rucio-Butler integration](https://df-ops.lsst.io/usdf-applications/drp/rucio-butler-integration/info.html)
- Non-WLCG token needs: the 2024 workshop flagged token timelines for non-WLCG communities as an unmet need. (snippet) — [Development roadmap discussion 2024](https://indico.cern.ch/event/1343110/sessions/566779/attachments/2940421/5165695/Development%20roadmap%20discussion.pdf)
- Mitigation for small communities: hosted multi-VO Rucio-as-a-service (STFC), and a CERN talk on pitfalls commissioning Rucio prototype clusters for SMEs. (snippet) — [STFC Rucio service](https://www.scd.stfc.ac.uk/Pages/SCD-STFC-Rucio-Data-Management-Service.aspx), [Rucio Workshop 2024 contribution](https://indico.cern.ch/event/1343110/contributions/6105511/contribution.pdf)
- Other adopters named by the project's community page include many beyond those above (snippet, list not read in full). — [Rucio community page](https://rucio.cern.ch/community.html)

### Inferences
- Minimum viable Rucio = DB + REST server + auth + ~10+ daemons + FTS3 (itself a service with its own DB) + message broker + monitoring stack + storage endpoints speaking WebDAV/XRootD/S3 with third-party copy. That floor is the main barrier for small collaborations; "as a service" hosting is the community's answer rather than a lighter single-binary mode.
- The DIRAC/Belle II and Butler/Rubin cases show Rucio adopted as a subsystem under an existing framework — the integration surface (events, registration, name mapping) matters as much as the core features.

### Gaps
- No quantified ops effort (FTE) for any community found. No full text of the two "beyond ATLAS" papers, so per-experiment pitfalls are thinner here than those papers would support.
- CTAO, XENON, IGWN, ESCAPE specifics for 2025-26 not verified in this session (they are documented adopters historically; current status unconfirmed).

## 7. Current status: releases, cadence, governance, license, roadmap

### Takeaway
Healthy, steadily maintained project with a predictable cadence (major every ~4 months, LTS yearly). As of 2026-11-23 the current LTS is 41 ("Donkémon", July 2026; latest 41.5.2 released today), 38 LTS is still patched, and 42.0.0 is at rc1. Recent majors are dominated by clean-up (CLI rewrite, legacy protocol/auth removal, policy-package extraction, security fixes) rather than new data-model capability.

### Cited Findings
- Release table (official policy page): 35 LTS 2024-07 (supported to 2026-08); 36 2024-12; 37 2025-04; 38 LTS "Donkirk" 2025-08 (to 2027-08); 39 "Grand Theft Donkey" 2025-12; 40 "Guardians of the Donkeyverse" 2026-04; 41 LTS "Donkémon" 2026-07 (to at least 2028-07); 42 "How to Train Your Donkey" planned 2026-11. — [Release policy (rucio/documentation)](https://github.com/rucio/documentation/blob/main/docs/started/releasepolicy.md)
- Policy: semantic versioning; a major roughly every 4 months; patch/minor releases roughly every two weeks; no schema/API-breaking changes within a line; one LTS per year with ~2 years of security/critical patches and >=12 months overlap; a server supports clients back to the second-previous LTS. WebUI, JupyterLab extension and Helm charts share the server major version. — [Release policy](https://github.com/rucio/documentation/blob/main/docs/started/releasepolicy.md)
- Latest releases (GitHub, dates shown without year; year inferred from policy table): 41.5.2 and 38.6.5 on 23 Nov 2026; 42.0.0rc1 on 20 Nov 2026 (MySQL 8.4 / Postgres 16 test images, `is_opendata` DID column, Globus SDK 3+, QoS policy client functions removed, old CLI deprecated); 41.5.0/38.6.3 on 16 Oct (four security advisories); 41.0.0 on 20 Jul; 40.0.0 on 20 Apr. — [rucio/rucio releases](https://github.com/rucio/rucio/releases)
- 40.0.0 and 41.0.0 both state "This major release does not include any schema changes". — [40.0.0](https://github.com/rucio/rucio/releases/tag/40.0.0), [41.0.0](https://github.com/rucio/rucio/releases/tag/41.0.0)
- 41.2.2 added an AGENTS.md "to guide AI-assisted and new contributions"; a third-party `rucio-mcp` (MCP server for Rucio) exists. — [rucio/rucio releases](https://github.com/rucio/rucio/releases), [rucio-mcp docs](https://rucio-mcp.readthedocs.io/en/latest/tools/)
- License Apache-2.0; ~326 GitHub stars, ~425 forks, ~422 open issues (Nov 2026). — [rucio/rucio](https://github.com/rucio/rucio)
- Governance: open-source project led from the ATLAS collaboration with project management shared with several other organisations (2019 paper); weekly Rucio meetings with per-release roadmap tracking on CERN Indico; annual Community Workshop (7th: 30 Sep-4 Oct 2024; 8th: 3-7 Nov 2025 at SKAO HQ); Special Interest Groups (SIG Metadata, SIG Tokens). (snippet) — [Barisits et al. 2019](https://link.springer.com/article/10.1007/s41781-019-0026-3), [Rucio Workshop 2025](https://indico.cern.ch/event/1545309/contributions/), [SIG Tokens](https://rucio.cern.ch/documentation/sig_tokens)
- Roadmap themes 2024-26 (from workshop/meeting notes): token (OIDC/WLCG) transition; moving all metadata to JSON and deprecating the column plugin, toward a "schema-governed catalog"; new WebUI aimed at "getting communities running quickly"; removing community policy from upstream into policy packages; open-data support. (snippet) — [Development roadmap discussion 2024](https://indico.cern.ch/event/1343110/sessions/566779/attachments/2940421/5165695/Development%20roadmap%20discussion.pdf), [Rucio 36 roadmap notes](https://indico.cern.ch/event/1431526/contributions/6022681/note/), [Rucio Workshop 2025](https://indico.cern.ch/event/1545309/contributions/)

### Inferences
- More stars than forks is inverted (326 stars vs 425 forks): the project is used and contributed to by institutional operators, not discovered by the general data-engineering community. Mindshare outside physics/astronomy is minimal.
- Roadmap items (tokens, metadata redesign, policy-package extraction) have been "in progress" across several years of workshops; core development capacity is limited and ATLAS/CMS operational needs dominate priorities.

### Gaps
- 42.0.0 final not yet released as of 2026-11-23 (only rc1 seen). Details of 39.0.0 (Dec 2025) release notes not fetched.
- No 2026 community workshop information found (whether a 9th workshop took place in 2026 is unverified).
- Formal governance documents (advisory board composition, funding) not verified.

## 8. What Rucio lacks relative to modern data catalogs

### Takeaway
Measured against Unity Catalog / Iceberg REST / DataHub / OpenMetadata-style catalogs, Rucio is missing essentially the entire "data science" half: structured entities, schemas, lineage, table formats, search/discovery UX, fine-grained governance. These are not on its near-term roadmap either; its roadmap is about making the existing replica manager cleaner.

### Cited Findings
- Provenance/lineage is absent from Rucio's own model: DUNE keeps "file provenance" in MetaCat explicitly because Rucio only replaces the replica manager. (snippet) — [DUNE GDB 2020](https://indico.cern.ch/event/813752/contributions/4051564/attachments/2122693/3573056/DUNE_Rucio_GDB_2020_Oct_Wenlong.pdf)
- No schema governance yet: metadata is fixed columns or free key:value JSON; a "schema-governed catalog" was a 2025 workshop proposal. (snippet) — [Rucio Workshop 2025](https://indico.cern.ch/event/1545309/contributions/)
- Limited query: comparison operators over keys + name wildcards; no joins, full-text search, facets or aggregation in the core API; search does not scale indefinitely. (snippet) — [DID metadata docs](https://rucio.cern.ch/documentation/operator/did_meta), [Rucio/DIRAC evaluation 2020](https://indico.cern.ch/event/876274/contributions/3692660/attachments/1986191/3309617/RucioDiRAC3Evaluation20200212.pdf)
- Discovery UI: the new WebUI is an operations/data-management UI (DIDs, rules, RSEs) still expanding its coverage; it is not a faceted data-discovery portal. — [rucio/webui](https://github.com/rucio/webui)
- Authorization gaps are real, not theoretical: "missing authorization on Open Data catalog endpoints" was a 2026 advisory; `rucio upload` only began checking the `add_did` permission in 41.0.0 (#6058). — [rucio/rucio releases](https://github.com/rucio/rucio/releases), [41.0.0](https://github.com/rucio/rucio/releases/tag/41.0.0)
- Small-site unfriendliness: complexity is "a significant obstacle for smaller scientific teams". (snippet) — [JENA WP2 report](https://arxiv.org/pdf/2503.09213)
- Not a backup system, not a software-management system. (snippet) — [Rucio/DIRAC evaluation 2020](https://indico.cern.ch/event/876274/contributions/3692660/attachments/1986191/3309617/RucioDiRAC3Evaluation20200212.pdf)

### Inferences
(These are assessments from the absence of features in the sources reviewed; I found no document claiming these features exist.)
- No table/column/schema entities; no Iceberg/Delta/Hudi awareness; no notion of snapshot or time travel; no ML-dataset concepts (splits, feature schemas, model/dataset cards).
- No lineage graph (file parentage, job/process entities, OpenLineage/PROV export).
- No per-object ACLs, row/column policies, tags/classifications, or audit-oriented governance; access control is account + scope + policy-package Python.
- No POSIX/hierarchical namespace, no native mount/FUSE as a first-class path (data is reached via client download or via protocol URLs from `list-replicas`).
- No lightweight mode: cannot run usefully as a single process against one bucket; assumes FTS3 and third-party-copy-capable storage for transfers.
- No content-addressing/dedup; checksums (adler32/md5) are for integrity only.
- Conversely, modern data catalogs lack what Rucio does best: declarative multi-site replica placement, quotas, lifetime-driven deletion, tape-aware transfer orchestration, bad-replica recovery. A system combining both halves would be novel.

### Gaps
- No published side-by-side of Rucio vs Unity Catalog / Iceberg REST / DataHub was found; the comparison above is my own gap analysis and should be presented as such.

## 9. Published criticisms and comparisons (iRODS, DIRAC DMS, Globus); lessons learned

### Takeaway
Rigorous independent comparisons are scarce; most published comparisons are written by Rucio adopters or the Rucio team and are positional. The consistent, credible criticisms are: high complexity/operational floor, weak metadata and search, WLCG-flavoured assumptions, painful upgrades, and poor fit for hierarchical or small-file data.

### Cited Findings
- 2019 Rucio paper's own positioning: iRODS operates at smaller scale and is typically targeted at single data centres; DIRAC (from LHCb) offers "less advanced data management features" at smaller scale; neither provides the same catalog + policy + lifecycle combination; Globus is a transfer service, usable underneath Rucio as a transfer tool. (snippet; Rucio-authored, so partisan) — [Barisits et al. 2019](https://link.springer.com/article/10.1007/s41781-019-0026-3), [ASTERICS repository note](https://repository.asterics2020.eu/node/155)
- UK Rucio vs DIRAC evaluation (Feb 2020) for a multi-community service: noted Rucio managed >0.5 EB at the time, is not a backup or software-management system, and its metadata search "does not scale indefinitely". (snippet) — [RucioDiRAC3Evaluation 2020-02-12](https://indico.cern.ch/event/876274/contributions/3692660/attachments/1986191/3309617/RucioDiRAC3Evaluation20200212.pdf)
- Coexistence rather than replacement is common: Belle II runs Rucio beneath DIRAC; Pierre Auger used iRODS as the main access route for simulations (IN2P3 support ending end-2025, CNAF taking over) with some collaborators using DIRAC DMS. (snippet) — [OSTI 1839557](https://www.osti.gov/biblio/1839557), [Auger software report, arXiv:2503.20650](https://arxiv.org/pdf/2503.20650)
- SKAO's 2024 assessment (see sections 1 and 6): trade-off between decentralised fault tolerance and per-site operational burden; hierarchical data unresolved. (snippet) — [SKAO DDM assessment](https://indico.cern.ch/event/1343110/contributions/6089547/attachments/2937876/5160662/JSC%20-%20DDM%20Assessment%20-%20Rucio%20Workshop%2030-09-24%20(1).pdf)
- JENA WP2 (2025): complexity is a significant obstacle for small teams. (snippet) — [arXiv:2503.09213](https://arxiv.org/pdf/2503.09213)
- ESCAPE/SKA (2022): major-version upgrades are not user friendly. (snippet) — [ESCAPE DIOS deployment experience](https://indico.in2p3.fr/event/26307/contributions/105835/attachments/69440/98054/ESCAPE%20DIOS%20Workshop,%20Deployment%20Experience,%20March%202022.pdf)

### Inferences — consolidated strengths and weaknesses
Strengths
- Proven exabyte / billion-file / 120-site scale in two independent production instances over 10+ years.
- Declarative replication rules + locks + lifetimes + quotas: a clean, general placement and lifecycle model.
- Storage abstraction (RSEs, attributes, expressions, protocol plugins) spanning disk, tape, object stores, clouds.
- Self-healing loop (bad/suspicious replica recovery, consistency audits, dark-data cleanup).
- Permanent identifiers; deterministic path mapping.
- Vendor-neutral, Apache-2.0, multi-DB via SQLAlchemy; pluggable transfer tools (FTS3, Globus); event stream for integration.
- Real community: predictable releases with LTS, workshops, SIGs, policy-package mechanism for per-community customisation.

Weaknesses
- Not a metadata catalog: every major user runs a second system (AMI, DBS, MetaCat, Butler); no lineage, no schemas, weak queries; redesign still pending in 2026.
- File-only granularity; per-file-per-replica-per-lock bookkeeping; small-file and hierarchical-data workloads fit poorly.
- Heavy operational footprint (DB, servers, many daemons, FTS, broker, monitoring); complexity repeatedly cited as a barrier; major upgrades painful; PostgreSQL tuning under-documented relative to Oracle.
- Central RDBMS is the bottleneck and single point of coordination; analytics must be exported elsewhere.
- Eventual consistency by reconciliation: stuck rules/locks, permanently non-zero backlog of unavailable files and failed transfers.
- WLCG heritage: X.509 legacy, slow token migration, grid storage protocols and FTS assumed; customisation requires writing Python policy packages.
- Coarse authorization; 2026 advisories for command injection/RCE and missing authz.
- Minimal discovery UX; low visibility outside physics/astronomy; slow feature velocity on data-model capability (no schema changes across 40 and 41).

Lessons for a new catalog design
- Keep placement policy declarative and separate from user intent to delete (rules/locks/tombstones), but make it an optional layer.
- Put rich, schema-governed metadata and lineage in the same system as replica state so policies can key on them and there is one source of truth.
- Model collections/manifests/tables as first-class units so cost is not strictly per-file.
- Provide a single-node/embedded mode and a scale-out mode from the same codebase.
- Emit a standard event stream (the Rubin/Butler bridge shows this is how downstream registries stay in sync).
- Treat names from users as untrusted input; design per-object authorization early.

### Gaps
- No peer-reviewed, independent head-to-head benchmark of Rucio vs iRODS vs DIRAC DMS vs Globus was found; available comparisons are positional or adopter evaluations.
- Full text of the 2020 UK evaluation, the SKAO assessment and the two "Rucio beyond ATLAS" papers was not accessible in this session; only search extracts were used.
