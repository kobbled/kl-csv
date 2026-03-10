# lib/csv — AI Context File

## Purpose

CSV file I/O for typed FANUC robot data. Solves the problem of persisting and reading back structured data (INTEGER, REAL, STRING, VECTOR, XYZWPR) on the robot controller filesystem, where Karel has no serialization built-ins. Sits at Layer 4, above `files` (raw I/O) and below `sensors`, `layout`, and `paths` modules that consume CSV data.

Dual API: a thin **procedural** wrapper set (`csv__*`) and a richer **OOP class** (`csvclass`) that encapsulates a file handle and an internal pipe buffer for deferred writes. Both surfaces ultimately delegate to `lib/files`.

---

## Repository Layout

```
lib/csv/
├── package.json                  rossum manifest; 8 direct deps
├── readme.md                     developer-facing reference
├── CLAUDE.md                     this file
├── include/
│   ├── csv.klt                   CSV_STRUCT type, constants, predefined headers
│   ├── csv.klh                   procedural public API (26 routines)
│   ├── csv.private.klh           private: csv__read_type_array
│   └── class/
│       ├── csvclass.klh          OOP public API (44 routines)
│       ├── csvclass.private.klh  OOP private: clear_fieldnames
│       ├── csvclass.klc          OOP implementation (~460 lines)
│       └── csvclass.friends.klt  macro for friend access to internal data
├── src/
│   └── csv.kl                    procedural implementation (~254 lines)
└── test/
    ├── write_test/
    │   ├── tst_csv_main.kl       master harness (spawns pipe + flush tasks)
    │   ├── tst_csv_pipe.kl       writer task (writes random XYZWPR)
    │   └── tst_csv_writ.kl       flush-monitor task (pipes to file on full)
    └── read_test/
        ├── tst_csv_rdln.kl       simple read demo (raw lines → XYZWPR)
        ├── tst_csv_typ.kl        structured read demo (multi-field rows)
        ├── read_data.csv         50 XYZWPR test rows
        └── read_typ_data.csv     506 complex multi-field rows (layer/code/depth/pose)
```

---

## Full API Reference

### Constants (csv.klt)

```
CSV_MAX_COLUMNS      = 12       max columns per row
CSV_CELL_LENGTH      = 16       max chars per cell
CSV_LINE_CHAR_LENGTH = 127      max chars per line

CARTESIAN_HEADER              = 'x,y,z'
CYLINDRICAL_HEADER            = 'theta,z,r'
CYLINDRICAL_SCANNING_HEADER   = 'theta,r,z'
POSITION_HEADER               = 'x,y,z,w,p,r'
```

### Types (csv.klt)

```
CSV_STRUCT
  filename   : STRING[20]
  fieldnames : ARRAY[CSV_MAX_COLUMNS] OF STRING[CSV_CELL_LENGTH]
  delimeter  : STRING[1]         -- note: module uses this spelling
  columns    : INTEGER
  writeLines : INTEGER
  readLines  : INTEGER
  maxpipe    : INTEGER
```

---

### Procedural API (csv.klh / csv.kl)

All aliases follow `funcname(csv, shortname)` macro pattern.

#### Lifecycle

| Routine | Alias | Description |
|---------|-------|-------------|
| `csv__clear(filename, fl)` | `csv__clr` | Truncate file to zero bytes |
| `csv__create(filename, fl)` | `csv__create` | Create file if not exists |
| `csv__open(filename, access_typ, fl)` | `csv__open` | Open with mode: `'RO'`,`'RW'`,`'AP'`,`'UD'` |
| `csv__close(filename, fl)` | `csv__close` | Close file handle |
| `csv__append_open(filename, fl)` | `csv__apopen` | Open in append mode (CLR_IO_STAT) |
| `csv__write_open(filename, fl)` | `csv__rwopen` | Open for overwrite (CLR_IO_STAT) |

#### Write

| Routine | Alias | Description |
|---------|-------|-------------|
| `csv__write_pipe(pipname, pip_fl, filename, write_fl, handle_file)` | `csv__wrtpipe` | Flush pipe file to disk file |
| `csv__write_to_display(filename, fl)` | `csv__wrtdisp` | Print file contents to TP display |
| `csv__write_string(filename, fl, str)` | `csv__wrtstr` | Write raw string line |
| `csv__write_row_string(filename, fl, delimeter, row[])` | `csv__rowstr` | Write STRING array as CSV row |
| `csv__write_row_val(filename, fl, delimeter, row[])` | `csv__rowvals` | Write REAL array as CSV row |
| `csv__write_row_vector(filename, fl, delimeter, row)` | `csv__rowvec` | Write VECTOR (x,y,z) as CSV row |
| `csv__write_row_xyzwpr(filename, fl, delimeter, row)` | `csv__rowpos` | Write XYZWPR (x,y,z,w,p,r) as CSV row |

#### Read

| Routine | Alias | Description |
|---------|-------|-------------|
| `csv__read_open(filename, fl)` | `csv__rdopen` | Open file for reading |
| `csv__read_close(fl)` | `csv__rdclose` | Close read handle |
| `csv__read_line(filename, fl) : STRING` | `csv__rdline` | Read next line, return as STRING |
| `csv__read_line_ref(filename, fl, out_str) : BOOLEAN` | `csv__rdlner` | Read next line into out_str; FALSE = EOF |
| `csv__read_rarr(filename, fl, delimeter, out_arr[], uninit_arr[]) : BOOLEAN` | `csv__rdarr` | Parse next line into REAL array |
| `csv__read_iarr(filename, fl, delimeter, out_arr[], uninit_arr[]) : BOOLEAN` | `csv__idarr` | Parse next line into INTEGER array |
| `csv__read_sarr(filename, fl, delimeter, out_arr[], uninit_arr[]) : BOOLEAN` | `csv__sdarr` | Parse next line into STRING array |

#### Private

| Routine | Alias | Description |
|---------|-------|-------------|
| `csv__read_type_array(filename, fl, delimeter, typ, real_arr[], int_arr[], str_arr[]) : BOOLEAN` | `csv__rdtpar` | Dispatch typed array read; returns FALSE at EOF |

---

### OOP Class API (csvclass.klh / csvclass.klc)

Instantiated via `%class mycsv('csvclass.klc','csvclass.klh')`.

#### Lifecycle

| Method | Description |
|--------|-------------|
| `new(filename, delim, header)` | Constructor; sets filename, delimiter, parses header |
| `delete` | Destructor; closes file |
| `clear` | Truncate file to empty |
| `open` | Open for read/write |
| `open_read` | Open read-only |
| `open_append` | Open in append mode |
| `open_write` | Open for overwrite |
| `close` | Close file handle |
| `isopen : BOOLEAN` | Returns TRUE if file open |
| `isclosed : BOOLEAN` | Returns TRUE if file closed |
| `check_read : BOOLEAN` | Validate state for reading; posts error if invalid |

#### Getters

| Method | Returns | Description |
|--------|---------|-------------|
| `get_filename` | STRING | Stored filename |
| `get_line` | STRING | Current line buffer (after read_line) |
| `get_fieldnames` | STRING | Comma-delimited header string |
| `get_maxlines` | INTEGER | Pipe buffer capacity |
| `get_lines_written` | INTEGER | Cumulative lines written |
| `get_lines_read` | INTEGER | Cumulative lines read |
| `copy_data` | t_DATA_TYPE | Deep copy of last parsed cell |
| `get_data` | t_DATA_TYPE | Reference to last parsed cell |

#### Setters

| Method | Description |
|--------|-------------|
| `set_delimeter(delim)` | Set CSV delimiter (validated single char) |
| `set_fieldnames(header)` | Parse header string into fieldnames array |
| `set_maxlines(maxlines)` | Set pipe buffer flush threshold |

#### Write Methods

| Method | Description |
|--------|-------------|
| `write_fieldnames` | Write header row to pipe |
| `write_to_display` | Dump pipe to TP display |
| `write_to_file(filename, fl, handle_file)` | Flush pipe to disk file |
| `write_string(str)` | Append raw string to pipe |
| `write_row_string(row[])` | Write STRING array row using stored delimiter |
| `write_row_val(row[])` | Write REAL array row |
| `write_row_vector(row)` | Write VECTOR as x,y,z row |
| `write_row_xyzwpr(row)` | Write XYZWPR as x,y,z,w,p,r row |

#### Read Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `read_header` | — | Parse first line as field names |
| `read_line` | BOOLEAN | Advance to next line; FALSE = EOF |
| `read_int(offset)` | INTEGER | Cell at column `offset` |
| `read_real(offset)` | REAL | Cell at column `offset` |
| `read_string(offset)` | STRING | Cell at column `offset` |
| `read_bool(offset)` | BOOLEAN | Cell at column `offset` |
| `read_vector(offset)` | VECTOR | Columns `offset..offset+2` → x,y,z |
| `read_xyzwpr(offset)` | XYZWPR | Columns `offset..offset+5` → x,y,z,w,p,r |
| `read_rarr(offset, size, out_arr[])` | — | Parse `size` columns starting at `offset` into REAL array |
| `read_iarr(offset, size, out_arr[])` | — | Parse into INTEGER array |
| `read_sarr(offset, size, out_arr[])` | — | Parse into STRING array |
| `read_data(offset, typ, split)` | — | Generic typed read; result in `get_data` |

#### Friend Access (csvclass.friends.klt)

```
%include csvclass.friends.klt        -- must appear in .klt config
csvclass_friends(OBJECT_NAME)        -- exposes: VAR data FROM OBJECT_NAME : t_DATA_TYPE
```
Grants pipeline stages direct access to parsed `data` without re-parsing.

---

## Core Patterns

### Pattern 1: Simple Procedural Write

Write a series of XYZWPR poses to CSV without the OOP class.

```karel
-- From: lib/sensors/tof/test usage pattern
VAR
  csv_file : FILE
  pose     : XYZWPR

csv__create('UD1:scan.csv', csv_file)
csv__clear('UD1:scan.csv', csv_file)
csv__open('UD1:scan.csv', 'AP', csv_file)
csv__write_string('UD1:scan.csv', csv_file, POSITION_HEADER)
csv__write_row_xyzwpr('UD1:scan.csv', csv_file, ',', pose)
csv__close('UD1:scan.csv', csv_file)
```

### Pattern 2: Procedural Sequential Read

Read XYZWPR rows until EOF; standard loop pattern.

```karel
-- From: lib/csv/test/read_test/tst_csv_rdln.kl
VAR
  fl   : FILE
  str  : STRING[127]
  pose : XYZWPR
  b    : BOOLEAN

csv__read_open(CSV_NAME, fl)
-- consume header
str = csv__read_line(CSV_NAME, fl)
REPEAT
  b = csv__read_line_ref(CSV_NAME, fl, str)
  IF b THEN
    pose = s_to_xyzwpr(str, ',')
    WRITE TPDISPLAY(pose_to_s(pose, ','), CR)
  ENDIF
UNTIL(NOT b)
csv__read_close(fl)
```

### Pattern 3: Pipe-Buffered Async Write (Class API)

Used by TOF sensor scan_part_stat.klc — deferred writes to a pipe file, flushed periodically.

```karel
-- Setup
%class csvpip('csvclass.klc', 'csvclass.klh')

csv__create(csv_name, csv_file)
csv__clear(csv_name, csv_file)
csvpip__new(PIPE_FILE, ',', CYLINDRICAL_SCANNING_HEADER)
csvpip__set_maxlines(MAX_PIPE)

-- During scan loop
csv__open(csv_name, 'AP', csv_file)
REPEAT
  -- write measurement to pipe
  csvpip__write_row_vector(measurement)
  -- periodic flush on IO error or when full
  status = IO_STATUS(csv_file)
  IF status <> 0 THEN
    csv__close(csv_name, csv_file)
    csv__open(csv_name, 'AP', csv_file)
  ENDIF
  csvpip__write_to_file(csv_name, csv_file, FALSE)
  csvpip__clear
UNTIL(scan_done)

-- Cleanup
csvpip__write_to_file(csv_name, csv_file, FALSE)
csv__close(csv_name, csv_file)
csvpip__delete
```

### Pattern 4: Typed Multi-Field Row Read

Read structured records with mixed types; from tst_csv_typ.kl and layout module usage.

```karel
-- From: lib/csv/test/read_test/tst_csv_typ.kl
%class tstpip('csvclass.klc', 'csvclass.klh')

tstpip__new(PIPE_NAME, ',', tstcsv__get_fieldnames)

REPEAT
  b = tstpip__read_line
  IF b THEN
    ndes[i].layer   = tstpip__read_int(1)
    ndes[i].code    = tstpip__read_int(2)
    ndes[i].depth   = tstpip__read_real(3)
    ndes[i].pose    = tstpip__read_xyzwpr(4)   -- columns 4-9
    ndes[i].rotary  = tstpip__read_real(10)
    i = i + 1
  ENDIF
UNTIL(NOT b)
```

### Pattern 5: Sensor Simulation from CSV

Load pre-recorded analog values from CSV to simulate sensor input; from tst_sim_tof.kl.

```karel
%from csv.klh %import read_open, read_close, read_line, read_rarr

VAR
  fl      : FILE
  val     : ARRAY[1] OF REAL
  uninit_r: ARRAY[1] OF REAL
  b       : BOOLEAN
  head    : STRING[127]

csv__read_open(sim_file, fl)
head = csv__read_line(sim_file, fl)   -- skip header

REPEAT
  b = csv__read_rarr(sim_file, fl, ',', val, uninit_r)
  IF b THEN
    SET_PORT_VAL(io_anin, SENSOR_PIN, TRUNC(val[1]), status)
  ENDIF
UNTIL(NOT b)

csv__read_close(fl)
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgetting to skip header before reading rows | First data row parsed as REAL gives junk values (string coerced to 0.0) | Call `csv__read_line` once or `read_header` before the read loop |
| Using `offset` as 0-based column index | Off-by-one in all cell reads | `offset` is 1-based; column 1 = first field |
| Not calling `clear` before appending | Old data accumulates in file across runs | Call `csv__clear` or `csvclass::clear` before writing |
| Exceeding `CSV_LINE_CHAR_LENGTH = 127` | Line truncated silently; read parses incomplete data | Keep rows under 127 chars; reduce decimal precision via `math__decimal` |
| Calling `write_to_file` with `handle_file=TRUE` then accessing the file again | File handle closed unexpectedly; next write fails | Use `handle_file=FALSE` for mid-loop flushes; only `TRUE` at final flush |
| Using multi-char delimiter | Delimiter validation rejects it; class init errors | Delimiter must be exactly 1 character (e.g. `','`) |
| Reading past `CSV_MAX_COLUMNS = 12` | Array index out of bounds abort on controller | Ensure row column count ≤ 12; redesign schema if needed |

---

## Dependencies

### Depends On
- `files` — all underlying file I/O (open, close, read_line, write_line, etc.)
- `errors` — `CHK_STAT`, `karelError`, named error codes
- `systemlib` / `system` — `t_DATA_TYPE` union type used in class internals
- `display` — TP display output for `write_to_display`
- `multitask` — semaphore coordination in concurrent write tests
- `math` — numeric utilities for type conversions
- `Strings` — `split_str`, `r_to_s`, `s_to_r`, `pose_to_s`, `s_to_xyzwpr`, etc.
- `ktransw-macros` — `declare_function`, `funcname`, header guards

### Depended On By
- `sensors` — TOF scan_part_stat writes scan data; tst_sim_tof reads pre-recorded data
- `layout` — wraps csvclass as generic buffered file reader for slicer layer data
- `pathplan` — declares csvpth instance for debug path export
- `draw` — uses CSV for polygon data input (DXF/SVG conversion outputs CSV)

---

## Build / Integration Notes

- **Import pattern:** `%from csv.klh %import open, close, read_line, ...` (selective import) or include the full header
- **Class instantiation:** `%class mycsv('csvclass.klc', 'csvclass.klh')` — generates all method names as `mycsv__*`
- **Friend access:** modules that need to read parsed `data` fields include `csvclass.friends.klt` and call `csvclass_friends(OBJECT_NAME)` in their `.klt` config before class expansion
- **File path format:** All filenames must include device prefix: `'UD1:filename.csv'` or `'MD:filename.csv'`
- **Pipe file naming:** Use a distinct device/name from the target write file (e.g. pipe on `UD1:`, write to `UD1:out.csv`)
- **`uninit_arr` parameter:** `csv__read_rarr/iarr/sarr` require a pre-allocated array of the same size initialized to uninitialized values — used by `files__read_type` for safe initialization
