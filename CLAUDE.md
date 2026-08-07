# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`mylogwatch` is a single executable bash script (plus a sourced config) that
buffers interesting log lines, coalesces them over a short window, and mails the
batch via `sendmail`. There is no build system, no dependency manifest, and no
test suite — the repo is the deliverable. See `README.md` for end-user setup and
the full config-option table; this file covers the internals and the dev loop.

## Development

There is nothing to build or install. Syntax-check after any edit:

```bash
bash -n mylogwatch
```

### Running it without mailing anything

Don't test against a real MTA. Work on a *copy* with the sendmail pipe stubbed
out and runtime state pointed somewhere disposable — this exercises the real
batching, locking, rDNS and label-substitution paths and leaves the mail in a
file you can read:

```bash
mkdir -p /tmp/mw/run && cp mylogwatch /tmp/mw/ && cd /tmp/mw
sed -i 's#SENDMAIL = "/usr/sbin/sendmail -t -f " SMTP_FROM#SENDMAIL = "cat >> /tmp/mw_sent.log"#' mylogwatch
cat > mylogwatch.cfg <<'EOF'                      # the cfg must sit next to the copy
RUNDIR="/tmp/mw/run"
WAIT_TO_SEND=2
AP_ETHERS='my-router      |AA:BB:CC:DD:EE:FF|^$|__END__'
EOF

./mylogwatch "Aug  7 10:00:00 host kernel: link up from 8.8.8.8"
./mylogwatch "Aug  7 10:00:01 host device AA:BB:CC:DD:EE:FF joined"
sleep 4 && cat /tmp/mw_sent.log
```

`./mylogwatch FLUSH` forces an immediate flush instead of waiting out
`WAIT_TO_SEND`. Diagnostics from non-tty runs land in `$RUNDIR/mylogwatch.log`,
which is the first place to look when a line silently disappears. Clean up
`$RUNDIR/mylogwatch*` between runs — a stale buffer or lock will skew the next
test.

To test a `PRIV_CMD` or `/etc/nullmailer/remotes` code path you need root;
otherwise set `EMAIL_DOMAIN` in the cfg and skip it.

## Architecture

**One script, two modes.** A normal invocation takes one log line as `$*`,
filters it, appends it to the shared buffer, and schedules a flush by
re-executing *itself* as `$0 FLUSH &`. The `FLUSH` branch is the entire second
half of the file. Both halves share the config and lock-path setup at the top,
so a change to `RUNDIR`/`PRGBNAME` naming affects both automatically.

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

**Config.** `. $0.cfg` sources site settings on every run, silently if absent.
Because it is `$0`-relative, the cfg must be named after and live beside the
script *as invoked* — a copy of the script needs its own copy of the cfg. The
stderr/stdout redirect to the log file happens only *after* the source, so
`RUNDIR` from the cfg can steer it; don't move that redirect earlier.

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
