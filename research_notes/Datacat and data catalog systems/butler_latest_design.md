# Rubin/LSST Data Butler: latest design thinking (as of 2026-11-23)

Research notes for the datacat redesign. Basics (registry/datastore split, dimensions, dataset types,
RUN/TAGGED/CALIBRATION/CHAINED, quantum graphs, DMTN-288 client/server retrofit) are assumed and not repeated.

## 0. Method and verification status

Sources were read from Git (lsst.io itself was not fetched; the tech-note source repos under
`github.com/lsst-dm/dmtn-NNN` are the primary sources). Dates are last-commit dates of the source repo.

Read in full (primary source):

- DMTN-346 "Towards immutable Butler collections", D. Irving, first draft 2026-07-28, merged 2026-10-15 — https://github.com/lsst-dm/dmtn-346 (https://dmtn-346.lsst.io)
- DMTN-352 "Enhancing the Portability of the Butler", T. Jenness + J. Bosch, 2026-09-30, plus its `config-feasibility.md` appendix — https://github.com/lsst-dm/dmtn-352
- DMTN-353 "Data download for the RSP", R. Allbery, 2026-10-02 — https://github.com/lsst-dm/dmtn-353
- DMTN-350 "Setting up Butler for DP2" (iDAC how-to), 2026-08-25 — https://github.com/lsst-dm/dmtn-350
- DMTN-345 "Re-Architecting Multi-Site Processing Data Flow" — `main` is an empty stub; the content exists only on
  UNMERGED draft branches (`u/ctslater/initial_intro` 2026-08-07, `u/airnandez/current-system-architecture` 2026-08-11,
  `u/atani/challenges-section` 2026-07-31, `u/eiger/requirements-section` 2026-08-06). "Proposed new architecture" and
  "Plan" sections are still empty. Treat as work-in-progress opinion of a working group, not an adopted design.
  https://github.com/lsst-dm/dmtn-345
- DMTN-271 "Reimagining quantum graphs for post-execution transfers and provenance", J. Bosch, revised 2025-12 ("Update to
  reflect implementation decisions", "Replace and expand plan for AP provenance"), merged 2026-01-16 — https://github.com/lsst-dm/dmtn-271
- DMTN-330 "Publication flow for Butler prompt data products", 2025-11-04 — https://github.com/lsst-dm/dmtn-330
- DMTN-310 "Reducing Butler database contention in Prompt Processing", 2025-03-20 — https://github.com/lsst-dm/dmtn-310
- DMTN-297 "Butler Databases for Community Science Users at the RSP", 2024-11-18 — https://github.com/lsst-dm/dmtn-297
- DMTN-303 "Writeable Butlers for Science Users", J. Bosch — `main` is a stub; the real text is on UNMERGED branch
  `tickets/DM-48071` (2025-01-09) — https://github.com/lsst-dm/dmtn-303
- DMTN-306 "Data Movement Model for the Vera C. Rubin Observatory" (Rucio/FTS/HermesK/ingestd), 2025-07-16 — https://github.com/lsst-dm/dmtn-306
- `lsst/pipe_base` `doc/lsst.pipe.base/recording-provenance.rst` and `CHANGES.rst` at commit 483a95d (2026-11-02)
- `lsst/daf_butler` `doc/lsst.daf.butler/CHANGES.rst` v28.0.0 (2024-11-20) through v30.0.12 (2026-10-27), `datastores.rst`,
  and server/client code at commit c99d1a6 (2026-11-13): `remote_butler/server/README.md`, `_gafaelfawr.py`, `_config.py`,
  `handlers/_file_info.py`, `handlers/_query_limits.py`, `handlers/_query_streaming.py`, `_uuid.py`, `_dataset_ref.py` (id generation).

Read in part (grep/sections only): DMTN-356 (computing-ecosystem overview paper draft, 2026-11-17; Butler/Rucio and
"challenges" paragraphs), DMTN-339 (serialization of data products, 2026-06; limitations/requirements/future-work sections),
DMTN-167 (collection conventions; 2026 diff only), DMTN-203/204 (2026-10 status notes only), `queries.rst` (first third),
query driver code (signatures and constants only), `butler_transform` and `prompt_publication_service` READMEs.

Search snippets only (not read): SPIE 2024 "Converting Rubin Observatory's data butler to a client/server architecture"
(=DMTN-288), arXiv 2501.00544 (SIAv2 over Butler, =DMTN-300).

Not found / could not do:

- No "Butler v2 / next-generation Butler" document exists among lsst-dm tech notes up to DMTN-363 (2026-11-18). The design
  evolves through narrow tech notes plus code.
- No ADASS 2025/2026 or SPIE 2026 Butler-architecture paper surfaced in web search (only the 2022 and 2024 SPIE papers).
  A 2026 SPIE paper is cited inside DMTN-204 for the exposure-log database (`2026SPIE14151E..2TB`), not for Butler design.
- No relevant community.lsst.org design threads surfaced in search; the forum was not browsed directly.
- GitHub PR/issue listings for lsst/daf_butler were not accessible (API not enabled for this session), and Rubin Jira
  (rubinobs.atlassian.net) was not consulted. Ticket numbers below come from changelogs/tech notes.
- DMTN-333 ("Data Publication improvements roadmap", 2025-11) and DMTN-355 are empty stubs. DMTN-283/288/289/242/249/205
  were not re-read (pre-2025, assumed known).

## 1. Newest design documents

### Newest substantive Butler design document

**DMTN-346 "Towards immutable Butler collections" (Irving; merged 2026-10-15).** DMTN-352 (2026-09-30) is the other
candidate and is broader; both are current. DMTN-346 is the newest one that changes the data model direction.

Problem: collections are mutable, so they cannot be reliably replicated between sites, a campaign cannot record which
version of `LSSTCam/calib` it ran against, and two campaigns cannot use different versions concurrently.

Proposal: a `butler collection-snapshot {create,list-datasets,load,verify}` tool that produces pseudo-immutable,
content-addressed point-in-time copies, with NO core API or schema change:

- Walk the chain tree depth-first and copy each collection. CHAINED/TAGGED/CALIBRATION are copied as-is; a RUN becomes a
  TAGGED collection with the same members.
- Snapshot name = `SNAPSHOT/<original name>/<UUIDv5>`; the UUIDv5 hashes name, type, doc, the sorted member dataset UUIDs
  with data IDs, calibration timespans, and (for chains) the child names after conversion to snapshot names. This is a
  Merkle tree: unchanged children are reused, uncoordinated users converge on the same names, and integrity is verifiable
  from the database alone.
- The user-facing name is a single-child CHAINED alias, e.g. `LSSTCam/calib/2026-07-28 -> SNAPSHOT/LSSTCam/calib/<uuid>`.
  Campaign input chains should point at the dated alias, never at the mutable `LSSTCam/calib`. Open question in the note:
  whether the top-level name should also carry a hash, since otherwise users must promise not to remake it.
- Transport is a JSON Lines file, one collection per line, written depth-first; `load` is idempotent, verifies existing
  collections, and reports missing dataset UUIDs (to feed Rucio or `transfer-datasets`). Files move by other means.
- One transaction per collection, not one giant transaction, because "transactions registering datasets to a calibration
  collection currently take a global lock on the table".
- Garbage collection: any `SNAPSHOT/` collection unreachable from a non-`SNAPSHOT/` chain can be pruned.
- Subsetting by dataset type or data ID yields a distinct hash, useful for bringing back partial outputs.
- Stated failure case: `LSSTCam/raw/all` is one RUN growing ~200,000 datasets/day; every snapshot would duplicate it
  into a huge TAGGED collection and a JSON file that eventually will not fit in memory. Mitigations listed: partition raws
  by `day_obs`, one JSON file per child, store snapshot files in the Butler, and "immutable collections with their hashes
  stored directly in the Butler database". Interim workaround: snapshot a `defaults/non_raw` subtree.

Not implemented in daf_butler as of 2026-11-13 (no snapshot command in the tree).

### Other 2024-2026 notes

**DMTN-352 "Enhancing the Portability of the Butler" (Jenness, Bosch; 2026-09-30).** Synthesises feedback from external
adopters (SPHEREx, PFS, WFST, HSC). Proposals with effort estimates: (a) replace the inherited gen2 config system
(untyped dict hierarchy, six composing discovery mechanisms, no validation, typos silently ignored) with Pydantic
models, split per-butler config from per-data-release config, and discover storage class/formatter definitions via
Python entry points keyed by `(namespace, version)` such as `("rubin-lsst","dp1")` — 8 weeks; (b) replace the
per-process `StorageClassFactory` singleton with a keyed cache like `DimensionUniverse`; (c) entry points for
`daf_butler_migrate` keyed by universe namespace — under 1 week; (d) remove the `afw` dependency from `obs_base` — 4
weeks; (e) include datastore records in quantum graphs by default and make `pipetask run` execute through the
quantum-backed butler like BPS does — 4 weeks; (f) "incremental processing" (overwriting a coadd as data arrive), which
the data model forbids because a data ID is unique within a collection; considered a research topic, possibly a special
dimension holding a UUIDv7 — 2-4 weeks to demonstrate; (g) persistence correction (quantum depends on the previous
observation), compatible with the data model but disruptive to the quantum graph format.

**DMTN-353 "Data download for the RSP" (Allbery; 2026-10-02).** Butler-issued signed URLs are the only download route
and are judged inadequate: they expire, they can be passed to people without data rights while valid, there is no rate
limiting, and there is no persistent URL per dataset. Proposal: a separate download service with persistent URLs that
does authn/authz on every access, applies per-user byte quotas at signing time, and redirects to a ~15-minute signed
URL; Butler would return download-service URLs instead of store URLs; a format-conversion service next to the object
store handles ZIP-member extraction and FITS-to-HDF5. Open problems: authorization knowledge lives only in Butler
(object paths carry no data-release structure), and backend choice needs client location that Butler does not receive
because requests arrive via DataLink.

**DMTN-345 "Re-Architecting Multi-Site Processing Data Flow" (working group; drafts July-August 2026, unmerged).**
Diagnosis only so far. See section 6.

**DMTN-350 "Setting up Butler for DP2" (2026-08-25).** Operational: an iDAC gets the registry as a tarball of parquet
files exported by `lsst-dm/butler_transform`, bulk-loaded into Postgres 17 with `btree_gist`; datastore paths are either
relative (ChainedDatastore of two FileDatastores with separate roots and separate records tables) or rewritten to
site-absolute paths through a per-site path-mapping function.

**DMTN-271 (Bosch; revised 2025-12, merged 2026-01-16).** File-based provenance and new quantum graph formats; now
implemented. See section 2.

**DMTN-330 "Publication flow for Butler prompt data products" (2025-11-04).** A Prompt Publication Service with its own
state database (one row per dataset, status per repository) drives datasets through
embargo -> `prompt_prep` -> `/repo/main` -> Google `prompt` -> expired. See sections 5 and 6.

**DMTN-310 "Reducing Butler database contention in Prompt Processing" (2025-03-20).** Workers stop writing to the
registry; they send JSON records over Kafka to a single writer service that batches inserts. See section 5.

**DMTN-297 "Butler Databases for Community Science Users at the RSP" (2024-11-18).** Taxonomy of registry deployments
(static / batch-updated / online) and an inconclusive analysis of user-writeable repositories. See sections 4 and 5.

**DMTN-303 "Writeable Butlers for Science Users" (Bosch; draft branch 2025-01-09, unmerged).** "Satellite" personal
DirectButler repositories that reference official datasets served by RemoteButler. See section 4.

**DMTN-306 "Data Movement Model" (2025-07-16)** and **DMTN-356 (overview paper draft, 2026-11-17).** Butler-Rucio
bridge as built, and its acknowledged weaknesses. See section 6.

**DMTN-339 "Challenges serializing Rubin Observatory data products" (2026-06).** Not a Butler note but it drives storage
classes: the DP1 FITS serialization (C++ archive tables, pickle) could not be documented or partially read from object
stores; DP2 moves to per-product Python classes (`lsst.images`) with public schemas, a data model separated from file
format (FITS, JSON, HDF5, Zarr backends), and a FITS convention storing an index of HDU byte offsets.

**DMTN-203 / DMTN-204 status notes (2026-10-20/21).** DMTN-204 (annotations/quality flags in the registry) is now
marked deprecated: "we decided that it was simpler to continue to track dataset metadata ... in a separate database ...
and not expand the Butler data model". DMTN-203: metrics went to a write-only `SasquatchDatastore`; "there is as yet no
way to perform graph builds constrained by metrics"; the expected route is to query ConsDB for exposure IDs first.

**DMTN-167 (collection conventions; updated 2026-03/05 for RFC-1159).** Prompt run collections were renamed from
hash-based `<inst>/prompt/output-YYYY-MM-DD/<pipeline>/pipelines-<hash>-config-<hash>` to
`<inst>/runs/prompt/YYYYMMDD/<pipeline>/<deployment-id>`, with daily chains to which Daytime-AP catch-up runs are
prepended.

## 2. Provenance

Decision (DMTN-271): provenance is stored in FILES that are themselves Butler datasets, not in registry tables. This
"partially supersedes" DMTN-205 (2022), which assumed quantum/edge tables in the database. Stated reasons: simpler to
implement; "provenance schema changes are much easier"; better separation of concerns between `daf_butler` (which
knows nothing about quanta) and `pipe_base`. Stated costs: external tools expecting a SQL schema shaped like IVOA
provenance cannot query it directly, and some operations need shallow reads of a whole-RUN file.

What exists now (pipe_base v30.0.0, 2026-01-16, and later; `recording-provenance.rst`):

- `PredictedQuantumGraph` (`.qg`) replaces `QuantumGraph` (`.qgraph`): a zip archive of zstd-compressed JSON
  components, each a Pydantic model, plus "multi-block" files of concatenated compressed blocks indexed by a binary
  address table sorted by UUID (O(log N) lookup, partial reads over object stores). Zip members were rejected for
  per-quantum data because zip overhead is significant at millions of entries.
- `ProvenanceQuantumGraph`: one per RUN collection, stored as dataset type `run_provenance` with empty dimensions.
  Bipartite quantum/dataset graph. Per quantum: UUID, task label, data ID, status, exception info, caveats, every
  attempt, input and output UUIDs. Per dataset: UUID, type, data ID, existence at execution time, producer and
  consumers. Includes all predicted datasets, so non-retained intermediates stay traceable. Also embeds the full
  pipeline graph with task configs, and the content of all per-quantum log and task-metadata datasets.
- One artifact, many datasets: `{task}_metadata`, `{task}_log` and config datasets are re-ingested so that their
  datastore records point into the provenance file, and the original small files are deleted. Planned further types:
  `{task}_provenance` (UUID equal to the quantum UUID) so that data-ID-first provenance questions can use the ordinary
  query system.
- Workflow: `pipetask run-qbb` writes outputs with no database access; `butler aggregate-graph` (replaces
  `transfer-from-graph`) scans outputs, ingests datasets, and writes the provenance graph; `butler ingest-graph` later
  ingests it and swaps the small datasets over. The ordering is rigid: `ingest-graph` must not go in the BPS `finalJob`,
  `bps restart` is unsafe after it, and an interrupted `ingest-graph` must be rerun or files are orphaned.
  `--promise-ingest-graph` skips ingesting the small files at aggregation time at the risk of orphans and of wrong
  `--skip-existing-in` results downstream.
- Reporting: `butler provenance-report` (table or JSON) replaces `pipetask report`/`QuantumProvenanceGraph`.
- In-process executors write provenance too (v30.0.0, v30.0.6), including when a task raises.
- Prompt processing: each worker writes a small `{visit, detector}` provenance graph as a normal dataset; these are to
  be merged into per-RUN `run_provenance` during the day before unembargo.

Per-dataset provenance (daf_butler v29.0.0, 2025-03-25, DM-35396): `DatasetProvenance` (quantum ID plus input refs) is
passed to `Butler.put()` and formatters write it into file headers/metadata (FITS, Parquet). v30.0.0 added limits "to
prevent excessive numbers of input datasets being recorded in file metadata" (2,000 inputs for Parquet).

Granularity summary: registry holds no edges at all; RUN collection is the unit of graph storage; quantum and dataset
are the node granularity; files carry their own direct inputs.

Problems that drove the change (DMTN-271): the old graph imported every task class and executed config code on read, so
persisted graphs rotted with the software stack and were a security concern; it was quantum-quantum only, while
provenance questions are dataset-first; one instance per file made cross-run questions awkward; execution-only content
(dimension and datastore records) was mixed with post-execution status; and the per-quantum small files were a storage
and transfer burden.

Acknowledged gaps: the high-level provenance query system is "out of scope" and undesigned; cross-RUN queries must open
several files; finding the RUN of a non-retained dataset from its UUID alone is not possible; "all raws contribute to
every coadd" through global calibration, so naive ancestor traversal is useless without edge filtering; direct
`pipetask run` use with clobber/restart is "provenance-hostile" and may get this later. Annotations of datasets and
provenance-propagated quality flags (DMTN-204) were dropped.

## 3. Query system

- `Butler.query()` (context manager returning a `Query`), `query_datasets`, `query_data_ids`,
  `query_dimension_records` became public in v28.0.0 (2024-11-20). In v30.0.0 (2026-01-15) the legacy
  `Butler.registry.query*` methods and `find_dataset` were re-implemented on the same framework and the `daf_relation`
  dependency was removed entirely. `Butler.registry` is "being phased out"; `queryDatasetTypes` is the main method with
  no replacement yet.
- A query is a serialisable `QueryTree` (Pydantic) plus a result spec, executed by a `QueryDriver`
  (`DirectQueryDriver` builds SQL; the remote driver posts the tree to the server). `Query` methods: `data_ids`,
  `datasets`, `dimension_records`, `general`, `materialize`, `join_dataset_search`, `join_data_coordinates`,
  `join_data_coordinate_table`, `join_dimensions`, `where`.
- `Butler.query_all_datasets()` (v30.0.0) queries many dataset types in one call.
- Spatial: `OVERLAPS` with `POINT(ra, dec)` (v28) and ADQL-like `CIRCLE`, `BOX`, `POLYGON`, `REGION` (v30.0.0).
  Constraining by `htmN`/`healpixN` values in data IDs was removed (except `htm7`). Spatial joins between dimension
  elements are added automatically; SQL does a coarse join and exact region overlap is applied by a Python
  `Postprocessing` step, which means LIMIT sometimes has to be applied after fetching and filtering.
- Temporal: calibration lookups are timespan joins. v30.0.0 changed the conceptual timespan of RUN and TAGGED
  collections from NULL to unbounded, because a RUN earlier in the search path "would be silently ignored in favor of a
  later CALIBRATION collection" in quantum graph builds. Find-first now works in calibration collections; a
  post-processing check requires exactly one matching calibration per data ID.
- Expression language additions: `:name` bind syntax, `GLOB()`, `UUID('...')` literals, boolean columns, `ingest_date`
  comparisons; case-insensitive identifiers were dropped.
- Paging: OFFSET was removed (deprecated `offset` parameter deleted in v30.0.0); only ORDER BY + LIMIT. CLI commands
  default to `--limit 10000` with a warning when hit. Direct driver reads 2,000 raw rows per page from a server-side
  cursor; rows are deduplicated.
- Server side (code): results are streamed as JSON Lines pages with a keep-alive every 15 s; at most 25 concurrent
  streaming queries per server replica (HTTP 503 + Retry-After 5 s beyond that) and 2 per user per replica (HTTP 429),
  which the code calls "kind of a bad hack"; thread pool 40 and 40 DB connections per repository; data-ID uploads capped
  at 100,000 rows. `materialize` is emulated remotely by resending the sub-tree and re-running it. Butler server needs
  Python 3.13+ after a Postgres-cursor concurrency segfault (v30.0.4).
- Known performance problems, from sources: v30.0.1 pipe_base guards against graph builds whose initial query returns
  far more rows than end up in the graph; v29.1.0 pushed WHERE terms into subqueries; spatial-join bugs fixed in v29.0.0
  produced over-coarse joins or false overlaps; a `temporary_tables: False` registry option was added (v30.0.0);
  DMTN-303 says graph building leans on temporary tables, which the remote client only emulates, and suggests dropping
  temp-table support in favour of re-uploading data IDs; DMTN-303 also says the single-query approach to graph building
  "already fails for some hard" problems (hence `--dataset-query-constraint`, "more of a workaround than a real
  solution") and cannot span two repositories at all.
- Query results depend on per-client caches of dataset types and governor values; concurrent writers are not seen
  until `Registry.refresh`.

## 4. Client/server

- RemoteButler is in production for end users (DP0.2, DP1, DP2, prompt products) and is read-only. In current code
  `put`, `ingest`, `ingest_zip`, `import_`, `export`, `transfer_from` (as destination), `pruneDatasets`, `removeRuns`,
  `transaction` all raise `NotImplementedError`; there is no upload endpoint. It can be the SOURCE of `transfer_from`
  (v29.1.0). About fifteen external routes.
- The server holds no science code: it never resolves storage classes or formatters. It returns a location plus the
  recorded formatter name; the client downloads and deserialises. DMTN-352's appendix treats "server must never import
  payload code" as an invariant to be enforced by test.
- Authn: Gafaelfawr tokens (Rubin) or CADC mode. Authz: a per-REPOSITORY list of Gafaelfawr groups in server config
  (`authorized_groups`, `*` for everyone), cached per user. There are no per-collection or per-dataset ACLs in the code.
  DMTN-303 says the centralised model of DMTN-182 is one "we no longer consider viable" for writes but its
  ownership/ACL description "still seems appropriate"; where ACL state would live is "an open question". DMTN-169 and
  DMTN-182 were only reformatted in 2026, not updated.
- Signed URLs: the server pre-signs GET URLs with a 1-hour lifetime; a code comment says there is no technical reason
  it could not be a day, the limit is political. In CADC mode it returns the plain URL and the client authenticates to
  the store. DMTN-353 plans to take signing away from Butler (section 1).
- Multi-location serving exists only as server policy: for DP2 "the Butler server knows which classes of images are
  available in which locations and chooses to hand back signed URLs for either Google Cloud Storage or SLAC's object
  store, depending on the type of requested image" (DMTN-353). That is per dataset type, not per dataset replica.
- Caching: server clones a template DirectButler per request. Caches assume the repository does not change; adding or
  changing governor dimension records, or deleting/redefining a dataset type, requires a server restart. "We don't yet
  have a way to configure mutable vs immutable repositories in the server."
- What stays direct-DB: all pipeline execution, graph building, ingest, calibration certification, collection
  management, deletion, prompt processing, and every staff repository at the data facilities; iDACs also run
  DirectButler against their own Postgres with a read-only account (DMTN-350).
- User-writeable repositories are unresolved. DMTN-297 lists three options (shared with the release DB, separate
  Postgres, local SQLite) and says "all of these potential solutions have significant problems". DMTN-303's draft picks
  personal "satellite" repositories (SQLite or personal Postgres, DirectButler) that reference official datasets,
  needing: graph building against RemoteButler, a Datastore that obtains signed URLs from the server (plain
  `transfer=None` references fail because signed URLs expire), and later a moderated "publish" flow. Graph building
  across two repositories is called a hard, possibly never fully general, problem.

## 5. Scale and registry problems

Numbers stated in sources: DP0.2 registry ~250 million datasets (DMTN-297); a production repository with 1.5 billion
datastore records (CHANGES v30.0.12); ~200,000 new raws per night into one RUN (DMTN-346); ~3 million prompt datasets
published per day (DMTN-330); USDF registry "multiple terabytes" (DMTN-345 draft, flagged unsure by its author);
"billions" of files per data release (DMTN-356).

What broke or cost, and the response:

- Concurrent small writers. In Operations Rehearsal 5, registry inserts from hundreds of prompt pods took "up to 10
  minutes"; dataset insertion does sanity checks suited to big batches and uses "an excessive number of queries and
  temporary tables" for a few datasets of many types; "collection chain updates take a lock that could cause one
  stalled pod to block completion of all other work". Fix: Kafka + single batching writer service, and a physical read
  replica for the pods' input queries (DMTN-310). The Kafka log then became the source of truth for publication
  (DMTN-330).
- Random UUIDs. New dataset IDs switched from UUIDv4 to UUIDv7 (v30.0.0, DM-52719) because random keys scatter index
  updates; quantum graph files also use UUIDv7 directly because they compress nearly as well as integers.
- Deletion. Emptying datastore trash joined every artifact against trashed ones; on 1.5 billion records
  `pruneDatasets`/`remove-runs` took over an hour per chunk until rewritten as a correlated EXISTS (v30.0.12).
  `removeRuns` now works in restartable chunks and parallel artifact deletes (v29.1.0). `remove-runs` on runs in the
  same chain used to block each other (v29.1.0). Purging intermediates from both Rucio and Butler at campaign end "is
  slow at this scale" (DMTN-356).
- Schema migrations. "At the largest Butler repository at the USDF ... we have a migration that has been pending for
  nearly a year due to the difficulty of completing it in an online environment" (DMTN-297). Each data release keeps
  the dimension universe current at release time, so release databases have "incompatible database schemas"; v30.0.10
  added `conform_to` to move dataset types/refs between compatible universes; `dimensions.yaml` is at version 8.
  Schema-version skew between sites is a campaign risk nothing enforces (DMTN-345 draft).
- Calibration collections: certification takes a global table lock (DMTN-346; `LOCK TABLE ... IN EXCLUSIVE MODE`
  exists in the Postgres backend).
- Loading and copying registries. DP1 took 30-60 minutes to load at Google in one transaction; early releases used
  `pg_dump`/`pg_restore`; DP2 uses `butler_transform`, which exports the registry to parquet files (dimension records,
  datasets, datastore records, associations, collections, manifest) that can be transformed offline and bulk-loaded.
- "Published vs unpublished" is not a registry concept; batch release would "require us to modify the Butler database
  schema", so incremental publication was chosen (DMTN-330). Expiry is expressed with TAGGED collections
  (`Prompt/Outputs/Available` vs `Prompt/Outputs/Expired`), keeping registry rows for deleted files so IDs and
  provenance remain resolvable.
- Performance depends on data being "structured correctly (not too many different dataset types, collection chains not
  too long, etc.)"; read-after-write consistency is assumed by library and pipelines, which conflicts with read
  replicas (DMTN-297).
- Postgres specifics: `btree_gist` required (timespan exclusion); ObsCore-in-Postgres needs `pg_sphere`, unavailable in
  Google Cloud SQL, so live ObsCore can only be hosted at USDF and static releases export ObsCore to Qserv; AlloyDB is
  used for prompt repositories at Google; stale-connection liveness checks (v28); large `IN` lists of dataset IDs broke
  queries until v30.0.0; `Butler.close()`/context managers added to release connections deterministically (v30.0.0).

## 6. Datastore, small files, multi-site, replicas

- File templates: alternates within a field (`{exposure.day_obs|visit.day_obs:?}`, v28), type-conversion fields
  (v29), unsafe characters replaced by underscores (v30.0.8). Templates are per-butler config; changing them affects
  only new datasets because the path is recorded per dataset.
- Datastore records: one row per artifact (path, formatter name, storage class, component, size, optional checksum) in
  an opaque registry table named per datastore. Absolute URIs are allowed and may use a different scheme from the root.
  Checksums are off by default; the DMTN-345 draft states "the Butler stores checksums only for raw images and never
  verifies them". v30.0.0 added an API to compute records for datasets not yet written.
- ChainedDatastore is used to stitch different roots/buckets into one repository (DMTN-350), not as a replica model.
- Ingest: `direct` mode registers external files whose lifetime Butler does not manage (used for raws); hard links
  share one physical copy between `/repo/main` and `prompt_prep`; `Butler.ingest(skip_existing=True)` and a
  pre-check for conflicting datastore records (v30.0.0) make retries safe; zero-byte S3 "directory" objects are no
  longer created (v30.0.0); `--no-track-file-attrs` skips size lookups on object stores (v28).
- Zip/aggregation (v29.0.0, DM-46776): `retrieve_artifacts_zip` and `ingest_zip` with an embedded `ZipIndex` of dataset
  refs; a member is addressed by a `zip-path` URI fragment on the datastore record, so many datasets share one
  artifact. Raws are stored as one zip per exposure (all detectors). Costs acknowledged: DataLink/Portal expect one URL
  per file, so "the current service chain of Butler and DataLink does not correctly support retrieving individual raw
  images" and a zip-extraction microservice is needed (DMTN-330, DMTN-353); bundling "adds complexity to the Butler and
  the data access services" (DMTN-356). Processed outputs are not re-bundled, which Rucio/FTS "handle poorly".
- Small files: JSON/YAML ancillary files are as numerous as FITS/Parquet files (DMTN-306). Current mitigations: fold
  logs/metadata/configs into the provenance graph file (section 2), zip per-quantum outputs, and do not persist
  intermediates. DMTN-271 anticipates a declarative per-dataset-type retention/aggregation policy that does not exist
  yet (pipe_base v30.0.9 added `retained_dataset_types` to the graph builder).
- Orphans: quantum-backed execution writes files before any registry row exists; DMTN-271 proposes registering graphs
  as an "uningested dataset manifest" abstraction so files are never fully orphaned. Not built.

Multi-site and Rucio as built (DMTN-306): each site has an independent Butler that knows only local files.
`rucio_register` selects datasets from a Butler, registers them in Rucio datasets and attaches ~500 bytes of JSON
Butler metadata to each Rucio file record; RSEs use identity logical-to-physical mapping so relative datastore paths
are preserved; FTS moves files; HermesK publishes completion events to a Kafka topic named after the RSE; `ingestd` at
the destination ingests into the local Butler. Only replicated files are known to Rucio.

How replicas are modelled: they are not. A dataset has datastore records only in the repository where the file is
physically present; "replica" means a separate registry row set in another site's database with the same UUID.
Existence at another site is knowable only from Rucio or by asking that site's Butler.

Authors' own assessment (DMTN-345 draft branches, DMTN-356):

- "We are building a distributed system, but the data management system doesn't know that the system is distributed."
  Truth is spread across "three Butler registries, a Rucio catalog, and the memories of the people operating the
  system".
- Rucio synchronisation is lossy: only top-level dataset information travels; collection structure is not
  transferred; dimension-record corrections at the source cannot be propagated; ingest fails if dimension records are
  missing at the destination.
- Collections are moving targets; chains are rebuilt by hand at the destination ("sorting alphabetically by DM ticket
  number mostly works, but not always").
- Failures are silent (a stalled ingest daemon cost several days across three institutions); no end-to-end "is
  everything my campaign needs present and correct at site X" check; early DP2 distribution needed a daily stand-up.
- Goals stated for the redesign: less duplicated state between systems, visibility of data-product status at all sites
  without querying each, versioned input selection. DMTN-356: "One approach under consideration is to extend the Butler
  to carry multi-site location information natively." No design for that is written yet.

## 7. Dataset identity and versioning

- Dataset ID: UUID. `UNIQUE` mode now generates UUIDv7 (was v4). `DATAID_TYPE` and `DATAID_TYPE_RUN` modes generate
  UUIDv5 from dataset type, data ID (and run) under a fixed namespace; used where independent ingests must agree (raws
  ingested separately at each site). Quantum IDs are UUIDs as well, and `{task}_provenance` datasets reuse them.
- Uniqueness: one dataset per (dataset type, data ID) per RUN; datasets are immutable; there is no version field. A
  "new version" is the same data ID in a different RUN, and precedence is the order of a CHAINED search path.
  DMTN-352 names the cost: incremental/overwriting workflows cannot be expressed "without using a new collection or
  including some discriminator in the dataID".
- Collection versioning is the gap being closed by DMTN-346 (content-hashed snapshots; dated aliases).
- External identifiers: IVOA IDs per DMTN-302 (`ivo://org.rubinobs/<release>?repo=<label>&id=<uuid>`), with
  `Butler.parse_dataset_uri` / `get_dataset_from_uri` (v29); `butler://` URIs deprecated. No persistent HTTP URL per
  dataset yet (DMTN-353).
- Storage classes: read-time conversion via `storageClass=` overrides and registered converters; components can be
  resolved against the read storage class with a full-read-then-convert fallback and a warning (v30.0.11); a
  "downgrading" converter lets old repository definitions work with new pipelines at the cost of dropped information
  (v30.0.11); repository config may now override dataset type names and storage classes (v30.0.8).
- Formatters: `FormatterV2` (v28) is file-I/O only; the formatter name is stored per dataset and is permanent, so reads
  need no formatter configuration, while writes do (which is why graphs must carry a config label under DMTN-352).
- Weakness found by the DMTN-352 feasibility study: dataset type compatibility in transfers short-circuits on equal
  storage class NAME, which is only sound because of the process-wide singleton; once definitions are versioned per
  release, both transfer guards "fail open". Proposed fix: carry one composite `schema` handle (dimension universe +
  storage class label) as context excluded from equality, and check compatibility before `conform_to`.

## 8. Things the authors call mistakes, limitations, or future redesign

1. Provenance in registry tables (DMTN-205) was abandoned for files-as-datasets (DMTN-271).
2. Annotations/quality flags in the registry (DMTN-204) were abandoned for a separate database; metric-constrained
   graph building never happened (DMTN-203).
3. Mutable, unversioned collections are now regarded as the root of replication and reproducibility trouble; immutable
   collections with stored hashes are the stated direction (DMTN-346, DMTN-345 draft).
4. One giant RUN for all raws does not work with snapshotting; partitioning by `day_obs` is suggested (DMTN-346).
5. The Butler has no notion of sites or replicas; the Rucio bridge is described as fragile, lossy and manual
   (DMTN-345 draft, DMTN-356). Redesign is in progress with no proposal text yet.
6. Checksums are effectively unused and never verified (DMTN-345 draft).
7. Config system inherited from gen2: untyped, unvalidated, six discovery mechanisms, client and server settings
   intermingled; shipping 120 Rubin storage classes and 94 formatter entries in the core package (most unusable without
   the LSST stack); `StorageClassFactory` singleton "to simplify development in the beginning" (DMTN-352).
8. Uniqueness of data ID per collection blocks incremental processing (DMTN-352).
9. Butler-signed URLs as the user download mechanism: no persistent URL, no quota, leakable while valid (DMTN-353).
10. Direct database writes from many concurrent workers do not scale; writes were moved behind a queue (DMTN-310).
11. Authorization is per repository only; finer ACLs and user write access remain undesigned (DMTN-297, DMTN-303).
12. Server caches cannot be invalidated for some changes; no mutable/immutable repository flag (server README).
13. Online schema migration of the largest repository stalled for about a year (DMTN-297).
14. Quantum graph building as one SQL query fails on hard cases and cannot span repositories; temporary tables are a
    liability for the remote client (DMTN-303 draft).
15. Old `QuantumGraph` format executed code on load; replaced (DMTN-271).
16. Zip bundling broke the one-URL-per-file assumption of downstream services (DMTN-330, DMTN-353, DMTN-356).
17. File-before-registry writes in quantum-backed execution can orphan files; several documented procedures rely on
    operators rerunning tools to avoid it (DMTN-271, `recording-provenance.rst`).
18. DP1 file serialization could not be documented or partially read remotely; replaced for DP2 (DMTN-339).

## 9. Implications worth carrying into datacat (my reading, not the Butler authors')

- Their biggest current pain is exactly the thing datacat intends to model: per-site replicas and a global view. They
  bolted a file catalog (Rucio) beside a dataset catalog and now regret having two systems that each record locations.
- Mutable named groupings need a content-addressed immutable form from the start; they are retrofitting a Merkle scheme
  through naming conventions.
- Time-ordered IDs (UUIDv7) for random assignment, deterministic IDs (UUIDv5) where independent sites must agree.
- Keep graph provenance out of hot relational tables if schema churn is expected, but note they have no cross-run
  query story yet and lineage of deleted intermediates relies on reading per-run files.
- Many small artifacts must be aggregated, and the catalog needs first-class "member of container at offset/path"
  locations that download services understand.
- High-fan-in writers should go through a batching ingest queue; the queue log doubles as a change feed for
  publication and replication.
- Checksums recorded and verified, publication state, and per-collection ACLs are all things they lack and now want.
