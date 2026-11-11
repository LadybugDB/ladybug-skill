# Migrate from Kuzu to LadybugDB

LadybugDB is the continuation of Kuzu. The migration involves updating package imports and connection code.

## Before You Start

- The Python package is `ladybug` and `real_ladybug` is obsolete; it stopped at 0.15.3.
- Copy every `.kuzu` database file before changing anything, and keep the original until the new database is verified: row counts for every node and relationship table match the old database. Do not delete it as part of the migration.

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
uv pip install ladybug
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

## Database Files

LadybugDB upgrades its own database files in place (since v0.17.0); once a file is upgraded, an older LadybugDB version cannot open it.

Kuzu database files are not opened directly: ladybug 0.22.3 refuses a file written by kuzu 0.11.3 with "not a valid Lbug database file". Move the data with EXPORT/IMPORT, each step in its own process (`kuzu` and `ladybug` cannot be loaded in one Python process):

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
