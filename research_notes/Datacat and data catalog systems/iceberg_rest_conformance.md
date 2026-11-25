# Iceberg REST catalog: conformance testing

Reading notes, November 2026. Sources: the `open-api` module of `apache/iceberg` at `main` (README, `RESTCompatibilityKitSuite.java`, `RESTCompatibilityKitCatalogTests.java`, `RCKUtils.java`), `CatalogTests.java` and `ViewCatalogTests.java` in `core`, the REST OpenAPI document, and the test READMEs of Lakekeeper, Polaris and PyIceberg.

## The REST Compatibility Kit (RCK)

The README describes it as "a Technology Compatibility Kit (TCK) implementation for the Iceberg REST Specification. This includes a series of tests based on the Java reference implementation of the REST Catalog that can be executed against any REST server that implements the spec." It was added by pull request apache/iceberg #10908.

**Structure.**

- `RESTCompatibilityKitSuite` is a JUnit suite that selects two classes.
- `RESTCompatibilityKitCatalogTests extends CatalogTests<RESTCatalog>`: namespace and table tests. `CatalogTests` has 106 test methods at the time of reading.
- `RESTCompatibilityKitViewCatalogTests`: view tests. `ViewCatalogTests` has 52 test methods.
- `RCKUtils` builds the REST client from configuration, can start a local server, and purges test namespaces and tables at start (`rck.purge-test-namespaces`).

**What the table tests cover** (from the method names): create, load, list, update and drop of namespaces, with properties, nested namespaces and names with slashes or dots; basic and complete table creation; default and overridden table properties; default column values and variant columns; load of missing tables and metadata tables; rename, including missing source and existing destination; drop with and without purge; schema updates with server-side retry, conflict and assignment conflict; partition spec and sort order updates with the same cases; removal of unused specs and schemas; append, and concurrent appends to empty and non-empty tables; update, create, replace and create-or-replace transactions, including concurrent ones with schema, spec and sort-order conflicts; removal of old metadata files after commit; register and unregister table; a custom metrics reporter.

All transactions in these tests are on one table. The multi-table commit endpoint is not exercised by `CatalogTests`.

**Configuration.** Environment variables with the prefix `CATALOG_` (prefix stripped, `_` becomes `.`, `__` becomes `-`, lower-cased) or Java system properties with the prefix `rck.`:

```
rck.uri=https://my_rest_server.io/
rck.warehouse=test_warehouse
rck.io-impl=org.apache.iceberg.aws.s3.S3FileIO
rck.credential=<oauth_key>:<oauth_secret>
```

**Behaviour switches,** because "not all behaviors are strictly defined by the REST Specification":

| Option | Default |
|---|---|
| `rck.requires-namespace-create` | true |
| `rck.supports-serverside-retry` | true |
| `rck.overrides-requested-location` | false |
| `rck.supports-names-with-dot` | false |
| `rck.supports-variant` | false |

In the test class, names with slashes are fixed to unsupported and unregister is fixed to supported.

**Running it.**

```
./gradlew :iceberg-open-api:test --tests RESTCompatibilityKitSuite \
  -Drck.local=false \
  -Drck.requires-namespace-create=true \
  -Drck.uri=https://my_rest_server.io/ \
  -Drck.warehouse=test_warehouse \
  -Drck.credential=<oauth_key>:<oauth_secret>
```

"The default behavior is to run a local http server with a jdbc backend for testing purposes, so `-Drck.local=false` must be set to point to an external REST server."

## What "server-side retry" means

The kit's `...ServerSideRetry` tests make two concurrent changes whose requirements do not contradict each other and expect both to succeed without a client retry. In the Java reference server this is a loop: on a failed pointer swap the server reloads the table, re-validates the commit's requirements against the new base and re-applies the updates. The requirements, not the base the client saw, decide whether there is a conflict.

## Endpoint discovery

The configuration response has an optional `endpoints` field: "A list of endpoints that the server supports. The format of each endpoint must be `<HTTP verb> <resource path from OpenAPI REST spec>`." If a server does not send it, a default set is assumed, which includes namespaces, tables, register, metrics, rename and `POST /v1/{prefix}/transactions/commit`. A server with a reduced scope must therefore send the field explicitly.

The same schema has an idempotency setting for mutation endpoints: "If absent, clients MUST assume idempotency is not supported."

## How other catalogs test

- **Lakekeeper** has its own integration harness (`tests/run.sh`) that runs the catalog against Spark (several Iceberg versions and storage back ends: S3 with STS, S3 remote signing, s3a, ADLS, WASBS, OneLake), PyIceberg, StarRocks and Trino, including variants with OpenFGA and Open Policy Agent authorization.
- **Polaris** has an `integration-tests` module of "black box tests that can be run against a local or remote Polaris instance".
- **PyIceberg** tests itself against the `apache/iceberg-rest-fixture` container image, the reference REST server, together with a Spark container.

No evidence was found in these files that Lakekeeper or Polaris run the RCK itself.

## Consequences for mudcat

- The kit cannot run against a read-only catalog: every test creates tables and commits to them through a FileIO, so it needs a writable gateway, a warehouse location and credentials.
- Views can be left out by running only `RESTCompatibilityKitCatalogTests` and by not listing view endpoints in the configuration response.
- The kit purges namespaces and tables, so it needs a tenant of its own.
- The Iceberg specification makes the catalog write the new table metadata file at commit, which is the one object-storage write in a mudcat commit.
- The kit uses one client and tests no access control, policy, tenancy or lineage, so engine tests (PyIceberg, Spark, Trino, DuckDB) and OpenAPI schema validation are needed beside it.
- The kit changes with each Iceberg release; the release to pass must be pinned.
