# Index Hints Feature for MySQL Shell Dump

## Overview

This feature adds support for specifying which index to use for table chunking during parallel dumps. This is useful when the primary key has poor data distribution and an alternative unique index would provide better-balanced chunks across threads.

## Implementation Summary

### Files Modified

1. **modules/util/dump/ddl_dumper_options.h**
   - Added `m_index_hints` member variable to store index hints
   - Added `index_hints()` getter method
   - Added `set_index_hints()` setter method declaration

2. **modules/util/dump/ddl_dumper_options.cc**
   - Implemented `set_index_hints()` with validation
   - Added `indexHints` option to the option pack

3. **modules/util/dump/indexes.h**
   - Modified `select_index()` signature to accept optional hint parameter

4. **modules/util/dump/indexes.cc**
   - Implemented hint lookup logic at the beginning of `select_index()`
   - Checks primary key, primary key equivalents, and unique keys for matching hint

5. **modules/util/dump/dumper.cc**
   - Modified `create_table_task()` to lookup and pass hints to `select_index()`
   - Added validation to ensure hinted index exists, with helpful error message

6. **modules/util/mod_util.cc**
   - Added documentation for the `indexHints` option in both dump and copy operations

## Usage Example

```javascript
// Dump a table using an alternative index for better chunking
util.dumpTables(
  "myschema",
  ["mytable"],
  "/path/to/dump",
  {
    chunking: true,
    threads: 8,
    indexHints: {
      "myschema.mytable": "idx_created_user"  // Use this index instead of PK
    }
  }
)
```

## Use Case Example

Consider a table with the following structure:

```sql
CREATE TABLE orders (
  order_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  created_at DATETIME NOT NULL,
  UNIQUE KEY idx_created_user (created_at, user_id)
);
```

If `order_id` values are not evenly distributed (e.g., due to bulk imports or deletions), chunking by primary key may create unbalanced chunks. The `idx_created_user` index might provide better distribution if orders are created relatively evenly over time.

```javascript
util.dumpTables(
  "sales",
  ["orders"],
  "/backup/orders",
  {
    threads: 16,
    indexHints: {
      "sales.orders": "idx_created_user"
    }
  }
)
```

## Algorithm Details

### Default Index Selection (without hints)

The dumper selects indexes in this priority order:

1. **Primary Key** (if exists and not ENUM-based)
2. **Primary Key Equivalents** (non-NULL unique indexes)
   - Prefers shorter indexes
   - Prefers longer runs of integer columns
   - Prefers NOT NULL columns
3. **Unique Keys** (even with nullable columns)
4. **No chunking** (if no suitable index found)

### With Index Hints

When a hint is provided:

1. Search for the index name in primary key
2. Search in primary key equivalents
3. Search in unique keys
4. If found, use that index regardless of default selection logic
5. If not found, throw error with list of available indexes

## Error Handling

If an invalid index is specified, a clear error message is displayed:

```
Error: Could not find index 'idx_nonexistent' specified in indexHints for table
myschema.mytable. Available indexes: PRIMARY, idx_created_user, idx_user_status
```

## Benefits

1. **Better chunk balance** - Use indexes with better data distribution
2. **Faster parallel dumps** - More evenly distributed work across threads
3. **Flexibility** - Override automatic selection when you know better
4. **Validation** - Clear errors if index doesn't exist

## Limitations

1. Only works with unique indexes (primary key, primary key equivalents, or unique keys)
2. Index must exist at dump time
3. Format is "schema.table" -> "index_name" mapping
4. No support for wildcards or patterns

## Testing Recommendations

1. Test with tables having multiple unique indexes
2. Verify error handling with non-existent index names
3. Compare chunk distribution with and without hints
4. Test with composite indexes
5. Verify it works with `dumpTables()`, `dumpSchemas()`, and `dumpInstance()`

## Future Enhancements

Potential improvements for future versions:

1. Support for pattern matching (e.g., "db%.%": "idx_name")
2. Automatic distribution analysis to suggest better indexes
3. Support for non-unique indexes (with additional logic)
4. Performance metrics showing chunk balance effectiveness

