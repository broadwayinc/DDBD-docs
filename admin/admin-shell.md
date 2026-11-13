# Admin Shell

The admin shell is the terminal tool of the operating system user who runs DDBD. With it that user adds users and sets their access levels, reads and deletes any record, manages agent keys, backs up, exports and imports, and turns the Web Admin on or off. It reaches the server through a Unix socket in the data folder, never over the network, and no access level gives a network user these powers.

## Opening the shell

Open it while the server runs, as the same operating system user:

```bash
$ ddbd admin
```

Without `--data` it uses `$DDBD_DATA`, else `~/ddbd-data` (see [The data folder](/admin/command-line#the-data-folder)). For a server with another data folder, name it:

```bash
$ ddbd admin --data /srv/ddbd
```

It then waits at its own prompt, `ddbd admin>`. Every command on this page is typed there:

```
DDBD admin shell. Type `help` for commands.
ddbd admin> user add alice --level 10
Password for alice: correct-horse
Added alice (user id DS4gIE0n, level 10).
ddbd admin> status
  ...
ddbd admin> exit
```

`help` lists the commands; `exit`, `quit` or Ctrl-D leaves. Errors are printed as `<code>: <message>`, for example `NOT_FOUND: There is no user "bob".`

### One command without opening the shell

Any command on this page can also be run straight from the operating system shell, by putting `ddbd admin` (and `--data <folder>` if needed) in front of it. It runs once, prints the answer and exits, which suits scripts and cron:

| typed in the admin shell | the same, from the OS shell |
|---|---|
| `ddbd admin> status` | `$ ddbd admin status` |
| `ddbd admin> user list` | `$ ddbd admin --data /srv/ddbd user list` |
| `ddbd admin> collections >= prod` | `$ ddbd admin collections '>=' prod` |

::: warning Quote `**`, `>`, `<` and `|` in the OS shell
At the shell's own prompt these characters are plain text. Typed straight into bash or zsh they are not: `**` is expanded into file names, `>` and `<` redirect to and from files, and `|` starts a pipe. Put them in single quotes there:

```bash
$ ddbd admin records --level '**'
$ ddbd admin collections '>=' prod
```
:::

Put `--data` before the command: every word after the command name belongs to the command. Both forms work only while the server runs. The shell talks to the server over a Unix socket, `admin.sock`, in the data folder (mode 0600), and the server answers only connections from the operating system user who started it. From another account, run it as that user:

```bash
$ sudo -u ddbd ddbd admin --data /srv/ddbd status
```

::: warning
Only the OS user can use the socket. There is no admin password for the shell and no network path to it: the admin API of the [Web Admin](/admin/web-admin) is a separate, password-protected thing that the admin turns on with `web-password` from here.
:::

## Typing values

Commands take **arguments** (words in a fixed place, written `<like this>`) and **options** (`--name value`, in any order after the arguments). An option can be written with a space or with `=`: `--level 10` or `--level=10`. A switch is an option with no value, such as `--json`. Quote any value that has spaces: `--password "two words"`.

The types used in the tables below:

| type | what to type | examples |
|---|---|---|
| username | a username, or the user's id | `alice`, `DS4gIE0n` |
| level | two digits `00` to `99`; one digit is read as two (`5` = `05`). Where noted, `**` (private records) | `00`, `10`, `99`, `**` |
| password | any text, not empty; quote it if it has spaces | `correct-horse` |
| record id | the id DDBD gave the record (16 characters) | `VZOat4ixBwfABGAq` |
| key id | the id of an agent key (6 characters), from `agents` | `MFgYtk` |
| collection | a collection name, 1 to 256 characters | `products` |
| number | a whole number | `20` |
| folder, file | a path on the server machine | `/srv/backups/ddbd-2026-11-13` |
| JSON | a JSON object in single quotes | `'{"query":{"collection":"products"}}'` |
| switch | nothing: the option alone turns it on | `--json` |

### Catalog conditions

The catalog commands (`collections`, `tags`, `indexes`, `uniques`) take an optional condition after their arguments: an **operator**, a space, and a **text**. They are separate words, so put spaces around the operator.

| condition | finds names that | example |
|---|---|---|
| (nothing) | (lists everything) | `collections` |
| `<text>` | are exactly `<text>` | `collections products` |
| `= <text>` | are exactly `<text>` | `collections = products` |
| `> <text>` | sort after `<text>` | `collections > m` |
| `< <text>` | sort before `<text>` | `collections < m` |
| `>= <text>` | **start with** `<text>` | `collections >= prod` |
| `<= <text>` | **end with** `<text>` | `collections <= _log` |

## Users

There is no sign-up and no password reset: every account is made here (or on the Users page of the [Web Admin](/admin/web-admin)), and a user who lost the password gets a fresh account with `user del` and `user add`. See [Login and Sessions](/users/login).

### `user add`

Adds a user.

```
user add <username> [--level <level>] [--password <password>]
```

| argument | type | meaning |
|---|---|---|
| `<username>` | text, 1 to 256 characters, no spaces or control characters | the new user's name; must be unique |

| option | type | default | meaning |
|---|---|---|---|
| `--level` | level `00` to `99` | `00` | the user's access level |
| `--password` | password | asked: `Password for <username>:` (shown as you type) | the user's password |

```
ddbd admin> user add alice --level 10
Password for alice: correct-horse
Added alice (user id DS4gIE0n, level 10).
ddbd admin> user add bob --password "bob's secret"
Added bob (user id Lq0aZ7xe, level 00).
```

### `user list`

Lists every user.

```
user list [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off: a table | print JSON instead |

```
ddbd admin> user list
USERNAME             USER ID    LEVEL   RECORDS  SUBSCRIBERS  SUBSCRIBED
alice                DS4gIE0n   10            3            0           0
bob                  Lq0aZ7xe   00            0            0           0
```

### `user info`

Shows one user as JSON.

```
user info <username>
```

| argument | type | meaning |
|---|---|---|
| `<username>` | username | the user to show |

```
ddbd admin> user info alice
```

### `user level`

Sets a user's access level. It applies to the user's next request.

```
user level <username> <level>
```

| argument | type | meaning |
|---|---|---|
| `<username>` | username | the user to change |
| `<level>` | level `00` to `99` | the new level |

```
ddbd admin> user level alice 20
alice is now level 20.
```

### `user del`

Deletes the account and ends its sessions at once. Its records, grants, subscriptions, agent keys, feed and notifications are removed in the background.

```
user del <username> [--yes]
```

| argument | type | meaning |
|---|---|---|
| `<username>` | username | the user to delete |

| option | type | default | meaning |
|---|---|---|---|
| `--yes` | switch | off: asks you to type the username to confirm | delete without asking |

```
ddbd admin> user del bob
This deletes user bob and the 0 records bob uploaded.
Type the username to confirm: bob
Deleted bob. Removing records in the background (outbox 1).
ddbd admin> user del carol --yes
Deleted carol. Removing records in the background (outbox 1).
```

## Records

The admin reads every record whatever its level, including `**`, and deletes any record, including read-only ones.

### `records`

Lists records, newest first, private ones included. A listing prints one record per line: id, collection/level, indexes, tags, unique id, reference and data (shortened).

```
records [--collection <collection>] [--user <username>] [--level <level>] [--limit <number>] [--asc] [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--collection` | collection | every collection | only this collection |
| `--user` | username | every user | only records this user uploaded |
| `--level` | level `00` to `99`, or `**` | every level | only this level |
| `--limit` | number, `1` to `1000` | `50` | records per page |
| `--asc` | switch | off: newest first | oldest first |
| `--json` | switch | off: one line per record | print JSON instead |

```
ddbd admin> records --collection products --limit 2
VZOat4ixBwfABGAq  products/00  [price=1200 title="Desk Lamp"]  tags:sale  {"color":"red"}
VZOaq1aUWxKcBGAq  products/00  [price=800]  null
(2 shown; type `more` for the next page)
ddbd admin> records --user alice --level ** --json
```

### `record get`

Shows one record as JSON.

```
record get <record id>
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | the record to show |

```
ddbd admin> record get VZOat4ixBwfABGAq
```

### `record del`

Deletes one record, read-only ones too.

```
record del <record id>
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | the record to delete |

```
ddbd admin> record del VZOat4ixBwfABGAq
Deleted record VZOat4ixBwfABGAq.
```

### `query`

Runs a [`getRecords`](/api-reference/database#getrecords) request with admin rights. It takes the same JSON body a client would send, so conditions are written as in the API: `"condition": ">="`.

```
query '<json>' [--json]
```

| argument | type | meaning |
|---|---|---|
| `'<json>'` | JSON | a `getRecords` body; see [Fetching Records](/database/fetch) and [Indexing](/database/indexing) |

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off: one line per record | print JSON instead |

```
ddbd admin> query '{"query":{"collection":"products","index":{"name":"price","value":1000,"condition":">="}},"fetch":{"limit":20}}'
ddbd admin> query '{"query":{"collection":"products","tag":"sale"}}' --json
```

### `more`

Shows the next page of the last listing (`records`, `query` or a catalog command). It takes nothing.

```
ddbd admin> more
```

## Catalog

These list names in use, with counts. Each takes an optional [catalog condition](#catalog-conditions).

### `collections`

```
collections [<condition>] [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off | print JSON instead |

```
ddbd admin> collections
ddbd admin> collections >= prod
```

### `tags`

```
tags <collection> [<condition>] [--json]
```

| argument | type | meaning |
|---|---|---|
| `<collection>` | collection | the collection whose tags to list |

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off | print JSON instead |

```
ddbd admin> tags products
ddbd admin> tags products >= sa
```

### `indexes`

```
indexes <collection> [<condition>] [--order <order>] [--desc] [--json]
```

| argument | type | meaning |
|---|---|---|
| `<collection>` | collection | the collection whose index names to list |

A condition text ending in `.` (such as `shop.`) lists the children of that compound name.

| option | type | default | meaning |
|---|---|---|---|
| `--order` | one of `index_name`, `number_of_records`, `average_number`, `total_number`, `number_count`, `average_bool`, `total_bool`, `bool_count`, `string_count` (see [`getIndexes`](/api-reference/database#getindexes)) | `index_name` | the order of the list; with any order but `index_name`, a condition text ending in `.` keeps only its children and any other condition is ignored |
| `--desc` | switch | off: ascending | largest or last first |
| `--json` | switch | off | print JSON instead |

```
ddbd admin> indexes products
ddbd admin> indexes products shop.
ddbd admin> indexes products --order number_of_records --desc
```

### `uniques`

Lists unique ids and the record each belongs to.

```
uniques [<condition>] [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off | print JSON instead |

```
ddbd admin> uniques
ddbd admin> uniques >= order-
```

## Agent keys

Users make and revoke their own keys over the API (see [Agent Keys](/users/agent-keys)); the admin sees all of them and can revoke any. The key strings themselves are never shown.

### `agents`

Lists agent keys: all of them, or one user's.

```
agents [--user <username>]
```

| option | type | default | meaning |
|---|---|---|---|
| `--user` | username | every user | only this user's keys |

```
ddbd admin> agents
ddbd admin> agents --user alice
```

### `agent revoke`

Revokes any user's key. It stops working at once.

```
agent revoke <key id>
```

| argument | type | meaning |
|---|---|---|
| `<key id>` | key id | the key to revoke, from `agents` |

```
ddbd admin> agent revoke MFgYtk
Revoked agent key MFgYtk.
```

## Server

### `status`

Shows the version, data folder, port, TLS, counts (users, records, collections, sessions, connections, requests, listeners), outbox depth, limits in force, whether the Web Admin is on and its session count, and the loose-rule versions. It takes nothing.

```
ddbd admin> status
```

### `settings`

Shows the settings file path and every setting in force. A setting whose saved value differs is marked `(saved <value>, applies on next start)`. It takes nothing. Which settings apply at once and which wait for the next start is on the [Settings](/admin/settings) page.

```
ddbd admin> settings
```

### `backup`

Makes a consistent copy of the database while it runs. A copy of the settings file is put inside as `ddbd.toml.copy`. How to restore one is on the [Operations](/admin/operations) page.

```
backup <folder>
```

| argument | type | meaning |
|---|---|---|
| `<folder>` | folder, absolute path, must not exist yet | where to write the copy. The server process writes it, so it needs permission there |

```
ddbd admin> backup /srv/backups/ddbd-2026-11-13
Backup written to /srv/backups/ddbd-2026-11-13.
```

### `export`

Writes users, records, grants, subscriptions and agent keys as JSON lines (see [the export file format](/admin/operations#export-file-format)) and reports the counts.

```
export <file> [--collection <collection>] [--user <username>]
```

| argument | type | meaning |
|---|---|---|
| `<file>` | file; a relative path is resolved against the folder the command is run in; that folder must exist | the file to write |

| option | type | default | meaning |
|---|---|---|---|
| `--collection` | collection | every collection | only records of this collection |
| `--user` | username | every user | only this user and their records |

```
ddbd admin> export /srv/exports/all.jsonl
ddbd admin> export /srv/exports/products.jsonl --collection products
```

### `import`

Loads an export file. Reports the counts, the skipped and failed lines, and the first 20 failed lines with their line numbers.

```
import <file> [--replace | --skip-existing]
```

| argument | type | meaning |
|---|---|---|
| `<file>` | file; a relative path is resolved against the folder the command is run in | the export file to load |

| option | type | default | meaning |
|---|---|---|---|
| `--replace` | switch | off | overwrite users and records that already exist |
| `--skip-existing` | switch | on | keep users and records that already exist, and skip those lines |

```
ddbd admin> import /srv/exports/all.jsonl
ddbd admin> import /srv/exports/all.jsonl --replace
```

### `web-password`

Turns the [Web Admin](/admin/web-admin) on, or changes its password. Signs out every web admin session.

```
web-password [--password <password>]
web-password off
```

| option | type | default | meaning |
|---|---|---|---|
| `--password` | password, at least 8 characters | asked (shown as you type) | the new Web Admin password |

`web-password off` turns the Web Admin off and signs out every web admin session.

```
ddbd admin> web-password
New web admin password (shown as you type, at least 8 characters): a-long-password
ddbd admin> web-password off
```

### `secret rotate`

Makes a new server secret: every [private access key](/database/access-levels) stops working at once.

```
secret rotate [--yes]
```

| option | type | default | meaning |
|---|---|---|---|
| `--yes` | switch | off: asks you to type `rotate` to confirm | rotate without asking |

```
ddbd admin> secret rotate
This cancels every private access key. Type rotate to confirm: rotate
```

### `rebuild-loose`

Rewrites every loose-match key with this build's rules. Run it when `ddbd start` prints `NOTE: loose search rules changed`. It takes nothing.

```
ddbd admin> rebuild-loose
Rebuilt loose keys for 1520 records (1520 removed, 1520 written).
```

### Not in the shell

`backup`, `export` and `import` take paths on the server machine, and the server process reads or writes the files, so it needs permission there. Changing settings, saving records as a user, and the batch deletes have no shell command; they are on the [Web Admin](/admin/web-admin) and in its admin API.
