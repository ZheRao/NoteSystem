# Generator with `yield`

What you are really reaching for is:
1. separate traversal from storage
2. stream records out one-by-one
3. let the caller decide what to do with them

That is exactly where `yield` shines.

## First, Type Hint

Better option: `Iterator` because
- Most of the time you **don't care** that something is specifically a generator.
- You only care that it is **iterable**.

```python
from typing import Iterator

def crawler(rows, cols) -> Iterator[dict]:
    ...
    yield record
```

or: 

```python
from typing import Generator

# Generator[YieldType, SendType, ReturnType]

def crawler(rows, cols) -> Generator[dict, None, None]:
    ...
    yield record
```

Meaning:

| position | meaning                              |
| -------- | ------------------------------------ |
| `dict`   | values yielded                       |
| `None`   | type accepted by `.send()`           |
| `None`   | return value when generator finishes |


## The core idea

A normal function `return`s once and dies.

A function with `yield` becomes a generator:
- it pauses at each `yield`
- gives one value back to the caller
- remembers where it was
- resumes from that exact spot when asked for the next value

So instead of:
```python
def crawler(obj):
    records = []
    # walk JSON
    records.append(record1)
    records.append(record2)
    records.append(record3)
    return records
```
you do:
```python
def crawler(obj):
    # walk JSON
    yield record1
    yield record2
    yield record3
```
Now the function does **not** build the full list in memory first. It emits records one at a time.

## Minimal example first
```python
def numbers():
    yield 1
    yield 2
    yield 3
```

Using it:
```python
g = numbers()
print(g)          # generator object
print(next(g))    # 1
print(next(g))    # 2
print(next(g))    # 3
```

Or more normally:
```python
for x in numbers():
    print(x)
```

Output:
```text
1
2
3
```

So the caller pulls values one at a time.

## Why this matters for your crawler

Suppose your JSON traversal finds row after row.

Instead of doing:
```python
records = []
for row in all_rows:
    record = extract_record(row)
    records.append(record)
return records
```

you do:
```python
for row in all_rows:
    record = extract_record(row)
    yield record
```

Now the consumer can decide:
- turn it into a list
- write row-by-row to disk
- feed Spark later
- feed pandas in chunks
- debug first 5 records only

That makes your code much more generic.

**Example usages:**

```python
for rec in crawler(obj, cols):
    print(rec)
```
Or
```python
records = list(crawler(obj, cols))
df = pd.DataFrame(records)
```

Notice what happened:
- the crawler does not care about pandas
- does not care about Spark
- does not care about storage
- it only knows how to emit records

That is the right abstraction.

## The most important pattern: *recursive generator*

Your JSON sounds nested and hierarchical, so the real power comes when recursion and yield combine.

Suppose rows can contain child rows:
```python
def crawler(rows, cols):
    for row in rows:
        # if this row is a real data row
        if "ColData" in row:
            values = [item.get("value") for item in row["ColData"]]
            record = dict(zip(cols, values))
            yield record
        
        # if this row has children
        if "Rows" in row and "Row" in row["Rows"]:
            for child_record in crawler(row["Rows"]["Row"], cols):
                yield child_record
```

This says:
- emit current row if it is a real record
- then continue into children
- emit all child records too

That inner part:
```python
for child_record in crawler(...):
    yield child_record
```
means:

“delegate to the recursive call, and pass its yielded records upward.”

This is extremely important.

## Even cleaner: `yield from`

Python gives a shortcut for that:
```python
def crawler(rows, cols):
    for row in rows:
        if "ColData" in row:
            values = [item.get("value") for item in row["ColData"]]
            record = dict(zip(cols, values))
            yield record
        
        if "Rows" in row and "Row" in row["Rows"]:
            yield from crawler(row["Rows"]["Row"], cols)
```
`yield from subgenerator` means:

“yield every item produced by that subgenerator.”

So this:
```python
for x in subgen:
    yield x
```
is basically:
```python
yield from subgen
```
For recursive crawlers, `yield from` is beautiful.

# Appendix: `yield` example

When `yield` becomes magical to people  
Usually because it feels like values are “coming out of nowhere.”

But mechanically, it is simple:
```python
def f():
    print("A")
    yield 1
    print("B")
    yield 2
    print("C")
```

Then:
```python
g = f()
next(g)
```
Output:
```text
A
1
```
Then:
```python
next(g)
```
Output:
```text
B
2
```
Then:
```python
next(g)
```
Output:
```text
C
StopIteration
```
So the function is paused and resumed. That is all.

It keeps:
- local variables
- loop state
- recursion state
- current instruction position

That is why it is perfect for tree traversal.