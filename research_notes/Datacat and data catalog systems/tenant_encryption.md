# Tenant encryption: reading notes

Notes on how multi-tenant data systems encrypt the data of each tenant with a key of its own, and what that means for mudcat. Read on 2026-11-26.

How to read the confidence marks: **[paper]** means I read the statement in the paper or the document itself. **[search]** means it comes from a search extract of the vendor's documentation, not the full page. **[memory]** means it is from memory and not checked.

## 1. Salesforce Database (SalesforceDB)

Source: Agrawal, Arora, Baker et al., "Salesforce Database: A Cloud-Native Multi-Tenant OLTP Database", PVLDB 19, pages 4426–4439. Section 5 (LSM virtualization), subsection "Tenant Encryption", and the measurements in section 7.

### What the paper says [paper]

- **The need.** "Compliance and regulatory obligations have introduced the need for per-tenant data encryption, where each tenant's data is protected by its own encryption key."
- **Storage model.** Storage is immutable. An *extent* is like a file and a *fragment* is like a block within a file. A fragment cannot be changed after it is written. An extent cannot grow after it is closed. Fragments have variable size.
- **Tenant clustering.** The tenant ID leads each key, so "all data associated with a single tenant forms a contiguous key range". This is shared-table multi-tenancy: many tenants in the same tables.
- **The unit of encryption is the fragment.** "All data stored within a fragment belongs exclusively to a single tenant. Data extents are written to the LSM during flush and compaction, and each fragment is encrypted using a tenant-specific key, with the key ID embedded in the fragment."
- **Decryption** happens on each access to a fragment: for a query, for compaction, or for another process.
- **Why immutability matters.** "In traditional databases, any page mutation and subsequent eviction from the buffer pool would result in re-encrypting the page." With immutable fragments, encryption is done once, at flush or compaction.
- **Keys.** A key management system holds the keys. "Encryption keys can be rotated, and the data rotation is triggered via LSM compaction. Since a key's identifier is embedded in the fragment, data encrypted with the old encryption key can still be read, until the data rotation is completed."
- **Scale and cost.** More than 7,000 tenants are encrypted, with sizes up to 80+ TB. Encryption adds 13.96 µs to a fragment write (p95 49.58 µs) against 2.7 ms for the whole write. Decryption adds 12.52 µs to a fragment read (p95 25.77 µs) against 0.49 ms for the whole read.
- **How it compares.** The paper sets its approach against column encryption and against one database per tenant (it cites Aurora for the second). It names Shield as a system that also encrypts LSM files at flush and compaction, but with no per-tenant keys.
- **The general lesson** (section 9): expensive derived work, "building Bloom and range filters, calculating integrity checksums, and applying fragment-level encryption", is "computed once during asynchronous flush and compaction and reused indefinitely".

### What the paper does not say

- **The transaction log and the Memstore.** The encryption text covers data extents only. The paper does not say whether log extents carry tenant keys. Log records of many tenants are interleaved by nature, so this is the hard case, and the paper is silent on it.
- **Clones.** A sandbox tenant refers to the extents of its source through a tenant mapping in the extent reference. Those extents carry the key ID of the source. The paper does not say which key a clone needs, or whether the clone gets a key of its own before compaction rewrites the data.
- **Customer-held keys.** The paper says only that a KMS manages the keys. It does not describe keys that the customer holds or revokes.
- **What the encryption protects against.** The database decrypts on every access. The encryption protects data at rest and limits the damage of one lost key to one tenant. It does not hide data from the running service.

### The three properties that make it work

1. The tenant ID leads every key, so the data of a tenant is physically together.
2. Blocks are immutable, so each block is encrypted one time.
3. A block never holds data of two tenants, and it names its key.

Rotation needs no new mechanism: compaction rewrites blocks anyway, and writes them with the newest key.

## 2. Apache Iceberg table encryption

Source: `docs/docs/encryption.md` in the apache/iceberg repository, main branch [paper]; a mailing-list thread on interoperability [search].

- **What is encrypted:** data files, delete files, manifests and manifest lists. "The `metadata.json` file does not contain data or stats, and is therefore not encrypted."
- **The key tree follows the file tree.** Each data file gets a key and an AAD prefix from a secure random generator on the worker. The parent manifest stores them in the `key_metadata` field of the entry. The manifest is encrypted with its own key, which the manifest list stores in its `key_metadata`. The key metadata of the manifest list is encrypted with a key encryption key (KEK). The KEK is wrapped by the table master key through the KMS and stored in the `encryption-keys` list of the table metadata.
- **Formats:** Parquet modular encryption for Parquet data files. "AES GCM Stream" for Avro files, manifests and manifest lists. An encrypted Parquet file starts with `PARE`, and an encrypted manifest with `AGS1`.
- **Rotation:** the KEK is used for a period that NIST SP 800-57 permits, then replaced. Old KEKs stay for old snapshots.
- **The unit of the master key is the table,** set by the table property `encryption.key-id`. There is no tenant concept.
- **Duties of the catalog:** it must not let `encryption.key-id` change, and it must protect the table metadata against tampering.
- **Support:** Hive and REST catalogs, Parquet and Avro data. KMS clients for AWS, GCP and Azure.
- **Open point** [search]: implementers of iceberg-rust reported that some formats needed for interoperability are "implementation-specific" in the specification. A pull request proposes a standard binary format for `key_metadata`. I did not confirm whether it merged.

The property that matters for mudcat: a wrapped key sits in the **reference** to a file, not in the file. One KMS call opens a whole tree.

## 3. Snowflake

Source: Snowflake documentation on encryption key management [search].

- **Four levels:** a root key in a hardware security module, account master keys, table master keys and file keys. Each account has its own account master key. Each file (a micro-partition, which is immutable) has its own key.
- **Rotation and rekeying are separate.** Rotation gives new data a new key and retires the old key for encryption. Rekeying re-encrypts old data, when the retired key is older than one year, as an option.
- **Tri-Secret Secure:** a key that the customer manages is combined with a key that Snowflake manages into a composite master key. The customer can remove access to its key.

## 4. FoundationDB

Sources: `design/encryption-data-at-rest.md` and the command-line reference in the apple/foundationdb repository, branches release-7.3, release-7.4 and main [paper].

- The design note (written by Snowflake) proposes encryption at rest in which "the encryption domain matches with tenant partition semantics". A mode `domain_aware` gives per-tenant keys, and `cluster_aware` gives one key.
- The proxy encrypts mutations **before** they go to the transaction log, with a key that it derives for the tenant of the request. So the log is covered. The Redwood storage engine encrypts each tree node. Backup files are encrypted by key range. Each encrypted unit has a header that names its key.
- Processes cache keys in memory for a short time (10 minutes in the design note).
- Only the Redwood engine is supported. The property is set at the creation of the database and cannot change.
- **Status:** the command-line reference of 7.3 and 7.4 lists `encryption_at_rest_mode={aes_256_ctr|disabled}`, and tenants as `optional_experimental` and `required_experimental`, with a warning that tenants "are not recommended for use in production". On the main branch, the design note and the tenant page are absent, and the command-line reference names neither option. This suggests that the functions were removed, but I did not read a statement that says so.

The design is the closest published one to SalesforceDB for a key-value store, and the only one in these notes that states how the log is handled. It is not something to depend on.

## 5. PostgreSQL

Source: Percona documentation for `pg_tde` [search].

- PostgreSQL has no transparent data encryption in its core [memory].
- `pg_tde` has one principal key per **database**. The internal keys of that database are wrapped by it. A default key covers databases with no key of their own.
- WAL encryption is one setting and one key for the whole cluster. It is not per database.
- `pg_tde` needs Percona's distribution of PostgreSQL [memory].
- Managed PostgreSQL services take one KMS key per instance or cluster [memory].

Consequence: PostgreSQL can give a tenant its own key only when the tenant has its own database, and then the WAL is still under one key for all tenants of the cluster. Shared tables cannot have per-tenant keys at the page level, for the reason that the Salesforce paper gives: pages are mutable, and a page holds rows of several tenants.

## 6. What this means for mudcat

mudcat already has property 1 (the tenant ID leads every key, requirement ID-12). It has properties 2 and 3 only for the parts that are immutable files owned by one tenant: manifests, element lists, annotation sets, statistics, stream segments, exports, and data files on managed storage. For those, the Salesforce mechanism transfers almost unchanged, and the Iceberg key tree shows where to keep the wrapped keys.

The catalog rows in PostgreSQL do not have properties 2 and 3. That is the open question. The options, and a recommendation, are in `docs/tenant-encryption.md` of the mudcat repository.

## Sources

- Salesforce Database paper: PVLDB 19, pages 4426–4439 (local copy read in full for the sections named).
- https://github.com/apache/iceberg/blob/main/docs/docs/encryption.md
- https://github.com/apple/foundationdb/blob/release-7.4/design/encryption-data-at-rest.md
- https://github.com/apple/foundationdb/wiki/Encryption-data-at-rest
- https://docs.snowflake.com/en/user-guide/security-encryption-manage
- https://docs.percona.com/pg-tde/architecture/index.html
- https://docs.percona.com/pg-tde/wal-encryption.html
