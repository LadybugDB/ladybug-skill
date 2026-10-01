# Data Definition Language (DDL)

## Query Icebug-Disk Format Parquet Files

Ladybug supports querying Icebug/Iceberg-format Parquet files directly from disk without importing them into the database.

### Example Dataset

The demo dataset at https://github.com/LadybugDB/ladybug/tree/master/dataset/demo-db/icebug-disk contains:

- `demo_nodes_user.parquet` - User node data
- `demo_nodes_city.parquet` - City node data
- `demo_indices_follows.parquet` - Follows relationship indices
- `demo_indptr_follows.parquet` - Follows indirection pointers
- `demo_indices_livesin.parquet` - LivesIn relationship indices
- `demo_indptr_livesin.parquet` - LivesIn indirection pointers
- `demo_mapping_user.parquet` - User ID mapping
- `demo_mapping_city.parquet` - City ID mapping
- `demo_metadata.parquet` - Metadata

### Schema Definition

```cypher
CREATE NODE TABLE city(id INT32, name STRING, population INT64, PRIMARY KEY(id)) WITH (storage = 'demo');
CREATE NODE TABLE user(id INT32, name STRING, age INT64, PRIMARY KEY(id)) WITH (storage = 'demo');
CREATE REL TABLE follows(FROM user TO user, since INT32) WITH (storage = 'demo');
CREATE REL TABLE livesin(FROM user TO city) WITH (storage = 'demo');
```

The `storage` parameter references the Icebug catalog name. In this case, `'demo'` refers to the Icebug metadata file at `demo_metadata.parquet`.

### Full Example

```cypher
-- Create node tables with Icebug-disk storage
CREATE NODE TABLE city(id INT32, name STRING, population INT64, PRIMARY KEY(id)) WITH (storage = 'demo');
CREATE NODE TABLE user(id INT32, name STRING, age INT64, PRIMARY KEY(id)) WITH (storage = 'demo');

-- Create relationship tables with Icebug-disk storage
CREATE REL TABLE follows(FROM user TO user, since INT32) WITH (storage = 'demo');
CREATE REL TABLE livesin(FROM user TO city) WITH (storage = 'demo');

-- Query the data
MATCH (u:user)-[:follows]->(f:user) RETURN u.name, f.name LIMIT 5;
MATCH (u:user)-[:livesin]->(c:city) RETURN u.name, c.name;
```

### Using with Local Files

To use with local Icebug-disk format files, reference the directory path:

```cypher
CREATE NODE TABLE my_node (id INT64, name STRING, PRIMARY KEY(id)) 
WITH (storage = '/path/to/icebug-disk/');
```

### Notes

- The `WITH (storage = '...')` clause connects the table to external Parquet files
- No data is copied into Ladybug - queries run directly against the Parquet files
- Works with Icebug format datasets that have proper metadata
- Useful for large datasets where importing would be impractical
- Icebug-disk export/compat targets **Icebug 13.2** with **Ladybug 0.21.2**

## Query Arrow Memory

Query Arrow memory directly by registering it with the database and using the `arrow://` storage prefix:

### Python Example

Use the `create_arrow_table()` helper function:

```python
import ladybug as lb
import pyarrow as pa

db = lb.Database(":memory:")
conn = lb.Connection(db)

# Register an Arrow table
table = pa.table({
    "city": ["New York", "Los Angeles", "Chicago"],
    "population": [8419000, 3980000, 2716000],
})
conn.create_arrow_table("cities", table)

# Query like a regular table
result = conn.execute(
    "MATCH (n:cities) WHERE n.population > 3000000 RETURN n.city ORDER BY n.population"
)
```

### C++ Example

Use explicit arrow IDs from registration:

```cpp
#include <ladybug/ladybug.h>

int main() {
    ladybug::Database db(":memory:");
    ladybug::Connection conn(db);

    // Create and register Arrow table
    auto arrow_id = conn.registerArrowTable("employees", arrow_table);

    // Create table with explicit arrow ID
    conn.execute(
        "CREATE NODE TABLE Employee (id INT64, name STRING, salary DOUBLE, PRIMARY KEY(id)) "
        "WITH (storage='arrow://" + arrow_id + "')"
    );

    // Query the table
    auto result = conn.execute("MATCH (e:Employee) WHERE e.salary > 50000 RETURN e.name");
}
```

### Notes

- Python: Use `conn.create_arrow_table()` helper for convenience
- C++/other: Use `conn.registerArrowTable()` to get explicit arrow ID
- The `arrow://` prefix references the registered table name/ID
- Data remains in Arrow memory - no copying occurs
