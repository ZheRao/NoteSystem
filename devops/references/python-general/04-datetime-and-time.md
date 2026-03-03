
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

**Extract String Month Example**
---

```python
import datetime

# Create a datetime object
date_object = datetime.datetime(2024, 3, 15)

# Get the full month name
full_month_name = date_object.strftime("%B")
print(f"Full month name: {full_month_name}")

# Get the abbreviated month name
abbreviated_month_name = date_object.strftime("%b")
print(f"Abbreviated month name: {abbreviated_month_name}")
```

Output:
```text
Full month name: March
Abbreviated month name: Mar
```

**Time Sleep**
---

```python
import time
time.sleep(30)
```

