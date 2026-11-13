# User Shell

The user shell is a terminal client for the HTTP API. It signs in as a user (or with an agent key), and each command sends an ordinary API call, so it is a way to try a query or look at a record before writing the same request in code.

## Opening the shell

Open it with [`ddbd connect`](/admin/command-line#ddbd-connect), giving the server's address and port:

```bash
$ ddbd connect 127.0.0.1:7070 --user alice
```

It asks for the password, prints `Signed in as alice (level 10).`, then waits at a prompt made of the username and `>`. Every command on this page is typed there:

```
alice> post products --index price=1200 --index "title=Desk Lamp" --tags sale --data '{"color":"red"}'
Saved record VZOat4ixBwfABGAq
alice> get products --index "price >= 1000"
VZOat4ixBwfABGAq  products/00  [price=1200 title="Desk Lamp"]  tags:sale  {"color":"red"}
alice> exit
```

With an agent key the prompt is the username and the key name, such as `alice/shop-bot>`. `help` lists the commands; `exit`, `quit`, `logout` or Ctrl-D signs out and leaves. Errors are printed as `<code>: <message>`, for example `NO_ACCESS: You have no access to record VZOat4ixBwfABGAq.`

A command can also be run once from the operating system shell, by putting it after the `connect` options. The shell signs in, runs it, signs out and exits:

```bash
$ ddbd connect 127.0.0.1:7070 --user alice --password correct-horse get products --limit 5
```

There, the operating system shell reads the line first: put conditions in quotes as a whole (`get products --index "price >= 1000"`), and quote `**`, `>`, `<` and `|` when they stand alone (`get orders --level '**'`, `collections '>=' prod`). Unquoted, `**` is expanded into file names and `>` redirects into a file.

## Typing values

This section is the reference for every value you type in the shell: what each type looks like, how a value becomes a number, a boolean or a string, how to set an index, and how to write a condition. The command sections below link back here.

Commands take **arguments** (words in a fixed place, written `<like this>`) and **options** (`--name value`, in any order after the command name). An option can be written with a space or with `=`: `--level 05` or `--level=05`. A **switch** is an option with no value, such as `--json`.

The line is split into words the way a Unix shell splits it: spaces separate words, and quotes keep a value with spaces (or with characters such as `<`, `>`, `|`, `$`) together. Use double quotes `"..."` or single quotes `'...'`; single quotes are easiest for JSON and for `$` names:

```
alice> get products --index "title = Desk Lamp"
alice> post notes --data '{"text":"hello world"}'
alice> get products --by '$updated'
```

### Types

| type | what to type | examples |
|---|---|---|
| collection | a collection name: 1 to 256 characters, no control characters | `products` |
| level | two digits `00` to `99`, or `**` for private; one digit is read as two (`5` = `05`) | `00`, `10`, `**` |
| record id | the id DDBD gave a record (16 characters) | `VZOat4ixBwfABGAq` |
| user id | the id of a user (8 characters), shown by `whoami` and in records | `DS4gIE0n` |
| unique id | an id you chose for a record | `order-1042` |
| value | a number, `true`/`false`, or text; see [Values](#values-numbers-booleans-and-text) | `1200`, `-3.5`, `true`, `Desk Lamp` |
| index | an index name and a value joined by `=`; see [Setting an index](#setting-an-index-post-and-update) | `price=1200`, `"title=Desk Lamp"` |
| condition | an index condition, in quotes; see [Index conditions](#index-conditions-get) | `"price >= 1000"`, `"price 100..500"` |
| list | items separated by commas, no spaces needed | `sale,new` |
| JSON | a JSON object in single quotes | `'{"color":"red"}'` |
| number | a whole number | `20` |
| duration | a whole number and a unit: `ms`, `s`, `m`, `h`, `d` or `w` (see [Durations](/admin/command-line#durations)) | `30m`, `12h`, `30d` |
| switch | nothing: the option alone turns it on | `--json` |

### Values: numbers, booleans and text

Wherever a **value** is typed (in `--index price=1200` and in conditions such as `"price >= 1000"`), the shell decides its type the same way:

| you type | it is sent as | type |
|---|---|---|
| `1200`, `-5`, `3.14` | `1200`, `-5`, `3.14` | number |
| `true`, `false` | `true`, `false` | boolean |
| `Desk Lamp`, `red`, `0x1F` | `"Desk Lamp"`, `"red"`, `"0x1F"` | string (text) |
| `"0042"` (inner double quotes) | `"0042"` | string, even though it looks like a number |

Numbers, booleans and strings are different values: an index saved as the number `42` is not found by the string `"42"`. To save or search digits as text (a zip code, a code with leading zeros), put double quotes **around the value, inside the outer quotes**:

```
alice> post products --index 'code="0042"'
alice> get products --index 'code = "0042"'
```

Without the inner quotes, `code=0042` is the number 42.

### Setting an index (`post` and `update`)

`--index` on `post` and `update` sets one index: the **name**, an equals sign `=`, and the **value**, with no spaces around `=`. Repeat `--index` for more indexes.

```
--index price=1200
```

| part | type | rule |
|---|---|---|
| name | text, 1 to 256 characters, before the first `=` | your own index name, such as `price`, or a compound name with dots, such as `shop.brand`. Names cannot start with `$` (those are built in) or end with `.` |
| `=` | | separates the name from the value. Only the first `=` counts, so the value may contain `=` |
| value | [value](#values-numbers-booleans-and-text) | everything after the first `=` |

A value with spaces needs quotes around the whole option value:

```
alice> post products --index price=1200 --index "title=Desk Lamp" --index shop.brand=acme
alice> post products --index in_stock=true --index 'sku="00731"'
```

### Index conditions (`get`)

`--index` on `get` is different from `post`: it takes **one condition** that selects records, in quotes. A condition is the index name, an **operator**, and a value. Spaces around the operator are optional (`"price>=1000"` works), but they make it easier to read.

| operator | meaning for numbers and booleans | meaning for text | example |
|---|---|---|---|
| (none) | every record that has the index, in value order | the same | `--index price` |
| `=` | equal | equal | `--index "color = red"` |
| `>` | greater | after, in character order | `--index "price > 1000"` |
| `<` | less | before, in character order | `--index "price < 1000"` |
| `>=` | greater or equal | **starts with** | `--index "price >= 1000"`, `--index "title >= Desk"` |
| `<=` | less or equal | **ends with** | `--index "price <= 1000"`, `--index "file <= .pdf"` |
| `..` | from the first value to the second, both included (a space after the name, no operator) | the same | `--index "price 1000..2000"` |

Put `~` before an operator (`~=`, `~>`, `~<`, `~>=`, `~<=`) or before a range (`name ~low..high`) for a **loose match**: case, accents, width, spacing and punctuation are ignored, and the value is always sent as text. `--ignore-case` makes the whole condition loose too.

```
alice> get products --index "title ~= desk lamp"
alice> get products --index "title ~>= desk"
alice> get products --index "title >= Desk" --ignore-case
```

Two kinds of index names work with these operators besides your own:

| name | what it is | example |
|---|---|---|
| `<compound name>.` (ending in `.`) | the children of a compound name, such as `shop.brand` and `shop.color` under `shop.`. The condition is on the child's **name**: equal, starts with or ends with | `--index "shop. = brand"`, `--index "shop. >= b"` |
| `$uploaded`, `$updated` | upload and update time, in milliseconds | `--index '$updated > 1793230000000'` |
| `$referenced_count` | how many records point to the record | `--index '$referenced_count >= 2'` |
| `$user_id` | the uploader's user id | `--index '$user_id = DS4gIE0n'` |

Write `$` names in single quotes so that nothing replaces them. See [Indexing](/database/indexing) for how each condition is answered.

### Catalog conditions

The catalog commands (`collections`, `tags`, `indexes`, `uniques`) take an optional condition on the **name** after their arguments: an operator, a space, and a text, as **separate words**.

| condition | finds names that | example |
|---|---|---|
| (nothing) | (lists everything) | `collections` |
| `<text>` | are exactly `<text>` | `collections products` |
| `= <text>` | are exactly `<text>` | `collections = products` |
| `> <text>` | sort after `<text>` | `collections > m` |
| `< <text>` | sort before `<text>` | `collections < m` |
| `>= <text>` | **start with** `<text>` | `tags products >= sa` |
| `<= <text>` | **end with** `<text>` | `uniques <= -2026` |

### Removing a value on `update`

On `update`, an empty value removes what the record had: `--tags ""` removes all tags, `--unique ""` the unique id, `--ref ""` the reference. `--no-index` removes all indexes.

## Records

### `post`

Saves a new record. Prints its record id, or the whole record with `--json`.

```
post <collection> [--level <level>] [--index <name=value>]... [--tags <list>] [--unique <unique id>] [--ref <record id>] [--readonly] [--data '<json>'] [--source '<json>'] [--subscription '<json>'] [--notification "<title>|<body>"] [--json]
```

| argument | type | meaning |
|---|---|---|
| `<collection>` | collection | the collection to save into; it is created with its first record |

| option | type | default | meaning |
|---|---|---|---|
| `--level` | level `00` to `99`, or `**` | `00` | who can read it: users at this level or above; `**` = private (you, and users you grant) |
| `--index` | [index](#setting-an-index-post-and-update) `name=value`; repeat for more | no indexes | an index to find the record by |
| `--tags` | list | no tags | tags, separated by commas |
| `--unique` | unique id | none | an id of your choice; no other record may have it |
| `--ref` | record id or unique id | none | the record this one points to (see [Referencing](/database/referencing)) |
| `--readonly` | switch | off | only the admin can change or delete it afterwards |
| `--data` | JSON (or any JSON value) | `null` | the record's data |
| `--source` | JSON | all off | rules for records that point to this one, such as `'{"referencing_limit":10}'` (see [Referencing](/database/referencing)) |
| `--subscription` | JSON with `true`/`false` fields: `is_subscription_record`, `upload_to_feed`, `notify_subscribers`, `feed_referencing_records`, `notify_referencing_records` | all `false` | feed and notification behaviour (see [Subscriptions](/database/subscription)) |
| `--notification` | text: a title, `\|`, then a body, in quotes | none | the notification sent when `notify_subscribers` is on |
| `--json` | switch | off: prints the record id | print the whole saved record |

```
alice> post notes
Saved record VZObA2kq1xLdBGAq
alice> post products --index price=1200 --index "title=Desk Lamp" --tags sale,new --data '{"color":"red"}'
Saved record VZOat4ixBwfABGAq
alice> post orders --level ** --unique order-1042 --ref VZOat4ixBwfABGAq --data '{"qty":2}'
alice> post posts --subscription '{"upload_to_feed":true,"notify_subscribers":true}' --notification "New post|Read it now"
```

### `update`

Changes a record: yours, and not read-only. Only the options given change; everything else stays.

```
update <record id> [--collection <collection>] [--level <level>] [--index <name=value>]... [--no-index] [--tags <list>] [--unique <unique id>] [--ref <record id>] [--readonly] [--data '<json>'] [--source '<json>'] [--subscription '<json>'] [--json]
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | the record to change |

| option | type | default | meaning |
|---|---|---|---|
| `--collection` | collection | unchanged | move the record to another collection |
| `--level` | level `00` to `99`, or `**` | unchanged | the new level |
| `--index` | [index](#setting-an-index-post-and-update) `name=value`; repeat for more | unchanged | **replaces all** the record's indexes with these |
| `--no-index` | switch | off | removes all indexes |
| `--tags` | list, or `""` | unchanged | replaces the tags; `""` removes them |
| `--unique` | unique id, or `""` | unchanged | the new unique id; `""` removes it |
| `--ref` | record id or unique id, or `""` | unchanged | the new reference; `""` removes it |
| `--readonly` | switch | off | make the record read-only |
| `--data` | JSON | unchanged | replaces the data |
| `--source` | JSON | unchanged | as for `post` |
| `--subscription` | JSON | unchanged | as for `post` |
| `--json` | switch | off: one line | print the whole updated record |

```
alice> update VZOat4ixBwfABGAq --index price=990 --index "title=Desk Lamp"
Updated VZOat4ixBwfABGAq  products/00  [price=990 title="Desk Lamp"]  tags:sale,new  {"color":"red"}
alice> update VZOat4ixBwfABGAq --tags ""
alice> update VZOat4ixBwfABGAq --data '{"color":"blue"}' --level 05
```

### `get`

Reads records. Prints one record per line: record id, collection/level, indexes in `[...]`, tags, unique id, reference, and the data (shortened). `--json` prints full JSON. A listing with more pages ends with ``(<number> shown; type `more` for the next page)``.

```
get <collection> [--index "<condition>"] [--ignore-case] [--by <index name>] [--tag <tag>] [--ref <record id>] [--user <user id>] [--level <level>] [--sub <user id>] [--limit <number>] [--desc] [--json]
get --id <record id> [--key <private key>] [--json]
get --unique <unique id> [--key <private key>] [--json]
```

| argument | type | meaning |
|---|---|---|
| `<collection>` | collection | the collection to read. Required, unless `--id` or `--unique` is given. `--collection <collection>` works too |

Options for a collection:

| option | type | default | meaning |
|---|---|---|---|
| `--index` | [condition](#index-conditions-get), in quotes | none | one index condition |
| `--ignore-case` | switch | off | makes the `--index` condition a loose match, like `~` |
| `--by` | index name, such as `price` or `'$updated'` | upload order, oldest first | sort by this index, without a condition: only records that have it, in value order. Ignored when `--index` is given |
| `--tag` | text | none | only records with this tag |
| `--ref` | record id or unique id | none | only records pointing to this record |
| `--user` | user id | none | only records this user uploaded |
| `--level` | level `00` to `99`, or `**` | every level you can read | only this level |
| `--sub` | user id | none | that user's subscription records (you must subscribe to them) |
| `--limit` | number, `1` to `1000` | `50` | records per page |
| `--desc` | switch | off: ascending | descending order |
| `--json` | switch | off: one line per record | print full JSON |

Options for one record:

| option | type | default | meaning |
|---|---|---|---|
| `--id` | record id | none | read this one record |
| `--unique` | unique id | none | read the record with this unique id |
| `--key` | private access key, from `key` | none | read a record you could not otherwise read (see [`key`](#key)) |
| `--json` | switch | off | print full JSON |

```
alice> get products
alice> get products --index "price >= 1000" --limit 10
alice> get products --index "price 1000..2000" --desc
alice> get products --index "title ~>= desk"
alice> get products --by '$updated' --desc
alice> get products --tag sale --level 05
alice> get reviews --ref VZOat4ixBwfABGAq
alice> get --id VZOat4ixBwfABGAq --json
alice> get --unique order-1042 --key pk_...
```

### `more`

Shows the next page of the last listing (`get`, `feed`, `subscriptions` or a catalog command) in the same shell session. It takes nothing.

```
alice> get products --limit 2
VZOat4ixBwfABGAq  products/00  [price=1200 title="Desk Lamp"]  tags:sale  {"color":"red"}
VZOaq1aUWxKcBGAq  products/00  [price=800]  null
(2 shown; type `more` for the next page)
alice> more
```

### `del`

Deletes one or more of your records and reports each one.

```
del <record id> [<record id> ...]
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id; one or more, separated by spaces | the records to delete |

```
alice> del VZOat4ixBwfABGAq VZOaq1aUWxKcBGAq
Deleted VZOat4ixBwfABGAq
Not deleted VZOaq1aUWxKcBGAq: You cannot delete record VZOaq1aUWxKcBGAq: only its uploader can.
```

## Catalog

These list names in use, with counts, as JSON lines. Each takes an optional [catalog condition](#catalog-conditions).

### `collections`

```
collections [<condition>] [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off: JSON lines | print one JSON document |

```
alice> collections
alice> collections >= prod
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
| `--json` | switch | off | print one JSON document |

```
alice> tags products
alice> tags products >= sa
```

### `indexes`

```
indexes <collection> [<condition>] [--order <order>] [--desc] [--json]
```

| argument | type | meaning |
|---|---|---|
| `<collection>` | collection | the collection whose index names to list. A condition text ending in `.` (such as `shop.`) lists the children of that compound name |

| option | type | default | meaning |
|---|---|---|---|
| `--order` | one of `index_name`, `number_of_records`, `average_number`, `total_number`, `number_count`, `average_bool`, `total_bool`, `bool_count`, `string_count` (see [`getIndexes`](/api-reference/database#getindexes)) | `index_name` | the order of the list |
| `--desc` | switch | off: ascending | largest or last first |
| `--json` | switch | off | print one JSON document |

```
alice> indexes products
alice> indexes products shop.
alice> indexes products --order number_of_records --desc
```

### `uniques`

```
uniques [<condition>] [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--json` | switch | off | print one JSON document |

```
alice> uniques >= order-
```

## Private access

A grant lets named users read a record they could not read by level, such as a private (`**`) one. A private access key opens one record for a limited time (`private_key_ttl`, default 1 hour). See [Access levels](/database/access-levels).

### `grant`

Lets up to 100 users read one of your records.

```
grant <record id> <user id> [<user id> ...]
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | your record |
| `<user id>` | user id; one to 100, separated by spaces | the users to let in |

```
alice> grant VZOat4ixBwfABGAq Lq0aZ7xe 9fKp2RtA
```

### `revoke`

Removes users' access that `grant` gave.

```
revoke <record id> <user id> [<user id> ...]
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | your record |
| `<user id>` | user id; one or more | the users to remove |

```
alice> revoke VZOat4ixBwfABGAq Lq0aZ7xe
```

### `grants`

Lists who is granted a record.

```
grants <record id>
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | your record |

```
alice> grants VZOat4ixBwfABGAq
```

### `key`

Asks for a private access key to one record. Read the record with it using `get --id <record id> --key <private key>`.

```
key <record id> [--ref <record id>]
```

| argument | type | meaning |
|---|---|---|
| `<record id>` | record id | the record to open |

| option | type | default | meaning |
|---|---|---|---|
| `--ref` | record id | none | a record you can read that points to this one, which gives you the right to the key |

```
alice> key VZOat4ixBwfABGAq
```

## Subscriptions

See [Subscriptions and feed](/database/subscription).

### `subscribe`

Subscribes to a user. Options not given are set to off.

```
subscribe <user id> [--feed] [--notify]
```

| argument | type | meaning |
|---|---|---|
| `<user id>` | user id | the user to subscribe to |

| option | type | default | meaning |
|---|---|---|---|
| `--feed` | switch | off | put their feed records in your `feed` |
| `--notify` | switch | off | send you their notifications (see `listen`) |

```
alice> subscribe Lq0aZ7xe --feed --notify
Subscribed to Lq0aZ7xe (feed true, notifications true).
```

### `unsubscribe`, `block`, `unblock`

```
unsubscribe <user id>
block <user id>
unblock <user id>
```

| command | argument | type | what it does |
|---|---|---|---|
| `unsubscribe` | `<user id>` | user id | ends your subscription to that user |
| `block` | `<user id>` | user id | stops a user who subscribes to you from getting your feed and notifications |
| `unblock` | `<user id>` | user id | lets them again |

```
alice> unsubscribe Lq0aZ7xe
alice> block 9fKp2RtA
```

### `subscriptions`

Lists subscriptions. Without options: the users you subscribe to.

```
subscriptions [--subscriber <user id>] [--subscription <user id>]
```

| option | type | default | meaning |
|---|---|---|---|
| `--subscriber` | user id | you | subscriptions made by this user |
| `--subscription` | user id | none | subscriptions to this user (who subscribes to them) |

```
alice> subscriptions
alice> subscriptions --subscription DS4gIE0n
```

### `feed`

Lists the feed records of the users you subscribe to with `--feed`, newest first.

```
feed [--limit <number>] [--json]
```

| option | type | default | meaning |
|---|---|---|---|
| `--limit` | number, `1` to `1000` | `50` | records per page |
| `--json` | switch | off | print full JSON |

```
alice> feed --limit 10
```

### `listen`

Prints notifications as they arrive, until Ctrl-C. Listening again in the same session resumes after the last one seen; notifications older than `notify_keep` (default 1 day) are gone, and a gap is reported. It takes nothing.

```
alice> listen
Listening for notifications. Ctrl-C to stop.
```

## Agent keys

Agent keys cannot create, list or revoke agent keys, so these three commands work only in a shell signed in with a username and password, not with `--key`. See [Agent Keys](/users/agent-keys).

### `agent create`

Makes an agent key and prints it **once**.

```
agent create <key name> [--level <level>] [--read-only] [--private] [--collections <list>] [--expires <duration>]
```

| argument | type | meaning |
|---|---|---|
| `<key name>` | text, 1 to 256 characters; quote it if it has spaces | a name to recognise the key by, such as `shop-bot` |

| option | type | default | meaning |
|---|---|---|---|
| `--level` | level `00` to `99` | your level | the highest level the key can read and write |
| `--read-only` | switch | off: read and write | the key can only read |
| `--private` | switch | off | the key may use `**` (private) records |
| `--collections` | list of collections | every collection | the only collections the key may use |
| `--expires` | [duration](/admin/command-line#durations) | never | the key stops working after this long |

```
alice> agent create shop-bot --level 05 --collections products,orders --expires 30d
Agent key MFgYtk for "shop-bot"
Key (shown once): dk_...
alice> agent create reporter --read-only
```

### `agent list`, `agent revoke`

```
agent list
agent revoke <key id>
```

| command | argument | type | what it does |
|---|---|---|---|
| `agent list` | | | lists your keys (the key strings are never shown again) |
| `agent revoke` | `<key id>` | key id (6 characters), from `agent list` | revokes the key; it stops working at once |

```
alice> agent list
alice> agent revoke MFgYtk
Revoked agent key MFgYtk.
```

## Other

### `whoami`

Shows your account, `enc_salt`, `kdf`, effective level, and agent key details when using one. It takes nothing.

```
alice> whoami
```

### `raw`

Sends any API call and prints the answer. It is the escape hatch: anything the shell has no command for can still be sent as the method name and its JSON body. Conditions are then written as in the API: `"condition": ">="`.

```
raw <method> ['<json>']
```

| argument | type | meaning |
|---|---|---|
| `<method>` | an API method name, such as `getRecords` (see [API Reference](/api-reference/database)) | the call to make |
| `'<json>'` | JSON; optional, default `{}` | the request body |

```
alice> raw getCollections
alice> raw getRecords '{"query":{"collection":"products","index":{"name":"price","value":1000,"condition":">="}},"fetch":{"limit":20}}'
```
