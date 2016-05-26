- Feature Name: sql_column_groups
- Status: draft
- Start Date: 2015-12-14
- Authors: Daniel Harrison, Peter Mattis
- RFC PR: [#6712](https://github.com/cockroachdb/cockroach/pull/6712)
- Cockroach Issue:

# Summary

Store multiple columns for a row in a single value to amortize the
overhead of keys and the fixed costs associated with values. The
mapping of a column to a column group will be determined when a column
is added to the table.

# Motivation

The SQL to KV mapping currently utilizes a key/value pair per
non-primary key column. The key is structured as:

```
/<tableID>/<indexID>/<primaryKeyColumns...>/<columnID>
```

Keys also have an additional suffix imposed by the MVCC
implementation: ~10 bytes of timestamp.

The value is a proto (note, this is what is stored on disk, we also
include a timestamp when transmitting over the network):

```
message Value {
  optional bytes raw_bytes;
}
```

The raw_bytes field currently holds a 4 byte CRC followed by the bytes
for a single "value" encoding (as opposed to the util/encoding "key"
encodings) of the `<columnVal>`. The value encoding is a tag byte
followed by a payload.

Consider the following table:

```
CREATE TABLE kv (
  k INT PRIMARY KEY,
  v INT
)
```

A row for this table logically has 16 bytes of data (`INTs` are 64-bit
values). The current SQL to KV mapping will create 2 key/value pairs:
a sentinel key for the row and the key/value for column `v`. These two
keys imply an overhead of ~28 bytes (timestamp + checksum) when stored
on disk and more than that when transmitted over the network (disk
storage takes advantage of prefix compression of the keys). We can cut
this overhead in half by storing only a single key/value for the
row. The savings are even more significant as more columns are added
to the table.

Note that there is also per key overhead at the transaction
level. Every key written within a transaction has a "write intent"
associated with it and these intents need to be resolved when the
transaction is committed. Reducing the number of keys per row reduces
this overhead.

# Detailed design

The `ColumnDescriptor` proto will be extended with a `GroupID` field.
`GroupIDs` will be allocated per-table using a new
`TableDescriptor.NextGroupID` field. `GroupID(0)` will be always be
written for a row and will take the place of the sentinel key. A
`GroupDescriptor` proto will be added along with a repeated field for
them in `TableDescriptor`. It will contain the id, a user provided or
autogenerated name, and future metadata.

The structure for table keys will be changed to:

```
/<tableID>/<indexID>/<primaryKeyColumns...>/<groupID>
```

The value proto will remain almost the same. The same 4 byte CRC will
be followed by a series of `<columnID>/<columnVal>` pairs where
`<columnID>` is varint encoded and `<columnVal>` is encoded using
(almost) the same value encoding as before. Unfortunately, the
bytes, string, and decimal encodings do not self-delimit their length,
so additional ones will be added with a length prefix. These new
encodings will also be used by distributed SQL, which also needs self-
delimiting value encodings. Similar to the existing convention, `NULL`
column values will not be present in the value.

For backwards compatibility, groups with only a single column will use
exactly the old encoding. Columns without a group will use the
`<columnID>` as the `<groupID>`. Finally, the `/<groupID>` key suffix
will be omitted when encoding `GroupID(0)` to make the encoding
identical to the previous sentinel key encoding.

When a column is added to a table it will be assigned to a group. This
assignment will be done with a set of heuristics, but will be
override-able by the user at column creation using a SQL syntax
extension.

## SQL Syntax Examples

```
CREATE TABLE t (
  a INT PRIMARY KEY,
  b INT,
  c STRING,
  GROUP primary (a, b),
  GROUP (c)
)
```

`ALTER TABLE t ADD COLUMN d DECIMAL GROUP d`

## Heuristics for Fitting Columns into Groups

- Fixed sized columns (`INT`, `DECIMAL` with precision, `FLOAT`, `BOOL`, `DATE`,
`TIMESTAMP`, `INTERVAL`, `STRING` with length, `BYTES` with length) will be
packed into a group, up to a threshold (initially 256 bytes).

- `STRING` and `BYTES` columns without a length restriction (and `DECIMAL`
without precision) get their own group.

- If the declared length of a `STRING` or `BYTES` column is changed with an
`ALTER COLUMN`, group assignments are unaffected.

- Columns in the primary index are declared as group 0, but they're stored in
the key, so they don't count against the byte limit.

- Group 0 (the sentinel) always has at least one column (though it might be a
primary index column stored in the key). Non-nullable and fixed sized columns
are preferred.

Note that these heuristics only apply at column creation, so there's no
backwards compatibility issues in revisiting them at any time.

# Drawbacks

* We have to support the old format for backwards compatibility.
Luckily, we can re-frame most of the complexity as a one column group
optimization instead of pure technical debt.

* `UPDATE` will now have to fetch the previous values of every column
in a group being modified. However, we already have to scan before
every `UPDATE` so, at worst, this is just returning more fields from
the scan than before.

# Alternatives

* We could introduce a richer KV layer api that pushes knowledge of
columns from the SQL layer down to the KV layer. This is not as
radical as it sounds as it is essentially the Bigtable/HBase
API. Specifically, we could enhance KV to know about rows and columns
where columns are identified by integers but not restricted to a
predefined schemas. This is actually a somewhat more limited version
of the Bigtable API which allows columns to be arbitrary strings and
also has the concept of column families. The upside to this approach
is that the encoding of the data at the MVCC layer would be
unchanged. We'd still have a key/value per column. But we'd get
something akin to prefix compression at the network level where
setting or retrieving multiple columns for a single row would only
send the row key once. Additionally, we could likely get away with a
single intent per row as opposed to an intent per column in the
existing system. The downside to this approach is that it appears to
be much more invasive than the column group change. Would every
consumer of the KV api need to change?

* Another alternative would be to omit the sentinel key when there is
no non-NULL/non-primary-key column. For example, we could omit the
sentinel key in the following table because we know there will always
be one KV pair:

  ```
CREATE TABLE kv (
  k INT PRIMARY KEY,
  v INT NOT NULL
)
```

# Unresolved questions

* Some of the complexity around legacy compatibility could be reduced
if `GroupDescriptor`s could be backfilled for tables missing them on
cluster upgrade. Is there a reasonable mechanism for this? Or should
it be hidden as a TableDescriptor transformation in the
`getAliasedTableLease` call?

# Future Work

* Allow changing of a column's group using ALTER COLUMN and schema changes.

  * Mark column as write-only in new group.
  * Write column to both old and new groups for queries, reading it only from old group.
  * Backfill column data to new group.
  * Remove column from old group, mark readable in new group.
  * TODO(dan): Consider if we can instead write only to the new group
    and read from both old and new, preferring new. Maybe some issues
    with NULLs.

* Add metadata to column groups. @bdarnell mentions this would let us
map some groups to an alternative storage model (perhaps using rocksdb
column families) and that this was very useful with bigtable column
families/locality groups. Once the schema changes are supported, it
will also be needed to keep track of which column, if any, was used
with the single column optimization.

# Performance Experiments

Note: This is from @petermattis's original doc, so it might be out of
date. The conclusion is still valid though.

For the above `kv` table, we can approximate the benefit of this
change for a benchmark by not writing the sentinel key for each
row. The numbers below show the delta for that change using the `kv`
table structure described above (instead of the 1 column table
currently used in the `{Insert,Scan}` benchmarks).

```
name                   old time/op    new time/op    delta
Insert1_Cockroach-8       983µs ± 1%     948µs ± 0%   -3.53%    (p=0.000 n=9+9)
Insert10_Cockroach-8     1.72ms ± 1%    1.34ms ± 0%  -22.05%   (p=0.000 n=10+9)
Insert100_Cockroach-8    8.52ms ± 1%    4.99ms ± 1%  -41.42%  (p=0.000 n=10+10)
Scan1_Cockroach-8         348µs ± 1%     345µs ± 1%   -1.07%  (p=0.002 n=10+10)
Scan10_Cockroach-8        464µs ± 1%     419µs ± 1%   -9.68%  (p=0.000 n=10+10)
Scan100_Cockroach-8      1.33ms ± 1%    0.95ms ± 1%  -28.61%  (p=0.000 n=10+10)
```

While the benefit is fairly small for single row insertions, this is
only benchmarking the simplest of tables. We'd expect a bigger benefit
for tables with more columns.