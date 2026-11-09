# Migrate from Kuzu to LadybugDB

LadybugDB is the continuation of Kuzu. The migration involves updating package imports and connection code.

## Package Changes

| Language | Old Package | New Package |
|----------|-------------|-------------|
| Python | `kuzu` | `ladybug` |
| Node.js | `kuzu` | `@ladybugdb/core` |
| Go | `github.com/kuzudb/go-kuzu` | `github.com/LadybugDB/go-ladybug` |
| Rust | `kuzu` | `lbug` |

## Python Migration

### Installation

```bash
# Old
pip install kuzu

# New
pip install ladybug
```

### Code Changes

```python
# Old (kuzu)
import kuzu

db = kuzu.Database("myDB")
conn = kuzu.Connection(db)
result = conn.execute("MATCH (n) RETURN n LIMIT 5")
```

```python
# New (ladybug)
import ladybug as lb

db = lb.Database("myDB")
conn = lb.Connection(db)
result = conn.execute("MATCH (n) RETURN n LIMIT 5")
```

## kuzu and ladybug in One Process

`kuzu` and `ladybug` register the same pybind11 types, so they cannot be used in one Python process, in either order. Whichever native module loads second fails. With `kuzu` imported first, ladybug's first call fails with:

```
RuntimeError: Could not find lbug C API shared library. Set LBUG_C_API_LIB_PATH or download a shared lib (...). The bundled _lbug extension, which is tried first, failed to load: generic_type: type "Database" is already registered!
```

With ladybug loaded first, `import kuzu` fails with `ImportError: generic_type: type "Database" is already registered!`.

The cause is the end of the message. Do not follow its hint to download a shared library or set `LBUG_C_API_LIB_PATH`, and do not patch pybind11's type registry: neither fixes the clash.

- Migrate every module in one change. The Python API is the same, so this is usually only the import and the `kuzu.` prefix.
- If some code has to stay on `kuzu` (a dependency still requires it, or you were asked to leave a module on Kuzu for now), run that code in its own process and never import `kuzu` in the process that uses `ladybug`. A small worker that prints JSON is enough:

```python
# kuzu_worker.py: run as a script; the only entry point that imports kuzu code
import json, sys
import legacy  # your module that still uses kuzu

print(json.dumps(legacy.query(sys.argv[1])))
```

```python
# in the ladybug process
import json, subprocess, sys

out = subprocess.run([sys.executable, "kuzu_worker.py", "some-arg"],
                     capture_output=True, text=True, check=True).stdout
result = json.loads(out)
```

## Database Files

LadybugDB upgrades its own database files in place (since v0.17.0); once a file is upgraded, an older LadybugDB version cannot open it.

Kuzu database files are not opened directly: ladybug 0.22.3 refuses a file written by kuzu 0.11.3 with "not a valid Lbug database file". Move the data with EXPORT/IMPORT, each step in its own process (see above):

```python
# 1. With kuzu: export
import kuzu
conn = kuzu.Connection(kuzu.Database("old.kuzu", read_only=True))
conn.execute("EXPORT DATABASE '/path/to/export'")
```

```python
# 2. With ladybug, in a separate process: import into a new database
import ladybug as lb
conn = lb.Connection(lb.Database("new.lbdb"))
conn.execute("IMPORT DATABASE '/path/to/export'")
```

Point the code at the new path and keep the old `.kuzu` file as a backup until you have checked the new database (for example, `count(*)` per node and relationship table in both). A database that is rebuilt from source files (CSV loaders, fixtures) needs no export: run the loader on ladybug.

## Connection String Changes

```python
# Embedded (unchanged)
db = lb.Database("myDB")

# In-memory (unchanged)
db = lb.Database(":memory:")

# Remote API server
db = lb.Database("http://localhost:8123")
```

## Notes

- KuzuDB is now archived; LadybugDB continues development
- Some extension names may differ between versions
- Check LadybugDB documentation for new features like object storage support

## Reference

- [Cypher Queries](../reference/cypher.md)
- [DDL](../reference/ddl.md)
- [Data Import/Export](../reference/foreign.md)
- [Data Science](../reference/data-science.md)
