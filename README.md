# logbark

A small bash+awk tool that watches for interesting log lines, coalesces
them into a batch over a short window, and mails the batch out —
looking up reverse DNS for any IPs it finds and swapping known device
identifiers (MAC/IMSI/IMEI/...) for friendly labels along the way.

Each invocation receives one log line as its argument:

```
logbark "Aug  5 12:00:00 host kernel: something happened"
```

Quoting it is good practice, but not load-bearing — any extra arguments
are joined with spaces rather than dropped.

Called with no arguments at all, it reads log lines from standard input
instead, one per line, until stdin closes:

```
tail -Fn0 /var/log/syslog | logbark
```

That mode is for callers that start a single process and keep it alive,
feeding it a stream — an Apache piped log, or rsyslog's `omprog`. Those
can name the script directly, with no read-loop wrapper in between. The
one thing it costs is config reloading: a long-lived process reads
`logbark.cfg` once at startup rather than per line.

The first call in a batch schedules a flush `WAIT_TO_SEND` seconds
later (5s by default); further calls in that window just append to the
buffer. When the flush fires, everything buffered since is mailed as a
single message and the cycle starts over on the next new line — a line
arriving while the mail is still being sent simply opens the next
batch.

**Renamed from `mylogwatch`.** Upgrading takes one extra step: rename
the config alongside the script. It is sourced as `$0.cfg` and silently
skipped when missing, so a `logbark` next to a `mylogwatch.cfg` runs
with no site config at all — no `IGNORES`, no `AP_ETHERS`, no
`EMAIL_DOMAIN`, and no error either.

```bash
mv /usr/local/sbin/mylogwatch.cfg /usr/local/sbin/logbark.cfg
```

Everything else follows the script's own name by itself, including the
address the mail comes from: `mylogwatch.<host>@<domain>` is now
`logbark.<host>@<domain>`, so a mail filter matching the old one needs
updating, as does any caller naming the old path.

- [Requirements](#requirements)
- [Setup](#setup)
  - [Wiring it into rsyslog](#wiring-it-into-rsyslog)
  - [Debian 13 (trixie) and other sandboxed rsyslog units](#debian-13-trixie-and-other-sandboxed-rsyslog-units)
  - [Wiring it into Apache](#wiring-it-into-apache)
- [Configuration](#configuration)
- [The mail it sends](#the-mail-it-sends)
- [Notes](#notes)
- [Development](#development)
  - [Testing without mailing anything](#testing-without-mailing-anything)

## Requirements

- `bash`
- `gawk` (the script uses GNU-only features: `gensub()` and the
  3-argument form of `match()` — plain POSIX/mawk `awk` will not work)
- `flock` (util-linux) and `mktemp` (coreutils)
- a `sendmail`-compatible MTA at `/usr/sbin/sendmail` (Postfix,
  nullmailer, msmtp's sendmail shim, ...)
- optionally, `/etc/nullmailer/remotes` for automatic smart-host
  detection, and `getent` for reverse DNS lookups

## Setup

1. Copy the script somewhere on `$PATH` (or wherever you'll invoke it
   from) and make it executable:

   ```bash
   cp logbark /usr/local/sbin/logbark
   chmod +x /usr/local/sbin/logbark
   ```

   The install directory does not need to be writable — all runtime
   state goes to `RUNDIR` (see below).

2. Copy the config template next to it and edit it for your site:

   ```bash
   cp logbark.cfg.example /usr/local/sbin/logbark.cfg
   ```

   `logbark.cfg` is sourced by the script on every run (silently
   skipped if missing) and is **not** meant to be checked into git —
   it's exactly where site-specific things like device identifiers,
   your outgoing mail domain, or a privilege-escalation command belong.
   It must sit next to the script, named after it.

3. Feed it log lines. `logbark` doesn't tail anything itself — wire
   it into whatever produces the lines you care about. For example, to
   follow a plain text log:

   ```bash
   tail -Fn0 /var/log/syslog | /usr/local/sbin/logbark
   ```

   or point a syslog daemon's program/exec action (see
   [Wiring it into rsyslog](#wiring-it-into-rsyslog) below), an Apache
   piped log (see [Wiring it into Apache](#wiring-it-into-apache)), a
   udev rule, or a cron job at it — anything that can either call it
   once per line of interest or hand it a stream of them.

4. If it won't run as root, set `PRIV_CMD` in `logbark.cfg` (e.g.
   `PRIV_CMD="sudo"`) — root is needed to read
   `/etc/nullmailer/remotes` for smart-host detection.

### Wiring it into rsyslog

If `rsyslog` is already collecting your logs, this is the least-effort
way to get instant mail about reboots, disk errors, USB disconnects and
the like: one template that emits the raw message, plus one `if` whose
conditions list every alert you care about, one per line. Drop
something like this into `/etc/rsyslog.d/xlog-all.conf`:

```
template(name="mylogargs" type="string" string="%rawmsg%")

if  $rawmsg contains "sector"
 or $rawmsg contains "FAILED"
 or $rawmsg contains "ERROR"
 or $rawmsg contains "Error"
 or $rawmsg contains "EXT4-fs error"
 or re_match($rawmsg, "activated.*BogoMIPS")
 or re_match($rawmsg, "usb .*: (Product:|USB disconnect)")
then ^/usr/local/sbin/logbark;mylogargs
```

Check it with `rsyslogd -N1 -f /etc/rsyslog.d/xlog-all.conf`, then
`systemctl restart rsyslog`. That's the whole integration — adding a
new alert is one more `or` line.

A few things worth knowing about that config:

- `^/path/prog;template` is rsyslog's shell-execute action, usable as
  the `then` branch of a RainerScript `if`: it runs the program once per
  matching message, with the formatted template as its single argument.
  The path must be absolute and the file executable by rsyslog —
  and, on a distribution that sandboxes the unit, reachable from
  inside that sandbox, which rules out `/root` and `/home`; see
  [Debian 13 (trixie) and other sandboxed rsyslog units](#debian-13-trixie-and-other-sandboxed-rsyslog-units).
- `%rawmsg%` passes the message as received, which for anything arriving
  over a socket includes the numeric priority prefix (`<13>`) ahead of
  the timestamp. That is what `logbark` wants — it strips the prefix
  and does its own tidying (kernel timestamp in the subject, reverse
  DNS, device labels).
- `contains` is a plain case-sensitive substring test and is cheaper
  than a regex, so it covers most of the list; `re_match()` is there
  only for the two conditions that actually need POSIX ERE. Both are
  case-sensitive, which is why `ERROR` and `Error` are listed
  separately — `contains_i`/`re_match_i` would fold in lowercase
  `error` too, and with it every line mentioning `EXT4-fs error`.
- Keeping this to a single `if` matters: rsyslog evaluates rules
  independently, so splitting the conditions into one rule each makes a
  line matching two of them invoke the script twice and show up twice
  in the mail. One `if` hands each line over exactly once, however many
  of its conditions hold.
- `logbark` returns as soon as it has buffered the line — the flush
  is backgrounded — so rsyslog is never held up by a slow `sendmail`.
- The shell-execute action forks a process per matching message, which
  is fine at the rate these conditions fire. For one that matches
  hundreds of lines a second, rsyslog's `omprog` keeps a single process
  alive instead and feeds messages on stdin rather than as arguments —
  point it at the script with no arguments and it reads them directly.
  Mind the caveats that come with any long-lived caller, listed under
  [Wiring it into Apache](#wiring-it-into-apache).
- The legacy one-line form does the same job:
  `$template mylogargs,"%rawmsg%"` plus
  `:rawmsg, ereregex, "sector|FAILED|..."  ^/usr/local/sbin/logbark; mylogargs`.
  RainerScript is used above only because its condition list can span
  lines; the legacy filter cannot be wrapped, so that pattern grows into
  one very long line.

The `IGNORES` config option is the second half of this: rsyslog decides
what reaches the script at all, `IGNORES` drops the known-boring
remainder without touching the rsyslog config.

### Debian 13 (trixie) and other sandboxed rsyslog units

Debian 13 ships `rsyslog.service` with a systemd sandbox that earlier
releases did not have, and a `^program` action inherits all of it. Two
directives in it break `logbark`:

- `ProtectHome=yes` replaces `/root` and `/home` with an empty,
  mode-`000` directory inside rsyslogd's mount namespace. A script
  installed under `/root/bin` simply does not exist as far as rsyslog is
  concerned; the `execve()` fails and the only thing logged is

      rsyslogd: program '/root/bin/logbark' (pid N) exited with status 1

  Installing to `/usr/local/sbin` (or anywhere outside `/root` and
  `/home`) is enough — the script never writes next to itself, so a
  read-only location is fine.

- `NoNewPrivileges=yes` is the awkward one, because it survives the move.
  Every descendant inherits it, and it makes the kernel ignore the setuid
  bit on exec. With `nullmailer` that strips the privilege from
  `/usr/sbin/nullmailer-queue` (mode `4755`, owner `mail`), so the queue
  file is created `root:root` `0600` and `nullmailer-send`, which runs as
  user `mail`, can never open it:

      nullmailer-send: Can't open file '1787077005.7947'

  This one is silent from `logbark`'s side — the script exits 0, the
  mail is accepted, and it accumulates in `/var/spool/nullmailer/queue`
  forever. `journalctl -u nullmailer` is the only place it shows.

`FLUSH_WRAPPER` is the fix that does not weaken the sandbox. Only the
`FLUSH` pass needs the privileged path — the buffering half is happy
inside it — so launching just that half as a transient unit re-parents it
to PID 1 and leaves rsyslog's hardening untouched:

    RUNDIR=/var/lib/logbark
    FLUSH_WRAPPER="systemd-run --quiet --collect --property=ExitType=cgroup --unit=logbark-flush-$$"

`RUNDIR` is not optional here. `rsyslog.service` also sets
`PrivateTmp=yes`, so the default `RUNDIR=/tmp` resolves to a per-service
private tmpfs for the buffering half, while the transient unit sees the
real `/tmp` — the flush would find an empty buffer and mail nothing. Any
path outside `/tmp` that both can reach works; `/var` stays writable
under `ProtectSystem=full`.

`ExitType=cgroup` is not optional either, once the MTA delivers from a
child of the submitting process — which is what exim does by default. A
transient unit is finished the instant its main process exits, and
`KillMode=` defaults to `control-group`, so systemd kills whatever that
process left behind. `sendmail` spools the message and returns, exim's
forked delivery process is killed a moment later, and the mail sits on
the queue. The signature in `/var/log/exim4/mainlog` is an acceptance
line with no delivery line after it:

    2026-01-01 05:50:49 1abCdE-000000012345-6xYz <= logbark.host@example.com U=root P=local S=316

and then nothing at all until a queue runner picks it up minutes later —
up to a full `QUEUEINTERVAL`. Setting `ExitType=cgroup` keeps the unit
running until every process in its cgroup has exited, which is the
missing piece. It needs systemd 250 or newer, and it costs nothing under
nullmailer or Postfix: both only spool and return, leaving delivery to a
daemon that was never inside the transient unit. That is why the recipe
went so long without it.

The blunter alternative is a drop-in that turns the offending directive
off:

    # /etc/systemd/system/rsyslog.service.d/logbark.conf
    [Service]
    NoNewPrivileges=no

That works — nothing else in the shipped unit forces it back on — but it
relaxes the hardening for everything rsyslog runs, not just this script.

### Wiring it into Apache

Apache can hand a request straight to `logbark` as a piped log. Two
lines in the vhost — one tagging the requests worth knowing about, one
logging them — are the whole integration:

```
SetEnvIf Request_URI "/toh/ota" WATCHED
CustomLog "|/usr/local/sbin/logbark" combined env=WATCHED
```

`env=WATCHED` restricts this `CustomLog` to the requests `SetEnvIf`
tagged, so nothing else reaches the script and the normal access log
goes on recording every request as before. Check it with `apachectl -t`
and `systemctl reload apache2`; from then on a device pulling firmware
from `/toh/ota` — or anything else probing it — turns into a mail.

Apache starts a piped-log program once, at server startup, and writes
one line per matching request to its stdin. That is exactly the
no-argument mode, so there is no wrapper script in the picture. What
follows from it:

- The program is spawned by the parent httpd process and inherits its
  uid, so it runs as **root**. `PRIV_CMD` is unnecessary and
  `/etc/nullmailer/remotes` is readable.
- `"|$..."` is the same thing spawned through `/bin/sh -c` rather than
  directly. Nothing here needs a shell, so plain `"|..."` is the better
  form — one process fewer, and no shell parsing between Apache and the
  script.
- **`logbark.cfg` is read once**, when Apache starts the process,
  not per line the way a per-invocation caller re-reads it. An edit to
  `IGNORES` takes effect on the next `systemctl reload apache2`, not on
  the next request.
- The process inherits `apache2.service`'s sandbox. Check
  `systemctl show apache2 -p PrivateTmp`: if it is on, the default
  `RUNDIR=/tmp` is Apache's private tmpfs, so runtime state is
  invisible to a hand-run of the script and is wiped whenever Apache
  restarts — and a `FLUSH_WRAPPER` would need `RUNDIR` pinned outside
  `/tmp`, for the reason described
  [above](#debian-13-trixie-and-other-sandboxed-rsyslog-units).
- Every line `IGNORES` drops writes one `ignored` to
  `$RUNDIR/logbark.log`, which nothing rotates. That is nothing at
  the rate an `env=`-gated log fires, but it is a reason not to point an
  ungated `CustomLog` at the script.
- Apache restarts a piped-log program that dies, so a crash is not
  fatal — but the requests logged in between are lost, which is why the
  read loop keeps going rather than exit on a line it dislikes.

## Configuration

All of these are optional; see `logbark.cfg.example` for the exact
format of the two list-shaped ones.

| Setting | Default | Purpose |
|---|---|---|
| `IGNORES` | *(empty)* | `\|`-joined regex alternatives; a line matching any of them is dropped instead of buffered. Unanchored, so an entry matches anywhere in the line. Blank lines are always dropped regardless. |
| `AP_ETHERS` | *(empty)* | `Label\|Identifier\|` pairs. Each identifier found in a buffered line is replaced by `[ Label ]`. A lookup table only — listing a device annotates its lines, it does not drop them. |
| `EMAIL_DOMAIN` | smart host from `/etc/nullmailer/remotes`, else `example.com` | Domain used for the `From:`/`To:` addresses. |
| `FLUSH_WRAPPER` | *(empty)* | Command prefix the `FLUSH` pass is launched through, e.g. `systemd-run --quiet --collect --property=ExitType=cgroup --unit=logbark-flush-$$`. Empty runs it as a plain background child. See [Debian 13 (trixie) and other sandboxed rsyslog units](#debian-13-trixie-and-other-sandboxed-rsyslog-units). |
| `PRIV_CMD` | *(empty)* | Command prefix used to run `awk` when not root, e.g. `sudo`. |
| `RUNDIR` | `$TMPDIR`, else `/tmp` | Where all runtime state is kept. |
| `WAIT_TO_SEND` | `5` | Seconds to collect lines before flushing a batch. |

## The mail it sends

One batch is one mail, sent from and to
`logbark.<short-hostname>@<EMAIL_DOMAIN>`. The subject is the first
line of the batch (with a leading kernel timestamp like
`[   12.345678] ` stripped off); the body holds every line of the
batch, in arrival order, first line included.

When a batch holds more than one line, the subject is prefixed with
`[ +N ]`, where N counts the lines *below* the first one. Without that
marker the extra lines are easy to miss, since only the first one is
ever visible in a mail client's subject list:

```
Subject: [ +2 ] EXT4-fs error (device sda1): ext4_find_entry:1455: inode #2

EXT4-fs error (device sda1): ext4_find_entry:1455: inode #2
sd 0:0:0:0: [sda] Unrecovered read error - auto reallocate failed sector 1234
usb 1-1: USB disconnect, device number 7
```

No prefix means the mail is a single line and the subject is the whole
story.

## Notes

- Runtime state all lives under `RUNDIR`, named after the script:
  the buffer (`logbark`), its lock (`.lock`), the flush debounce
  lock (`.lck`), the diagnostic log (`.log`), plus short-lived
  per-flush batch and `awk` files. Nothing is ever written next to the
  script, and nothing needs managing by hand.
- Buffer access is `flock`-guarded end to end: appends and flushes take
  the same lock, and a flush atomically hands the buffer off to a
  private file before mailing it, so a line arriving mid-flush is never
  silently dropped, and a slow mail send never blocks new appends.
- Force an immediate flush manually with `logbark FLUSH`.

## Development

There is nothing to build or install — the script is the deliverable.
Syntax-check it after any edit:

```bash
bash -n logbark
```

### Testing without mailing anything

Don't test against a real MTA. Work on a *copy* with the sendmail pipe
stubbed out and runtime state pointed somewhere disposable — that still
exercises the real batching, locking, reverse-DNS and label-substitution
paths, and leaves the message in a file you can read:

```bash
mkdir -p /tmp/lb/run && cp logbark /tmp/lb/ && cd /tmp/lb
sed -i 's#^ *SENDMAIL = .*#    SENDMAIL = "cat >> /tmp/lb_sent.log"#' logbark
cat > logbark.cfg <<'EOF'          # the cfg must sit next to the copy
RUNDIR="/tmp/lb/run"
WAIT_TO_SEND=2
AP_ETHERS='my-router      |AA:BB:CC:DD:EE:FF|^$|__END__'
EOF

./logbark "Aug  7 10:00:00 host kernel: link up from 8.8.8.8"
./logbark "Aug  7 10:00:01 host device AA:BB:CC:DD:EE:FF joined"
sleep 4 && cat /tmp/lb_sent.log
```

Feeding the same lines on stdin exercises the other calling convention,
against a single long-lived process rather than one per line:

```bash
printf '%s\n' "Aug  7 10:00:00 host kernel: link up from 8.8.8.8" \
              "Aug  7 10:00:01 host device AA:BB:CC:DD:EE:FF joined" | ./logbark
```

`./logbark FLUSH` cuts the wait short. Diagnostics from non-tty runs
land in `$RUNDIR/logbark.log`, which is the first place to look when
a line silently disappears, and `$RUNDIR/logbark*` is worth clearing
between runs — a stale buffer or lock will skew the next test.

Testing a `PRIV_CMD` or `/etc/nullmailer/remotes` code path needs root;
otherwise set `EMAIL_DOMAIN` in the config and skip it.
