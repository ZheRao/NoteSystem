


## Strings

**concatenating columns and add line break in between**
```DAX
Week Label =
"W" & FORMAT('YourTable'[week_num], "00")
    & UNICHAR(10)
    & FORMAT('YourTable'[week_end_date], "MMM d")
```

output
```
W01
Nov 7
```






















