
**Common String Operations**
---

```python
"a d c b".split(" ")
"|".join(["a", "b", "c"])
string.strip()
```

**Search & Replace**
---

```python
string.find(value, start, end)        # returns index or -1
string.replace(old, new, count)

import re
s = "..."
result = re.sub(r'\s+', "_", s.strip()) # replace one or several consecutive white spaces with _
```

**Prefix / Suffix**
---

```python
string.startswith(value)
string.endswith(value)
```

**String Classification**
---

```python
string.isnumeric()
string.isalnum()
string.isalpha()
```

**Formatting**
---

```python
string.title()  # Capitalize each word
```

