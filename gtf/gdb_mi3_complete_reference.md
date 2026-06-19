# GDB/MI3 — The Complete Reference Guide

> **Everything you need to know about GDB's Machine Interface version 3**

---

## Table of Contents

1. [What Is GDB/MI?](#1-what-is-gdbmi)
2. [MI Versions: History and MI3 Specifically](#2-mi-versions-history-and-mi3-specifically)
3. [Starting GDB in MI3 Mode](#3-starting-gdb-in-mi3-mode)
4. [General Design Philosophy](#4-general-design-philosophy)
5. [Command Syntax (Input)](#5-command-syntax-input)
6. [Output Syntax — Every Record Type](#6-output-syntax--every-record-type)
7. [Result Records](#7-result-records)
8. [Stream Records](#8-stream-records)
9. [Asynchronous Records](#9-asynchronous-records)
10. [Thread Groups](#10-thread-groups)
11. [Context Management — Thread and Frame Options](#11-context-management--thread-and-frame-options)
12. [Breakpoint Commands](#12-breakpoint-commands)
13. [Catchpoint Commands](#13-catchpoint-commands)
14. [Program Execution Commands](#14-program-execution-commands)
15. [Stack Manipulation Commands](#15-stack-manipulation-commands)
16. [Variable Objects (varobj)](#16-variable-objects-varobj)
17. [Data Manipulation Commands](#17-data-manipulation-commands)
18. [Thread Commands](#18-thread-commands)
19. [Program Context Commands](#19-program-context-commands)
20. [File Commands](#20-file-commands)
21. [Target Manipulation Commands](#21-target-manipulation-commands)
22. [Symbol Query Commands](#22-symbol-query-commands)
23. [Tracepoint Commands](#23-tracepoint-commands)
24. [Ada Tasking Commands](#24-ada-tasking-commands)
25. [Support and Miscellaneous Commands](#25-support-and-miscellaneous-commands)
26. [MI3-Specific Breakpoint Output Format](#26-mi3-specific-breakpoint-output-format)
27. [Asynchronous and Non-Stop Modes](#27-asynchronous-and-non-stop-modes)
28. [CLI Compatibility Inside MI](#28-cli-compatibility-inside-mi)
29. [Front-End Development Guide](#29-front-end-development-guide)
30. [Complete Annotated Session Example](#30-complete-annotated-session-example)

---

## 1. What Is GDB/MI?

GDB/MI (Machine Interface) is a **line-based, machine-oriented text protocol** sitting on top of GDB. Instead of the human-readable CLI output of `(gdb) break main`, MI gives structured, parseable output designed for programs — IDEs, graphical debuggers, test harnesses — that need to drive GDB programmatically.

### Why MI Exists

When an IDE or debugger front-end needs to set a breakpoint, it must:

1. Send a command to GDB
2. Receive a deterministic response it can parse
3. Update its UI accordingly

Human-readable CLI output like:

```
Breakpoint 1 at 0x08048564: file main.c, line 68.
```

…is fragile to parse. MI output like:

```
^done,bkpt={number="1",type="breakpoint",disp="keep",enabled="y",
addr="0x08048564",func="main",file="main.c",line="68"}
```

…is structured, consistent, and straightforward to parse programmatically.

### Key Properties

- **Line-based:** Every input command and every output record is one line (though output may be long).
- **Token-based:** Every command can carry a numeric token; every response carries that token back, enabling asynchronous matching.
- **Prefix-coded:** Every output line starts with a fixed character (`^`, `~`, `@`, `&`, `*`, `+`, `=`) indicating its type.
- **Structured values:** Results use a `key="value"`, `key={nested}`, `key=[list]` format.

---

## 2. MI Versions: History and MI3 Specifically

| Version | GDB Release | Key Changes |
|---------|-------------|-------------|
| mi1     | GDB 5.1–5.3 | Original MI; deprecated |
| mi2     | GDB 6.0     | Default for many years; still supported |
| **mi3** | **GDB 9.1** | **New multi-location breakpoint output; default as of GDB 9.1** |
| mi4     | GDB 13+     | New breakpoint `script` field output |

### What Changed in MI3

**MI3 became the default in GDB 9.1** (`--interpreter=mi` now means `--interpreter=mi3`).

The most significant change is the **multi-location breakpoint output format**. In MI2, when a breakpoint resolved to multiple locations (e.g., a C++ overloaded function), GDB emitted output that was technically syntactically invalid MI. MI3 fixes this by representing multi-location breakpoints correctly as a structured list of `locations`.

#### MI2 Multi-Location (problematic):
```
^done,bkpt={number="1",...,addr="<MULTIPLE>"},
bkpt={number="1.1",addr="0x..."},
bkpt={number="1.2",addr="0x..."}
```
The repeated `bkpt` key in a result record is not valid MI syntax.

#### MI3 Multi-Location (correct):
```
^done,bkpt={number="1",type="breakpoint",disp="keep",enabled="y",
addr="<MULTIPLE>",locations=[
  {number="1.1",enabled="y",addr="0x08048500",func="foo(int)",
   file="main.cpp",fullname="/home/user/main.cpp",line="12",
   thread-groups=["i1"]},
  {number="1.2",enabled="y",addr="0x08048600",func="foo(double)",
   file="main.cpp",fullname="/home/user/main.cpp",line="18",
   thread-groups=["i1"]}
]}
```

#### Other MI3 Additions (GDB 9.1)
- Backtraces and frames include an optional `addr_flags` field.
- New MI commands: `-complete`, `-catch-throw`, `-catch-rethrow`, `-catch-catch`, `-symbol-info-functions`, `-symbol-info-types`, `-symbol-info-variables`, `-symbol-info-modules`, and more.

#### Enabling MI3 Features in Older MI Versions
If you cannot upgrade to MI3 but want the new multi-location format:
```
-fix-multi-location-breakpoint-output
```
This command backports the MI3 breakpoint format to MI2. Similarly, `-fix-breakpoint-script-output` backports the MI4 `script` field.

---

## 3. Starting GDB in MI3 Mode

```bash
# Use MI3 explicitly (recommended for front-ends)
gdb --interpreter=mi3

# Use MI (which currently means MI3 — the latest)
gdb --interpreter=mi

# Use MI2 (explicitly request older version for compatibility)
gdb --interpreter=mi2

# Common real-world invocation
gdb --interpreter=mi3 --quiet /path/to/executable
```

> **Best practice for front-ends:** Always specify the exact MI version (e.g., `--interpreter=mi3`) rather than `--interpreter=mi`. This protects your front-end against future default version bumps.

Once running, GDB prints its version info and then the prompt:
```
~"GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1\n"
~"..."
(gdb)
```
The `(gdb)` prompt signals GDB is ready for a command.

---

## 4. General Design Philosophy

### Three-Part Interaction Model

Every interaction between a front-end and GDB/MI3 involves three parts:

**1. Commands** — sent from the front-end to GDB.
**2. Responses** — sent back for each command (exactly one per command).
**3. Notifications (async records)** — sent at any time, reporting state changes not directly associated with a command.

### Result Guarantee

Every command produces **exactly one result record**. This result is:
- `^done` — success with optional result data
- `^running` — the target was successfully resumed
- `^connected` — connected to a remote target
- `^error` — failure, with `msg` field
- `^exit` — GDB is quitting

### Error Handling Policy

When a command results in `^error`, the state of GDB and the target is **undefined** — it is not necessarily rolled back. The front-end should refresh all displayed state when it receives an error response.

---

## 5. Command Syntax (Input)

### Full Grammar

```
command       → cli-command | mi-command
cli-command   → [token] cli-command nl
mi-command    → [token] "-" operation (" " option)* [" --"] (" " parameter)* nl

token         → any sequence of digits (e.g., "001", "42", "1234")
option        → "-" parameter [" " parameter]
parameter     → non-blank-sequence | c-string
operation     → any MI command name (without the leading "-")
non-blank-seq → any chars except "-", newline, `"`, and space
c-string      → `"` seven-bit-ISO-C-string-content `"`
nl            → CR | CR-LF
```

### Token

The token is an **optional sequence of digits** prepended to any command. When GDB responds, it **echoes the same token** at the start of the result record. This lets a front-end send multiple commands and match responses even if they arrive out of order.

```
# No token
-break-insert main

# With token 42
42-break-insert main
```

Response with token:
```
42^done,bkpt={number="1",...}
```

### Options vs Parameters

Options start with `-` and may take a value. Parameters are plain values after `--`:

```
-break-insert --source main.c --line 42
-stack-list-frames --thread 1 --frame 0
-data-evaluate-expression "x + y"
```

Use `--` to separate options from parameters that might look like options:
```
-data-evaluate-expression -- "-my-tricky-variable"
```

### Complete Example Commands

```
# Set breakpoint at function main
-break-insert main

# Set conditional breakpoint at line 100 with condition
-break-insert -c "x > 5" main.c:100

# Step with a thread context
--thread 2 -exec-next

# Evaluate an expression in context of thread 1, frame 0
-data-evaluate-expression --thread 1 --frame 0 "some_variable"
```

---

## 6. Output Syntax — Every Record Type

Every line of GDB/MI output begins with one of these prefix characters:

| Prefix | Record Type | Description |
|--------|-------------|-------------|
| `^` | Result record | Response to the most recent command |
| `~` | Console stream | Human-readable console output from CLI commands |
| `@` | Target stream | Output produced by the running target program |
| `&` | Log stream | Internal GDB diagnostic messages |
| `*` | Exec async record | Target state changes (stopped, running) |
| `+` | Status async record | Progress of long-running operations |
| `=` | Notify async record | State changes GDB wants the front-end to know about |

### Value Types in Output

```
result     → variable "=" value
value      → const | tuple | list
const      → c-string             # "hello\n"
tuple      → "{}" | "{" result ("," result)* "}"
list       → "[]" | "[" value ("," value)* "]"
              | "[" result ("," result)* "]"
```

### Example Value Structures

```
# Simple string
addr="0x08048564"

# Tuple (key-value record)
frame={addr="0x08048564",func="main",file="main.c",line="68"}

# List of tuples
args=[{name="argc",value="1"},{name="argv",value="0xbfc4d4d4"}]

# List of strings
changed-registers=["0","1","2","3"]
```

---

## 7. Result Records

A result record is the **direct response** to a command. It is always prefixed by the token (if any was given) followed by `^`.

### Syntax

```
result-record → [token] "^" result-class ("," result)* nl
result-class  → "done" | "running" | "connected" | "error" | "exit"
```

### `^done`

The command succeeded. Zero or more result key-value pairs follow.

```
^done
^done,value="42"
^done,bkpt={number="1",type="breakpoint",...}
^done,stack=[frame={...},frame={...}]
```

### `^running`

The target was successfully resumed. This is returned by execution commands like `-exec-run`, `-exec-continue`, `-exec-next`, etc.

```
-exec-run
^running
(gdb)
```

The `(gdb)` prompt appears *before* the `*stopped` notification — the command succeeded, and the target runs asynchronously.

### `^connected`

Used when connecting to a remote target:

```
-target-select remote localhost:1234
^connected
(gdb)
```

### `^error`

The command failed. Always includes `msg` with an error description:

```
^error,msg="Undefined MI command: rubbish"
^error,msg="No symbol \"xyz\" in current context."
^error,code="undefined-command",msg="Undefined MI command: foo"
```

The optional `code` field provides a machine-readable error category (available in newer GDB).

### `^exit`

GDB is terminating. Returned by `-gdb-exit`:

```
-gdb-exit
^exit
```

---

## 8. Stream Records

GDB maintains three output streams, all funneled through MI as stream records.

### Console Stream (`~`)

Text meant for display in a console/terminal view. This is the output of CLI commands. Always a C-quoted string.

```
~"GNU gdb (Ubuntu 12.1) 12.1\n"
~"Reading symbols from a.out...\n"
~"Breakpoint 1 at 0x804856c: file main.c, line 32.\n"
```

### Target Stream (`@`)

Output produced by the program being debugged. Only present in truly asynchronous setups (notably remote targets):

```
@"Hello from the target program\n"
```

### Log Stream (`&`)

Internal GDB diagnostics and messages. Front-ends typically display these in a debug or log panel:

```
&"warning: Source file is more recent than executable.\n"
&"During symbol reading, incomplete CFI data.\n"
```

---

## 9. Asynchronous Records

Async records are **out-of-band** notifications. They can arrive at any time — before, during, or between command responses.

### Exec Async Records (`*`)

Report **execution state changes** of the target.

#### `*running`

The target (or a thread) started running:

```
*running,thread-id="all"
*running,thread-id="1"
```

The `thread-id` is either a global thread number or `"all"` if all threads are running.

#### `*stopped`

The target (or a thread) stopped. This is the most important async record. Fields:

| Field | Description |
|-------|-------------|
| `reason` | Why execution stopped (see table below) |
| `thread-id` | The thread that triggered the stop |
| `stopped-threads` | `"all"` or list of thread IDs that are now stopped |
| `core` | Processor core where the stop occurred (if available) |
| `frame` | The current frame (see frame format below) |

**Stop reasons:**

| `reason` value | Meaning |
|----------------|---------|
| `breakpoint-hit` | Hit a regular breakpoint |
| `watchpoint-trigger` | A write watchpoint triggered |
| `read-watchpoint-trigger` | A read watchpoint triggered |
| `access-watchpoint-trigger` | An access watchpoint triggered |
| `function-finished` | `-exec-finish` completed |
| `location-reached` | `-exec-until` reached its target |
| `watchpoint-scope` | A watchpoint went out of scope |
| `end-stepping-range` | A step command completed |
| `exited-signalled` | Program exited due to signal |
| `exited` | Program exited with exit code |
| `exited-normally` | Program exited normally (code 0) |
| `signal-received` | Program received a signal (e.g., SIGSEGV) |
| `solib-event` | A shared library event triggered |
| `fork` | A fork call was detected |
| `vfork` | A vfork call was detected |
| `syscall-entry` | Entering a syscall |
| `syscall-return` | Returning from a syscall |
| `exec` | An exec call occurred |

**Examples:**

```
*stopped,reason="breakpoint-hit",disp="keep",bkptno="1",thread-id="1",
stopped-threads="all",
frame={addr="0x08048564",func="main",
       args=[{name="argc",value="1"},{name="argv",value="0xbfc4d4d4"}],
       file="main.c",fullname="/home/user/main.c",line="68",arch="i386:x86_64"}

*stopped,reason="exited-normally"

*stopped,reason="signal-received",signal-name="SIGSEGV",
signal-meaning="Segmentation fault",
frame={addr="0x00000000",func="??",args=[],arch="i386:x86_64"}

*stopped,reason="watchpoint-trigger",
wpt={number="2",exp="x"},
value={old="0",new="42"},
frame={func="main",args=[],file="main.c",line="10"}
```

### Status Async Records (`+`)

Progress of slow operations (e.g., downloading a large binary to a remote target):

```
+download,{section=".text",section-sent="512",section-size="162580",
total-sent="512",total-size="463328"}
```

Front-ends can safely discard `+` records if they don't want to show progress.

### Notify Async Records (`=`)

Supplementary state change notifications. Front-ends **should** handle these to keep their UI accurate.

#### Thread Notifications

```
=thread-group-added,id="i1"
=thread-group-removed,id="i1"
=thread-group-started,id="i1",pid="9999"
=thread-group-exited,id="i1",exit-code="0"

=thread-created,id="2",group-id="i1"
=thread-exited,id="2",group-id="i1"
=thread-selected,id="2",frame={...}
```

#### Breakpoint Notifications

```
=breakpoint-created,bkpt={number="3",type="breakpoint",...}
=breakpoint-modified,bkpt={number="3",...,times="1"}
=breakpoint-deleted,id="3"
```

#### Library Notifications

```
=library-loaded,id="/lib/libc.so.6",
target-name="/lib/libc.so.6",
host-name="/lib/libc.so.6",
symbols-loaded="0",
ranges=[{from="0xb7ee4000",to="0xb7ff9000"}]

=library-unloaded,id="/lib/libc.so.6",
target-name="/lib/libc.so.6",
host-name="/lib/libc.so.6"
```

#### Record/Replay Notifications

```
=record-started,thread-group="i1",method="full"
=record-stopped,thread-group="i1"
```

#### Parameter Change

```
=cmd-param-changed,param="print pretty",value="on"
```

#### Memory Change

```
=memory-changed,thread-group="i1",addr="0x08049860",len="4"
```

---

## 10. Thread Groups

Thread groups are a key concept in GDB/MI for handling complex debugging scenarios: multiple processes, multiple cores, remote targets.

A **thread group** represents a unit that can contain threads — typically this corresponds to a single inferior (process). GDB assigns each inferior a thread group ID like `i1`, `i2`, etc.

### Thread Group Operations

```
# List all thread groups
-list-thread-groups
^done,groups=[
  {id="i1",type="process",pid="12345",
   num-inferior="1",cores=["0"],
   executable="/home/user/a.out"}
]

# List threads in a specific group
-list-thread-groups i1
^done,threads=[
  {id="1",target-id="Thread 0x...  (LWP 12345)",
   name="main",
   frame={...},
   state="stopped",
   core="0"}
]
```

### Why Thread Groups Matter

When debugging multi-process applications or using `follow-fork-mode detach`, GDB will have multiple inferiors. A breakpoint might belong to one thread group but not another. MI3 surfaces all of this through the `thread-groups` field in breakpoint output.

---

## 11. Context Management — Thread and Frame Options

Most MI commands accept `--thread` and `--frame` options to specify which thread and stack frame the command should act in.

```
--thread thread-id   # Apply command in context of this global thread ID
--frame frame-level  # Apply command in context of this frame level (0 = innermost)
```

### Example

```
# List local variables for thread 2, frame 1 (one level up from innermost)
-stack-list-locals --thread 2 --frame 1 --simple-values

# Evaluate an expression in thread 1's innermost frame
-data-evaluate-expression --thread 1 --frame 0 "ptr->field"
```

### Thread Selection State

GDB MI historically shares a "selected thread" with the CLI. To avoid relying on global state, front-ends should always pass `--thread` and `--frame` options explicitly.

---

## 12. Breakpoint Commands

### `-break-insert`

The primary command for setting breakpoints.

**Syntax:**
```
-break-insert [-t] [-h] [-f] [-d] [-a]
              [-c condition] [-i ignore-count] [-p thread-id]
              [--source filename] [--function name] [--label name]
              [--line linenum] [location]
```

**Options:**

| Option | Meaning |
|--------|---------|
| `-t` | Temporary breakpoint (deleted after first hit) |
| `-h` | Hardware breakpoint |
| `-f` | Force/pending: create even if location can't be resolved |
| `-d` | Create disabled |
| `-a` | Create as tracepoint |
| `-c condition` | Set break condition |
| `-i count` | Set ignore count |
| `-p thread-id` | Restrict to a specific thread |

**Location examples:**

```
# Function name
-break-insert main

# File and line
-break-insert main.c:42

# Explicit location
-break-insert --source main.c --line 42

# Address
-break-insert *0x08048564

# With condition
-break-insert -c "i == 5" main.c:100

# Temporary
-break-insert -t foo

# Pending (even if symbol not loaded yet)
-break-insert -f some_dynamic_lib_function
```

**Output in MI3:**

```
^done,bkpt={
  number="1",
  type="breakpoint",
  disp="keep",
  enabled="y",
  addr="0x08048564",
  func="main",
  file="main.c",
  fullname="/home/user/main.c",
  line="68",
  thread-groups=["i1"],
  times="0",
  original-location="main"
}
(gdb)
```

**Multi-location breakpoint in MI3:**

```
-break-insert foo   # foo is a C++ overloaded function

^done,bkpt={
  number="1",
  type="breakpoint",
  disp="keep",
  enabled="y",
  addr="<MULTIPLE>",
  times="0",
  original-location="foo",
  locations=[
    {number="1.1",enabled="y",addr="0x08048500",func="foo(int)",
     file="main.cpp",fullname="/home/user/main.cpp",line="12",
     thread-groups=["i1"]},
    {number="1.2",enabled="y",addr="0x08048560",func="foo(double)",
     file="main.cpp",fullname="/home/user/main.cpp",line="20",
     thread-groups=["i1"]}
  ]
}
```

---

### `-break-delete`

```
-break-delete breakpoint-id...

-break-delete 1
^done
(gdb)

# Delete multiple
-break-delete 1 2 3
^done
```

### `-break-enable` / `-break-disable`

```
-break-enable 1
^done

-break-disable 2
^done
```

### `-break-condition`

Set or change the condition of a breakpoint:

```
-break-condition 1 "x > 10"
^done

# Remove condition (empty string)
-break-condition 1 ""
^done
```

### `-break-ignore`

Set the ignore count:

```
-break-ignore 1 5
^done
# Breakpoint 1 will be ignored the next 5 times it's hit
```

### `-break-list`

List all breakpoints and watchpoints:

```
-break-list
^done,BreakpointTable={
  nr_rows="2",nr_cols="6",
  hdr=[
    {width="3",alignment="-1",col_name="number",colhdr="Num"},
    {width="14",alignment="-1",col_name="type",colhdr="Type"},
    {width="4",alignment="-1",col_name="disp",colhdr="Disp"},
    {width="3",alignment="-1",col_name="enabled",colhdr="Enb"},
    {width="10",alignment="-1",col_name="addr",colhdr="Address"},
    {width="40",alignment="2",col_name="what",colhdr="What"}
  ],
  body=[
    bkpt={number="1",type="breakpoint",disp="keep",enabled="y",
          addr="0x08048564",func="main",file="main.c",line="68",times="1"},
    bkpt={number="2",type="watchpoint",disp="keep",enabled="y",
          addr="",what="x",times="0"}
  ]
}
```

### `-break-watch`

Create a watchpoint:

```
# Write watchpoint (default)
-break-watch x
^done,wpt={number="2",exp="x"}

# Access watchpoint (read or write)
-break-watch -a x

# Read watchpoint
-break-watch -r x
```

Watchpoint hit notification:
```
*stopped,reason="watchpoint-trigger",
wpt={number="2",exp="x"},
value={old="-268439212",new="55"},
frame={func="main",args=[],file="main.c",line="15"}
```

### `-break-after`

Same as `-break-ignore` — set the hit count before triggering:

```
-break-after 1 10
^done
```

### `-break-commands`

Set a list of commands to execute when a breakpoint is hit (in MI4, the `script` field in breakpoint info represents this):

```
-break-commands 1 "silent" "print x" "continue"
^done
```

### `-dprintf-insert`

Insert a dynamic printf breakpoint — hits the location and prints a message:

```
-dprintf-insert main.c:42 "Reached line 42, x=%d\n" x
^done,bkpt={number="3",type="dprintf",...}
```

---

## 13. Catchpoint Commands

Catchpoints are special breakpoints that trigger on events rather than locations.

### `-catch-load` / `-catch-unload`

Catch shared library load/unload events:

```
-catch-load
^done,bkpt={number="4",type="catchpoint",...}

-catch-load libfoo.so
^done,bkpt={...}
```

### `-catch-fork` / `-catch-vfork` / `-catch-exec`

```
-catch-fork
^done,bkpt={number="5",type="catchpoint",what="fork"}
```

### `-catch-syscall`

Catch system calls:

```
-catch-syscall write read
^done,bkpt={...}
```

### `-catch-throw` / `-catch-rethrow` / `-catch-catch` (added in GDB 9.1 / MI3)

Catch C++ exceptions:

```
# Catch when any exception is thrown
-catch-throw
^done,bkpt={number="6",type="catchpoint",what="exception thrown"}

# Catch when a specific type is thrown
-catch-throw -r std::runtime_error

# Catch when exceptions are caught (landed in a catch block)
-catch-catch
^done,bkpt={...}

# Catch rethrows
-catch-rethrow
^done,bkpt={...}
```

---

## 14. Program Execution Commands

These commands resume or step the inferior. They return `^running`, then (later, asynchronously) `*stopped`.

### `-exec-run`

Start the program from the beginning:

```
-exec-run
^running
(gdb)
*stopped,reason="breakpoint-hit",bkptno="1",thread-id="1",
stopped-threads="all",
frame={addr="0x08048564",func="main",
       args=[{name="argc",value="1"},{name="argv",value="0xbfc4d4d4"}],
       file="main.c",fullname="/home/user/main.c",line="68"}
(gdb)
```

Optional `--start` flag stops at the first instruction of `main`:

```
-exec-run --start
```

### `-exec-continue`

Resume execution (continue):

```
-exec-continue
^running
(gdb)
*stopped,reason="exited-normally"
(gdb)
```

Continue all threads (in non-stop mode):
```
-exec-continue --all
```

Continue a specific thread group:
```
-exec-continue --thread-group i1
```

### `-exec-next`

Step over (next line, no stepping into function calls):

```
-exec-next
^running
(gdb)
*stopped,reason="end-stepping-range",
frame={addr="0x08048583",func="main",file="main.c",line="69"},
thread-id="1",stopped-threads="all"
(gdb)
```

### `-exec-step`

Step into (next line, stepping into function calls):

```
-exec-step
^running
(gdb)
*stopped,reason="end-stepping-range",
frame={addr="0x08048600",func="callee",file="main.c",line="10"},
thread-id="1",stopped-threads="all"
```

### `-exec-next-instruction`

Step one machine instruction (nexti equivalent):

```
-exec-next-instruction
^running
(gdb)
*stopped,reason="end-stepping-range",
frame={addr="0x0804857a",func="main",file="main.c",line="68"},
thread-id="1",stopped-threads="all"
```

### `-exec-step-instruction`

Step one machine instruction, entering calls (stepi equivalent):

```
-exec-step-instruction
^running
(gdb)
```

### `-exec-finish`

Run until current function returns:

```
-exec-finish
^running
(gdb)
*stopped,reason="function-finished",
frame={addr="0x08048600",func="main",file="main.c",line="75"},
gdb-result-var="$1",return-value="42"
```

### `-exec-until`

Run until a given location is reached (or the current line is passed):

```
-exec-until main.c:90
^running
(gdb)
*stopped,reason="location-reached",
frame={addr="0x08048700",func="main",file="main.c",line="90"}
```

### `-exec-jump`

Jump to a location without executing the intervening code:

```
-exec-jump main.c:100
^running
(gdb)
```

### `-exec-interrupt`

Interrupt a running program:

```
-exec-interrupt
^done
(gdb)
*stopped,reason="signal-received",signal-name="SIGINT",
signal-meaning="Interrupt",thread-id="1",stopped-threads="all",
frame={addr="0x00007ffff7a2f9d5",func="sleep",
       file="../sysdeps/unix/syscall-template.S",line="78"}
(gdb)
```

### `-exec-return`

Make the current function return immediately (without executing more code):

```
-exec-return
^done,frame={level="0",addr="0x08048600",func="main",
             file="main.c",fullname="/home/user/main.c",line="75"}
```

---

## 15. Stack Manipulation Commands

### `-stack-list-frames`

List stack frames:

```
-stack-list-frames
^done,stack=[
  frame={level="0",addr="0x00108733",func="callee4",
         file="basics.c",fullname="/home/user/basics.c",line="8"},
  frame={level="1",addr="0x0010871f",func="callee3",
         file="basics.c",fullname="/home/user/basics.c",line="18"},
  frame={level="2",addr="0x00108709",func="callee2",
         file="basics.c",fullname="/home/user/basics.c",line="28"},
  frame={level="3",addr="0x001086e7",func="callee1",
         file="basics.c",fullname="/home/user/basics.c",line="42"},
  frame={level="4",addr="0x00108657",func="main",
         file="basics.c",fullname="/home/user/basics.c",line="74"}
]
```

Limit to a range of frames:
```
-stack-list-frames 0 2
```

### `-stack-info-frame`

Info about the current frame:

```
-stack-info-frame
^done,frame={level="0",addr="0x00108733",func="callee4",
             file="basics.c",fullname="/home/user/basics.c",line="8",
             arch="i386:x86_64"}
```

### `-stack-info-depth`

Get the depth of the stack:

```
-stack-info-depth
^done,depth="5"

# Don't count beyond 10
-stack-info-depth 10
^done,depth="5"
```

### `-stack-list-arguments`

List arguments for each frame:

```
-stack-list-arguments --simple-values
^done,stack-args=[
  frame={level="0",args=[{name="x",value="3"},{name="y",value="5"}]},
  frame={level="1",args=[{name="str",value="0x8048900 \"hello\""}]}
]
```

`--simple-values` prints only simple (non-composite) values. Alternatives: `--no-values`, `--all-values`.

### `-stack-list-locals`

List local variables in current frame:

```
-stack-list-locals --simple-values
^done,locals=[
  {name="i",value="0"},
  {name="result",value="0"}
]
```

### `-stack-list-variables`

List both locals and arguments in current frame:

```
-stack-list-variables --all-values
^done,variables=[
  {name="argc",arg="1",value="1"},
  {name="argv",arg="1",value="0xbfc4d4d4"},
  {name="i",value="0"}
]
```

### `-stack-select-frame`

Change the selected frame:

```
-stack-select-frame 2
^done
```

---

## 16. Variable Objects (varobj)

Variable objects are a high-level MI feature for building interactive variable/watch panels. They are "object-oriented" wrappers around expressions that support:

- Hierarchical display (expanding structs, arrays)
- Change detection via update
- Frozen (non-updating) objects
- Python pretty-printer integration

### `-var-create`

Create a variable object:

```
# Create fixed variable object (bound to current thread/frame)
-var-create myvar @ x
^done,name="myvar",numchild="0",value="42",type="int",
has_more="0",thread-id="1"

# Floating variable (re-evaluated in whatever the current frame is)
-var-create myvar * x
^done,name="myvar",numchild="0",value="42",type="int"
```

For composite types (structs):
```
-var-create point @ pt
^done,name="point",numchild="2",value="{...}",type="Point",
displayhint="",has_more="0"
```

### `-var-delete`

Delete a variable object (and all its children):

```
-var-delete myvar
^done,ndeleted="1"

# Delete only children, keep the root
-var-delete -c myvar
^done,ndeleted="2"
```

### `-var-set-format`

Change the display format:

```
-var-set-format myvar hex
^done,format="hexadecimal",value="0x2a"

-var-set-format myvar decimal
^done,format="decimal",value="42"

-var-set-format myvar binary
^done,format="binary",value="00101010"
```

Formats: `binary`, `decimal`, `hexadecimal`, `octal`, `natural`, `zero-hexadecimal`.

### `-var-show-format`

```
-var-show-format myvar
^done,format="hexadecimal"
```

### `-var-info-num-children`

```
-var-info-num-children point
^done,numchild="2"
```

### `-var-list-children`

Expand a composite variable object:

```
-var-list-children point
^done,numchild="2",children=[
  child={name="point.x",exp="x",numchild="0",type="int",value="3"},
  child={name="point.y",exp="y",numchild="0",type="int",value="4"}
],has_more="0"
```

With values:
```
-var-list-children --all-values point
^done,numchild="2",children=[
  child={name="point.x",exp="x",numchild="0",type="int",value="3"},
  child={name="point.y",exp="y",numchild="0",type="int",value="4"}
],has_more="0"
```

### `-var-info-type`

```
-var-info-type point
^done,type="Point"
```

### `-var-info-expression`

```
-var-info-expression point.x
^done,lang="C",exp="x"
```

### `-var-info-path-expression`

Get an expression that can be evaluated to reach this variable:

```
-var-info-path-expression point.x
^done,path_expr="pt.x"
```

### `-var-show-attributes`

```
-var-show-attributes myvar
^done,attr="editable"
```

Possible attributes: `editable`, `noneditable`.

### `-var-evaluate-expression`

Evaluate the variable object's current value:

```
-var-evaluate-expression myvar
^done,value="42"

# Evaluate in a specific format
-var-evaluate-expression -f hex myvar
^done,value="0x2a"
```

### `-var-assign`

Assign a new value to the variable:

```
-var-assign myvar 100
^done,value="100"
```

### `-var-update`

Check which variable objects have changed (and get their new values):

```
-var-update *
^done,changelist=[
  {name="myvar",value="100",in_scope="true",type_changed="false",has_more="0"}
]

# Update a specific variable only
-var-update myvar
^done,changelist=[
  {name="myvar",value="200",in_scope="true",type_changed="false",has_more="0"}
]
```

Change record fields:

| Field | Description |
|-------|-------------|
| `name` | Variable object name |
| `value` | New value (if changed) |
| `in_scope` | `"true"`, `"false"`, or `"invalid"` |
| `type_changed` | Whether the type changed |
| `new_type` | New type string (if type changed) |
| `new_num_children` | New child count (dynamic varobjs) |
| `has_more` | Whether there are more children not listed |

### `-var-set-frozen`

Freeze/unfreeze a variable object (frozen objects are never auto-updated):

```
-var-set-frozen myvar 1   # freeze
^done

-var-set-frozen myvar 0   # unfreeze
^done
```

### `-var-set-update-range`

Set a slice range for array-like variables:

```
-var-set-update-range arr 0 9   # update only indices 0-9
^done
```

### `-var-set-visualizer`

Set a Python pretty-printer for this variable object:

```
-var-set-visualizer myvar "gdb.default_visualizer"
^done
```

### Dynamic Varobjs and Pretty-Printers

When Python pretty-printers are enabled (via `-enable-pretty-printing`), GDB may create **dynamic varobjs**. These work similarly but:

- `numchild` is unreliable — check `has_more` instead
- Children may be lazily instantiated
- The `value` comes from the Python `to_string()` method

Enable pretty-printing:
```
-enable-pretty-printing
^done
```

---

## 17. Data Manipulation Commands

### `-data-evaluate-expression`

Evaluate any GDB expression:

```
-data-evaluate-expression "1 + 1"
^done,value="2"

-data-evaluate-expression "ptr->field"
^done,value="42"

-data-evaluate-expression "&my_array[5]"
^done,value="0x804b02c"
```

### `-data-disassemble`

Disassemble code:

```
# Disassemble by address range
-data-disassemble -s 0x000107c0 -e 0x000107e0 -- 0
^done,asm_insns=[
  {address="0x000107c0",func-name="main",offset="4",inst="mov 2, %o0"},
  {address="0x000107c4",func-name="main",offset="8",inst="sethi ..."},
  ...
]

# Disassemble by address (whole function)
-data-disassemble -a 0x000107c0 -- 0
^done,asm_insns=[...]

# Disassemble by source file+line
-data-disassemble -f main.c -l 32 -- 1
^done,asm_insns=[
  src_and_asm_line={line="32",file="main.c",
                   fullname="/home/user/main.c",
                   line_asm_insn=[
                     {address="0x...",func-name="main",offset="0",
                      inst="push   %rbp"}
                   ]}
]
```

The last `--` argument is the mode:
- `0`: plain disassembly
- `1`: disassembly with source
- `2`: disassembly with source, raw opcodes
- `3`: disassembly with raw opcodes only
- `4`: disassembly with source (no source filename)
- `5`: like 4 with opcodes

The `--opcodes` option (GDB 10+) provides more control: `--opcodes none`, `--opcodes bytes`, `--opcodes display`.

### `-data-read-memory`

Read raw memory:

```
-data-read-memory 0x08049860 x 4 2 4
^done,
addr="0x08049860",
nr-bytes="32",
total-bytes="32",
next-row="0x08049880",
prev-row="0x08049840",
next-page="0x08049880",
prev-page="0x08049840",
memory=[
  {addr="0x08049860",data=["0x00000000","0x00000001","0x00000002","0x00000003"]},
  {addr="0x08049870",data=["0x00000004","0x00000005","0x00000006","0x00000007"]}
]
```

Parameters: `address format word-size nr-rows nr-cols [aschar]`
- `format`: `x` (hex), `d` (decimal), `u` (unsigned), `o` (octal), `t` (binary), `a` (address), `c` (char), `f` (float)
- `word-size`: 1, 2, 4, or 8 bytes

### `-data-read-memory-bytes`

Read memory as a flat byte array:

```
-data-read-memory-bytes 0x08049860 8
^done,memory=[
  {begin="0x08049860",offset="0x00000000",end="0x08049868",
   contents="0102030405060708"}
]
```

### `-data-write-memory`

Write to memory (address, format, word-size, value):

```
-data-write-memory 0x08049860 d 4 42
^done
```

### `-data-write-memory-bytes`

Write a hex string to memory:

```
-data-write-memory-bytes 0x08049860 "0102030405060708"
^done
```

### `-data-list-register-names`

```
-data-list-register-names
^done,register-names=["eax","ecx","edx","ebx","esp","ebp","esi","edi",
"eip","eflags","cs","ss","ds","es","fs","gs",...]
```

### `-data-list-register-values`

Read current register values:

```
-data-list-register-values x
^done,register-values=[
  {number="0",value="0x0"},
  {number="1",value="0x0"},
  {number="2",value="0x0"},
  {number="3",value="0x8"},
  ...
]
```

Format codes: same as `-data-read-memory`.

### `-data-list-changed-registers`

Detect which registers changed since last step:

```
-data-list-changed-registers
^done,changed-registers=["0","1","2","4","5","8","14","15","17","18"]
```

### `-data-write-register-values`

Write to registers:

```
-data-write-register-values x [{number="0",value="0x42"}]
^done
```

---

## 18. Thread Commands

### `-thread-info`

Get info about one or all threads:

```
-thread-info
^done,threads=[
  {id="2",target-id="Thread 0x7ffff7fc6700 (LWP 14994)",
   name="worker",
   frame={level="0",addr="0x...",func="pthread_cond_wait",
          args=[],file="pthread_cond_wait.S",line="185"},
   state="stopped",core="1"},
  {id="1",target-id="Thread 0x7ffff7fc7720 (LWP 14990)",
   name="main",
   frame={level="0",addr="0x...",func="main",
          args=[],file="main.c",line="20"},
   state="stopped",core="0"}
],
current-thread-id="1"

# Info about specific thread
-thread-info 2
```

### `-thread-list-ids`

```
-thread-list-ids
^done,thread-ids={thread-id="2",thread-id="1"},
current-thread-id="1",number-of-threads="2"
```

### `-thread-select`

Select a thread as current:

```
-thread-select 2
^done,new-thread-id="2",
frame={level="0",addr="0x...",func="pthread_cond_wait",...}
```

### `-list-thread-groups`

List inferiors (thread groups):

```
-list-thread-groups
^done,groups=[
  {id="i1",type="process",pid="14990",
   num-inferior="1",
   executable="/home/user/a.out"}
]

# Include all threads in output
-list-thread-groups --available
```

---

## 19. Program Context Commands

### `-exec-arguments`

Set the arguments for the next program run:

```
-exec-arguments --foo bar baz
^done
```

### `-environment-cd`

Change the working directory:

```
-environment-cd /home/user/project
^done
```

### `-environment-directory`

Show or set source path:

```
# Add to source search path
-environment-directory /home/user/src

# Reset source search path
-environment-directory -r

^done,source-path="/home/user/src:/usr/src"
```

### `-environment-path`

Show or set the executable search path:

```
-environment-path /usr/bin /bin
^done,path="/usr/bin:/bin"
```

### `-environment-pwd`

Show the current working directory:

```
-environment-pwd
^done,cwd="/home/user/project"
```

---

## 20. File Commands

### `-file-exec-and-symbols`

Load an executable (and its symbols):

```
-file-exec-and-symbols /home/user/a.out
^done
(gdb)
```

### `-file-exec-file`

Load an executable without reading symbols:

```
-file-exec-file /home/user/a.out
^done
```

### `-file-symbol-file`

Load symbols from a file:

```
-file-symbol-file /home/user/a.out
^done
```

### `-file-list-exec-sections`

List sections of the executable:

```
-file-list-exec-sections
^done,section-info=[
  {section="0x08048114->0x0804813e at 0x00000114: .interp ALLOC LOAD READONLY DATA HAS_CONTENTS"},
  {section="0x08048138->0x08048158 at 0x00000138: .note.ABI-tag ALLOC LOAD READONLY DATA HAS_CONTENTS"},
  ...
]
```

### `-file-list-shared-libraries`

List shared libraries:

```
-file-list-shared-libraries
^done,shlib-info=[
  {num="1",name="/lib/libc.so.6",kind="-",syms-read="yes",
   addr="0xb7ee4000",data-addr="0xb7ff5000",
   filename="/lib/libc.so.6"}
]
```

---

## 21. Target Manipulation Commands

### `-target-select`

Connect to a remote/other target:

```
-target-select remote localhost:1234
^connected,frame={...}
(gdb)

-target-select extended-remote /dev/ttyS0
^connected
```

### `-target-detach`

Detach from the target (keep it running):

```
-target-detach
^done
```

### `-target-disconnect`

Disconnect from a remote target:

```
-target-disconnect
^done
```

### `-target-download`

Download the program to a remote target:

```
-target-download
+download,{section=".text",section-sent="512",
           section-size="162580",total-sent="512",total-size="463328"}
+download,{section=".text",section-sent="1024",...}
...
^done,address="0x00100000",load-size="463328",
     transfer-rate="1294564",write-rate="512"
```

### `-target-flash-erase`

Erase flash memory:

```
-target-flash-erase
^done
```

---

## 22. Symbol Query Commands

### `-symbol-list-lines`

List the source lines for a given source file:

```
-symbol-list-lines main.c
^done,lines=[
  {pc="0x08048564",line="68"},
  {pc="0x08048582",line="69"},
  ...
]
```

### `-symbol-info-functions` (added in GDB 9.1 / MI3 era)

List functions matching a pattern:

```
-symbol-info-functions
^done,symbols={
  debug=[
    {filename="main.c",fullname="/home/user/main.c",
     symbols=[
       {line="68",name="main",type="int (int, char **)"},
       {line="10",name="callee",type="void (int)"}
     ]}
  ]
}

# With a filter
-symbol-info-functions --name "calc.*"

# Limit results
-symbol-info-functions --max-results 10
```

### `-symbol-info-variables` (added in GDB 9.1 / MI3 era)

```
-symbol-info-variables
^done,symbols={
  debug=[
    {filename="main.c",fullname="/home/user/main.c",
     symbols=[
       {line="5",name="global_counter",type="int"}
     ]}
  ]
}
```

### `-symbol-info-types` (added in GDB 9.1 / MI3 era)

```
-symbol-info-types --name "My.*"
^done,symbols=[
  {typeclass="struct",name="MyStruct"},
  {typeclass="enum",name="MyEnum"}
]
```

### `-symbol-info-modules` (added in GDB 9.1 / MI3 era — for Fortran/Ada)

```
-symbol-info-modules
^done,symbols=[...]
```

---

## 23. Tracepoint Commands

Tracepoints are non-stopping observation points, mainly used with `gdbserver` in `--multi` mode or similar. They collect data without stopping the program.

### `-trace-define-variable`

```
-trace-define-variable "$counter" "0"
^done
```

### `-trace-find`

Find a trace frame:

```
-trace-find frame-number 3
^done,found="1",tracepoint="1",traceframe="3",frame={...}

-trace-find pc 0x08048564
^done,found="1",...
```

### `-trace-list-variables`

```
-trace-list-variables
^done,trace-variables=[
  {name="$counter",initial="0",current="42"}
]
```

### `-trace-save`

Save trace data to a file:

```
-trace-save /home/user/trace.tfl
^done
```

### `-trace-start` / `-trace-stop`

```
-trace-start
^done

-trace-stop
^done,traceframes="100",tracepoints="1",
stop-reason="request",stopping-tracepoint="1"
```

### `-trace-status`

```
-trace-status
^done,supported="1",running="0",frames="100",
frames-created="100",buffer-size="5242880",buffer-free="5241288",
disconnected="0",circular="0",user-name="",notes="",
start-time={secs="1600000000",usecs="0"},
stop-time={secs="1600000001",usecs="500000"}
```

---

## 24. Ada Tasking Commands

For programs written in Ada, GDB/MI exposes task (lightweight coroutine) control.

### `-ada-task-info`

```
-ada-task-info
^done,tasks={
  nr_rows="2",nr_cols="8",
  hdr=[...],
  body=[
    {current="*",id="1",task-id="0x...",
     thread-id="1",parent-id="0",
     priority="48",state="Runnable",
     name="main_task"},
    {id="2",task-id="0x...",thread-id="2",
     parent-id="1",priority="48",
     state="Delay Sleep",name="worker_task"}
  ]
}
```

---

## 25. Support and Miscellaneous Commands

### `-list-features`

Find out which optional features GDB supports:

```
-list-features
^done,features=[
  "frozen-varobjs",
  "pending-breakpoints",
  "python",
  "thread-info",
  "data-read-memory-bytes",
  "breakpoint-notifications",
  "ada-task-info",
  "language-option",
  "info-gdb-mi-command",
  "undefined-command-error-code",
  "exec-run-start-option",
  "data-disassemble-a-option"
]
```

### `-list-target-features`

```
-list-target-features
^done,features=["async","reverse"]
```

### `-gdb-version`

```
-gdb-version
~"GNU gdb (Ubuntu 12.1-0ubuntu1~22.04) 12.1\n"
~"Copyright (C) 2022 Free Software Foundation, Inc.\n"
...
^done
```

### `-gdb-set`

Set a GDB variable:

```
-gdb-set print pretty on
^done

-gdb-set target-async on
^done

-gdb-set non-stop on
^done
```

### `-gdb-show`

Show a GDB variable:

```
-gdb-show print pretty
^done,value="on"

-gdb-show language
^done,value="auto; currently c"
```

### `-gdb-exit`

Exit GDB:

```
-gdb-exit
^exit
```

### `-interpreter-exec`

Run a command in a specific interpreter, from within another:

```
# Run a CLI command from within the MI interpreter
-interpreter-exec console "info break"
~"Num     Type           Disp Enb Address    What\n"
~"1       breakpoint     keep y   0x08048564 in main at main.c:68\n"
^done
```

### `-enable-pretty-printing`

Enable Python-based pretty-printers for variable objects:

```
-enable-pretty-printing
^done
```

Once enabled, this cannot be disabled in the same GDB session.

### `-complete`  (added in GDB 9.1 / MI3)

List completions for a partial command:

```
-complete "break mai"
^done,completion="break main",matches=["break main","break main.c"],
max_completions_reached="0"
```

### `-fix-multi-location-breakpoint-output`

Backport MI3 multi-location breakpoint format to MI2:

```
-fix-multi-location-breakpoint-output
^done
```

### `-info-gdb-mi-command`

Check if a specific MI command exists:

```
-info-gdb-mi-command data-read-memory-bytes
^done,command={exists="true"}

-info-gdb-mi-command nonexistent-command
^done,command={exists="false"}
```

### `-add-inferior`

Add a new inferior (for multi-process debugging):

```
-add-inferior
^done,inferior="i2"
=thread-group-added,id="i2"
```

### `-remove-inferior`

```
-remove-inferior i2
^done
```

---

## 26. MI3-Specific Breakpoint Output Format

This section details the complete structure of a breakpoint record as output in MI3.

### Single-Location Breakpoint Fields

| Field | Type | Description |
|-------|------|-------------|
| `number` | string | Breakpoint number |
| `type` | string | `"breakpoint"`, `"watchpoint"`, `"hw watchpoint"`, `"read watchpoint"`, `"acc watchpoint"`, `"tracepoint"`, `"catchpoint"`, `"dprintf"` |
| `catch-type` | string | For catchpoints: `"fork"`, `"exec"`, `"throw"`, etc. |
| `disp` | string | `"keep"`, `"del"` (temporary), `"dis"` (disable after hit) |
| `enabled` | string | `"y"` or `"n"` |
| `addr` | string | Address, `"<MULTIPLE>"` (multi-location), or `"<PENDING>"` |
| `func` | string | Function name |
| `file` | string | Source file (relative) |
| `fullname` | string | Source file (absolute) |
| `line` | string | Source line number |
| `at` | string | Short description of location |
| `pending` | string | Original location string (pending breakpoints) |
| `evaluated-by` | string | `"host"` or `"target"` |
| `thread` | string | Thread ID if thread-specific |
| `thread-groups` | list | Thread group IDs where this breakpoint is active |
| `cond` | string | Condition expression (if conditional) |
| `ignore` | string | Ignore count |
| `enable` | string | Enable count |
| `traceframe-usage` | string | Trace frame usage |
| `static-tracepoint-marker-string-id` | string | Static tracepoint ID |
| `mask` | string | Watchpoint mask |
| `pass` | string | Tracepoint pass count |
| `original-location` | string | The original location string as given to the command |
| `times` | string | Hit count |
| `installed` | string | Whether a tracepoint is installed on target |
| `what` | string | What the watchpoint watches |
| `script` | list | Commands to execute on hit (MI4+) |
| `locations` | list | Sub-locations for multi-location breakpoints (MI3+) |

### Multi-Location Breakpoint (MI3)

```
bkpt={
  number="1",
  type="breakpoint",
  disp="keep",
  enabled="y",
  addr="<MULTIPLE>",
  times="0",
  original-location="foo",
  locations=[
    {
      number="1.1",
      enabled="y",
      addr="0x08048500",
      func="foo(int)",
      file="main.cpp",
      fullname="/home/user/main.cpp",
      line="12",
      thread-groups=["i1"]
    },
    {
      number="1.2",
      enabled="y",
      addr="0x08048560",
      func="foo(double)",
      file="main.cpp",
      fullname="/home/user/main.cpp",
      line="20",
      thread-groups=["i1"]
    }
  ]
}
```

---

## 27. Asynchronous and Non-Stop Modes

### All-Stop Mode (default)

When any thread hits a breakpoint, **all threads stop**. `*stopped` will have `stopped-threads="all"`.

### Non-Stop Mode

Individual threads can be stopped/resumed independently. Enable it:

```
-gdb-set non-stop on
^done

-exec-run
^running
*running,thread-id="1"
(gdb)
```

In non-stop mode, `*running` and `*stopped` carry specific thread IDs, not `"all"`.

### Asynchronous Command Execution

In asynchronous mode (default for remote targets and non-stop), commands return immediately:

```
-exec-continue
^running         # command succeeded immediately
(gdb)
*stopped,...     # arrives later when thread stops
```

In synchronous mode (default for local targets), the `(gdb)` prompt is not returned until after `*stopped`.

### Checking Async Support

```
-list-target-features
^done,features=["async","reverse"]
```

---

## 28. CLI Compatibility Inside MI

CLI commands can be issued directly in MI mode:

```
break main
&"break main\n"
~"Breakpoint 1 at 0x8048564: file main.c, line 68.\n"
=breakpoint-created,bkpt={number="1",...}
^done
```

The output mixes stream records (`~`, `&`) with the standard result record. This is useful for testing, but **not recommended for production front-ends** because:

- Output format is less predictable
- Some CLI commands query the user (GDB always answers "yes" in MI mode)
- Breakpoint command lists are not executed
- Multi-line CLI commands (like `define`, `if`, `while`) emit `>` prompts that are not valid MI

The **correct approach** is to use `-interpreter-exec console "cmd"`:

```
-interpreter-exec console "info threads"
~"  Id   Target Id         Frame \n"
~"* 1    Thread 0x...      main () at main.c:42\n"
^done
```

---

## 29. Front-End Development Guide

### Startup Sequence

A robust front-end startup sequence:

```
# 1. Start GDB with specific MI version
gdb --interpreter=mi3 --quiet

# 2. Check GDB version
-gdb-version

# 3. Enable pretty printing (optional)
-enable-pretty-printing

# 4. Configure GDB settings
-gdb-set print pretty on
-gdb-set print object on
-gdb-set print static-members on
-gdb-set print vtbl on
-gdb-set print demangle on

# 5. Load the executable
-file-exec-and-symbols /path/to/program

# 6. Set initial breakpoints
-break-insert main

# 7. Run
-exec-run
```

### Handling Errors

Always check the result class:

```python
def send_mi_command(token, cmd):
    write(f"{token}{cmd}\n")

def handle_response(line):
    if line.startswith(f"{token}^done"):
        handle_success(line)
    elif line.startswith(f"{token}^running"):
        handle_running()
    elif line.startswith(f"{token}^error"):
        handle_error(line)
    elif line.startswith("*stopped"):
        handle_stopped(line)
    elif line.startswith("="):
        handle_notification(line)
    elif line == "(gdb)":
        ready_for_next_command()
```

### Token Usage

Use monotonically increasing tokens to match responses to requests:

```
1-break-insert main
2-exec-run
# Response: 1^done,bkpt={...}
#           2^running
```

### Handling `*stopped`

The `*stopped` record is the most important async record. Always parse:
- `reason` to know why we stopped
- `thread-id` to know which thread
- `frame` to update the UI
- `bkptno` (for breakpoint hits) to highlight the relevant breakpoint

### Handling Notifications

Subscribe to `=` records to keep UI fresh:
- `=breakpoint-created/modified/deleted` → refresh breakpoint panel
- `=thread-created/exited` → refresh thread panel
- `=library-loaded/unloaded` → refresh module panel
- `=thread-selected` → highlight current thread

### Non-Stop UI Guidelines

In non-stop mode:
- Multiple threads may be running simultaneously
- Always pass `--thread` to commands
- Use separate per-thread variable object sets
- Refreshing the stopped thread's stack doesn't affect running threads

---

## 30. Complete Annotated Session Example

Here is a complete session debugging a simple C program, with every command and response annotated.

```c
// main.c
#include <stdio.h>
int add(int a, int b) { return a + b; }
int main(int argc, char **argv) {
    int x = 10, y = 20, result;
    result = add(x, y);
    printf("Result: %d\n", result);
    return 0;
}
```

**Compile:** `gcc -g -o a.out main.c`

**Session:**

```
# Start GDB with MI3
$ gdb --interpreter=mi3 a.out --quiet
# GDB outputs version info then prompt:
=thread-group-added,id="i1"
~"Reading symbols from a.out...\n"
(gdb)

# Load the program (already loaded from command line, but showing the command)
1-file-exec-and-symbols a.out
1^done
(gdb)

# Enable pretty printing
2-enable-pretty-printing
2^done
(gdb)

# Set a breakpoint at main
3-break-insert main
3^done,bkpt={number="1",type="breakpoint",disp="keep",
             enabled="y",addr="0x000000000040056a",
             func="main",file="main.c",
             fullname="/home/user/main.c",line="4",
             thread-groups=["i1"],times="0",
             original-location="main"}
(gdb)

# Set a watchpoint on result
4-break-watch result
# Can't set yet since result doesn't exist — we'll set it after stopping

# Run the program
5-exec-run
5^running
(gdb)
# Program stops at main breakpoint:
*stopped,reason="breakpoint-hit",disp="keep",bkptno="1",
frame={addr="0x000000000040056a",func="main",
       args=[{name="argc",value="1"},{name="argv",value="0x7fffffffe0d8"}],
       file="main.c",fullname="/home/user/main.c",line="4",
       arch="i386:x86_64"},
thread-id="1",stopped-threads="all",core="0"
(gdb)

# Examine the stack
6-stack-list-frames
6^done,stack=[
  frame={level="0",addr="0x000000000040056a",func="main",
         file="main.c",fullname="/home/user/main.c",line="4",
         arch="i386:x86_64"}
]
(gdb)

# Create a variable object for x
7-var-create x @ x
7^done,name="x",numchild="0",value="10",type="int",
has_more="0",thread-id="1"
(gdb)

# Create a variable object for y
8-var-create y @ y
8^done,name="y",numchild="0",value="20",type="int",
has_more="0",thread-id="1"
(gdb)

# Step to next line
9-exec-next
9^running
(gdb)
*stopped,reason="end-stepping-range",
frame={addr="0x0000000000400571",func="main",
       file="main.c",fullname="/home/user/main.c",line="5"},
thread-id="1",stopped-threads="all"
(gdb)

# Step into the add() call
10-exec-step
10^running
(gdb)
*stopped,reason="end-stepping-range",
frame={addr="0x0000000000400547",func="add",
       file="main.c",fullname="/home/user/main.c",line="2"},
thread-id="1",stopped-threads="all"
(gdb)

# List arguments in the current frame
11-stack-list-arguments --all-values
11^done,stack-args=[
  frame={level="0",args=[
    {name="a",value="10"},
    {name="b",value="20"}
  ]},
  frame={level="1",args=[
    {name="argc",value="1"},
    {name="argv",value="0x7fffffffe0d8"}
  ]}
]
(gdb)

# Step out of add() back to main
12-exec-finish
12^running
(gdb)
*stopped,reason="function-finished",
frame={addr="0x0000000000400580",func="main",
       file="main.c",fullname="/home/user/main.c",line="5"},
gdb-result-var="$1",return-value="30",
thread-id="1",stopped-threads="all"
(gdb)

# Update variable objects to see changes
13-var-update *
13^done,changelist=[]
(gdb)

# Create variable object for result now that we're past its declaration
14-var-create result @ result
14^done,name="result",numchild="0",value="30",type="int",has_more="0"
(gdb)

# Evaluate an expression directly
15-data-evaluate-expression "x + y + result"
15^done,value="60"
(gdb)

# Read memory at the address of result
16-data-evaluate-expression "&result"
16^done,value="0x7fffffffe00c"
(gdb)
17-data-read-memory-bytes 0x7fffffffe00c 4
17^done,memory=[
  {begin="0x7fffffffe00c",offset="0x00000000",end="0x7fffffffe010",
   contents="1e000000"}
]
# 0x1e = 30 decimal — correct!
(gdb)

# Continue to end of program
18-exec-continue
18^running
(gdb)
@"Result: 30\n"
*stopped,reason="exited-normally"
(gdb)

# Clean up and exit
19-gdb-exit
19^exit
```

---

## Summary: Quick Reference Card

### Output Prefix Characters

| Prefix | Meaning |
|--------|---------|
| `^` | Result record (response to last command) |
| `~` | Console text (CLI output) |
| `@` | Target program output |
| `&` | GDB internal log |
| `*` | Exec async (running/stopped) |
| `+` | Status async (progress) |
| `=` | Notify async (state change) |

### Result Classes

| Class | Meaning |
|-------|---------|
| `^done` | Success |
| `^running` | Target resumed |
| `^connected` | Remote target connected |
| `^error` | Failure |
| `^exit` | GDB exiting |

### Essential Commands

| Command | Purpose |
|---------|---------|
| `-file-exec-and-symbols` | Load binary |
| `-break-insert` | Set breakpoint |
| `-break-delete` | Delete breakpoint |
| `-break-watch` | Set watchpoint |
| `-exec-run` | Start program |
| `-exec-continue` | Continue |
| `-exec-next` | Step over |
| `-exec-step` | Step into |
| `-exec-finish` | Step out |
| `-exec-interrupt` | Interrupt |
| `-stack-list-frames` | Show call stack |
| `-stack-list-locals` | Show local vars |
| `-var-create` | Create varobj |
| `-var-update` | Check for changes |
| `-data-evaluate-expression` | Evaluate expression |
| `-thread-info` | List threads |
| `-gdb-set` | Set GDB option |
| `-gdb-exit` | Quit |

---

*This document is based on the official GDB documentation at [sourceware.org](https://sourceware.org/gdb/current/onlinedocs/gdb.html/GDB_002fMI.html) and GDB 9.1 release notes. MI3 is the default GDB/MI version as of GDB 9.1.*
