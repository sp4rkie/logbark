# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`mylogwatch` is a single executable bash script (plus a sourced config) that
buffers interesting log lines, coalesces them over a short window, and mails the
batch via `sendmail`. There is no build system, no dependency manifest, and no
test suite — the repo is the deliverable. `README.md` carries end-user setup,
the full config-option table, and the dev loop (`bash -n`, plus the
stubbed-sendmail recipe for exercising a change without a real MTA) — start
there, then come back here for the internals behind it.

## Architecture

**One script, two modes.** A normal invocation takes one log line as `$*`,
filters it, appends it to the shared buffer, and schedules a flush by
re-executing *itself* as `$FLUSH_WRAPPER "$SELF" FLUSH &`. The `FLUSH` branch is
the entire second half of the file. Both halves share the config and lock-path
setup at the top, so a change to `RUNDIR`/`PRGBNAME` naming affects both
automatically.

**A third entry point, not a third mode.** With no arguments at all the script
reads lines from stdin in a loop and hands each to `buffer_line()` — the same
function the argument path calls, so there is exactly one filter/append/schedule
implementation. It is there for callers that start one process and keep it
alive (an Apache piped log, rsyslog's `omprog`); `README.md` has the wiring.
Three things about it are load-bearing:

- **It keys on `$# -eq 0`, not on a flag.** Zero arguments was already a no-op
  (`MSG=""` is blank, and blank is always ignored), so nothing regressed, and
  the caller's config stays a bare path.
- **`FLUSH` is argv-only.** A line off stdin is caller-controlled text — in the
  Apache case it carries a request URL — so the `FLUSH` test is made before the
  loop is entered and never against a line the loop read.
- **The flush child reads `/dev/null`.** Without it the backgrounded flush
  inherits the caller's pipe as stdin. That is harmless only because the
  generated awk program is BEGIN-only, and gawk skips reading input for those;
  give that program a main rule and it would start eating the log stream.

The loop's one cost is that the cfg is sourced once at process start rather than
per line. That is inherent to a long-lived caller and is documented in the
README rather than worked around — re-sourcing per line would mean moving the
whole `RUNDIR`/`LOGBUFF`/`IGNORE_RE` derivation into `buffer_line()`.

**The awk program is generated, not shipped.** The FLUSH branch writes a gawk
program to an `mktemp` file and `exec`s it; that program does the mail
formatting, reverse DNS (`getent hosts`), and device-label substitution. The
heredoc delimiter (`!`) is **unquoted**, so the shell interpolates into it. This
is load-bearing and easy to break:

- `$LOGBATCH`, `$PRGBNAME`, `$AWKPROG`, `$EMAIL_DOMAIN`, `$(hostname -s)` are
  deliberately expanded by the shell at generation time.
- Everything meant for awk must be escaped: `\$` for awk's own field/variable
  sigils, and triple backslashes (`\\\.`, `\\\[`, `\\\1`) for regex atoms and
  backreferences, because the string passes through both the heredoc and awk's
  string-literal parser.
- `AP_ETHERS` is passed with `awk -v`, **not** interpolated, so quotes or
  backslashes in a site's device labels cannot break the generated program.
  Follow that pattern for any new cfg value that carries user text.
- The generated file deletes itself from awk's `BEGIN` block, so it never
  accumulates in `RUNDIR`.
- The subject carries a `[ +N ]` prefix counting the lines after the first, so
  it can only be printed once the whole batch has been read. Hence the body is
  buffered in `COLLECTED` and emitted after the headers, and the first line is
  appended to `COLLECTED` before the `getline` loop — that loop reuses `line`,
  and printing `line` after it emits the *last* line instead.

Requires **gawk** — `gensub()` and 3-argument `match()` are GNU extensions;
mawk/POSIX awk will fail at runtime, not at `bash -n`.

**Locking is the part to be careful with.** Three distinct pieces of state
under `RUNDIR`, all named after the script:

- `mylogwatch` — the shared buffer, appended by every invocation.
- `mylogwatch.lock` — guards the buffer. Held by appends, and by the flush for
  only the instant of the `mv` that rotates the buffer to a private
  `mylogwatch.$$` batch file. Mailing happens *after* the lock is released, so a
  slow `sendmail` never blocks log writers and a line arriving mid-flush lands
  in the next batch rather than being dropped.
- `mylogwatch.lck` — debounces scheduling (`flock -n`), so a burst of lines
  produces one mail. It is released before the flush runs, so a line arriving
  while `sendmail` is still going can schedule the next flush instead of sitting
  unsent.

The inline comments at these spots record bugs that were actually hit; preserve
that behavior when refactoring.

**`FLUSH_WRAPPER` launches the flush, and only the flush.** It is a command
*prefix* the `FLUSH` re-exec is placed behind, empty by default. It exists
because the two halves have different privilege needs: the half that runs
`sendmail` sometimes has to escape a sandbox the appending half sits in quite
happily. `README.md` carries the motivating case (Debian 13's hardened
`rsyslog.service`, where `NoNewPrivileges=yes` strips the setuid bit from
nullmailer's queue helper and mail is accepted but never delivered). What
matters here is the mechanics:

- **It is unquoted on purpose.** A prefix has to word-split into argv, so
  quoting it would hand the whole string over as one argv[0] and break every
  multi-word wrapper. The line reads like a quoting bug and carries a comment
  saying it isn't — keep that comment if you touch it.
- **`$SELF` is `$0` made absolute**, computed once at the top with a `case`, and
  deliberately *not* with `realpath`/`readlink -f`. `systemd-run` rejects a
  relative executable, but resolving symlinks would change which cfg gets
  sourced: `$SELF.cfg` has to keep naming exactly the file `$0.cfg` names. Don't
  "improve" it into a symlink-resolving form.
- **A wrapper constrains `RUNDIR`.** A wrapper that re-parents the flush out of
  the caller's mount namespace puts the two halves on different `/tmp`s if the
  caller has `PrivateTmp=yes` and `RUNDIR` is left at its default: the buffer is
  appended in one and read in the other, and the flush mails nothing. There is
  no error anywhere — it just goes quiet. Any cfg setting `FLUSH_WRAPPER` must
  set `RUNDIR` outside `/tmp` as well.
- **A transient unit name needs `$$`, not a fixed string.** The debounce lock is
  released before the flush runs (above), so two flushes can legitimately
  overlap and a fixed `--unit=` would make the second fail to start.

**Config.** `. $0.cfg` sources site settings on every run, silently if absent.
Because it is `$0`-relative, the cfg must be named after and live beside the
script *as invoked* — a copy of the script needs its own copy of the cfg, and
`$SELF` is built the way it is (above) so a wrapped flush still resolves to that
same cfg. The stderr/stdout redirect to the log file happens only *after* the
source, so `RUNDIR` from the cfg can steer it; don't move that redirect
earlier.

`IGNORES` is a `|`-joined regex the cfg conventionally terminates with
`__END__|`. The script collapses runs of `||` to a single `|`, then strips one
leading and one trailing `|`, because an empty alternative matches every line
and would silently discard the entire log. The order is load-bearing: `${…#|}`
and `${…%|}` each remove exactly one pipe, and are only sufficient because the
collapse loop has already run. Keep both steps, in that order, if you touch the
filter. `AP_ETHERS` is a lookup
table only — matching it must never drop a line; that distinction was a real
bug and the comment above `IGNORE_RE` explains it.

`mylogwatch.cfg` is gitignored: it holds real MAC/IMSI/IMEI values and host
names. Edit `mylogwatch.cfg.example` when adding or changing an option, and
never copy values out of the live cfg into tracked files or commit messages.
