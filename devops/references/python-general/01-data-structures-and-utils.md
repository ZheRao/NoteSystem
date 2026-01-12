
**Heap Queue (`heapq`)**
---

- Min-heap implementation for priority queues

```python
import heapq

a = []
heapq.heappush(a, (key, value))
heapq.heappop(a)

a[0]          # smallest (key, value)
for item in a:
    print(item)
```

**Random Utilities (`random`)**
---

```python
import random

random.choice(list)          # single random element
random.sample(list, k=n)     # k unique random elements
```

**Zip**
---

```python
list(zip([1, 2], [11, 22]))
# [(1, 11), (2, 22)]

dict(zip(["a", "b"], [1, 2]))
# {'a': 1, 'b': 2}
```


**Numpy Essentials**
---

**Array creation**

```python
np.arange(start=0, stop=9, step=1, dtype=int)
# stop is exclusive
```

**Saving & loading multiple arrays**

```python
with open("test.npy", "wb") as f:
    np.save(f, np.array([1, 2]))
    np.save(f, np.array([3, 4]))

with open("test.npy", "rb") as f:
    a = np.load(f)
    b = np.load(f)
```