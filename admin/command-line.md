# Command Line

DDBD is one program, `ddbd`. Install it on Linux with:

```bash
$ curl -fsSL https://cdn.broadwayinc.com/get/ddbd/v1 | sudo sh
```

On macOS, use `brew install broadwayinc/ddbd/ddbd` (see [Installing, upgrading and removing](/admin/operations#installing-upgrading-and-removing)). This page lists its commands and their options:

```
ddbd <command> [<options>]
```

| command | what it does |
|---|---|
| [`init`](#ddbd-init) | makes a data folder: settings, an empty database and (with `--tls`) a TLS certificate. Once per database |
| [`start`](#ddbd-start) | runs the server in the terminal. Every time |
| [`stop`](#ddbd-stop) | stops the running server cleanly. Only for the OS user who runs the server |
| [`restart`](#ddbd-restart) | stops the running server, then starts it again in this terminal |
| [`admin`](#ddbd-admin) | runs an admin command or opens the [Admin Shell](/admin/admin-shell). Only for the OS user who runs the server |
| [`connect`](#ddbd-connect) | opens the [User Shell](/admin/user-shell) against a running server, as a user |
| [`debug`](#ddbd-debug) | shows how values are encoded and sorted. Needs no server |
| [`version`](#ddbd-version) | prints the version |

Options can be written with a space or with `=`: `--data /srv/ddbd` or `--data=/srv/ddbd`. `ddbd --help` (or `ddbd <command> --help`) lists the options; `ddbd --version` prints the version.

## The data folder

Every command except `connect`, `debug` and `version` works on one **data folder**: the folder that holds the settings file and the database. It is chosen in this order:

| | where it comes from | example |
|---|---|---|
| 1 | the `--data` option | `ddbd start --data /srv/ddbd` |
| 2 | the environment variable `DDBD_DATA` | `export DDBD_DATA=/srv/ddbd`, then `ddbd start` |
| 3 | the default: `ddbd-data` in your home folder (`~/ddbd-data`, which is `$HOME/ddbd-data`) | `ddbd start` uses `/home/alice/ddbd-data` on Linux, `/Users/alice/ddbd-data` on macOS |

The default is the same folder whatever folder you run the command in, so `ddbd init` and then `ddbd start` from anywhere find the same database. Each operating system user has their own home folder, and so their own default: a server started by `alice` uses `/home/alice/ddbd-data`, while one started with `sudo` uses root's (`/root/ddbd-data`). For a server run as a service, give `--data` explicitly. Up to version 0.3.0 the default was `./ddbd-data` in the current folder; a data folder made that way keeps working with `--data <folder>`.

Quick start, with the default folder:

```bash
$ ddbd init
$ ddbd start
```

Or with a folder of your choice:

```bash
$ ddbd init --data /srv/ddbd
$ ddbd start --data /srv/ddbd
```

## `ddbd init`

Makes a data folder with settings and a database, and with `--tls` a TLS certificate.

```
ddbd init [--port <port>] [--bind <address>] [--data <folder>] [--tls]
```

| option | type | default | meaning |
|---|---|---|---|
| `--port` | number, `1` to `65535` | `7070` | the port to listen on |
| `--bind` | IP address | `0.0.0.0` (every network) | the address to listen on; `127.0.0.1` = this machine only |
| `--data` | folder path | `$DDBD_DATA`, else `~/ddbd-data` | the data folder to create (see [The data folder](#the-data-folder)) |
| `--tls` | switch (no value) | off: plain HTTP | DDBD serves HTTPS itself, with a self-signed certificate made now. Without it, DDBD speaks plain HTTP, which is meant for `localhost` or for running behind a load balancer or proxy that serves HTTPS |

```bash
$ ddbd init
$ ddbd init --port 7171 --bind 127.0.0.1
$ ddbd init --data /srv/ddbd --tls
```

The folder gets mode 0700. Inside it:

| path | what it is |
|---|---|
| `ddbd.toml` | the settings file (see [Settings](/admin/settings)) |
| `db/` | the database |
| `tls/cert.pem`, `tls/key.pem` | with `--tls`: the self-signed certificate and its key (key mode 0600) |

With `--tls`, the certificate is made for the names `localhost`, `127.0.0.1` and the machine's host name (from `/etc/hostname`), and `init` prints its SHA-256 fingerprint, which clients can check. Without `--tls` no certificate is made, and `init` warns that DDBD speaks plain HTTP. `init` also creates the empty database and prints the next two commands to run. It refuses a folder that already holds a `ddbd.toml`.

::: tip
`--port`, `--bind` and `--tls` only set the first values. To change them later, edit `ddbd.toml` or use the Web Admin's Settings page, and restart. See [Operations](/admin/operations#changing-the-configuration).
:::

## `ddbd start`

Runs the server in the terminal.

```
ddbd start [--data <folder>] [--notify-keep <duration>] [--private-key-ttl <duration>] [--max-indexes <number>] [--max-tags <number>]
```

| option | type | default | meaning |
|---|---|---|---|
| `--data` | folder path | `$DDBD_DATA`, else `~/ddbd-data` | the data folder (see [The data folder](#the-data-folder)) |
| `--notify-keep` | [duration](#durations) | `notify_keep` in `ddbd.toml`; `1d` (1 day) if unset | how long a notification is kept for a subscriber who is not listening, so that `listen` can still deliver it later |
| `--private-key-ttl` | [duration](#durations) | `private_key_ttl` in `ddbd.toml`; `1h` (1 hour) if unset | how long a [private access key](/database/access-levels) works after it is made |
| `--max-indexes` | whole number, `0` or more | `max_indexes` in `ddbd.toml`; `0` if unset | the most indexes one record may have; `0` = unlimited |
| `--max-tags` | whole number, `0` or more | `max_tags` in `ddbd.toml`; `0` if unset | the most tags one record may have; `0` = unlimited |

Options override the settings file for this run only; the file is not changed.

```bash
$ ddbd start
$ ddbd start --data /srv/ddbd
$ ddbd start --notify-keep 7d --private-key-ttl 30m
$ ddbd start --max-indexes 20 --max-tags 50
```

### Durations

`--notify-keep` and `--private-key-ttl` (and the `notify_keep`, `private_key_ttl` and `login_max_delay` settings in `ddbd.toml`) take a **duration**: a whole number followed by one unit, with no space needed.

| unit | means | example | equals |
|---|---|---|---|
| `ms` | milliseconds | `500ms` | half a second |
| `s` | seconds | `90s` | 1 minute 30 seconds |
| `m` | minutes | `30m` | 30 minutes |
| `h` | hours | `12h` | 12 hours |
| `d` | days | `7d` | 7 days |
| `w` | weeks | `2w` | 14 days |
| (none) | seconds | `3600` | 1 hour |

Rules:

- **Whole numbers only, above zero.** `1h`, `90m` and `5400s` are all fine; `0`, `1.5h` and `-1h` are refused.
- **One unit only.** Write `90m`, not `1h30m`.
- **Lowercase units.** `1d` works; `1D` and `1 day` do not.
- **No upper limit.** `52w` (a year) is accepted.

A value that does not follow these rules stops `start` before anything runs, naming the setting:

```
notify_keep = "1.5h" is not a duration above zero. Use forms like 30m, 12h, 1d.
```

Sensible values for the two options:

| option | typical values | what a smaller value does | what a larger value does |
|---|---|---|---|
| `--notify-keep` | `1h` to `30d`; default `1d` | a subscriber who is offline longer than this misses the notifications from that time (`listen` reports a gap) | more notifications wait on disk for users who never come back |
| `--private-key-ttl` | `5m` to `1d`; default `1h` | a private access key must be asked for again sooner | a key that leaks can be used for longer. Every key can be cancelled at once with `secret rotate` in the [Admin Shell](/admin/admin-shell#server) |

### What you see in the terminal

| output | meaning |
|---|---|
| a banner | version, data folder, address and whether TLS is on, the certificate fingerprint, the admin socket path, the limits in force, and the number of users, records and collections |
| `Ready.` | the server is listening |
| one line per request | time, user (the username and the key name joined by `/`, such as `alice/shop-bot`, for an agent key; `admin(web)` for the Web Admin), method, collection, milliseconds, and an error code if any |
| `[outbox <number> · req/s <number> · conns <number>]` | checked every 10 seconds and printed when it differs from the last one: pending background work, request rate, open connections |

The log lines are explained in [Operations](/admin/operations#reading-the-log). Ctrl-C (SIGINT) or SIGTERM stops it cleanly: new connections are refused, the requests in progress finish, the database closes, and the admin socket file is removed; queued background work continues at the next start. See [Operations](/admin/operations#stopping-the-server). With TLS on, a plain `http://` request to the port is refused and never redirected: it answers `400`, and the request is not run (see [Operations](/admin/operations#plain-http-on-a-tls-port)).

If the loose-match rules of this build differ from the ones stored in the database, the banner says so (`NOTE: loose search rules changed ...`) and tells you to run `rebuild-loose` in the [Admin Shell](/admin/admin-shell#server).

## Why `init` and `start` are separate

`init` is done **once** per data folder; `start` is done **every time** the server runs.

| | `ddbd init` | `ddbd start` |
|---|---|---|
| when | once, when you set up a new database | every time you run the server (after a reboot, from systemd, after changing settings) |
| what it does | creates the data folder (mode 0700), writes `ddbd.toml` with your port, address and TLS choice, makes the TLS certificate and key, creates the empty database and its server secret, prints the certificate fingerprint | reads `ddbd.toml`, opens the existing database, listens on the port and the admin socket, runs until stopped |
| on an existing folder | refuses (`<folder> is already set up. Start it with: ddbd start --data <folder>`) | runs it |
| on a missing folder | creates it | refuses (``Could not read <folder>/ddbd.toml: ... Run `ddbd init --data <folder>` first.``) |
| changes anything on disk | yes, it creates everything | `start` itself writes no settings and no certificate; it writes the database and the admin socket while running |

Why two steps instead of one command that does both:

- **Setup must never repeat by accident.** If starting also set up, a restart could make a new certificate, and every client that pinned the old fingerprint would refuse to connect; or it could rewrite the settings you edited.
- **A mistyped folder fails loudly.** `start --data /srv/dadabdaa` (typo) stops with an error. If `start` created missing folders, it would quietly start a new, empty database, and users would think their data was gone.
- **Choices made once are kept.** Port, address and TLS are chosen at `init` and saved; `start` needs no options, so a service manager only ever runs `ddbd start --data <folder>`.
- **One process per database.** `start` refuses if the database is already open in another DDBD process, so two servers can never write the same files (see below).

## Running more than one server

One machine can run several DDBD servers at once, each with **its own data folder and its own port**. The port is a setting of the data folder (`port` in its `ddbd.toml`, set by `init --port`), so give each folder a different one:

```bash
$ ddbd init --data /srv/ddbd-shop --port 7070
$ ddbd init --data /srv/ddbd-blog --port 7071
$ ddbd start --data /srv/ddbd-shop     # in one terminal or service
$ ddbd start --data /srv/ddbd-blog     # in another
```

Each server is a separate database with its own users, records, admin socket and Web Admin password. `ddbd admin`, `stop` and `restart` reach the right one through `--data`.

**Two servers cannot share one data folder.** The database inside allows one process at a time. A second `ddbd start` on a folder whose server is already running stops at once, before it listens on any port and without touching the running server or its files:

```
The database at /srv/ddbd-shop/db is already open in another DDBD process (...).
```

To serve the same data on another port, change `port` in `ddbd.toml` and restart the one server. To have two servers with the same data, copy it: make a [backup](/admin/admin-shell#server) and start the copy as its own folder (the two then change separately).

Two folders that ask for the same port cannot both run either: the second stops with `Could not listen on 0.0.0.0:7070: Address already in use (os error 98)`.

## `ddbd stop`

Stops the running server cleanly, the same way as Ctrl-C.

```
ddbd stop [--data <folder>]
```

| option | type | default | meaning |
|---|---|---|---|
| `--data` | folder path | `$DDBD_DATA`, else `~/ddbd-data` | the data folder of the server to stop |

```bash
$ ddbd stop
$ ddbd stop --data /srv/ddbd
```

It refuses new requests, finishes the ones in progress, closes the database and removes `admin.sock`. Queued background work stays in the outbox and continues at the next start, so a stop takes about a second. The command asks the server over the admin socket, so only the operating system user who runs the server can use it, and it works however the server was started (in a terminal, in the background or as a service). It waits until the server process has ended:

```
Stopping DDBD (pid 214013): it finishes the requests in progress first...
Stopped.
```

When no server is running for that folder, it says so and exits with status 1: `No DDBD is running for /srv/ddbd (could not open /srv/ddbd/admin.sock: ...)`.

For a server that runs as a systemd service, use `systemctl stop ddbd` instead, so systemd knows it was stopped on purpose. For a Homebrew service on macOS, use `brew services stop ddbd`, or it starts again at the next login.

## `ddbd restart`

Stops the running server (as `ddbd stop` does), then starts it again **in this terminal**, in the foreground.

```
ddbd restart [--data <folder>] [--notify-keep <duration>] [--private-key-ttl <duration>] [--max-indexes <number>] [--max-tags <number>]
```

It takes the same options as [`ddbd start`](#ddbd-start), with the same types and defaults:

| option | type | default | meaning |
|---|---|---|---|
| `--data` | folder path | `$DDBD_DATA`, else `~/ddbd-data` | the data folder of the server to restart |
| `--notify-keep` | [duration](#durations) | from `ddbd.toml` (`1d`) | as for `start` |
| `--private-key-ttl` | [duration](#durations) | from `ddbd.toml` (`1h`) | as for `start` |
| `--max-indexes` | whole number, `0` or more | from `ddbd.toml` (`0`) | as for `start` |
| `--max-tags` | whole number, `0` or more | from `ddbd.toml` (`0`) | as for `start` |

```bash
$ ddbd restart
$ ddbd restart --data /srv/ddbd --notify-keep 3d
```

If no server is running for the folder, it says so and starts one. A server that was running in the background or in another terminal ends up running in the terminal where `restart` was typed. To restart a server where it already runs (its terminal, its log file, its process id), use **Restart** on the Tools page of the [Web Admin](/admin/web-admin). For a systemd service, use `systemctl restart ddbd`; for a Homebrew service, `brew services restart ddbd`.

## `ddbd admin`

Runs one admin command and exits, or, without a command, opens the [Admin Shell](/admin/admin-shell).

```
ddbd admin [--data <folder>] [<admin command> ...]
```

| option | type | default | meaning |
|---|---|---|---|
| `--data` | folder path | `$DDBD_DATA`, else `~/ddbd-data` | the data folder of the running server |
| `<admin command>` | words | none: opens the shell | one [admin shell command](/admin/admin-shell), such as `status` or `user add alice --level 10` |

It talks to the running server over the admin socket in the data folder, so it works only while the server runs, and only for the operating system user who started it. When the server is not running it stops with `Could not reach DDBD at <folder>/admin.sock (...). Is it running?`. Put `--data` before the command: every word after the command name belongs to the command.

```bash
$ ddbd admin status
$ ddbd admin --data /srv/ddbd user add alice --level 10
$ ddbd admin
```

The last one opens the shell, which waits at its own prompt:

```
ddbd admin>
```

## `ddbd connect`

Opens the [User Shell](/admin/user-shell), or runs one command and exits (signing out afterwards).

```
ddbd connect <address>[:<port>] [--user <username>] [--password <password>] [--key <agent key>] [--tls] [--trust] [<user command> ...]
```

| argument or option | type | default | meaning |
|---|---|---|---|
| `<address>[:<port>]` | host name or IP address, then optionally `:` and a port | port `7070` | the server, such as `localhost`, `127.0.0.1:7070` or `db.example.com:7171` |
| `--user` | username | asked (`Username:`) | the user to sign in as |
| `--password` | text | asked (`Password:`, shown as you type) | that user's password |
| `--key` | agent key, starts with `dk_` | none | sign in with an [agent key](/users/agent-keys) instead of a username and password |
| `--tls` | switch (no value) | off | the server serves HTTPS itself (it was set up with `--tls`); the shell then checks its certificate as described below |
| `--trust` | switch (no value) | off | on the first connection to an address, trust its certificate without asking (and remember it) |
| `<user command>` | words | none: opens the shell | one [user shell command](/admin/user-shell), such as `get products` |

Put `connect`'s own options before the command: every word after the command name belongs to the command.

```bash
$ ddbd connect 127.0.0.1:7070 --user alice
$ ddbd connect db.example.com:7070 --tls --user alice
$ ddbd connect localhost --key dk_... get products --limit 5
```

The first one asks for the password, then waits at the shell's prompt:

```
alice> post products --index price=1200 --index "title=Desk Lamp" --tags sale --data '{"color":"red"}'
alice> get products --index "price >= 1000"
```

With `--tls`, on the first connection to an address the shell shows the certificate fingerprint and asks `Certificate fingerprint SHA256 ... - trust it? (yes/no)`; only `yes` continues. Trusted fingerprints are kept in `$XDG_CONFIG_HOME/ddbd/known_hosts` (normally `~/.config/ddbd/known_hosts`), one `<address>:<port> <fingerprint>` per line. If a server's certificate later changes, the connection is refused (`the server certificate changed (expected ..., got ...)`), and `--trust` does not override it; delete that line to trust the new certificate.

## `ddbd debug`

Shows how values are encoded. It does not need a server.

```
ddbd debug <form> <value> ...
```

| form | arguments | shows |
|---|---|---|
| `key` | an index name, then one value | the index key bytes for that value |
| `sort` | two or more values | the values in the order the database keeps them |
| `num` | one or more numbers | the 8 sortable bytes of each number, and the number decoded back |
| `loose` | text | the loose form, its reverse, and the Korean initials form; says if the text would search initials |
| `reverse` | text | the name reversed per `.` segment, and the text reversed |
| `id` | a user id | a sample record id and the time it carries |

Values are numbers if they read as numbers (negative ones too), `true`/`false` as booleans, otherwise strings.

```bash
$ ddbd debug sort -1.5 2 apple true
$ ddbd debug key price 1200
$ ddbd debug loose "Café Crème"
```

## `ddbd version`

Prints `ddbd` and the version.

```bash
$ ddbd version
ddbd 0.3.1
```
