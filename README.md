# libsqlite3-sys

SQLite is a C library that implements a small, fast, self-contained SQL
database engine. It reads and writes directly to ordinary files on disk,
and a complete database with its tables, indices and views is one file.
The library's interface is documented in
[the SQLite C/C++ API reference](https://sqlite.org/c3ref/intro.html).
This package declares forty-eight of that interface's entry points to
novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in libsqlite3. The package contains no logic
of its own, and it does nothing without the C library installed. The
forty-eight entry points are the ones a program needs to open a
database, run statements against it and read the answers; the section
"What is not included" says what a program still cannot do with them
alone.

## What it is

A **connection** is a handle on one open database. `sqlite3_open`
creates one from a filename and `sqlite3_close` releases it. The special
filename `":memory:"` asks for a private database held in memory, which
disappears when the connection closes.

A **prepared statement** is one SQL statement compiled into a form the
engine can run. `sqlite3_prepare_v2` compiles the text and answers a
handle. Compiling once and running many times is cheaper than
recompiling, and it is the only way to pass values into a statement
without building SQL text out of them.

A **parameter** is a placeholder in the statement text that a value is
supplied for at run time. It is written `?`, `?N`, `:name`, `@name` or
`$name`. The `sqlite3_bind_*` family supplies the values, and the index
of the leftmost parameter is 1.

Running a statement is a loop. `sqlite3_step` advances it. It answers
`SQLITE_ROW` (100) when a result row is ready, and the `sqlite3_column_*`
family reads that row's columns. It answers `SQLITE_DONE` (101) when
there are no more rows. `sqlite3_reset` puts the statement back to the
start so it can be run again, and `sqlite3_finalize` destroys it.

A **storage class** is the kind of value actually stored in a column.
SQLite has five of them, and a column may hold a different one in every
row.

| Code | Storage class | What it holds |
| --- | --- | --- |
| 1 | INTEGER | A signed integer of up to 64 bits. |
| 2 | FLOAT | An IEEE-754 double. |
| 3 | TEXT | A string. |
| 4 | BLOB | A block of bytes, stored exactly as given. |
| 5 | NULL | No value. |

A **result code** is the integer every fallible entry point answers.
Zero is `SQLITE_OK`. The primary codes run from 0 to 28.
`sqlite3_extended_result_codes` turns on the extended codes, which carry
the primary code in their low eight bits and a reason in the rest.

## Install

```
novo pkg add libsqlite3-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package
`libsqlite3-dev`:

```
sudo apt install libsqlite3-dev
```

On macOS the library ships with the operating system, and the Homebrew
formula is `sqlite`. On other systems it builds from the SQLite
amalgamation, a single C file the project publishes for that purpose.

## Example

One in-memory database, one table, one row read back:

```novo
use libsqlite3

fn main() [io, ffi]
    // sqlite3_open writes the connection handle into a slot the
    // caller owns.
    let slot = ptr.alloc_word()
    let rc = libsqlite3.sqlite3_open(":memory:", slot)
    let db = ptr.read_word(slot)
    if rc != 0
        println("open failed: " + ptr.read_str(libsqlite3.sqlite3_errmsg(db)))
        return
    let create = "CREATE TABLE city (name TEXT, people INTEGER)"
    let _ = libsqlite3.sqlite3_exec(db, create, 0, 0, 0)
    let insert = "INSERT INTO city VALUES ('Aarhus', 285273)"
    let _ = libsqlite3.sqlite3_exec(db, insert, 0, 0, 0)

    // A prepared statement is compiled once and stepped to its rows.
    let stmt_slot = ptr.alloc_word()
    let _ = libsqlite3.sqlite3_prepare_v2(db, "SELECT name, people FROM city", -1,
                                          stmt_slot, 0)
    let stmt = ptr.read_word(stmt_slot)
    // 100 is SQLITE_ROW: a row is ready and its columns may be read.
    while libsqlite3.sqlite3_step(stmt) == 100
        let name = ptr.read_str(libsqlite3.sqlite3_column_text(stmt, 0))
        let people = libsqlite3.sqlite3_column_int64(stmt, 1)
        println("${name} has ${people} people")

    let _ = libsqlite3.sqlite3_finalize(stmt)
    let _ = libsqlite3.sqlite3_close(db)
    ptr.free(stmt_slot)
    ptr.free(slot)
```

## What the package contains

| Module | Contents |
| --- | --- |
| `libsqlite3` | Every entry point, in six groups: the connection, the prepared statement, the parameter bindings, the result columns, the error reporting and the library. |

The six groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Connection | 10 | Opens and closes a database, runs statements in one call, and reads the connection's own state. |
| Prepared statement | 8 | Compiles one statement, advances it, resets it and destroys it. |
| Parameter bindings | 9 | Supplies a value for each placeholder, and looks a placeholder up by name or by number. |
| Result columns | 9 | Reads the columns of the current row, and names them. |
| Error reporting | 7 | Reads the code and the message for the last failure, and counts the rows changed. |
| Library | 5 | Reports the version, releases memory the library allocated, and tests whether a string is a complete statement. |

## How to choose an entry point

`sqlite3_exec` runs one or more semicolon-separated statements in a
single call. Use it for statements with no parameters and no results:
`CREATE TABLE`, `BEGIN`, `PRAGMA`. It is the shortest path and it
compiles the text every time.

`sqlite3_prepare_v2` and the step loop are for everything else. Use them
whenever a value comes from outside the program, because binding a value
is the only way to keep it out of the SQL text, and whenever the same
statement runs more than once.

`sqlite3_prepare_v3` is `sqlite3_prepare_v2` with a flags argument. Use
it only to pass `SQLITE_PREPARE_PERSISTENT` for a statement the program
keeps for its whole run.

`sqlite3_close_v2` is `sqlite3_close` for a program that cannot easily
finalize every statement first. `sqlite3_close` answers `SQLITE_BUSY`
and leaves the connection open in that case; `sqlite3_close_v2` accepts
the close and completes it when the last statement is finalized.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **An out-parameter is the address of a caller-owned slot.**
   `sqlite3_open` and `sqlite3_prepare_v2` write their handles into a
   slot rather than returning them. `ptr.alloc_word` in the standard
   library returns the address of such a slot, `ptr.read_word` reads it
   back, and `ptr.free` releases it.
3. **`sqlite3_open` writes a handle even when it fails.** The connection
   must be closed either way, and `sqlite3_errmsg` on it is what says
   what went wrong. SQLite C/C++ API reference, `sqlite3_open`.
4. **A parameter index starts at 1 and a column index starts at 0.**
   The two families disagree, and an off-by-one is not an error: a bind
   answers `SQLITE_RANGE` (25), and a column read out of range answers
   a zero or a null address.
5. **A string a column read returns is owned by the statement.** It is
   valid until the next `sqlite3_step`, `sqlite3_reset` or
   `sqlite3_finalize` on that statement. Copy it with `ptr.read_str`
   before any of those. SQLite C/C++ API reference, `sqlite3_column_blob`.
6. **`sqlite3_column_bytes` is called after the value is read, not
   before.** Reading the column is the step that fixes the length,
   because a conversion between storage classes can change it.
7. **The text and blob binds take a destructor argument.** Pass 0,
   `SQLITE_STATIC`, only when the bytes will outlive the statement.
   Pass -1, `SQLITE_TRANSIENT`, to have libsqlite3 take its own copy
   before the call returns. A novo-lang string is managed by the
   novo-lang runtime, so -1 is the safe value for it.
8. **The error message from `sqlite3_exec` is released with
   `sqlite3_free`.** It is the one address a caller gets that libsqlite3
   allocated and does not own afterwards.
9. **A negative length means "up to the terminator".** Every entry point
   that takes a byte length accepts a negative number in place of one,
   and reads the string up to its first zero byte.
10. **A connection is not safe to use from two threads at once** unless
    the library was built in serialized threading mode. Building for one
    connection per thread avoids the question.

## What is not included

- **Every entry point that passes or returns a structure by value.**
  The novo-lang foreign function interface passes integers, floats and
  strings, and nothing else.
- **The user-defined function surface.** `sqlite3_create_function_v2`,
  the whole `sqlite3_value_*` family and the whole `sqlite3_result_*`
  family exist to let a caller add SQL functions written in C. They take
  C function pointers, and a novo-lang function is not one.
- **The collation, authorizer, hook and tracing callbacks.**
  `sqlite3_create_collation`, `sqlite3_set_authorizer`,
  `sqlite3_update_hook`, `sqlite3_commit_hook`, `sqlite3_busy_handler`
  and `sqlite3_trace_v2` all take function pointers, for the same
  reason.
- **The incremental blob interface.** `sqlite3_blob_open`,
  `sqlite3_blob_read` and `sqlite3_blob_write` read and write a single
  blob without loading it. They are left out of the first release, and
  `sqlite3_bind_zeroblob` is here because it is how space for one is
  reserved.
- **The backup and serialization interfaces.** `sqlite3_backup_init`
  and `sqlite3_serialize` copy a whole database. They are left out of
  the first release.
- **The UTF-16 entry points.** Every `*16` variant is absent. A
  novo-lang string is UTF-8, and the UTF-8 entry point is the direct
  one.
- **The virtual table interface.** It is a set of structures with
  function pointers in them, and it has no shape a binding can carry.

## Related packages

`sql-engine-nv` is the SQL parser and query engine extracted from
novodb, written in novo-lang with no C library, and `pager-nv` is the
page cache and file format beneath it. Together they are the native
answer this package is the escape hatch from. They are planned and not
published yet, and the native port is read-only for now, so a program
that has to write uses this package.

Choose `sql-engine-nv` and `pager-nv` when the program must build for a
microcontroller or for WebAssembly, or when a C toolchain is not wanted.
Choose this package when the program needs the whole of SQLite's SQL
dialect, or has to write to a database file that other SQLite programs
read.

## Tests

`tests/libsqlite3_tests.nv` holds ten tests written against the
signatures. They call the C library, so `novo test` needs libsqlite3
installed and linkable:

```
novo test tests/libsqlite3_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

Every database in the suite is `":memory:"`, so the tests create no
file and leave nothing behind. They assert what the reference specifies:
that a fresh connection is in autocommit mode, that a statement
compiles and steps to `SQLITE_ROW` and then `SQLITE_DONE`, that a bind
past the last parameter answers `SQLITE_RANGE`, that each of the five
storage classes reads back with the code the reference gives it, and
that a failing `sqlite3_exec` writes a message the caller frees.

## Implementation status

| Group | State |
| --- | --- |
| Connection | Complete for the open, close and exec path. |
| Prepared statement | Complete. |
| Parameter bindings | Complete except the 64-bit length and pointer variants. |
| Result columns | Complete except the UTF-16 variants and the source-table lookups. |
| Error reporting | Complete. |
| Library | Complete. |
| User-defined functions | Absent. Every entry point takes a function pointer or an `sqlite3_value`. |
| Incremental blob | Absent. Left out of the first release. |
| Backup and serialization | Absent. Left out of the first release. |
| Virtual tables | Absent. The interface is structures of function pointers. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

SQLite itself is in the public domain, and installing it is the reader's
own step.
