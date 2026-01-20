# Regex Mental Model (Read This First)

Regex operates in **two modes**:

1️⃣ **Alphabet mode — *What characters are allowed***

Activated by **character classes**: `[...]`

You are defining **set membership**:
> “This character is allowed / forbidden”

Used heavily in:
- data cleaning
- schema enforcement
- sanitization

---
2️⃣ **Grammar mode — *How characters are arranged***

Everywhere **outside** `[...]`

You are defining **structure**:
- position
- repetition
- grouping
- context

Used heavily in:
- validation
- parsing
- extraction

> **Most regex confusion comes from not knowing which mode you’re in.**

# 1. Anchors — *Position in the string* (Grammar Mode)

**Anchors do not match characters.**  
They assert *where* a match must occur.

| Anchor | Meaning         |
| ------ | --------------- |
| `^`    | start of string |
| `$`    | end of string   |

```python
# exactly "ABC"
df["col"].str.contains(r"^ABC$", na=False)

# starts with 4 digits
df["col"].str.contains(r"^\d{4}", na=False)

# ends with .csv
df["col"].str.contains(r"\.csv$", na=False)
```

**Critical rule about `^`**

| Where `^` appears         | Meaning         |
| ------------------------- | --------------- |
| Outside `[]`              | Start of string |
| Inside `[]` as first char | NOT (negation)  |
| Inside `[]` not first     | Literal `^`     |

**Common pitfall**

Without anchors, regex matches **anywhere** in the string.

# 2. Character Classes — *One character from a set* (Alphabet Mode)

Character classes match **exactly one character** chosen from a set.
```regex
[abc]       # a OR b OR c
[A-Za-z]    # any letter
[0-9]       # any digit
[^0-9]      # any NON-digit
```
```python
# contains at least one letter
df["col"].str.contains(r"[A-Za-z]", na=False)

# contains any non-numeric character
df["col"].str.contains(r"[^0-9]", na=False)
```

### Special characters inside `[]`

Only three are special:
- `^` → NOT (only if first)
- `-` → range
- `\` → escape

Everything else is literal.

# 3. Predefined Character Classes — *Common alphabets*
Shorthand character sets:
| Pattern | Meaning                  |
| ------- | ------------------------ |
| `\d`    | digit `[0-9]`            |
| `\w`    | word char `[A-Za-z0-9_]` |
| `\s`    | whitespace               |

Uppercase = inverse:
| Pattern | Meaning        |
| ------- | -------------- |
| `\D`    | NOT digit      |
| `\W`    | NOT word char  |
| `\S`    | NOT whitespace |

```python
# contains whitespace
df["col"].str.contains(r"\s", na=False)

# contains only digits
df["col"].str.contains(r"^\d+$", na=False)
```

# 4. Word Boundaries — *Whole words vs substrings* (Grammar Mode)

Word boundaries detect *transitions* between:
- word characters (\w)
- non-word characters

| Token | Meaning             |
| ----- | ------------------- |
| `\b`  | word boundary       |
| `\B`  | NOT a word boundary |

```python
# "cat" as a word, not "concatenate"
df["text"].str.contains(r"\bcat\b", na=False)

# "cat" only inside larger words
df["text"].str.contains(r"\Bcat\B", na=False)
```

Word characters = `[A-Za-z0-9_]`

# 5. Quantifiers — How many times (Grammar Mode)

Quantifiers apply to the **immediately preceding token**.

| Quantifier | Meaning         |
| ---------- | --------------- |
| `*`        | 0 or more       |
| `+`        | 1 or more       |
| `?`        | optional        |
| `{m}`      | exactly m       |
| `{m,}`     | m or more       |
| `{m,n}`    | between m and n |

```python
# optional "u" in color/colour
df["col"].str.contains(r"colou?r", na=False)

# exactly 3 digits
df["col"].str.contains(r"^\d{3}$", na=False)

# 5–10 alphanumeric characters
df["col"].str.contains(r"^[A-Za-z0-9]{5,10}$", na=False)
```

# 6. Groups — *Treat multiple tokens as one* (Grammar Mode)

### Capturing groups `( … )`
Capture text for reuse or extraction.
```python
df["year"] = df["date"].str.extract(r"^(\d{4})")
```

### Non-capturing groups `(?: … )`
Group without capturing (cleaner, faster).
```python
# jpg or png
df["file"].str.contains(r"\.(?:jpg|png)$", na=False)
```

**Rule of thumb**  
Use `(?:...)` unless you explicitly need the captured value.

# 7. Lookarounds — Context without consumption (Grammar Mode)

Lookarounds are **zero-width assertions**:
- they check context
- they do **not** consume characters

## 7.1 Lookahead — *What must follow*
| Type     | Syntax    | Meaning                 |
| -------- | --------- | ----------------------- |
| Positive | `(?=...)` | must be followed by     |
| Negative | `(?!...)` | must NOT be followed by |
```python
# "USD" only if followed by a digit
df["col"].str.contains(r"USD(?=\d)", na=False)

# "USD" not followed by a digit
df["col"].str.contains(r"USD(?!\d)", na=False)
```

## 7.2 Lookbehind — *What must precede*
| Type     | Syntax     | Meaning                 |
| -------- | ---------- | ----------------------- |
| Positive | `(?<=...)` | must be preceded by     |
| Negative | `(?<!...)` | must NOT be preceded by |
```python
# digits preceded by $
df["col"].str.contains(r"(?<=\$)\d+", na=False)

# digits not preceded by $
df["col"].str.contains(r"(?<!\$)\d+", na=False)
```

**Python limitation**  
Lookbehinds must be **fixed width**.

## 7.3 Zero-Width Assertion Intuition

After a lookaround:
- the regex engine **does not move**
- only the *main match* is returned

This makes lookarounds perfect for:
filtering
validation
constraint checking

# 8. High-Value Pandas Regex Patterns (Intent-Driven)
```regex
# exact date
^\d{4}-\d{2}-\d{2}$

# at least one letter AND one digit
(?=.*[A-Za-z])(?=.*\d)

# valid identifier (Python-like)
^[A-Za-z_]\w*$

# comma-separated numbers
^\d+(?:,\d+)*$
```