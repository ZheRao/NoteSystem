
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

p = Path("/data/exports/out.csv")

p.exists()
p.mkdir(parents=True, exist_ok=True)        # create the file and all parent folders if not exist
p.parent
p.parents[1]
p.name
p.suffix
p.parts
p.relative_to("C:/A")
p.unlink(missing_ok=True)

p.with_name("tmp.csv")    # replaces only the filename, keeping the same parent directory.
# -> Path("/data/exports/tmp.csv")
```