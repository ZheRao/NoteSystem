# Collections: Array / Map / Struct

## 1. Arrays

**Construct / basics**
- `F.array`, `F.size`, `F.array_distinct`, `F.array_contains`

**Set-like ops**
- `F.array_union`, `F.array_except`, `F.array_intersect`

**Sort / shuffle**
- `F.sort_array`, `F.shuffle`

**Zip / flatten**
- `F.arrays_zip`
- `F.flatten`

**Slice / index / generate**
- `F.slice` *(works on arrays; on strings prefer `substr`)*
- `F.element_at`
- `F.sequence`

## 2. Maps

- `F.create_map`
- `F.map_keys`, `F.map_values`
- `F.element_at`
- `F.map_entries`, `F.map_from_entries`

## 3. Structs

- `F.struct("c1", "c2", ...)`
- Access: `F.col("s.field")` or `df.s.field`

