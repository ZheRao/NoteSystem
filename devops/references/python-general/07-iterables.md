# 1. List-related

**Extract scalar values**
- `.item()`  
    Get a single number from a list/tensor as a native Python `int` or `float`

**Range generation**
- `range(start, stop, step)`

**Uniqueness & sorting**
- `set(list)`  
    Create a set containing unique values from a list
- `sorted(list)`  
    Return a new list with sorted values
- `list(set(mylist))`  
    Remove duplicate values from a list

**Index-based access**
- `list.index(element, start, end)`  
    Locate index of an element within a slice
- `itemgetter(*index_list)(list)`  
    Retrieve multiple items by index - *(from `operator` module)*

**Element-wise transformations**
- `list(map(function, iterable))`  
    Apply a function to every element
    - `map`: applies a function to all elements
    - `filter`: applies a predicate and keeps only `True` results

**Mutation**
- `list.remove(item)`  
    Remove the first occurrence of `item`


# 2. Set-related

**Basic operations**
- `set_a.add(6)`
- `b = next(iter(set_a))`  
    Access an arbitrary element
- `set_a.pop()`  
    Remove and return an arbitrary element

**Removal**
- `set_a.remove(2)`  
    Raises error if element not found
- `set_a.discard(2)`  
    Does *not* raise error if element not found

**Set algebra**
- `set_a | set_b` or `set_a.union(set_b)`  
    All unique elements
- `set_a & set_b` or `set_a.intersection(set_b)`  
    Common elements
- `set_a - set_b` or `set_a.intersection(set_b)`  
    Elements in `a` but not in `b`
- `set_a ^ set_b` or `set_a.symmetric_difference(set_b)`  
    Elements appearing only once across both sets

**important constraint**
- Sets **cannot be indexed**


# 3. Dictionary

**Key & value access**

- `for key in dict` or `for key in dict.keys()`  
    Iterate over keys
- `for item in dict.items()`  
    Iterate over `(key, value)` pairs
- `dict.get(key, default)`  
    Safe lookup with fallback

**Value extraction**

- `list(dict[key].values())`  
    Convert nested dictionary values to a list
- `itemgetter(*keys)(dict)`  
    Retrieve multiple values by key

**Persistence**

- Save:

```python
with open(path, "wb") as f:
    pickle.dump(dict, f)
```

- Load:

```python
with open(path, "rb") as f:
    dict = pickle.load(f)
```

**Sorting & aggregation**

- `max(stats, key=dict.get)`  
    Get key with largest value
- `sorted(dict.items(), key=lambda x: x[1], reverse=True)`  
    Sort dictionary by value

**Updating & merging**

- `a.update(b)`  
    Merge dictionaries (in-place)
- Create new dictionaries:

```python
d1 = base | {"new": new1}
d2 = base | {"new": new2}
```

**Transformations**

- Invert dictionary:

```python
inv_map = {v: k for k, v in my_map.items()}
```

- Create with default values:

```python
items = [...]
d = dict.fromkeys(items, 0)
```

**JSON serialization**

```python
with open(path, "w") as file:
    json.dump(dictfile, file, indent=4)
```

# 4. Iterators

**Creating iterators**

```python
mytuple = ("apple", "banana", "cherry")
myit = iter(mytuple)
```

**Consuming iterators**

```python
next_item = next(myit)
```

**Enumerate**

- `enumerate(iterable, start)`
- Adds a counter alongside values

```python
for i, value in enumerate(iterable, start=0):
    ...
```
