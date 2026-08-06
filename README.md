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

   or point a syslog daemon's program/exec action, a udev rule, or a
   cron job at it — anything that can call it once per line of
   interest.

4. If it won't run as root, set `PRIV_CMD` in `mylogwatch.cfg` (e.g.
   `PRIV_CMD="sudo"`) — root is needed to read
   `/etc/nullmailer/remotes` for smart-host detection.

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
