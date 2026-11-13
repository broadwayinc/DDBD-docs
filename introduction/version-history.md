# Version History

The newest version is at `https://cdn.broadwayinc.com/get/ddbd/latest`. `ddbd version` prints the one you run.
To upgrade, install the new version over the old one ([Operations](/admin/operations)) and restart:
`ddbd restart --data <folder>`.

## 0.3.2

2026-11-13

**Web Admin: the selected tab is ringed.** The chosen tab of every tab strip, such as **Form** and **JSON** on
the Query page, now has an outline around it, so it reads as selected at a glance. Nothing else changes; a
data folder used by 0.3.1 opens as it is.

## 0.3.1

2026-11-13

**The default data folder is `~/ddbd-data`.** Without `--data` (and without `DDBD_DATA`), every command uses
`ddbd-data` in your home folder, on Linux and macOS alike, whatever folder it is run in. It used to be
`./ddbd-data` in the current folder, so running `ddbd` from another folder found no database, or made a new one there.
See [The data folder](/admin/command-line#the-data-folder).

- A data folder made with the old default keeps working: give its path with `--data`, set `DDBD_DATA`, or move it
  to `~/ddbd-data` while the server is stopped.
- Services that pass `--data` (systemd units, `brew services`) are not affected.

**Durations must be above zero on the command line too.** `ddbd start --notify-keep 0` (or `--private-key-ttl 0`)
is refused, as the Web Admin's Settings page already refused it. See [Durations](/admin/command-line#durations).

**Admin shell catalog commands fixed.** `tags`, `indexes`, and `collections` or `uniques` with a condition (such as
`collections >= prod`) answered `INVALID_REQUEST: Unknown key "collection"`. They now work, and `indexes` takes
`--desc` as in the User Shell.

**Install with `sudo`.** The install command is now `curl -fsSL https://cdn.broadwayinc.com/get/ddbd/v1 | sudo sh`,
which installs `ddbd` in `/usr/local/bin` for every user of the machine.

**Clearer documentation** for the [Command Line](/admin/command-line), the [Admin Shell](/admin/admin-shell) and
the [User Shell](/admin/user-shell): every command has its own table of options with their types, and examples;
[Typing values](/admin/user-shell#typing-values) explains values, indexes and conditions in one place; and the docs
font no longer joins `>=`, `<=` and `=>` into single symbols.

## 0.3.0

2026-11-13

**Reference reads follow one rule.** The user who uploaded a record, and the users it was shared with
([`grantPrivateRecordAccess`](/api-reference/database#grantprivaterecordaccess)), read every record that points to
it. Anyone else reads the levels up to their own, and no private records. See
[Access Rules](/database/access-rules#reading-by-reference-one-rule).

- With access to the referenced record, a `reference` read returns every level the collection uses and every `"**"`
  record pointing to it, and `collection.subscription` lists the uploader's subscription records without a
  subscription. Before, it stopped at your own level.
- Without access, a `reference` read returns the levels up to your own and **no** `"**"` records, not even your
  own (read those by `record_id`, or among your own `"**"` records without a reference). Asking for `"**"` or a
  level above yours together with a `reference` is refused with `NO_ACCESS`.
- A subscription record that points to a record you uploaded or were granted is readable without a subscription.

**`$user_id` with a `reference` keeps the reference.** `getRecords` with a record `reference` and the `$user_id`
index used to drop the reference and list every record of the user in the collection. It now lists that user's
records pointing to the target, by the rule above (the user themself always gets all of theirs). See
[One user's records that point to a record](/database/access-rules#one-user-s-records-that-point-to-a-record).

**New page:** [Access Rules](/database/access-rules), every read rule with worked examples.

::: warning Upgrading from 0.2.0
The first start of 0.3.0 on an existing data folder rewrites one internal key of every record that points to another
record (it now holds the collection, level and subscription of the pointing record). This runs once, before the
server answers requests, and takes about as long as reading every record once; nothing else changes on
disk. Starting 0.2.0 again on a folder 0.3.0 has opened is not supported.
:::

## 0.2.0

DADABADA is now **DDBD**: the program `ddbd`, the settings file `ddbd.toml`, the environment variable `DDBD_DATA`,
the default data folder `./ddbd-data`, and releases from `https://cdn.broadwayinc.com/get/ddbd`. See
[Moving from DADABADA](/admin/operations) in Operations.

## 0.1.x

Released as DADABADA. 0.1.5 added builds for macOS.
