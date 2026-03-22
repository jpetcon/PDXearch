<h1 align="center">
  The PDXearch DuckDB Extension
</h1>
<h3 align="center">
  A state-of-the-art IVF index for lightweight but fast (filtered) vector similarity search.
</h3>
<br>

## Why PDXearch?

DuckDB offers vector similarity search (VSS) out of the box, through its
fixed-size `ARRAY` column type and distance functions
([docs](https://duckdb.org/docs/stable/sql/data_types/array#functions)). These
functions return exact results, but are often too slow on large datasets.

The official DuckDB [VSS extension](https://duckdb.org/docs/stable/core_extensions/vss)
introduces a graph-based (HNSW) VSS index. You can create this index on your
table to speed up the vector search queries. Unfortunately, although
these graph-based indexes deliver fast search, they take up a considerable
amount of memory and take long to construct.

The PDXearch extension aims to address these drawbacks. It achieves competitive
search performance, while using less memory and being significantly faster to
construct. This is made possible by a state-of-the-art partition-based (IVF)
index. To be precise, we rely on the CWI's [PDX](https://github.com/cwida/pdx)
data layout and the accompanying search framework called PDXearch. Furthermore,
this extension integrates tightly with DuckDB's internals to parallelize across
row groups, allowing us to squeeze more performance out of modern hardware.

## Install

> [!WARNING]
> The extension is unstable and experimental. We're actively working on adding
> features and improving stability. The extension will be made available as a
> community extension once it's ready. For now the extension has to be built
> locally.

To build the extension locally, see [DEVELOPMENT.md](DEVELOPMENT.md).

## Usage

To create an index and run a search, we provide an interface similar to the
official VSS extension ([VSS docs](https://duckdb.org/docs/stable/core_extensions/vss)).

1. Start a DuckDB instance with an in-memory database and allow loading unsigned extensions.

    ```bash
    duckdb -unsigned
    ```

2. Load the locally built extension by providing a full path to it.

    ```sql
    LOAD '<Fill in>/PDXearch/build/release/extension/pdxearch/pdxearch.duckdb_extension';
    ```

3. Set up a table.

    ```sql
    CREATE TABLE t1 (id INTEGER, vec FLOAT[512]);
    ```

    ```sql
    INSERT INTO t1 (id, vec) SELECT i as id, repeat([i], 512) FROM range(20000) t(i);
    ```

4. Create the PDXearch index and set one of the index's options (n_probe).

    ```sql
    CREATE INDEX t1_idx ON t1 USING PDXEARCH (vec) WITH (n_probe = 64);
    ```

5. Run an approximate filtered vector similarity search where the top 100 rows are returned.

    ```sql
    SELECT * FROM t1 WHERE id < 500
        ORDER BY array_distance(vec, repeat([1000.51], 512)::FLOAT[512]) LIMIT 100;
    ```

> [!WARNING]
> If you're executing (filtered) search queries where `K <= 50`, then please
> disable DuckDB's late materialization optimization by running the following
> statement prior to your search: `SET late_materialization_max_rows = 0;`. Due
> to the query's low LIMIT (K), DuckDB will apply a late materialization
> optimization. Unfortunately, the extension does not handle this case optimally
> yet, leading to a suboptimal query plan when a `K <= 50` VSS query is
> optimized. We aim to address this behavior in the near future.

## Index Parameters

The following parameters can be set during index creation:

```sql
CREATE INDEX idx ON t USING PDXEARCH (vec) WITH (metric = 'l2sq', quantization = 'u8', n_probe = 24, seed = 42);
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `metric` | string | `'l2sq'` | Distance metric. One of: `'l2sq'` (Euclidean), `'cosine'`, `'ip'` (inner product). |
| `quantization` | string | `'u8'` | Quantization type. One of: `'f32'` (full precision), `'u8'` (8-bit scalar quantization). |
| `n_probe` | integer | `24` | Number of clusters to probe during search. `0` probes all clusters (exact search). Range: 0–100,000. |
| `seed` | integer | random | Seed for rotation matrix generation. Set for reproducible index builds. |

### Runtime Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `pdxearch_n_probe` | integer | (unset) | Overrides the index's `n_probe` at query time. Range: 0–100,000. Set with `SET pdxearch_n_probe = 64;`. |

## Index Diagnostics

Inspect all PDXearch indexes in the database:

```sql
CALL pdxearch_index_info();
```

Returns columns: `catalog_name`, `schema_name`, `index_name`, `table_name`, `metric`, `num_dimensions`, `quantization`, `n_probe`, `seed`, `is_normalized`, `approx_lower_bound_memory_usage_bytes`, `has_unindexed_data`.

The `has_unindexed_data` column is `true` if rows have been inserted, updated, or deleted since the index was created. When this is `true`, search results may be incomplete. Drop and recreate the index to incorporate the changes:

```sql
DROP INDEX idx;
CREATE INDEX idx ON t USING PDXEARCH (vec);
```

## Limitations

- **No incremental maintenance**: The index is a snapshot at creation time. DML
  operations (INSERT, UPDATE, DELETE) do not crash the database but will not be
  reflected in search results until the index is rebuilt. The `has_unindexed_data`
  flag in `pdxearch_index_info()` indicates when a rebuild is needed.

- **No persistence**: The index should only be created in in-memory DuckDB
  databases. For disk-resident databases you'll have to manually drop and
  rebuild the index when you reload the database.

- **Requires full row groups**: The extension currently requires all but the
  last row group to be completely filled with rows. For example, three row
  groups where they have 122880, 122880, 4000 rows respectively is valid.
  Inserting rows in batches of 122880 can help to create such a layout.

- **Late materialization**: If you're executing queries where `K <= 50`, disable
  DuckDB's late materialization: `SET late_materialization_max_rows = 0;`.

- **Filter types**: Only filters pushed down into the sequential scan by DuckDB
  are supported. Check with `EXPLAIN` whether a PDXearch operator appears in
  the query plan.

- **Row count limit**: Tables with more than ~4 billion rows are not supported
  (row IDs must fit in 32 bits).

- **Supported platforms**: Linux and macOS. Windows and WASM builds are not yet
  available.

## Troubleshooting

**Index not being used for my query:**
Prepend `EXPLAIN` to your query. If no PDXearch operator appears, the optimizer
could not match it. Ensure your query follows the pattern:
`SELECT ... FROM t ORDER BY distance_function(vec, query) LIMIT K;`

**Search returns incomplete results after INSERT:**
The index does not automatically update. Check `CALL pdxearch_index_info();` —
if `has_unindexed_data` is `true`, drop and recreate the index.

**Out of memory during index creation:**
The global index variant loads all embeddings into memory. For large tables,
use the default row-group parallel variant (the default build) which processes
one row group at a time.

**"PDXearch index requires a non-empty table" after restart:**
Index persistence is limited. Drop the index and recreate it:
```sql
SELECT sql FROM duckdb_indexes();  -- find the CREATE INDEX statement
DROP INDEX index_name;
-- then recreate it
```

## Acknowledgements

The extension would not be possible without the underlying technologies and the
lessons learned from other extensions.

- **[PDX](https://github.com/cwida/pdx)**: We use the PDX data layout and
  PDXearch framework.

- **[Super K-Means](https://github.com/lkuffo/SuperKMeans)**: We use
  the Super K-Means library for fast k-means clustering.

- **[VSS](https://github.com/duckdb/duckdb-vss)**: We've taken inspiration from
  the VSS interface and we reuse parts of the VSS extension's code.

## License

The extension is licensed under the [MIT license](LICENSE).
