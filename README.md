# orm

A database-agnostic ORM for Carp.

`derive-model` reads a `deftype`'s fields with `members` and emits CRUD
functions in the type's module. The SQL dialect and row marshalling are
delegated to a backend module, so the same model definition works against
any database with a backend.

## Installation

The package ships the ORM core and a set of backends. Load the backend
you want via the two-argument form of `load`:

```clojure
; SQLite (also pulls in the carpentry-org/sqlite3 package transitively)
(load "git@github.com:carpentry-org/orm@0.4.0" "backends/sqlite3.carp")
```

If you are writing your own backend you can load just the core:

```clojure
(load "git@github.com:carpentry-org/orm@0.4.0")
```

This gives you the `derive-model` macro without pulling in any database
driver.

## Usage

### Defining a model

```clojure
(deftype Item [id Int text String done Bool])
(derive-model Item SQLiteBackend [id Int])
```

The first argument to `derive-model` is the type, the second is the backend
module, and the third is the array of primary-key fields (one or more).

### Nullable columns

A field typed `(Maybe T)` becomes a nullable column. `Maybe.Nothing`
is written as SQL `NULL`, and a `NULL` read back becomes `Maybe.Nothing`:

```clojure
(deftype Profile [id Int handle String bio (Maybe String) age (Maybe Int)])
(derive-model Profile SQLiteBackend [id Int])

(ignore
  (Profile.insert &db
                  &(Profile.init 0 @"ann" (Maybe.Just @"hi") (Maybe.Nothing))))

(Profile.find-where &db "age IS NULL" &[])
; => (Success [(Profile 1 @"ann" (Just @"hi") (Nothing))])
```

`update` and `upsert` set a column back to `NULL` by writing
`Maybe.Nothing`, and `IS NULL` / `IS NOT NULL` work in any WHERE clause.
Aggregates follow SQL semantics and skip `NULL` rows, so `sum` over a
column that is `NULL` everywhere returns 0.0.

Primary-key fields cannot be nullable, and neither `(Maybe (Maybe T))`
nor a `Maybe` of an unsupported type is allowed; both raise a macro error
at expansion time.

`create-table` emits `NOT NULL` for every column that is not a `Maybe`,
so the schema enforces what the Carp types claim. Since the DDL is
`CREATE TABLE IF NOT EXISTS`, this only applies to tables created from
now on; existing databases keep their current schema.

### Composite primary keys

```clojure
(deftype Enrollment [sid Int cid Int grade String])
(derive-model Enrollment SQLiteBackend [sid Int cid Int])
```

For composite-PK models, `insert` includes all fields (PK values are
user-supplied, not auto-incremented) and returns `(Result () String)`.
`find-by-id` and `delete-by-id` accept one argument per PK field:

```clojure
(ignore (Enrollment.insert &db &(Enrollment.init 1 101 @"A")))
(Enrollment.find-by-id &db &1 &101)
(Enrollment.delete-by-id &db &1 &101)
```

### Creating the table

```clojure
(let-do [db (Result.unsafe-from-success (SQLite3.open "app.db"))]
  (ignore (Item.create-table &db))
  (SQLite3.close db))
```

`create-table` runs `CREATE TABLE IF NOT EXISTS`, so it is safe to call on
every startup. It returns `(Result () String)` — an error if the
statement fails (e.g. the database is read-only).

### Inserting

```clojure
(match (Item.insert &db &(Item.init 0 @"buy milk" false))
  (Result.Success new-id) (println* new-id)
  (Result.Error e) (IO.errorln &e))
```

`insert` writes the non-PK fields and returns the auto-assigned rowid
as a `Long`, wrapped in a `Result`. The PK field on the input row is
ignored, which is why we pass `0`.

### Reading

```clojure
; All rows
(match (Item.find-all &db)
  (Result.Success items) (println* &items)
  (Result.Error e) (IO.errorln &e))

; By primary key
(match (Item.find-by-id &db &1)
  (Result.Success item) (println* &item)
  (Result.Error _) (println* "not found"))
```

The PK argument is passed as a reference even for value types like `Int`.

```clojure
; By WHERE clause with parameterized values
(match (Item.find-where &db "done = ?1" &[(to-sqlite3 0)])
  (Result.Success items) (println* &items)
  (Result.Error e) (IO.errorln &e))

; Multiple conditions
(match (Item.find-where &db "done = ?1 AND text = ?2"
                        &[(to-sqlite3 1) (to-sqlite3 @"buy milk")])
  (Result.Success items) (println* &items)
  (Result.Error e) (IO.errorln &e))
```

`find-where` takes a SQL WHERE clause as a string and an array of
`SQLite3.Type` parameter values. Parameters use positional placeholders
(`?1`, `?2`, …) and are bound safely — they are never interpolated into
the SQL string. The function returns all matching rows.

### Ordering and pagination

```clojure
; All rows, sorted
(match (Item.find-ordered &db "text ASC")
  (Result.Success items) (println* &items)
  (Result.Error e) (IO.errorln &e))

; Filtered rows, sorted
(match (Item.find-where-ordered &db "done = ?1" &[(to-sqlite3 0)] "text ASC")
  (Result.Success items) (println* &items)
  (Result.Error e) (IO.errorln &e))

; Paginated: page 1 (first 10 rows, sorted by id)
(match (Item.find-page &db "id ASC" 10 0)
  (Result.Success page) (println* &page)
  (Result.Error e) (IO.errorln &e))

; Paginated: page 2
(match (Item.find-page &db "id ASC" 10 10)
  (Result.Success page) (println* &page)
  (Result.Error e) (IO.errorln &e))

; Filtered + paginated
(match (Item.find-where-page &db "done = ?1" &[(to-sqlite3 0)]
                             "text ASC" 10 0)
  (Result.Success page) (println* &page)
  (Result.Error e) (IO.errorln &e))
```

`find-ordered` and `find-where-ordered` accept an ORDER BY clause as a
string. `find-page` and `find-where-page` add LIMIT/OFFSET pagination.
All four return `(Result (Array T) String)`.

### Updating

```clojure
(ignore (Item.update &db &(Item.init 1 @"bought milk" true)))
```

`update` writes all non-PK fields, using the PK on the row for the WHERE
clause and returns `(Result () String)`. Partial updates are not
supported, so the typical pattern is `find-by-id` then mutate then
`update`.

### Deleting

```clojure
(ignore (Item.delete-by-id &db &1))
```

### Aggregates

```clojure
; Sum of a numeric column
(match (Item.sum &db "id")
  (Result.Success total) (println* total)
  (Result.Error e) (IO.errorln &e))

; Average with a WHERE clause
(match (Item.avg-where &db "id" "done = ?1" &[(to-sqlite3 0)])
  (Result.Success average) (println* average)
  (Result.Error e) (IO.errorln &e))

; Min and max
(ignore (Item.min-val &db "id"))
(ignore (Item.max-val &db "id"))
```

`sum`, `avg`, `min-val`, and `max-val` take a column name and return
`(Result Double String)`. Each has a `-where` variant that accepts a
WHERE clause and bound parameters. Results are always returned as
`Double`, even for integer columns. Empty tables or unmatched WHERE
clauses return 0.0.

### Transactions

The ORM provides transaction support through macros that work with any
backend.

#### Manual control

```clojure
(ignore (ORM.begin SQLiteBackend &db))
(ignore (Todo.insert &db &(Todo.init 0 @"buy milk" false)))
(ignore (ORM.commit SQLiteBackend &db))
```

`ORM.begin`, `ORM.commit`, and `ORM.rollback` each take a backend module
and a database reference, returning `(Result () String)`.

#### Automatic rollback

`ORM.with-transaction` begins a transaction, evaluates a body expression,
and commits on success or rolls back on error:

```clojure
(ORM.with-transaction SQLiteBackend &db
  (do
    (ignore (Todo.insert &db &item1))
    (Todo.insert &db &item2)))
```

The body must evaluate to `(Result a String)`. On `Success` the
transaction is committed and the value is returned. On `Error` (or if
the commit itself fails) the transaction is rolled back and the error
is propagated.

## Generated functions

Given `(derive-model T Backend [pk-field Pk])`, the macro adds the
following functions to the `T` module:

| Function             | Type                                           |
|----------------------|------------------------------------------------|
| `create-table`       | `(Fn [&Backend.Db] (Result () String))`        |
| `insert`             | `(Fn [&Backend.Db &T] (Result Long String))`   |
| `find-all`           | `(Fn [&Backend.Db] (Result (Array T) String))` |
| `find-by-id`         | `(Fn [&Backend.Db &Pk] (Result T String))`     |
| `find-where`         | `(Fn [&Backend.Db &String &(Array Backend.Type)] (Result (Array T) String))` |
| `find-ordered`       | `(Fn [&Backend.Db &String] (Result (Array T) String))` |
| `find-where-ordered` | `(Fn [&Backend.Db &String &(Array Backend.Type) &String] (Result (Array T) String))` |
| `find-page`          | `(Fn [&Backend.Db &String Int Int] (Result (Array T) String))` |
| `find-where-page`    | `(Fn [&Backend.Db &String &(Array Backend.Type) &String Int Int] (Result (Array T) String))` |
| `update`             | `(Fn [&Backend.Db &T] (Result () String))`     |
| `delete-by-id`       | `(Fn [&Backend.Db &Pk] (Result () String))`    |
| `sum`                | `(Fn [&Backend.Db &String] (Result Double String))` |
| `sum-where`          | `(Fn [&Backend.Db &String &String &(Array Backend.Type)] (Result Double String))` |
| `avg`                | `(Fn [&Backend.Db &String] (Result Double String))` |
| `avg-where`          | `(Fn [&Backend.Db &String &String &(Array Backend.Type)] (Result Double String))` |
| `min-val`            | `(Fn [&Backend.Db &String] (Result Double String))` |
| `min-val-where`      | `(Fn [&Backend.Db &String &String &(Array Backend.Type)] (Result Double String))` |
| `max-val`            | `(Fn [&Backend.Db &String] (Result Double String))` |
| `max-val-where`      | `(Fn [&Backend.Db &String &String &(Array Backend.Type)] (Result Double String))` |

For composite-PK models (`[pk1 T1 pk2 T2 ...]`), `insert` returns
`(Result () String)` (no rowid), and `find-by-id`/`delete-by-id` accept
one argument per PK field (`&Pk1 &Pk2 ...`).

## Backends

A backend is a module that defines six `defndynamic` helpers. The ORM
macro calls them at expansion time to build SQL strings and row marshalling
code.

```clojure
(defmodule MyBackend
  (defndynamic sql-type [t] ...)           ; Carp type -> SQL type string
  (defndynamic placeholder [n] ...)        ; parameter placeholder, 1-indexed
  (defndynamic query-fn [] ...)            ; static function the generated code calls
  (defndynamic last-insert-id-sql [] ...)  ; SQL to fetch the last inserted id
  (defndynamic extract-col [t var] ...)    ; form extracting a value from an owned col variable
  (defndynamic bind-value [t expr] ...)    ; form converting an expression for binding
  (defndynamic begin-sql [] ...)           ; SQL to begin a transaction
  (defndynamic commit-sql [] ...)          ; SQL to commit a transaction
  (defndynamic rollback-sql [] ...)        ; SQL to roll back a transaction
)
```

The `t` passed to `sql-type`, `extract-col`, and `bind-value` is the
field's declared type, so it is a list `(Maybe T)` for a nullable field
and a bare symbol otherwise.

`backends/sqlite3.carp` is the reference implementation. It is small
(under 150 lines) and a good starting point for a new backend.

## Limitations

- At least one non-PK field must be present, since `insert` and `update`
  bind data columns. A table of only PK fields raises a macro error.
- `update` overwrites all non-PK fields. The typical workflow is
  `find-by-id`, mutate the result, then `update`.
- The macro only understands the Carp value types the chosen backend
  registers. The SQLite backend supports `Int`, `Long`, `Double`,
  `Float`, `String`, and `Bool`, plus `(Maybe T)` of any of those.
  Anything else raises a macro error at expansion time.
- `with-transaction` requires the body to return a `(Result a String)`.
  Use `do` to group multiple expressions.

## Testing

```
carp -x test/orm.carp
```

<hr/>

Have fun!
