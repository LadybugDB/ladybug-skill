# Rust Interface

> **Note:** LadybugDB uses the `lbug` crate in Rust.

## Cargo.toml

```toml
[dependencies]
lbug = "0.21.2"
```

## Quick Start

```rust
use lbug::Database;
use lbug::Connection;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create an in-memory database
    let db = Database::new(":memory:")?;
    let conn = Connection::new(&db)?;

    // Create a node table
    conn.execute("CREATE NODE TABLE Person (id INT64, name STRING, age INT64, PRIMARY KEY (id))")?;

    // Insert data
    conn.execute("CREATE (p:Person {id: 1, name: 'Alice', age: 30})")?;
    conn.execute("CREATE (p:Person {id: 2, name: 'Bob', age: 25})")?;

    // Query
    let result = conn.execute("MATCH (p:Person) WHERE p.age > 20 RETURN p.name, p.age")?;

    // Iterate rows
    while result.has_next() {
        let row = result.get_next()?;
        println!("{}: {}", row.get_value(0)?, row.get_value(1)?);
    }

    Ok(())
}
```

## Query Results

```rust
let result = conn.execute("MATCH (p:Person) RETURN p.name, p.age")?;

// Check number of columns
let num_cols = result.get_num_columns();

// Iterate
while result.has_next() {
    let row = result.get_next()?;
    let name: String = row.get_value(0)?;
    let age: i64 = row.get_value(1)?;
    println!("{name}: {age}");
}
```

## Arrow Integration

Register Arrow tables via `registerArrowTable`:

```rust
let arrow_id = conn.registerArrowTable("employees", arrow_table)?;

conn.execute(
    "CREATE NODE TABLE Employee (id INT64, name STRING, salary DOUBLE, PRIMARY KEY(id)) "
    "WITH (storage='arrow://{arrow_id}')"
)?;
```

## Shell Commands in Rust

```rust
// List tables
let result = conn.execute("CALL show_tables()")?;

// List indexes
let result = conn.execute("CALL show_indexes()")?;

// Table info
let result = conn.execute("CALL table_info('Person')")?;

// List functions
let result = conn.execute("CALL show_functions()")?;

// List macros
let result = conn.execute("CALL show_macros()")?;
```

## Remote Connection

```rust
let db = Database::new("http://localhost:8123")?;
let conn = Connection::new(&db)?;
```

## Further Reading

- [lbug Shell](../SKILL.md) — shell meta-commands and attach operations
- [Python Interface](python-interface.md)
- [JavaScript Interface](javascript-interface.md)
- [Cypher Reference](cypher.md)
