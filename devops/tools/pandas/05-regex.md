
# 1. Anchors — *Position in the string*

Anchors **do not match characters**; they assert where a match must occur.
- `^` → start of string
- `$` → end of string

```python
# exactly "ABC"
df["col"].str.contains(r"^ABC$", na=False)

# starts with 4 digits
df["col"].str.contains(r"^\d{4}", na=False)

# end with .csv
df["col"].str.contains(r"\.csv$", na=False)
```

Common pitfall
- Without anchors, regex matches anywhere in the string

# 2. Word boundaries — *Whole words vs. substring*

Word boundaries define transitions between **word characters** (`\w`) and **non-word characters**.
- `\b` → word boundary
- `\B` → NOT a word boundary

```python
# matches "cat" as a word, not "concatenate"
df["text"].str.contains(r"\bcat\b", na=False)

# matches "cat" only inside larger words
df["text"].str.contains(r"\Bcat\B", na=False)
```

Word characters = `[A-Za-z0-9_]`


# 3. Character classes — *one character from a set*

Basic classes

```python
[abc]       # a OR b OR c
[A-Za-z]    # any letter
[0-9]       # any digit
[^0-9]      # NOT a digit
```

```python
# contains at least one letter
df["col"].str.contains(r"[A-Za-z]", na=False)

# contains any non-numeric character
df["col"].str.contains(r"[^0-9]", na=False)
```

Key rule
- Character classes match **exactly one character**.

# 4. Predefined chatacter classes

Shortcuts for common classes:

| Pattern | Meaning                          |
| ------- | -------------------------------- |
| `\d`    | digit `[0-9]`                    |
| `\w`    | word char `[A-Za-z0-9_]`         |
| `\s`    | whitespace (space, tab, newline) |

Uppercase = inverse

| Pattern | Meaning        |
| ------- | -------------- |
| `\D`    | NOT digit      |
| `\W`    | NOT word char  |
| `\S`    | NOT whitespace |

```python
# contains any whitespace
df["col"].str.contains(r"\s", na=False)

# contains only digits
df["col"].str.contains(r"^\d+$", na=False)
```

# 5. Quantifiers — *How many times the previous token repeats*

Quantifiers apply to the **token immeidately before them**.

| Quantifier | Meaning           |
| ---------- | ----------------- |
| `*`        | 0 or more         |
| `+`        | 1 or more         |
| `?`        | 0 or 1 (optional) |
| `{m}`      | exactly m         |
| `{m,}`     | m or more         |
| `{m,n}`    | between m and n   |

```python
# optional "u" in color/colour
df["col"].str.contains(r"colou?r", na=False)

# exactly 3 digits
df["col"].str.contains(r"^\d{3}$", na=False)

# 5-10 alphanumeric chars
df["col"].str.contains(r"^[A-Za-z0-9]{5,10}$", na=False)
```

# 6. Groups — *Treat multiple tokens as one unit*

**Capturing groups** `(...)`

- Capture text for reuse or extraction

```python
df["year"] = df["date"].str.extract(r"^(\d{4})")
```

**Non-capturing groups** `(?:...)`

- Group **without capturing** (cleaner, faster)

```python
# match jpg or png
df["file"].str.contains(r"\.(?:jpg|png)$", na=False)
```

**Rule of thumb**
- Use `(?:...)` unless you explicitly need the captured value 

# 7. Lookarounds — *Match based on context, without consuming characters*

Lookarounds are **zero-width assertions**: they check surroundings but don't include them in the match

## 7.1 Lookahead — *What must follow*
**Positive Lookahead** `(?=...)`
- Match **only if followed by** pattern

```python
# "USD" only if followed by a digit
df["col"].str.contains(r"USD(?=\d)", na=False)
```

**Negative Lookahead** `(?!...)`
- Match **only if NOT followed by** pattern

```python
# "USD" not followed by a digit
df["col"].str.contains(r"USD(?!\d)", na=False)
```

## 7.2 Lookbehind — *What must precede*

**Positive Lookbehind** `(?<=...)`
- Match **only if preceded by** pattern

```python
# digits preceded by $
df["col"].str.contains(r"(?<=\$)\d+", na=False)
```

**Negative Lookbehind** `(?<!...)`
- Matched **only if NOT preceded by** pattern

```python
# digits not preceded by $
df["col"].str.contains(r"(?<!\$)\d+", na=False)
```

**Lookbehind limitations**
- Must be **fixed width** in Python regex (no `+`, `*` inside)

## 7.3 Zero-Width Assertion Intuition

After a lookaround check:
- **Regex engine position does not move**
- Only the *main match* is returned

This is why lookarounds are perfect for **filtering without altering the matched text**


# 8. High-Value Pandas Regex Patterns

```python
# exact YYYY-MM-DD
r"^\d{4}-\d{2}-\d{2}$"

# contains at least one letter AND one digit
r"(?=.*[A-Za-z])(?=.*\d)"

# valid identifier (Python-like)
r"^[A-Za-z_]\w*$"

# comma-separated numbers
r"^\d+(?:,\d+)*$"
```
