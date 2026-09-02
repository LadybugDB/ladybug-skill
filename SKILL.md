---
name: ladybug-skill
description: >
  Use this skill when writing code to talk to LadybugDB
---

# LadybugDB Shell

LadybugDB is a high-performance embedded graph database that speaks the ladybug dialect of Cypher. This doc covers the interactive **lbug shell** (CLI).

For language-specific interfaces, see:
- [Python Interface](reference/python-interface.md)
- [JavaScript Interface](reference/javascript-interface.md)
- [Rust Interface](reference/rust-interface.md)
- [Migrating from KuzuDB](reference/migration-from-kuzu.md)

## Quick Start

```bash
# Start the interactive shell on an existing database
lbug my_database.lbdb

# Start with an in-memory database (data lost on exit)
lbug :memory:
```

## Shell Meta-Commands

These commands run inside the shell to inspect the database schema and metadata:

### `show_tables()`

List all tables (node tables, relationship tables, and foreign tables) in the current database:

```cypher
CALL show_tables() RETURN *; // correct syntax
show_tables(); // Abbreviated version. CALL and RETURN * are implied, but syntactically incorrect
```

Output:

| name      | type   |
|-----------|--------|
| Person    | NODE   |
| City      | NODE   |
| Follows   | REL    |
| LivesIn   | REL    |
| ts.person | NODE   |← foreign (DuckDB)

### `show_indexes()`

List all indexes defined on node table properties:

```cypher
show_indexes();
```

Output:

| table_name | index_name   | property | index_type |
|------------|-------------|----------|------------|
| Person     | pk_Person   | id       | PRIMARY KEY |
| City       | pk_City     | id       | PRIMARY KEY |

### `table_info('table_name')`

Show schema details for a specific table, including column names, types, and primary key:

```cypher
table_info('Person');
```

Output:

| property_name | type   | primary_key |
|---------------|--------|-------------|
| id            | INT64  | true        |
| name          | STRING | false       |
| age           | INT64  | false       |

### `show_functions()`

List all built-in and user-defined functions available in the query engine:

```cypher
show_functions();
```

Output:

| name       | type     |
|------------|----------|
| COUNT      | AGGREGATE |
| SUM        | AGGREGATE |
| AVG        | AGGREGATE |
| MIN        | AGGREGATE |
| MAX        | AGGREGATE |
| length     | SCALAR   |
| concat     | SCALAR   |
| timestamp  | SCALAR   |
| node_uri   | SCALAR   |
| ...        | ...      |

### `show_macros()`

List all user-defined macros (parametrized expressions defined with `CREATE MACRO`):

```cypher
show_macros();
```

Output:

| name   | parameters | expression |
|--------|-----------|------------|
| add    | (x, y)    | x + y      |
| is_adult| (age)    | age >= 18  |

## Running Queries

Enter any Cypher statement directly:

```cypher
CREATE NODE TABLE Person (id INT64, name STRING, age INT64, PRIMARY KEY (id));
CREATE (p:Person {id: 1, name: 'Alice', age: 30});
MATCH (p:Person) RETURN p.name, p.age;
```

## Configuration Settings

All `CALL`-settable session settings (threads, timeout, checkpointing, spill behavior, plan/optimizer knobs) are documented in [Configuration Settings](reference/config.md).

## Shell Options

| Flag | Description |
|------|-------------|
| `:memory:` | Run against an in-memory database |
| `<path>` | Open or create a persistent database at the given path |

---

## Attaching Foreign Tables

LadybugDB can attach external databases as foreign tables and query them with standard Cypher.

### DuckDB

```cypher
install duckdb;
load duckdb;

ATTACH '/path/to/database.db' AS ts
    (dbtype duckdb, skip_unsupported_table = true);
```

Query using the alias prefix:

```cypher
MATCH (a:ts.person) WHERE a.age > 30 RETURN count(*);
```

Create relationship tables referencing DuckDB tables:

```cypher
CREATE REL TABLE knows_rel (FROM ts.person TO ts.person)
    WITH (storage = 'ts.knows');

MATCH (a:ts.person)-[b:knows_rel]->(c:ts.person) RETURN count(*);
```

### PostgreSQL (v0.19.0+)

Use the native `pg_client` extension (preferred over the DuckDB-based postgres extension). Both libpq connection strings and URLs are accepted:

```cypher
install pg_client;
load pg_client;

-- Libpq connection string
ATTACH 'host=localhost port=5432 dbname=mydb user=user password=pass' AS pg
    (dbtype pg_client);

-- URL format (also accepted)
ATTACH 'postgresql://user:pass@localhost:5432/mydb' AS pg
    (dbtype pg_client);
```

### Detach

```cypher
DETACH ts;
```

---

## Attaching LadybugDB Databases

You can attach one LadybugDB database to another, allowing cross-database queries.

Attach a LadybugDB database file:

```cypher
ATTACH '/path/to/other_database.lbdb' AS other
    (dbtype ladybug);
```

Query foreign node/rel tables using the alias:

```cypher
MATCH (n:other.Person) RETURN n.name;
CREATE REL TABLE Links (FROM Person TO other.Product)
    WITH (storage = 'other.knows');
```

Detach when finished:

```cypher
DETACH other;
```

---

## Further Reading

- [Cypher Query Reference](reference/cypher.md) — Cypher extensions for JSON, open-type graphs, Parquet on disk, Arrow memory
- [DDL Reference](reference/ddl.md) — Icebug/Iceberg format Parquet, table creation with storage options
- [Foreign Data Wrappers](reference/foreign.md) — DuckDB and other foreign database attachment details
- [Configuration Settings](reference/config.md) — all `CALL <setting> = <value>` session settings with types and defaults
- [Data Science](reference/data-science.md) — Icebug (networkit fork), Arrow-native graph algorithms
- [Python Interface](reference/python-interface.md) — LadybugDB Python SDK
- [JavaScript Interface](reference/javascript-interface.md) — LadybugDB Node.js SDK
- [Rust Interface](reference/rust-interface.md) — LadybugDB Rust SDK (`lbug` crate)
- [Migrating from KuzuDB](reference/migration-from-kuzu.md) — Package and code migration guide
