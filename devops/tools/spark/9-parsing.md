# Complex Parsing

## 1. JSON

- `F.from_json(col, schema)` (JSON string → struct/array)
- `F.to_json(col)` (struct/array → JSON string)
- `F.get_json_object(col, "$.path")` (string extraction)
- `F.json_tuple(col, "path1", "path2")` (SQL-style extraction)

## 2. CSV

- `F.from_csv(col, schema, options)`
- `F.to_csv(col, options)`
