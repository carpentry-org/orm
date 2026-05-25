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
(load "git@github.com:carpentry-org/orm@0.1.0" "backends/sqlite3.carp")
```

If you are writing your own backend you can load just the core:

```clojure
(load "git@github.com:carpentry-org/orm@0.1.0")
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
wrapped in a `Result`. The PK field on the input row is ignored, which
is why we pass `0`.

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

| Function       | Type                                           |
|----------------|------------------------------------------------|
| `create-table` | `(Fn [&Backend.Db] (Result () String))`        |
| `insert`       | `(Fn [&Backend.Db &T] (Result Int String))`    |
| `find-all`     | `(Fn [&Backend.Db] (Result (Array T) String))` |
| `find-by-id`   | `(Fn [&Backend.Db &Pk] (Result T String))`     |
| `find-where`   | `(Fn [&Backend.Db (Ref String) (Ref (Array Backend.Type))] (Result (Array T) String))` |
| `update`       | `(Fn [&Backend.Db &T] (Result () String))`     |
| `delete-by-id` | `(Fn [&Backend.Db &Pk] (Result () String))`    |

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

`backends/sqlite3.carp` is the reference implementation. It is small
(under 100 lines) and a good starting point for a new backend.

## Limitations

- At least one non-PK field must be present, since `insert` and `update`
  bind data columns. A table of only PK fields raises a macro error.
- `update` overwrites all non-PK fields. The typical workflow is
  `find-by-id`, mutate the result, then `update`.
- The macro only understands the Carp value types the chosen backend
  registers. Anything else raises a macro error at expansion time.
- `with-transaction` requires the body to return a `(Result a String)`.
  Use `do` to group multiple expressions.

## Testing

```
carp -x test/orm.carp
```

<hr/>

Have fun!
