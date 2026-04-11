# orm

A database-agnostic ORM layer for Carp.

`derive-model` reads a type's field definitions at macro-expansion time and
generates CRUD functions in the type's module. The SQL dialect and row
marshalling are delegated to a pluggable backend, so the same model
definition works against any database with a backend module.

## Installation

```clojure
(load "git@github.com:carpentry-org/orm@0.1.0")
```

The SQLite backend transitively loads
[`carpentry-org/sqlite3`](https://github.com/carpentry-org/sqlite3), so you
do not need to install it separately.

## Usage

Load a backend and derive a model:

```clojure
(load "orm/backends/sqlite3.carp")

(deftype Item [id Int text String done Bool])
(derive-model Item SQLiteBackend [id Int])

(defn main []
  (match (SQLite3.open "app.db")
    (Result.Error e) (IO.errorln &e)
    (Result.Success db)
      (do
        (Item.create-table &db)

        ; insert returns the auto-assigned rowid
        (let [new-id (Item.insert &db &(Item.init 0 @"buy milk" false))]
          (IO.println &(fmt "inserted row %d" new-id)))

        ; read
        (match (Item.find-all &db)
          (Result.Success items) (println* &items)
          (Result.Error e) (IO.errorln &e))

        ; update
        (Item.update &db &(Item.init 1 @"bought milk" true))

        ; delete
        (Item.delete-by-id &db &1)

        (SQLite3.close db))))
```

## Generated functions

Given `(derive-model T Backend [pk-field Pk])`, the macro adds the following
to the `T` module:

| Function       | Type                                           | Notes                                   |
|----------------|------------------------------------------------|-----------------------------------------|
| `create-table` | `(Fn [&Backend.Db] ())`                        | Runs `CREATE TABLE IF NOT EXISTS`.      |
| `insert`       | `(Fn [&Backend.Db &T] Int)`                    | Returns the newly assigned rowid.       |
| `find-all`     | `(Fn [&Backend.Db] (Result (Array T) String))` |                                         |
| `find-by-id`   | `(Fn [&Backend.Db &Pk] (Result T String))`     |                                         |
| `update`       | `(Fn [&Backend.Db &T] ())`                     | Writes all non-PK fields.               |
| `delete-by-id` | `(Fn [&Backend.Db &Pk] ())`                    |                                         |

The primary-key argument is passed as a reference even for value types.

## Backends

A backend is a module providing six `defndynamic` helpers that the ORM
macro calls at expansion time:

```clojure
(defmodule MyBackend
  (defndynamic sql-type [t] ...)           ; carp type → SQL type string
  (defndynamic placeholder [n] ...)        ; parameter placeholder (1-indexed)
  (defndynamic query-fn [] ...)            ; static function the generated code calls
  (defndynamic last-insert-id-sql [] ...)  ; SQL to fetch the last inserted id
  (defndynamic extract-col [t var] ...)    ; form extracting a value from an owned col variable
  (defndynamic bind-value [t expr] ...)    ; form converting an expression for binding
)
```

`backends/sqlite3.carp` is a working reference.

## Limitations

- Single-field primary key only. Composite keys raise a macro error at
  expansion time.
- At least one non-PK field must be present. A table of only PK fields
  raises a macro error.
- `update` overwrites all non-PK fields. There is no partial update — the
  caller is expected to `find-by-id`, mutate, and `update`.
- The macro only understands Carp value types that the chosen backend knows
  about. Unsupported types raise a macro error at expansion time.
- No transaction support. `begin`/`commit`/`rollback` should be added as
  backend helpers in a future version.

<hr/>

Have fun!
