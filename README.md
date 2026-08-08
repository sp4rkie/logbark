# mylogwatch

A small bash+awk tool that watches for interesting log lines, coalesces
them into a batch over a short window, and mails the batch out —
looking up reverse DNS for any IPs it finds and swapping known device
identifiers (MAC/IMSI/IMEI/...) for friendly labels along the way.

Each invocation receives one log line as its argument:

```
mylogwatch "Aug  5 12:00:00 host kernel: something happened"
```

Quoting it is good practice, but not load-bearing — any extra arguments
are joined with spaces rather than dropped.

The first call in a batch schedules a flush `WAIT_TO_SEND` seconds
later (5s by default); further calls in that window just append to the
buffer. When the flush fires, everything buffered since is mailed as a
single message and the cycle starts over on the next new line — a line
arriving while the mail is still being sent simply opens the next
batch.

- [Requirements](#requirements)
- [Setup](#setup)
  - [Wiring it into rsyslog](#wiring-it-into-rsyslog)
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
   cp mylogwatch /usr/local/sbin/mylogwatch
   chmod +x /usr/local/sbin/mylogwatch
   ```

   The install directory does not need to be writable — all runtime
   state goes to `RUNDIR` (see below).

2. Copy the config template next to it and edit it for your site:

   ```bash
   cp mylogwatch.cfg.example /usr/local/sbin/mylogwatch.cfg
   ```

   `mylogwatch.cfg` is sourced by the script on every run (silently
   skipped if missing) and is **not** meant to be checked into git —
   it's exactly where site-specific things like device identifiers,
   your outgoing mail domain, or a privilege-escalation command belong.
   It must sit next to the script, named after it.

3. Feed it log lines. `mylogwatch` doesn't tail anything itself — wire
   it into whatever produces the lines you care about. For example, to
   follow a plain text log:

   ```bash
   tail -Fn0 /var/log/syslog | while IFS= read -r line; do
       /usr/local/sbin/mylogwatch "$line"
   done
   ```

   or point a syslog daemon's program/exec action (see
   [Wiring it into rsyslog](#wiring-it-into-rsyslog) below), a udev
   rule, or a cron job at it — anything that can call it once per line
   of interest.

4. If it won't run as root, set `PRIV_CMD` in `mylogwatch.cfg` (e.g.
   `PRIV_CMD="sudo"`) — root is needed to read
   `/etc/nullmailer/remotes` for smart-host detection.

### Wiring it into rsyslog

If `rsyslog` is already collecting your logs, this is the least-effort
way to get instant mail about reboots, disk errors, USB disconnects and
the like: one template that emits the raw message, plus one filter rule
whose pattern lists every alert you care about. Drop something like
this into `/etc/rsyslog.d/xlog-all.conf`:

```
$template mylogargs,"%rawmsg%"
:rawmsg, ereregex, "activated.*BogoMIPS|sector|FAILED|ERROR|Error|EXT4-fs error|usb .*: (Product:|USB disconnect)" ^/root/bin/mylogwatch; mylogargs
```

then `systemctl restart rsyslog`. That's the whole integration —
adding a new alert is one more alternative in the pattern.

A few things worth knowing about that config:

- `^/path/prog; template` is rsyslog's shell-execute action: it runs
  the program once per matching message, with the formatted template as
  its single argument. The path must be absolute and the file
  executable by rsyslog.
- `%rawmsg%` passes the message as received, timestamps and all, which
  is what `mylogwatch` wants — it does its own tidying (kernel
  timestamp in the subject, reverse DNS, device labels).
- The pattern is a POSIX extended regex and case-sensitive, which is
  why both `ERROR` and `Error` appear in it. Alternatives may nest, as
  the `usb` one does.
- Keeping it to a single rule matters: rsyslog evaluates rules
  independently, so splitting the alternatives into one rule each makes
  a line that matches two of them invoke the script twice and show up
  twice in the mail. One rule hands each line over exactly once, however
  many alternatives it hits. The cost is one long line in the config;
  split it back into a rule per pattern if you find that easier to
  read, and live with the occasional doubled line.
- `mylogwatch` returns as soon as it has buffered the line — the flush
  is backgrounded — so rsyslog is never held up by a slow `sendmail`.
- The shell-execute action forks a process per matching message, which
  is fine at the rate these filters fire. For a pattern that matches
  hundreds of lines a second, rsyslog's `omprog` keeps a single process
  alive instead, but it feeds messages on stdin rather than as
  arguments, so it needs a small read-loop wrapper around the script.

The `IGNORES` config option is the second half of this: rsyslog decides
what reaches the script at all, `IGNORES` drops the known-boring
remainder without touching the rsyslog config.

## Configuration

All of these are optional; see `mylogwatch.cfg.example` for the exact
format of the two list-shaped ones.

| Setting | Default | Purpose |
|---|---|---|
| `IGNORES` | *(empty)* | `\|`-joined regex alternatives; a line matching any of them is dropped instead of buffered. Unanchored, so an entry matches anywhere in the line. Blank lines are always dropped regardless. |
| `AP_ETHERS` | *(empty)* | `Label\|Identifier\|` pairs. Each identifier found in a buffered line is replaced by `[ Label ]`. A lookup table only — listing a device annotates its lines, it does not drop them. |
| `EMAIL_DOMAIN` | smart host from `/etc/nullmailer/remotes`, else `example.com` | Domain used for the `From:`/`To:` addresses. |
| `PRIV_CMD` | *(empty)* | Command prefix used to run `awk` when not root, e.g. `sudo`. |
| `RUNDIR` | `$TMPDIR`, else `/tmp` | Where all runtime state is kept. |
| `WAIT_TO_SEND` | `5` | Seconds to collect lines before flushing a batch. |

## The mail it sends

One batch is one mail, sent from and to
`mylogwatch.<short-hostname>@<EMAIL_DOMAIN>`. The subject is the first
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
  the buffer (`mylogwatch`), its lock (`.lock`), the flush debounce
  lock (`.lck`), the diagnostic log (`.log`), plus short-lived
  per-flush batch and `awk` files. Nothing is ever written next to the
  script, and nothing needs managing by hand.
- Buffer access is `flock`-guarded end to end: appends and flushes take
  the same lock, and a flush atomically hands the buffer off to a
  private file before mailing it, so a line arriving mid-flush is never
  silently dropped, and a slow mail send never blocks new appends.
- Force an immediate flush manually with `mylogwatch FLUSH`.

## Development

There is nothing to build or install — the script is the deliverable.
Syntax-check it after any edit:

```bash
bash -n mylogwatch
```

### Testing without mailing anything

Don't test against a real MTA. Work on a *copy* with the sendmail pipe
stubbed out and runtime state pointed somewhere disposable — that still
exercises the real batching, locking, reverse-DNS and label-substitution
paths, and leaves the message in a file you can read:

```bash
mkdir -p /tmp/mw/run && cp mylogwatch /tmp/mw/ && cd /tmp/mw
sed -i 's#^ *SENDMAIL = .*#    SENDMAIL = "cat >> /tmp/mw_sent.log"#' mylogwatch
cat > mylogwatch.cfg <<'EOF'          # the cfg must sit next to the copy
RUNDIR="/tmp/mw/run"
WAIT_TO_SEND=2
AP_ETHERS='my-router      |AA:BB:CC:DD:EE:FF|^$|__END__'
EOF

./mylogwatch "Aug  7 10:00:00 host kernel: link up from 8.8.8.8"
./mylogwatch "Aug  7 10:00:01 host device AA:BB:CC:DD:EE:FF joined"
sleep 4 && cat /tmp/mw_sent.log
```

`./mylogwatch FLUSH` cuts the wait short. Diagnostics from non-tty runs
land in `$RUNDIR/mylogwatch.log`, which is the first place to look when
a line silently disappears, and `$RUNDIR/mylogwatch*` is worth clearing
between runs — a stale buffer or lock will skew the next test.

Testing a `PRIV_CMD` or `/etc/nullmailer/remotes` code path needs root;
otherwise set `EMAIL_DOMAIN` in the config and skip it.
