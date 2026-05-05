# XTDB ADBC / Flight SQL — Known Issues

Running log of issues discovered while exercising XTDB's ADBC/FlightSQL surface
from non-JVM clients. Updated as new issues land or resolve.

**Tracking issue upstream:** [xtdb/xtdb#5132](https://github.com/xtdb/xtdb/issues/5132)

## Environment

| Item | Value |
|---|---|
| XTDB image | `ghcr.io/xtdb/xtdb-aws:edge` |
| Tested build | nightly `75472e4` (2026-04-14 rebuild) |
| FlightSQL endpoint | `grpc://xtdb:9833` |
| Python ADBC driver | `adbc-driver-flightsql v1.11.0` |
| Go ADBC driver | `github.com/apache/arrow-adbc/go/adbc/driver/flightsql` |

---

## Open

### 1. Malformed Arrow IPC on column-schema responses

**Symptom.** Both `GetTableSchema` and `GetObjects(depth=all)` (the call paths
that emit per-column schema information) return corrupt IPC bytes:

```
arrow/ipc: could not read message schema: could not read continuation indicator: EOF
```

**Reproducer (Python):**

```python
import adbc_driver_flightsql.dbapi as f
c = f.connect("grpc://xtdb:9833")
cur = c.cursor()
cur.executemany("INSERT INTO t (_id, n) VALUES (?, ?)", [(1, 42)])
cur.close()

# Both raise the IPC error:
c.adbc_get_table_schema("t", db_schema_filter="public")
c.adbc_get_objects(depth="all", table_name_filter="t").read_all()
```

**Impact.** Any ADBC client calling `GetTableSchema` or enumerating columns
via `GetObjects` gets a hard error. BI/notebook tools that introspect schemas
before querying (e.g. DBeaver, DataGrip, Trino-style metadata scans) will fail.

**Test coverage.** `python/tests/test_adbc_metadata.py` has two `xfail`-marked
cases (`test_returns_arrow_schema`, `test_all_depth_returns_columns`) that flip
to XPASS when fixed.

**Suspect.** Same IPC encoding path shared by both endpoints. Other shallower
`GetObjects` depths (`catalogs`, `db_schemas`, `tables`) serialize fine.

---

### 2. Literal DML via `cursor.execute()` returns `INTERNAL` instead of a useful error

**Symptom.** DML statements (INSERT/UPDATE/DELETE/ERASE) passed to
`cursor.execute()` hit the FlightSQL DoGet (query) path and fail:

```
INTERNAL: [FlightSQL] There was an error servicing your request. (Internal; ExecuteQuery). Vendor code: 13
```

**Correct client usage.** Python ADBC exposes `cursor.executescript()` for the
update path (as discussed on xtdb/xtdb#5082). Python's DB-API doesn't
distinguish execute-query vs execute-update, so `executescript()` is the
designated update route for literal DML.

```python
# Fails:
cur.execute("INSERT INTO products RECORDS {_id: 1, name: 'Widget'}")

# Works:
cur.executescript("INSERT INTO products RECORDS {_id: 1, name: 'Widget'}")

# Also works (parameterized path goes through DoPut automatically):
cur.executemany(
    "INSERT INTO products (_id, name) VALUES (?, ?)", [(1, "Widget")]
)
```

**Server-side ask.** The error message is useless. XTDB should either propagate
a structured message distinguishing "DML submitted via query endpoint" from
genuine internal faults, or accept the statement on DoGet and route it
internally. Tracked in xtdb/xtdb#5082.

**Test coverage.** `python/flight_sql_example.py` section `4b` exercises the
`executescript` path as the canonical shape.

---

### 3. Parser errors classified as `INTERNAL` instead of `INVALID_ARGUMENT`

**Symptom.** Syntactically-unrecognized SQL — e.g. `DROP TABLE IF EXISTS t`,
which XTDB has no parser rule for — comes back from ADBC as:

```
InternalError: INTERNAL: [FlightSQL] There was an error servicing your
  request. (Internal; Prepare). Vendor code: 13
```

The same error appears via both `cursor.adbc_prepare(sql)` and
`cursor.execute(sql)` (which prepares under the hood).

**Reproducer.**

```python
import adbc_driver_flightsql.dbapi as f
c = f.connect("grpc://xtdb:9833")
cur = c.cursor()
cur.adbc_prepare("SELECT 1")              # OK
cur.adbc_prepare("SELECT ? + 1")          # OK (binds supported)
cur.adbc_prepare("DROP TABLE IF EXISTS t")  # raises INTERNAL
```

**Impact.** The entire `adbc-drivers/validation` `test_get_objects_table_*`
setup path hits this because the suite's default `try_drop_table` quirk
issues `DROP TABLE`. Client error handling can't distinguish real server
faults from grammar mismatches.

**Server-side ask.** Classify parser errors as `INVALID_ARGUMENT` /
`UNIMPLEMENTED` rather than `INTERNAL`. Separately, either accept `DROP TABLE`
as a no-op (XTDB has no schemas) or surface a helpful "use ERASE instead"
message.

**Test coverage.** Visible in `python/validation/tests/` under the
`test_get_objects_*` errors.

**Note.** `AdbcStatement.prepare()` itself is now functional — see Resolved
below. This issue is specifically about how parser errors are *classified*
once prepare reaches them.

---

### 3b. FlightSQL bulk ingest (`ExecuteIngest`) not implemented

**Symptom.** `cursor.adbc_ingest(table, arrow_table, mode=…)` for any of
`create`, `append`, `replace`, `create_append`:

```
NotSupportedError: NOT_IMPLEMENTED: [FlightSQL] Not implemented.
  (Unimplemented; ExecuteIngest). Vendor code: 12
```

**Impact.** Loss of the headline Arrow ergonomic: loading a `pyarrow.Table`
directly into XTDB in one call. Clients have to manually shred the Arrow
table into rows and call `executemany("INSERT ... VALUES (?, …)")`, which
round-trips through parameterized INSERT and is much slower for wide tables.

**Reproducer.**

```python
import pyarrow as pa, adbc_driver_flightsql.dbapi as f
c = f.connect("grpc://xtdb:9833")
cur = c.cursor()
t = pa.table({"_id": [1, 2], "n": [10, 20]})
cur.adbc_ingest("ingest_probe", t, mode="create_append")  # raises
```

**Server-side ask.** Implement FlightSQL's `CommandStatementIngest` (added in
FlightSQL v13). Especially valuable for loading parquet/arrow snapshots into
XTDB for time-travel analysis.

**Test coverage.** `python/validation/tests/test_ingest.py` (via task b)
surfaces this already as `NotSupportedError`.

---

### 4. `adbc_get_info` NPEs in upstream `GetInfoMetadataReader` (Arrow client bug)

**Symptom.** Calling `getInfo()` through the FlightSQL ADBC client crashes
client-side before any data reaches the caller:

```
java.lang.NullPointerException: Cannot invoke "VectorLoader.load(...)" because "this.loader" is null
  at org.apache.arrow.vector.ipc.ArrowReader.loadRecordBatch(ArrowReader.java:213)
  at org.apache.arrow.adbc.driver.flightsql.BaseFlightReader.loadRoot(BaseFlightReader.java:149)
  at org.apache.arrow.adbc.driver.flightsql.GetInfoMetadataReader.loadNextBatch(GetInfoMetadataReader.java:173)
```

The same shape from Python (`adbc_driver_flightsql.dbapi.Connection.adbc_get_info`)
manifests as a hard ADBC error.

**Reproducer (Kotlin, against any XTDB nightly):**

```kotlin
val al = RootAllocator()
val db = FlightSqlDriver(al).open(mapOf("uri" to "grpc+tcp://127.0.0.1:9833"))
db.connect().use { conn ->
    conn.getInfo().use { rdr ->
        rdr.loadNextBatch()    // NPE
    }
}
```

**Diagnosis.** Not an XTDB-side bug. `XtdbProducer.getStreamSqlInfo` returns
the canonical FlightSQL `GET_SQL_INFO_SCHEMA` with two rows
(`FLIGHT_SQL_SERVER_NAME`, `FLIGHT_SQL_SERVER_VERSION`) — the wire response
verifies fine via raw FlightSQL (`FlightSqlClient.getSqlInfo` works in the
core test suite). The crash is in Arrow's
`adbc-driver-flightsql:GetInfoMetadataReader`, which allocates a fresh output
VSR but never initialises its `VectorLoader` before calling `loadRoot`.
First flagged in xtdb/xtdb #d6705d512a as
*"GetInfoMetadataReader.processRootFromStream allocates on the wrong root"*.

**Impact.** Any non-trivial ADBC client that probes `getInfo()` for vendor /
driver metadata gets a hard error. Most clients call this during connection
setup, so the failure can surface before the user runs a single query.
In-process JVM ADBC (`XtdbConnection.getInfo()`) is unaffected — that path
emits all four ADBC info codes correctly (xtdb/xtdb#5553).

**Server-side ask.** None (verify periodically — once Apache Arrow ADBC fixes
the loader-init path the wire route should start working without changes
here).

**Test coverage.** Captured indirectly via xtdb/xtdb#5553 — the closing
comment includes the full Kotlin repro.

---

## Resolved

### `AdbcStatement.prepare()` and `get_parameter_schema` not implemented

`prepare()` was a TODO on the in-process ADBC path and the FlightSQL
prepared-statement DoGet/DoPut/DoAction handlers had a parallel
implementation that didn't match. Both are now wired:

- xtdb/xtdb#5526 — in-process `XtdbStatement.prepare()` + bind/execute
  lifecycle.
- xtdb/xtdb#5554 — FSQL prepared-statement callbacks delegate to
  `XtdbStatement` rather than maintaining their own copy.

`statement_prepare` and `statement_get_parameter_schema` flip from `False`
to `True` in `validation/xtdb.py` once both land. Validation suite passes
jump 16 → 100 — most of the gain is downstream of `prepare()` since the
suite's `try_drop_table` quirk prepares under the hood, so any test
touching it errored before the fix.

The classification angle in #3 (parser errors → `INTERNAL`) is separate
and remains open.

### `adbc_current_catalog` / `adbc_current_db_schema` not exposed over FlightSQL

The Go-driver-based ADBC clients (Python / C / R) read `adbc_current_catalog`
and `adbc_current_db_schema` by querying FlightSQL session options under the
well-known keys `catalog` and `schema`. `XtdbProducer` didn't override
`getSessionOptions`, so the calls came back `UNIMPLEMENTED` and the driver
surfaced "current catalog not supported".

Wired up on branch `tim/adbc-session-options` — `XtdbProducer.getSessionOptions`
now reads `getCurrentCatalog()` / `getCurrentDbSchema()` from the per-database
default connection. Once it lands, `validation/xtdb.py` flips `current_catalog` /
`current_schema` from `None` to `"xtdb"` / `"public"` and the suite's
`test_current_catalog` actively asserts both.

The Java FlightSqlConnection ADBC client (0.23) doesn't query these (it
inherits the AdbcConnection default which throws `notImplemented`), so the
Java wire path stays unsupported until upstream wires it up. `setSessionOptions`
isn't implemented either — XtdbProducer has no per-session state, so accepting
a set would mutate the shared default connection's catalog visible to other
callers.

---

## Notes

- Non-JVM clients use the stock Apache ADBC FlightSQL driver (Go/C/Python/C#).
  There is no bespoke XTDB ADBC driver outside the JVM.
- In-process JVM ADBC and over-the-wire FlightSQL share most of the
  prepared-statement codepath now (xtdb/xtdb#5554). Residual ADBC items
  still tracked under issue #5132.
- Conformance target is
  [adbc-drivers/validation](https://github.com/adbc-drivers/validation). Wired
  into this repo under `python/validation/` — see
  [`python/validation/README.md`](python/validation/README.md) for the
  102 pass / 45 fail / 80 skip / 14 error breakdown and failure categories.
