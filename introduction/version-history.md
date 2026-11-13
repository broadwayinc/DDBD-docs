# Version History

The newest version is at `https://cdn.broadwayinc.com/get/ddbd/latest`. `ddbd version` prints the one you run.
To upgrade, install the new version over the old one ([Operations](/admin/operations)) and restart:
`ddbd restart --data <folder>`.

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
