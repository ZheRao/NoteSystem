
**File System Basics**
---

```python
import os

if not os.path.exists(path):
    os.makedirs(path)
```

**File Read / Write**
---

```python
with open(path, "w") as f:
    f.write(message)
    f.writelines(["line1\n", "line2\n"])

with open(path, "r") as f:
    f.read()
    f.readline()
    f.readlines()
```

Notes:
- Text files usually don’t use `"rb"` / `"wb"`
- `"r+"` allows read + write without reopening

**Copying Files (`shutil`)**
---

```python
import shutil
shutil.copyfile(src, dst)
```

**Pathlib (`Path`)**
---

```python
from pathlib import Path

p = Path("C:/A/B/c.exe")

p.parent
p.parents[1]
p.name
p.suffix
p.parts
p.exists()
p.relative_to("C:/A")
p.unlink(missing_ok=True)
```