# log_digest

Sends a digest of the warnings and errors of a time span, from the systemd
journal and from the files rsyslog writes, by mail and XMPP. A message that is
in both, as when journald forwards to rsyslog, is counted once.

The digest starts with a table of the hosts and programs with the most
messages, errors first, followed by the messages themselves, up to `max_lines`
of them. With more, only the table is sent, with high priority.

```
Subject: [log_digest] server1.example.org: 7 message(s) 2026-10-02 17:00:00 - 2026-10-02 18:00:00

Log digest of server1.example.org for 2026-10-02 17:00:00 - 2026-10-02 18:00:00
Severity: warning or more severe; 2 include rule(s); 1 exclude rule(s)
Sources: journal, rsyslog
7 message(s) (journal: 4, rsyslog: 3), 3 duplicate(s) merged

Most messages by host and program:
  Count  Errors  Host     Program
      1       1  server1  kernel
      1       1  server2  nginx
      2       0  server1  sshd
      ...

Messages:
2026-10-02 17:31:19 server1 kernel err: EXT4-fs error: disk full
2026-10-02 17:40:29 printer cupsd[1] warning+: paper jam
...
```

A severity with `+` comes from an rsyslog file and is a bound, see
[rsyslog](#rsyslog); `?` marks a message whose severity is unknown, which only
an include rule can report.

## Requirements

- Python 3.7 or newer; PyYAML for a configuration file
- `journalctl`, and `systemd-analyze` for the full time syntax of journalctl
- a mail server on localhost, e.g. postfix or exim, for mail
- [go-sendxmpp](https://salsa.debian.org/mdosch/go-sendxmpp) for XMPP

## Usage

```sh
log_digest --print                                   # the last hour, printed
log_digest --print --since today -p err              # errors since midnight
log_digest --since "2026-10-02 08:00" --until "2026-10-02 12:00" --mail-to admin@example.org
log_digest --config /etc/log_digest.yaml             # as the systemd service runs it
log_digest --show-sources                            # the rsyslog files and their severities
```

| Option | Meaning |
|---|---|
| `-C`, `--config` | YAML configuration file, see `log_digest.yaml.example` |
| `-S`, `--since`, `-U`, `--until` | Time span in the syntax of journalctl (default: `-1h` and `now`) |
| `-p`, `--priority` | Severity like journalctl `-p`: a level and everything more severe, or `FROM..TO` (default: `warning`) |
| `-n`, `--max-lines` | List up to this many messages; above it send the table only, with high priority (default: 100) |
| `--top` | Rows of the host and program table (default: 10) |
| `--include`, `--exclude` | `FIELD=REGEX` rules, see [Filters](#filters); repeatable |
| `--no-journal`, `--no-rsyslog` | Leave out one of the sources |
| `--rsyslog-config` | rsyslog's configuration (default: `/etc/rsyslog.conf`) |
| `--journal-arg` | Argument for journalctl instead of `--merge`, e.g. `--directory=/var/log/journal/remote`; repeatable |
| `--mail-to`, `--xmpp-to` | Recipients; repeatable |
| `--print` | Print the digest instead of sending it; also the default without recipients |
| `-v`, `--verbose` | Explain on stderr what is read |

The command line overrides the configuration file. `--since` and `--until`
are interpreted by `systemd-analyze timestamp`, so `-1h`, `"1 hour ago"`,
`today`, `"2026-10-02 08:00"` and `@1790000000` work as in journalctl. The
journal and the rsyslog files are read for exactly the same span.

## Filters

A rule matches a message if all its fields match. Fields are `host`,
`program` (the syslog identifier, also `source`), `message` (also `content`),
`unit` (the systemd unit of a journal entry) and `origin` (`journal` or the
path of an rsyslog file). Values are Python regular expressions, searched
anywhere in the field.

- **include**: report a matching message whatever its severity, e.g.
  `--include 'message=segfault|Out of memory'`.
- **exclude**: leave out a matching message, also one an include rule would
  report.

On the command line a rule has one field, `FIELD=REGEX`; in the configuration
file a rule can combine several:

```yaml
include:
  - {program: '^smartd$', message: 'unreadable|uncorrectable'}
exclude:
  - {program: '^kernel$', message: 'IN=.*OUT='}
```

Include rules make the journal and rsyslog's files be read for every
severity, not only the selected ones.

## Sources

### journal

`journalctl -o json --merge` for the time span. `--merge` includes the
journals of other hosts, e.g. those systemd-journal-remote receives; change
it with `journal: {args: [...]}` or `--journal-arg`.

### rsyslog

The files come from rsyslog's configuration, including `$IncludeConfig` and
`include()`, legacy selectors (`*.=warning;*.=err -/var/log/warn`),
`action(type="omfile" ...)`, `if $syslogseverity <= 4 then ...` and dynamic
files (`?Template`, `dynaFile=`): `/var/log/hosts/%HOSTNAME%/warn` is read as
`/var/log/hosts/*/warn`. Rotated copies (`warn.1`, `warn-20261001.xz`) are
read if they were written to within the span. `--show-sources` lists the
files with the severities they can hold.

The common file formats do not store a message's severity. log_digest
therefore takes it from the configuration:

- A file that can only hold selected severities counts entirely, with the
  least severe of them marked `+`: with `-p warning`, openSUSE's
  `/var/log/warn` (warning, err, crit, alert, emerg) is read as `warning+`.
- A file that can also hold other severities, such as `/var/log/messages` or
  Debian's `/var/log/syslog`, gives no severity, so its lines count only
  for include rules. Their messages are usually in the journal anyway.
- A line that carries its priority, as in the RFC 5424 format
  (`RSYSLOG_SyslogProtocol23Format`) or with a `<PRI>` prefix, has its exact
  severity.

Conditions that are more than a conjunction of severity comparisons, such as
`if $programname == 'x' or ...`, count as all severities. For files the
configuration does not reveal, name them with their severities:

```yaml
rsyslog:
  files:
    - {path: '/var/log/remote/*/errors.log', priority: err}
```

On a central log server that writes every remote host to one `*.*` file, add
a file for warnings and worse, or one in a format with the priority, for
log_digest to read:

```
*.warning  action(type="omfile" dynaFile="RemoteWarn")
```

### Duplicates

Messages from the same host (compared by its short name), program and text
(with rsyslog's `#011` escapes undone) within `duplicate_window` seconds
(default 5) are one message, if they come from different origins: the
journal, or different rsyslog files. The journal's copy is kept, with its
exact severity. Repetitions within one origin stay separate messages.

## Sending

The digest goes by mail, the summary (the header and the table) by XMPP.
Without recipients it is printed. A digest without messages and problems is
not sent, unless `send_empty: true`.

- **Mail** goes to `localhost:25` by SMTP, which postfix and exim accept;
  `mail: {method: sendmail}` uses `/usr/sbin/sendmail -t` instead. With more
  than `max_lines` messages the mail carries `X-Priority: 1`,
  `Importance: high` and `Priority: urgent`, and the subject starts with
  `HIGH PRIORITY:`.
- **XMPP** messages have no priority: XMPP's priority belongs to the presence
  of a resource, not to a message. A high priority digest therefore starts
  with `HIGH PRIORITY:` as well, and also goes to the JIDs of
  `xmpp: {high_priority_to: [...]}`. go-sendxmpp reads its account from its
  configuration, or the file given as `xmpp: {config: ...}`.

Problems, such as an unreadable journal or rsyslog configuration, are listed
in the digest, so a broken setup does not stay silent.

## systemd

`systemd/log_digest.service` reads the last hour with
`/etc/log_digest.yaml`; `systemd/log_digest.timer` starts it every hour and
10 minutes after boot. The service runs as root, as rsyslog's configuration
and files are usually readable only by root, and waits for
`network-online.target`.

```sh
install -m 0755 log_digest /usr/bin/
install -m 0644 log_digest.yaml.example /etc/log_digest.yaml
install -m 0644 systemd/log_digest.service systemd/log_digest.timer /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now log_digest.timer
```

The hourly span and the run after boot can overlap or leave a gap of a few
seconds; the run after boot covers the hour before it, including the
shutdown.

## Tests

```sh
python3 -m unittest -v
```

The tests use a fake journalctl, rsyslog configurations in the layouts of
openSUSE and Debian, a fake SMTP server and a fake go-sendxmpp.

## License

GPL-2.0-or-later, see [LICENSE](LICENSE).
