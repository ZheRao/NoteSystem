
**`datetime` Essentials**
---

```python
import datetime as dt

dt.date.today()
dt.date.today().isoformat()
dt.timedelta(days=3)
```

**Weekday Offset Example**
---

```python
dt.timedelta(days=dt.date.today().weekday())
```

**Time Sleep**
---

```python
import time
time.sleep(30)
```

