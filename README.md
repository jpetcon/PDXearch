<h1 align="center">
  The PDXearch DuckDB Extension
</h1>
<h3 align="center">
  Fast vector similarity search for DuckDB
</h3>
<br>

## What is this?

**Vector similarity search** finds the rows in your database whose vector
(embedding) column is most similar to a given query vector. This is the core
operation behind semantic search, recommendation systems, RAG pipelines, and
image retrieval.

This extension adds a **PDXearch index** to [DuckDB](https://duckdb.org/) that
makes vector searches fast, even on tables with millions of rows. Without an
index, DuckDB computes distances to every row (exact but slow). With a PDXearch
index, only a subset of the data is scanned, returning approximate results in a
fraction of the time.

### How does it compare?

| | Exact scan (no index) | DuckDB VSS (HNSW) | **PDXearch** |
|---|---|---|---|
| Result quality | Exact | Approximate | Approximate |
| Search speed | Slow on large tables | Fast | Fast |
| Index build time | N/A | Slow | **Fast** |
| Memory usage | N/A | High | **Low** |
| Filtered search | N/A | Not supported | **Supported** |

PDXearch uses an **IVF (Inverted File Index)** approach: it groups similar
vectors into clusters at index creation time, then only searches the most
relevant clusters at query time. This is fundamentally different from the
graph-based (HNSW) approach used by the official VSS extension.

## Install

> [!NOTE]
> The extension is not yet available as a community extension. For now it must
> be built locally. See [DEVELOPMENT.md](DEVELOPMENT.md) for build instructions.

## Quick Start

### 1. Start DuckDB

```bash
duckdb -unsigned
```

The `-unsigned` flag is required to load locally built extensions.

### 2. Load the extension

```sql
LOAD '/path/to/PDXearch/build/release/extension/pdxearch/pdxearch.duckdb_extension';
```

### 3. Create a table with vector data

```sql
-- Create a table with an ID and a 128-dimensional embedding column
CREATE TABLE items (id INTEGER, embedding FLOAT[128]);

-- Insert 50,000 rows of sample data
INSERT INTO items
    SELECT i AS id, list_apply(range(128), x -> (i + x)::FLOAT)::FLOAT[128]
    FROM range(50000) t(i);
```

`FLOAT[128]` is DuckDB's fixed-size array type with 128 dimensions. Your
embeddings can have any number of dimensions (typically 128 to 1536, depending on
your embedding model).

### 4. Create the index

```sql
CREATE INDEX items_idx ON items USING PDXEARCH (embedding);
```

### 5. Search for similar vectors

Find the 10 items most similar to a query vector:

```sql
SELECT * FROM items
    ORDER BY array_distance(embedding, [1.0, 2.0, 3.0, ...]::FLOAT[128])
    LIMIT 10;
```

DuckDB's optimizer automatically detects this pattern (ORDER BY distance ...
LIMIT K) and uses the PDXearch index instead of scanning the full table.

### 6. Filtered search

You can add a WHERE clause and the index will still be used:

```sql
SELECT * FROM items
    WHERE id BETWEEN 1000 AND 5000
    ORDER BY array_distance(embedding, [1.0, 2.0, ...]::FLOAT[128])
    LIMIT 10;
```

> [!WARNING]
> For queries with small result limits (K <= 50), disable DuckDB's late
> materialization optimization first:
> ```sql
> SET late_materialization_max_rows = 0;
> ```
> This avoids a suboptimal query plan. This will be fixed in a future release.

## Distance Functions

The index supports three distance metrics. The metric determines which SQL
function triggers the index:

| Metric | SQL function | Operator | Meaning |
|--------|-------------|----------|---------|
| `l2sq` (default) | `array_distance(a, b)` | `a <-> b` | Euclidean distance. Lower = more similar. |
| `cosine` | `array_cosine_distance(a, b)` | `a <=> b` | Cosine distance (1 - cosine similarity). Lower = more similar. |
| `ip` | `array_negative_inner_product(a, b)` | — | Negative inner product. Lower = more similar. |

To create an index with a specific metric:

```sql
CREATE INDEX idx ON items USING PDXEARCH (embedding) WITH (metric = 'cosine');
```

## Index Parameters

```sql
CREATE INDEX idx ON t USING PDXEARCH (vec)
    WITH (metric = 'l2sq', quantization = 'u8', n_probe = 24, seed = 42);
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `metric` | string | `'l2sq'` | Distance metric: `'l2sq'`, `'cosine'`, or `'ip'`. See table above. |
| `quantization` | string | `'u8'` | How vectors are stored internally. `'f32'` keeps full 32-bit precision. `'u8'` compresses to 8-bit (less memory, slightly less accurate). |
| `n_probe` | integer | `24` | How many clusters to search at query time. Higher values give more accurate results but slower queries. Set to `0` to search all clusters (exact results). Range: 0–100,000. |
| `seed` | integer | random | Random seed for index construction. Set to a fixed value for reproducible builds. |

### Tuning accuracy vs speed

The `n_probe` parameter controls the accuracy-speed tradeoff:

- **Low n_probe** (e.g. 5–10): Faster queries, may miss some relevant results.
- **Default n_probe** (24): Good balance for most workloads.
- **High n_probe** (e.g. 100+): Slower queries, closer to exact results.
- **n_probe = 0**: Searches all clusters. Equivalent to exact search.

You can override `n_probe` at query time without rebuilding the index:

```sql
SET pdxearch_n_probe = 64;
```

## BLOB Functions

The extension provides utility functions for working with compact binary
representations of vectors. These are useful when you receive embeddings from
an external API as binary data, or when you want to store vectors more
compactly.

| Function | Description |
|----------|-------------|
| `pdxearch_encode_blob(list)` | Converts a `FLOAT[]` list to a compact BLOB (2 bytes per dimension). |
| `pdxearch_decode_blob(blob)` | Converts a BLOB back to a `FLOAT[]` list. |
| `pdxearch_blob_to_base64(blob)` | Converts a BLOB to a base64 string. |
| `pdxearch_base64_to_blob(str)` | Converts a base64 string back to a BLOB. |

BLOB vectors can be used directly in distance functions:

```sql
-- Compare a BLOB-encoded vector against a FLOAT array
SELECT array_distance(blob_column, [1.0, 2.0, 3.0]::FLOAT[3]) FROM my_table;
```

## Index Diagnostics

Inspect all PDXearch indexes in the database:

```sql
CALL pdxearch_index_info();
```

| Column | Description |
|--------|-------------|
| `index_name` | Name of the index. |
| `table_name` | Table the index is built on. |
| `metric` | Distance metric (`l2sq`, `cosine`, or `ip`). |
| `num_dimensions` | Number of dimensions in the indexed vectors. |
| `quantization` | Storage format (`f32` or `u8`). |
| `n_probe` | Default number of clusters probed during search. |
| `seed` | Random seed used during index construction. |
| `is_normalized` | Whether vectors are normalized internally (true for `cosine` and `ip`). |
| `approx_lower_bound_memory_usage_bytes` | Approximate memory used by the index. |
| `has_unindexed_data` | `true` if the table has been modified since the index was created. Search results may be incomplete. Rebuild the index to fix. |

When `has_unindexed_data` is `true`, drop and recreate the index:

```sql
DROP INDEX idx;
CREATE INDEX idx ON t USING PDXEARCH (vec);
```

## Limitations

- **No incremental updates**: The index is a snapshot at creation time. INSERT,
  UPDATE, and DELETE operations will not crash the database, but new or modified
  rows will not appear in search results until you rebuild the index. Use
  `CALL pdxearch_index_info()` to check `has_unindexed_data`.

- **In-memory only**: The index should only be created in in-memory DuckDB
  databases. For disk-resident databases, you will need to drop and recreate
  the index after reopening the database.

- **Row groups must be full**: DuckDB stores data in row groups (blocks of up to
  122,880 rows). All but the last row group must be completely full. If you are
  loading data, insert in batches of 122,880 rows to ensure this layout.

- **Small-K workaround**: For queries with `LIMIT` 50 or less, run
  `SET late_materialization_max_rows = 0;` before your search query.

- **Simple filters only**: The index accelerates queries where DuckDB pushes the
  WHERE clause filter down into the table scan. Complex or composite filters may
  not be optimized. Use `EXPLAIN` to verify the index is being used.

- **Table size**: Tables with more than ~4 billion rows are not supported.

- **Platforms**: Linux and macOS. Windows and WASM are not yet supported.

## Troubleshooting

**My query is not using the index:**

Use `EXPLAIN` to check the query plan:
```sql
EXPLAIN SELECT * FROM items
    ORDER BY array_distance(embedding, [1.0, ...]::FLOAT[128])
    LIMIT 10;
```
Look for `PDXEARCH_INDEX_SCAN` or `PDXEARCH_INDEX_FILT_SCAN` in the output. If
it's not there, check that:
1. Your query matches the pattern: `SELECT ... ORDER BY distance(col, query) LIMIT K`
2. The distance function matches the index metric (e.g. `array_distance` for `l2sq`)
3. The query vector has the same dimensions as the indexed column

**Search results are missing rows I recently inserted:**

The index is a snapshot. Check if the index is stale:
```sql
CALL pdxearch_index_info();
```
If `has_unindexed_data` is `true`, rebuild:
```sql
DROP INDEX items_idx;
CREATE INDEX items_idx ON items USING PDXEARCH (embedding);
```

**Out of memory during index creation:**

Try reducing the data size, or use 8-bit quantization (the default) which uses
4x less memory than `f32`:
```sql
CREATE INDEX idx ON t USING PDXEARCH (vec) WITH (quantization = 'u8');
```

**Error after reopening a database file:**

Index persistence is limited. Drop and recreate:
```sql
-- Find your indexes
SELECT index_name, sql FROM duckdb_indexes() WHERE index_type = 'PDXEARCH';
-- Drop the stale index
DROP INDEX index_name;
-- Recreate using the SQL from above
```

## Acknowledgements

- **[PDX](https://github.com/cwida/pdx)**: The PDX data layout and PDXearch
  search framework by CWI.
- **[Super K-Means](https://github.com/lkuffo/SuperKMeans)**: Fast k-means
  clustering library.
- **[DuckDB VSS](https://github.com/duckdb/duckdb-vss)**: Inspiration for the
  SQL interface and extension structure.

## License

The extension is licensed under the [MIT license](LICENSE).
