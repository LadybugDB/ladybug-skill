# Configuration Settings

Client/session settings are read and written via `CALL`:

```cypher
CALL current_setting('enable_plan_optimizer') RETURN *;   // read
CALL enable_plan_optimizer=false;                          // write
```

Complete list of settings (settable with `CALL <setting> = <value>;`):

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `threads` | UINT64 | all cores | Number of threads for query execution (0 = use all). |
| `timeout` | UINT64 | `0` | Query timeout in milliseconds; `0` disables. |
| `warning_limit` | UINT64 | `8192` | Maximum number of cached warnings retained. |
| `progress_bar` | BOOL | `false` | Show a progress bar for long-running queries. |
| `var_length_extend_max_depth` | INT64 | `30` | Maximum depth for variable-length extends (`[:R*1..n]`). |
| `sparse_frontier_threshold` | INT64 | `1000` | Frontier size at which BFS switches from sparse to dense frontier. |
| `enable_semi_mask` | BOOL | `true` | Use semi-mask reduction in joins. |
| `enable_zone_map` | BOOL | `true` | Use zone maps (min/max pruning) during scans. |
| `disable_map_key_check` | BOOL | `true` | Skip map key checks during binding. |
| `recursive_pattern_semantic` | STRING | `'WALK'` | Semantic for recursive patterns: `WALK`, `TRAIL`, or `ACYCLIC`. |
| `recursive_pattern_factor` | INT64 | `100` | Cardinality scale factor for recursive pattern estimation. |
| `home_directory` | STRING | (system) | Home directory for the database. |
| `file_search_path` | STRING | (empty) | Search path for file inputs (CSV/Parquet loads). |
| `checkpoint_threshold` | INT64 | `16777216` (16 MB) | WAL size threshold that triggers auto-checkpointing. |
| `auto_checkpoint` | BOOL | `true` | Checkpoint automatically when the threshold is hit. |
| `force_checkpoint_on_close` | BOOL | `true` | Force a checkpoint when the database is closed. |
| `spill_to_disk` | BOOL | DB-dependent | Allow operators (sort/hash join) to spill intermediate results to disk. Cannot be enabled for in-memory or read-only databases. |
| `pk_validator_spill_threshold` | UINT64 | `8589934592` (8 GB) | Memory budget (bytes) for the primary-key uniqueness buffer during `COPY` into a PK table with no hash index; beyond it, sorted runs spill to disk. `0` disables spilling (may OOM). |
| `enable_plan_optimizer` | BOOL | `true` | Enable the cost-based plan optimizer. |
| `enable_internal_catalog` | BOOL | `false` | Use the internal catalog during binding. |
| `enable_packed_path_extend` | BOOL | `false` | See below. |
| `enable_cached_prepared_statement` | STRING | `'BOTH'` | See below. |
| `debug_enable_multi_writes` | BOOL | `false` | See below. |
| `enable_default_hash_index` | BOOL | `true` | See below. |

### `enable_packed_path_extend` (BOOL, default `false`)

Enables a physical *packed extend* operator for rel scans: the scan reuses the
adjacency-list (CSR) structure directly to extend over multi-parent / nested
cases instead of materializing separate extend states. Useful for
variable-length / multi-parent path workloads. This is an optimizer
experimental path — if you hit lifetime or correctness issues with deep
multi-hop patterns, retry with it disabled.

### `enable_cached_prepared_statement` (STRING, default `'BOTH'`)

Controls which statement kinds may reuse a cached **physical plan** when a
parameterized prepared statement is re-executed. One of (case-insensitive):

| Value | Meaning |
|-------|---------|
| `READS` | only read statements populate/reuse the plan cache |
| `WRITES` | only write statements do |
| `BOTH` | both (default) |
| `NONE` | disables plan caching entirely |

It is a safety valve for latent plan-cache state-reuse bugs: if repeated
executions of a prepared statement return stale/incorrect results, set it to
`NONE` as a workaround.

```cypher
CALL enable_cached_prepared_statement='reads';
CALL current_setting('enable_cached_prepared_statement') RETURN *;  // READS
```

### `enable_default_hash_index` (BOOL, default `true`)

Database-level setting (settable per-session via `CALL`). If `true`, node tables
create the default primary-key **hash index** automatically. Set to `false` to
skip creating hash indexes (e.g. when using `ART` primary-key indexes or bulk
loading where the index is built later). Changes propagate to the storage
managers of the current context.

```cypher
CALL enable_default_hash_index=false;
CALL current_setting('enable_default_hash_index') RETURN *;
```

### `debug_enable_multi_writes` (BOOL, default `false`)

Enables multiple **concurrent write transactions** (MVCC). This is Ladybug's
name for what Kùzu called `enable_concurrent_writes`. With it disabled (the
default) only one write transaction may run at a time; enabling it allows
concurrent writers with optimistic concurrency control. The `debug_` prefix
signals it is still under validation — enable it for concurrency testing or
when concurrent writers are required.

```cypher
CALL debug_enable_multi_writes=true;
```
