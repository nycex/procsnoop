# procsnoop

mostly "vibecoded" version of `{exec,exit}snoop` featuring (non-exhaustive):
- fancier formatting (colors etc.)
- filtering
  - by whole process tree branches (processes and their children) (by PID or name/comm/exe regex)
  - only show top-level procs (procs whose parents did not spawn during run procsnoop's run) 
- get FORK, EXEC and EXIT events (and see how long proc ran until EXIT) in a single chronological output
- select which events to output

## requirements

- Linux **5.8 or newer** (BPF ring buffer).
- The **BCC** Python module:
  - Debian/Ubuntu: `python3-bpfcc`
  - Fedora: `python3-bcc`
  - Arch: `python-bcc`
- Python **3.9 or newer**.

## usage

```bash
# Watch every exec and exit (the default)
sudo procsnoop

# Only top-level commands, not their subprocesses
sudo procsnoop -T

# Hide a process and its whole subtree
sudo procsnoop -P 1234

# Show ONLY one process and its subtree
sudo procsnoop -o -P 1234

# Watch all three event types
sudo procsnoop -e all
```

`sudo` is optional: run it unprivileged and `procsnoop` re-executes itself
through `sudo` for you. Use `--no-sudo` to make it fail instead.

## recipes

| Goal | Command |
| --- | --- |
| Only top-level commands (hide subprocesses) | `procsnoop -T` |
| Same, but also cover sessions opened later | `procsnoop -T --strict` |
| Hide one process and its subtree | `procsnoop -P 1234` |
| Hide several processes | `procsnoop -P 1234,5678` |
| Hide sshd and everything it spawns | `procsnoop -P "$(pgrep -d, -x sshd)"` |
| Show *only* one process and its subtree | `procsnoop -o -P 4242` |
| Hide all kernel threads (kthreadd's subtree) | `procsnoop -P 2 -e all` |
| Hide Chrome / VS Code and their children | `procsnoop -m '^(chrome\|code)$'` |
| Watch every event type | `procsnoop -e all` |
| Follow a build, showing only the top-level jobs | `procsnoop -o -f 'make .*-j' -T` |

Roots can be combined: `-P`, `-m`, `-f` and `-X` may all be used together, and
a process is a root if it matches any of them.

## reading the output

```text
TIME         EVENT     PID    PPID COMM             DETAILS
14:03:21.114 EXEC     4821    4790 bash             /usr/bin/make -j8 all
14:03:21.119 EXEC     4822    4821 make             cc -c -o foo.o foo.c
14:03:21.204 EXIT     4822    4821 make             exit 0  after 84.9ms
14:03:21.310 EXIT     4821    4790 bash             exit 0  after 12.44s
```

| Column | Meaning |
| --- | --- |
| `TIME` | Wall clock (`HH:MM:SS.mmm`), relative seconds, or omitted — see `-t`. |
| `EVENT` | `FORK`, `EXEC` or `EXIT`. A trailing `*` marks a hidden line shown with `--show-hidden`. |
| `PID` | Subject process (for `FORK`, the child). |
| `PPID` | The process that *created* the subject; survives reparenting. |
| `UID` | Owning UID (with `-U`). |
| `COMM` | Process name (`comm`, at most 15 characters). |
| `PCOMM` | Parent's name, works even for dead parents (with `-M`). |
| `DETAILS` | `EXEC`: the quoted command line. `EXIT`: exit status or signal, and process lifetime. |

## options

### root selection

| Option | Description |
| --- | --- |
| `-P, --pid PID[,PID...]` | Roots: PID(s) whose subtree is hidden (or, with `-o`, shown); comma-separated, repeatable |
| `-m, --match PATTERN` | Roots: processes whose name (`comm`, max. 15 chars) matches, like `pgrep` |
| `-f, --match-full PATTERN` | Roots: processes whose command line (arguments joined by spaces) matches, like `pgrep -f` |
| `-X, --match-exe PATTERN` | Roots: processes whose executable matches (the loaded binary: `/proc/PID/exe`, or the interpreter for scripts) |
| `-F, --fixed-strings` | Patterns are literal strings, not regular expressions |
| `-x, --exact` | Patterns must match the whole value (like `pgrep -x`) |
| `-i, --ignore-case` | Match patterns case-insensitively |
| `-o, --only` | Invert: print only the subtrees instead of hiding them |
| `-r, --roots` | Count the root processes themselves as part of their subtrees (default) |
| `-R, --no-roots` | Filter only the descendants; the roots are treated like any other process |

### output

| Option | Description |
| --- | --- |
| `-e, --events EV[,EV...]` | Events to print: `FORK`, `EXEC`, `EXIT` or `ALL` (default: `EXEC,EXIT`); all three are always traced internally |
| `-T, --top-level` | Print only processes none of whose ancestors were printed (scripts and programs, not their subprocesses); interactive shells don't count as printed ancestors |
| `--strict` | With `-T`: interactive shells count as printed ancestors too; any session opened after startup hides everything run in it |
| `--paired` | Print `EXIT` only for processes whose `FORK` or `EXEC` line was printed (default whenever `--events` allows it) |
| `--no-paired` | Print every `EXIT` that passes the filters |
| `--show-hidden` | Print filtered lines too, dimmed and marked with `*` (debugging) |
| `-t, --time {wall,rel,none}` | Timestamp column: `wall` (`HH:MM:SS.mmm`), `rel` (seconds since the trace started) or `none` (default: `wall`) |
| `-U, --uid` | Add a UID column |
| `-M, --pcomm` | Add the parent's name (from the tracked tree, works for dead parents) |
| `--exe` | Show the loaded binary when it differs from `argv[0]` |
| `--color [{auto,always,never}]` | Colorize output (default: `auto`) |
| `--no-header` | Don't print the column header |

### tuning

| Option | Description |
| --- | --- |
| `--arg-bytes N` | Max bytes of argv captured per exec; a power of two between 256 and 16384 (default: 4096) |
| `--buffer-pages N` | Ring buffer size in pages; a power of two between 16 and 65536 (default: 1024 = 4 MiB) |
| `--no-sudo` | Don't re-run through sudo when not root; fail instead |

## how it works

`procsnoop` attaches to `sched_process_fork`, `sched_process_exec` and
`sched_process_exit` via eBPF (through BCC) and streams events to user space
over a BPF ring buffer. It **attaches before** snapshotting `/proc`, so
processes that already exist are covered and nothing started during startup is
missed. Each event is matched against the tracked tree:

- A **root** is a process selected by `-P` or a pattern.
- A **descendant** is any process whose ancestry includes a root; this is
  decided from the fork chain, not from the current `/proc` parent.
- By default roots and descendants are hidden; `-o` inverts this to show only
  them.
- `-T` additionally hides a process once an ancestor's start line has been
  printed, so you see the first command in each chain but not its children.

