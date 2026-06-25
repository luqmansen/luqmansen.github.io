

https://howqueryengineswork.com/02-apache-arrow.html


```
RecordBatch:
  Schema: {id: Int32, name: Utf8, salary: Float64}
  Columns:
    id:     [1, 2, 3]
    name:   ["Alice", "Bob", "Carol"]
    salary: [95000.0, 87000.0, 102000.0]
  Length: 3
```
Early observation of Apache arrow's Record batches
- As this is just an in-memory layout, the column format is so simple. There's no need specific ordering/tricks/indexes because, again, this is just an in-memory format

```
Schema:
  - id: Int32, not nullable
  - name: Utf8, nullable
  - hire_date: Date32, nullable
  - salary: Float64, not nullable
```

>Having explicit nullability in the schema lets the query engine optimize. If a column cannot be null, we can skip null checks entirely.

I'm really curious how much actually does it cost for nullability checks from cpu standpoint