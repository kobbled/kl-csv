# lib/csv

FANUC Karel CSV library — read and write typed robot data (INTEGER, REAL, VECTOR, XYZWPR) to and from CSV files on the controller filesystem.

## Overview

Karel has no serialization built-ins. `lib/csv` fills that gap for tabular data: it wraps `lib/files` with type-aware read/write helpers and an optional OOP class that buffers writes to a pipe file before flushing to disk. Reach for this module whenever you need to persist structured data across power cycles, exchange data between Karel tasks, or load pre-computed slicer paths from offline tools.

The module has two APIs:
- **Procedural** (`csv__*`): lightweight, caller manages the `FILE` handle
- **OOP class** (`csvclass`): encapsulates the file handle and an internal pipe buffer; better for concurrent or long-running writes

---

## Files

| File | Purpose |
|------|---------|
| `include/csv.klt` | `CSV_STRUCT` type definition, constants, predefined headers |
| `include/csv.klh` | Procedural public API declarations (26 routines) |
| `include/csv.private.klh` | Private helper declaration |
| `include/class/csvclass.klh` | OOP class public method declarations (44 routines) |
| `include/class/csvclass.private.klh` | OOP private method declarations |
| `include/class/csvclass.klc` | OOP class implementation (~460 lines) |
| `include/class/csvclass.friends.klt` | Macro for friend access to internal parsed data |
| `src/csv.kl` | Procedural implementation (~254 lines) |
| `test/write_test/tst_csv_main.kl` | Write test harness (concurrent tasks) |
| `test/write_test/tst_csv_pipe.kl` | Writer task (random XYZWPR to pipe) |
| `test/write_test/tst_csv_writ.kl` | Flush-monitor task |
| `test/read_test/tst_csv_rdln.kl` | Simple sequential read demo |
| `test/read_test/tst_csv_typ.kl` | Typed multi-field structured read demo |
| `test/read_test/read_data.csv` | 50 XYZWPR test rows |
| `test/read_test/read_typ_data.csv` | 506 complex rows (layer/code/depth/pose/rotary) |

---

## Constants

```karel
CSV_MAX_COLUMNS      = 12       -- max columns per row
CSV_CELL_LENGTH      = 16       -- max chars per cell
CSV_LINE_CHAR_LENGTH = 127      -- max chars per line

CARTESIAN_HEADER             = 'x,y,z'
CYLINDRICAL_HEADER           = 'theta,z,r'
CYLINDRICAL_SCANNING_HEADER  = 'theta,r,z'
POSITION_HEADER              = 'x,y,z,w,p,r'
```

---

## API Reference

### Procedural API (`csv.klh`)

Include with `%from csv.klh %import <routines>` or include the full header.

#### File Lifecycle

```karel
csv__create(filename : STRING; fl : FILE)
    -- Create file if it does not exist

csv__clear(filename : STRING; fl : FILE)            -- alias: csv__clr
    -- Truncate to zero bytes

csv__open(filename : STRING; access_typ : STRING; fl : FILE)
    -- Open with mode: 'RO' (read-only), 'RW' (read-write), 'AP' (append), 'UD' (update)

csv__close(filename : STRING; fl : FILE)
    -- Close file handle

csv__append_open(filename : STRING; fl : FILE)       -- alias: csv__apopen
csv__write_open(filename : STRING; fl : FILE)        -- alias: csv__rwopen
    -- Convenience: open for append / overwrite (issue CLR_IO_STAT automatically)
```

#### Write

```karel
csv__write_string(filename, fl, str : STRING)                              -- alias: csv__wrtstr
csv__write_row_string(filename, fl, delimeter, row : ARRAY[*] OF STRING)  -- alias: csv__rowstr
csv__write_row_val(filename, fl, delimeter, row : ARRAY[*] OF REAL)       -- alias: csv__rowvals
csv__write_row_vector(filename, fl, delimeter, row : VECTOR)              -- alias: csv__rowvec
csv__write_row_xyzwpr(filename, fl, delimeter, row : XYZWPR)             -- alias: csv__rowpos
csv__write_pipe(pipname, pip_fl, filename, write_fl, handle_file)         -- alias: csv__wrtpipe
    -- Flush a pipe file to a disk file; handle_file=TRUE closes disk file after flush
csv__write_to_display(filename, fl)                                        -- alias: csv__wrtdisp
    -- Print file contents to TP display
```

#### Read

```karel
csv__read_open(filename : STRING; fl : FILE)          -- alias: csv__rdopen
csv__read_close(fl : FILE)                            -- alias: csv__rdclose

csv__read_line(filename, fl) : STRING                 -- alias: csv__rdline
    -- Read next line; empty string at EOF

csv__read_line_ref(filename, fl, out_str) : BOOLEAN   -- alias: csv__rdlner
    -- Read next line into out_str; returns FALSE at EOF (use this in loops)

csv__read_rarr(filename, fl, delimeter, out_arr[], uninit_arr[]) : BOOLEAN   -- alias: csv__rdarr
csv__read_iarr(filename, fl, delimeter, out_arr[], uninit_arr[]) : BOOLEAN   -- alias: csv__idarr
csv__read_sarr(filename, fl, delimeter, out_arr[], uninit_arr[]) : BOOLEAN   -- alias: csv__sdarr
    -- Parse next delimited row into typed array; returns FALSE at EOF
    -- uninit_arr: same-size array with uninitialized values (required by files__read_type)
```

---

### OOP Class API (`csvclass.klh`)

Instantiate with:
```
%class mycsv('csvclass.klc', 'csvclass.klh')
```
All methods are then available as `mycsv__<method>`.

#### Lifecycle

```karel
mycsv__new(filename : STRING; delim : STRING; header : STRING)
    -- Constructor. Pass header='' to skip headers.
mycsv__delete
mycsv__clear
mycsv__open / open_read / open_append / open_write
mycsv__close
mycsv__isopen : BOOLEAN
mycsv__isclosed : BOOLEAN
mycsv__check_read : BOOLEAN    -- validate state before reading; posts error if invalid
```

#### Getters / Setters

```karel
mycsv__get_filename : STRING
mycsv__get_line : STRING         -- line buffer after most recent read_line
mycsv__get_fieldnames : STRING   -- comma-delimited header string
mycsv__get_maxlines : INTEGER    -- pipe buffer capacity
mycsv__get_lines_written : INTEGER
mycsv__get_lines_read : INTEGER
mycsv__get_data : t_DATA_TYPE    -- last parsed cell value
mycsv__copy_data : t_DATA_TYPE   -- deep copy of last parsed cell

mycsv__set_delimeter(delim : STRING)
mycsv__set_fieldnames(header : STRING)
mycsv__set_maxlines(maxlines : INTEGER)
```

#### Write

```karel
mycsv__write_fieldnames
mycsv__write_string(str : STRING)
mycsv__write_row_string(row : ARRAY[*] OF STRING)
mycsv__write_row_val(row : ARRAY[*] OF REAL)
mycsv__write_row_vector(row : VECTOR)         -- writes x,y,z
mycsv__write_row_xyzwpr(row : XYZWPR)        -- writes x,y,z,w,p,r
mycsv__write_to_file(filename, fl, handle_file : BOOLEAN)
    -- Flush pipe buffer to disk file; handle_file=TRUE closes fl after flush
mycsv__write_to_display
```

#### Read

```karel
mycsv__read_header             -- parse first line as column names
mycsv__read_line : BOOLEAN     -- advance cursor; FALSE = EOF

mycsv__read_int(offset : INTEGER) : INTEGER
mycsv__read_real(offset : INTEGER) : REAL
mycsv__read_string(offset : INTEGER) : STRING
mycsv__read_bool(offset : INTEGER) : BOOLEAN
mycsv__read_vector(offset : INTEGER) : VECTOR      -- columns offset, offset+1, offset+2
mycsv__read_xyzwpr(offset : INTEGER) : XYZWPR     -- columns offset..offset+5

mycsv__read_rarr(offset, size, out_arr[])
mycsv__read_iarr(offset, size, out_arr[])
mycsv__read_sarr(offset, size, out_arr[])
    -- Parse 'size' columns starting at 'offset' into typed array

mycsv__read_data(offset : INTEGER; typ : INTEGER; split : STRING)
    -- Generic read; result accessible via mycsv__get_data
```

---

## Common Patterns

### 1. Write XYZWPR Poses (Procedural)

Simple case: no class overhead, caller owns the file handle.

```karel
VAR
  csv_file : FILE
  pose     : XYZWPR

csv__create('UD1:scan.csv', csv_file)
csv__clear('UD1:scan.csv', csv_file)
csv__open('UD1:scan.csv', 'AP', csv_file)
csv__write_string('UD1:scan.csv', csv_file, POSITION_HEADER)

-- write loop
csv__write_row_xyzwpr('UD1:scan.csv', csv_file, ',', pose)

csv__close('UD1:scan.csv', csv_file)
```

---

### 2. Read Rows Until EOF (Procedural)

Standard EOF loop — always use `csv__read_line_ref` (returns BOOLEAN) rather than `csv__read_line` (returns STRING) for loop control.

```karel
VAR
  fl   : FILE
  str  : STRING[127]
  pose : XYZWPR
  b    : BOOLEAN

csv__read_open('UD1:data.csv', fl)
-- skip header
str = csv__read_line('UD1:data.csv', fl)

REPEAT
  b = csv__read_line_ref('UD1:data.csv', fl, str)
  IF b THEN
    pose = s_to_xyzwpr(str, ',')
    -- process pose
  ENDIF
UNTIL(NOT b)

csv__read_close(fl)
```

---

### 3. Pipe-Buffered Write During Motion (Class API)

Used by the TOF sensor scanner: accumulate measurements in a pipe buffer, flush to disk periodically without stopping motion.

```karel
%class csvpip('csvclass.klc', 'csvclass.klh')

VAR
  csv_file : FILE
  status   : INTEGER

-- Setup
csv__create('UD1:scan.csv', csv_file)
csv__clear('UD1:scan.csv', csv_file)
csvpip__new('UD1:scan_pipe.csv', ',', CYLINDRICAL_SCANNING_HEADER)
csvpip__set_maxlines(50)

-- Motion + write loop
csv__open('UD1:scan.csv', 'AP', csv_file)
REPEAT
  csvpip__write_row_vector(measurement)
  -- recover from IO errors mid-write
  status = IO_STATUS(csv_file)
  IF status <> 0 THEN
    csv__close('UD1:scan.csv', csv_file)
    csv__open('UD1:scan.csv', 'AP', csv_file)
  ENDIF
  csvpip__write_to_file('UD1:scan.csv', csv_file, FALSE)
  csvpip__clear
UNTIL(scan_done)

-- Final flush and cleanup
csvpip__write_to_file('UD1:scan.csv', csv_file, FALSE)
csv__close('UD1:scan.csv', csv_file)
csvpip__delete
```

---

### 4. Typed Multi-Field Row Read (Class API)

Read structured records with mixed column types — the standard pattern used by `layout` and path loading.

```karel
%class tstcsv('csvclass.klc', 'csvclass.klh')
%class tstpip('csvclass.klc', 'csvclass.klh')

-- Open file, read into pipe buffer
tstcsv__new('UD1:paths.csv', ',', '')
tstcsv__open
tstcsv__read_header
tstpip__new('UD1:pipe.csv', ',', tstcsv__get_fieldnames)

REPEAT
  b = tstcsv__read_line
  IF b THEN tstpip__write_string(tstcsv__get_line) ENDIF
UNTIL(NOT b)
tstcsv__close

-- Read back with typed accessors
REPEAT
  b = tstpip__read_line
  IF b THEN
    ndes[i].layer  = tstpip__read_int(1)
    ndes[i].code   = tstpip__read_int(2)
    ndes[i].depth  = tstpip__read_real(3)
    ndes[i].pose   = tstpip__read_xyzwpr(4)   -- uses columns 4..9
    ndes[i].rotary = tstpip__read_real(10)
    i = i + 1
  ENDIF
UNTIL(NOT b)
```

---

### 5. Friend Access for Pipeline Stages

When a pipeline stage (e.g. `layout`) needs access to the parsed `data` field of a CSV class instance without re-parsing, use the friend macro.

In your `.klt` config file:
```
%include csvclass.friends.klt
csvclass_friends(MY_CSV_OBJECT)
```

This exposes `VAR data FROM MY_CSV_OBJECT : t_DATA_TYPE`, allowing direct access to the last-parsed cell value.

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Not skipping the header before reading data rows | First data row parsed as REAL gives 0.0 or junk; string coerced to number | Call `csv__read_line` once (or `read_header`) before entering the read loop |
| Using 0-based `offset` in cell accessors | Off-by-one on all reads; first field missed | `offset` is 1-based — column 1 is the first field |
| Not clearing the file before writing a new run | Old data appended to; reader sees duplicate rows | Call `csv__clear` or `csvclass::clear` before the first write |
| Row length exceeds `CSV_LINE_CHAR_LENGTH = 127` | Line silently truncated; reads parse incomplete data | Keep rows under 127 chars; reduce decimal digits via `math__decimal(val, n)` |
| Passing `handle_file=TRUE` on mid-loop `write_to_file` | File handle closed; next write fails with IO error | Use `handle_file=FALSE` for intermediate flushes; only `TRUE` on the final call |
| Multi-character delimiter | Class init posts an error; writes produce malformed rows | Delimiter must be exactly one character (e.g. `','`) |
| Column count exceeds `CSV_MAX_COLUMNS = 12` | Array index out of bounds abort on controller | Keep schema ≤ 12 columns; split into multiple files if needed |

---

## Build Flow

`lib/csv` compiles as part of the standard rossum + ninja pipeline. It has no generated classes of its own — the OOP surface comes from instantiating `csvclass.klc` with `%class` in the *consuming* module's `.klt` config.

```shell
cd lib/<your_module>
rossum .. -w -o -b    # resolve deps including csv transitively
ninja
kpush
```

To build and test csv itself:
```shell
cd lib/csv
rossum .. -w -o -t    # include test programs
ninja
kpush
```

See the [Ka-Boost readme](../../readme.md) for full build instructions, robot setup, and controller cleanup steps.
