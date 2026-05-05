# XTDB × adbc-drivers/validation

Wiring for the upstream [adbc-drivers/validation](https://github.com/adbc-drivers/validation)
conformance suite, pointed at XTDB's Flight SQL server.

## Layout

```
validation/
├── xtdb.py                  # XtdbQuirks — feature matrix + SQL dialect overrides
├── tests/
│   ├── conftest.py          # driver + driver_path fixtures; re-exports suite fixtures
│   ├── test_connection.py   # thin wrapper around suite's connection tests
│   ├── test_query.py        # thin wrapper around suite's query tests
│   ├── test_ingest.py       # thin wrapper around suite's ingest tests
│   └── test_statement.py    # thin wrapper around suite's statement tests
└── queries/
    └── type/select/
        └── int32.txtcase    # XTDB-specific override demonstrating the pattern
```

## Running

```bash
# With venv + XTDB live on grpc://localhost:9833
XTDB_HOST=localhost \
XTDB_FLIGHT_SQL_URI=grpc://localhost:9833 \
  venv/bin/python -m pytest validation/tests/ -q --tb=no
```

## Current status (XTDB nightly w/ xtdb/xtdb#5526 + #5554, validation suite HEAD)

| bucket | count |
|---|---|
| passed | 100 |
| skipped (declared-unsupported feature) | 80 |
| failed | 47 |
| errored (setup) | 14 |

Up from 16 / 77 / 49 / 13 once `statement_prepare` and
`statement_get_parameter_schema` flip on (see `xtdb.py`). The +84-pass jump
is mostly downstream of `prepare()` — the suite's `try_drop_table` quirk
prepares under the hood, so any test touching it errored before #5526 made
prepare a real operation.

### Failure categories

1. **`type/bind/*` — parameter-binding type coverage (28 cases)** — single
   biggest cluster. Splits into suite-side feature gaps and server-side type
   coercion. Needs targeted probes to apportion blame.
2. **`type/select/*` — schema-strict comparisons (13 cases)** — `INT`
   columns declared as `int32` come back as `int64` because XTDB's planner
   promotes numeric literals. Either the override widens the expected schema
   to int64, or the server preserves declared widths. `type/literal/*` (2
   cases) is the same shape.
3. **`test_get_objects_column_*` — IPC encoding on column-depth GetObjects
   (12 errors + 1 failure)** — column-depth `GetObjects` returns malformed
   Arrow IPC. See [`adbc-bugs.md`](../../adbc-bugs.md) #1.
4. **`test_parameter_execute` — multi-row params (1 failure)** — sending 4
   rows of bound parameters expects 4 rows of result; XTDB executes once
   against the first row. Server/FSQL-side; tracked on xtdb/xtdb#5132.
5. **`test_execute_schema_noalias` — `adbc_execute_schema` (1 error)** —
   not implemented; declared-unsupported in `xtdb.py`.

### Paths to reduce the failure count

- Fast win: override a handful of `type/select/*.txtcase` to use
  `INSERT RECORDS` + widen expected schemas. This bumps the pass rate without
  server changes.
- Server-side: fix the column-depth `GetObjects` IPC encoding (one cluster
  unblocks 13 cases) and tighten declared-width preservation. Both tracked
  under xtdb/xtdb#5132.
- Server-side: implement the multi-row params expansion path so a 4-row
  bound batch produces 4 result rows.

## Not submoduled

The validation suite is pip-installed editable from a local clone
(`pip install --editable /tmp/adbc-validation` in setup). For CI or
`requirements.txt`, prefer the git URL form once upstream publishes releases:

```
adbc-drivers-validation @ git+https://github.com/adbc-drivers/validation.git
```

See the top-level README for why we don't submodule.
