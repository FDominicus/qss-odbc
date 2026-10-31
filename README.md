# qss-odbc — ODBC databases from Emacs Lisp

Any database with an ODBC driver — PostgreSQL, MariaDB, SQL Server,
Access (Windows), SQLite … — from Emacs Lisp. Emacs has no FFI, so the
calls into ODBC live in a small dynamic module (`module/qss-odbc-module.c`);
`qss-odbc.el` is the library on top of it.

```elisp
(require 'qss-odbc)
(setq con (qss-odbc-connect "test"))          ; a DSN, or a connection string
(qss-odbc-query con "SELECT id, name FROM person WHERE id > ?" 10)
;; => (("id" "name") (11 "Müller") (12 "Ærø"))
(qss-odbc-do-query ((id name) con "SELECT id, name FROM person")
  (message "%s %s" id name))
(qss-odbc-with-transaction con
  (qss-odbc-execute con "UPDATE person SET name = ? WHERE id = ?" "X" 11))
(qss-odbc-tables con)
(qss-odbc-table-columns con "person")      ; optional: a schema
(qss-odbc-primary-key con "person" "public")
(qss-odbc-disconnect con)
```

[qss](https://github.com/FDominicus/qss) uses it as a source of tables
with the vocabulary of Nushell: `(qss-> (odbc "test" "person") (where (> :id 10)))`.

## Install

Put the directory on the `load-path`, then build the module once:

```
M-x qss-odbc-build-module
```

It runs `make` in `module/` and loads the result. It needs a C compiler
and the ODBC headers (Debian/Ubuntu: `unixodbc-dev`; macOS: unixODBC or
iODBC, `ODBC=iodbc`; Windows: MinGW-w64, the driver manager is built in).
If `emacs-module.h` is not installed, set `qss-odbc-make-arguments` to
`("EMACS_INCLUDE=/path/to/emacs/src")`. Emacs 29.1 or later, built with
modules (`module-file-suffix` is not nil).

## Values

From the database:

| Type | Lisp |
|---|---|
| NULL | `nil` |
| integers | integers (bignums if need be) |
| REAL, FLOAT, DOUBLE | floats |
| NUMERIC(p,s), DECIMAL | integers scaled by 10^s: 1094.80 in a `NUMERIC(10,2)` is `109480`; `qss-odbc-decimal-string` shows it |
| BIT, BOOLEAN | `t` or `nil` |
| DATE, TIME, TIMESTAMP | text: `"2026-10-30"`, `"09:56:00"`, `"2026-10-30 09:56:00.123456"`, the wall clock as stored; `qss-odbc-parse-time` makes a Lisp time |
| binary | unibyte strings |
| text, everything else | strings |

Parameters (the `?` in the SQL): `nil` is NULL, `t` TRUE, integers,
floats and strings as such, and `(qss-odbc-decimal 109480 2)`,
`(qss-odbc-binary BYTES)`, `(qss-odbc-bool nil)`, `(qss-odbc-date "2026-10-30")`,
`(qss-odbc-timestamp "2026-10-30 09:56")`.

Errors are signalled as `qss-odbc-error` with the message, the SQLSTATE,
the native error and all diagnostic records (`qss-odbc-error-sqlstate`).
Connections start with autocommit on.

## Drivers

Unicode drivers get UTF-16 (the W functions). ANSI drivers that expect
UTF-8 — SQLite, and psqlODBC's ANSI driver, often configured by mistake
under the name "PostgreSQL Unicode" — get SQL text, parameters,
character columns and messages as UTF-8 bytes; through the W functions
unixODBC would turn them into Latin-1, and psqlODBC ANSI rejects UTF-16
parameters (HYC00 "Unrecognized C_parameter type"). Which drivers count
as such: `qss-odbc-ansi-utf-8-drivers`, or `:utf-8 t` at `qss-odbc-connect`.

What the tests found: the SQLite driver reports VARCHAR through
SQLColAttribute for BIGINT and BLOB columns (the type comes from
SQLDescribeCol), keeps 15 digits of a double and has no exact numbers;
psqlODBC reports booleans as VARCHAR "1"/"0" (the type name decides) and
splits long messages into several records. MariaDB and MySQL have no
boolean type: BOOLEAN is TINYINT(1) and comes back as 1 or 0; their TEXT
and BLOB end at 64 KB (LONGTEXT, LONGBLOB hold more), TIMESTAMP covers
1970 to 2038 only (DATETIME more). MariaDB Connector/ODBC before 3.1.24
and 3.2.10 miscounts characters U+xxD8 to U+xxDB (such as "Ø", MariaDB
ODBC-502).

The rules (types, lengths in which units, buffer sizes) follow
plain-odbc in the QSS version for Common Lisp, tested against Access,
SQLite, PostgreSQL and MariaDB.

## Tests

```
QSS_ODBC_DSNS="test mytest sqlitetest" emacs --batch -L . -l qss-odbc-test -f ert-run-tests-batch-and-exit
```

Unit tests run always; database tests against each DSN in
`QSS_ODBC_DSNS` (default `test`), skipped with the reason if the module
is not built or the DSN not reachable. They create and drop the table
`qss_odbc_t`. Tested here with PostgreSQL 16 (psqlODBC Unicode and ANSI,
also with a database that sorts by German rules), MariaDB 10.11
(Connector/ODBC 3.1.15) and SQLite3 ODBC; the module under valgrind
without errors or leaks.

## Manual

`doc/qss-odbc-manual.pdf` (about 10 pages): building, connecting,
values, drivers, tests. Build it with
`cd doc && lualatex qss-odbc-manual.tex && lualatex qss-odbc-manual.tex`
(LuaLaTeX, KOMA-Script, DejaVu Sans Mono).

## License

MIT, see LICENSE.
