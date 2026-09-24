# Changelog

All notable changes to libsqlite3-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-24

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-15

The first release: forty-eight entry points of the libsqlite3 C API,
one `@ffi` declaration each, and no logic.

### Added

- `libsqlite3` — the whole surface, in six groups.
  - The connection: `sqlite3_open`, `sqlite3_open_v2`,
    `sqlite3_close`, `sqlite3_close_v2`, `sqlite3_exec`,
    `sqlite3_busy_timeout`, `sqlite3_db_filename`,
    `sqlite3_get_autocommit`, `sqlite3_interrupt` and
    `sqlite3_extended_result_codes`.
  - The prepared statement: `sqlite3_prepare_v2`,
    `sqlite3_prepare_v3`, `sqlite3_step`, `sqlite3_finalize`,
    `sqlite3_reset`, `sqlite3_clear_bindings`, `sqlite3_sql` and
    `sqlite3_stmt_readonly`.
  - The parameter bindings: the six `sqlite3_bind_*` value calls and
    the three parameter lookups.
  - The result columns: the count, the storage class, the four
    readers, the byte length, the column name and the declared type.
  - The error reporting: `sqlite3_errcode`,
    `sqlite3_extended_errcode`, `sqlite3_errmsg`, `sqlite3_errstr`,
    the two change counters and `sqlite3_last_insert_rowid`.
  - The library: `sqlite3_libversion`, `sqlite3_libversion_number`,
    `sqlite3_sourceid`, `sqlite3_free` and `sqlite3_complete`.
- `tests/libsqlite3_tests.nv` — ten tests over the signatures. They
  call the C library, so they need libsqlite3 installed. Every
  database in the suite is `":memory:"`, so it creates no file.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**Passing a structure by value.** The novo-lang foreign function
interface passes integers, floats and strings. Every libsqlite3 entry
point built around `sqlite3_value` — the whole user-defined function
and collation surface — takes or answers one behind a pointer whose
contents only libsqlite3 can read, and every entry point that takes a
callback takes a C function pointer, which a novo-lang function is not.
Those are absent. A program that needs them writes a small C function
of its own and calls that.
