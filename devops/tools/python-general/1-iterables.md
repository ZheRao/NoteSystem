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
