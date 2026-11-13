# Access Rules

This page puts every rule about **who reads which record** in one place, with the questions people actually ask:
"can I list the comments on my post?", "why don't I see my own private reply?", "what does a reference read
return?". The rules are short; most of the page is examples.

## The basic rules

A user can read a record when **one** of these is true:

| # | Rule | Example |
|---|---|---|
| 1 | The record's level is at or below the user's level | a level `05` user reads `"00"` to `"05"` |
| 2 | The user uploaded the record | your own records, at any level, `"**"` included |
| 3 | The record was [granted](/database/access-levels#granting-access-to-a-record) to the user | a private record shared with you, or one above your level |
| 4 | The record **points to** a record the user uploaded or was granted ([references](/database/referencing)) | every comment on your article, at any level, private and subscription comments included |
| 5 | The user holds a valid [private access key](/database/access-levels#private-access-keys) for it (in a `record_id` or `unique_id` lookup) | |

And two limits apply on top:

- **Subscription records** (`is_subscription_record`) need an active, unblocked subscription to their uploader,
  except for their uploader, and except through rule 4.
- **Agent keys** read only within their own limits: their collections, their `max_level`, and private records only
  with `private: true`.

The admin (the operating system user running DDBD) reads everything. No network user does, whatever their level.

## What a query needs

[`getRecords`](/api-reference/database#getrecords) reads either **one record** or **one collection**:

| You give | Collection | You get |
|---|---|---|
| `record_id` or `unique_id` | not needed | that record, if one of the basic rules lets you read it |
| anything else | **required** | the records of that collection you can read |

`access_level` is optional. Without it, a query reads **every level you can read**: the levels up to your own,
plus the `"**"` records you uploaded or were granted. With it, only that level. Everything else combines freely with it:

```json
{ "query": { "collection": "comments" } }
{ "query": { "collection": "comments", "tag": "pinned" } }
{ "query": { "collection": "comments", "index": { "name": "likes", "value": 10, "condition": ">=" } } }
{ "query": { "collection": { "name": "comments", "access_level": "05" }, "tag": "pinned" } }
```

Each one is one read: DDBD keeps a sorted key range per level, so "every level you can read" is a handful of
exact ranges merged in order, whatever the `tag` or `index`.

## Reading by reference: one rule

`query.reference` lists the records that point to a record (the **target**). Which of them you get follows one
rule:

> **The user who uploaded the target, and the users it was shared with, read every record that points to it.**
> Anyone else reads the levels up to their own, and no private records.

| You ... | Without `access_level` | `access_level` above yours, or `"**"` | `collection.subscription: <uploader>` |
|---|---|---|---|
| uploaded the target, or it was shared with you | every level the collection uses, and every `"**"` record pointing to it | allowed | the uploader's subscription records, without subscribing |
| anyone else | the levels up to your own; **no** `"**"` records, **not even your own** | refused: `NO_ACCESS` | only as a subscriber |

Why "not even your own": a reference listing answers "what is attached to this record", and the private records
attached to a record belong to the people who have access to it. Your own private record is still yours: you read
it by `record_id`, or among your own `"**"` records without a `reference`.

The decision is made once per query, from the target, so DDBD reads either the whole private range or none of
it. Nothing is read and then thrown away.

### The example

| User | Level | |
|---|---|---|
| alice | `05` | uploads the article `ART` |
| carol | `01` | alice shares `ART` with her (`grantPrivateRecordAccess`) |
| bob | `01` | |
| dave | `99` | |

Comments in `comments`, all pointing to `ART`:

| Comment | By | Level |
|---|---|---|
| `C00` | dave | `"00"` |
| `C50` | dave | `"50"` |
| `CDP` | dave | `"**"` |
| `CBP` | bob | `"**"` |
| `CSUB` | dave | `"00"`, a subscription record |

**alice lists the comments on her article:**

```json
{ "query": { "collection": "comments", "reference": "ART" } }
```

`C00`, `C50`, `CDP`, `CBP`: every level, and every private comment, whoever wrote it. `CSUB` comes with
`"collection": { "name": "comments", "subscription": "<dave's user id>" }`, without alice subscribing to dave.

**carol, whom alice shared the article with:** the same as alice, although carol is level `01`.

**bob, who has no access to the article:**

```json
{ "query": { "collection": "comments", "reference": "ART" } }
```

`C00` only. Not `C50` (above his level), not `CDP`, and not his own `CBP`. Asking for more is refused:

```json
{ "query": { "collection": { "name": "comments", "access_level": "**" }, "reference": "ART" } }
```

```json
// Response (403)
{
  "error": {
    "code": "NO_ACCESS",
    "message": "You have no private access to record ART: only its uploader and the users it was shared with read the private records, and the records above their level, that reference it."
  }
}
```

bob still reads `CBP` by its `record_id`, and in `{ "collection": { "name": "comments", "access_level": "**" } }`
(his own private records, no reference).

**dave, level `99`, who wrote most of the comments but not the article:** `C00` and `C50`. A high level opens
the levels, not the private records attached to someone else's article.

## One user's records

The built-in index `$user_id` (or `reference: { "user_id": ... }`) lists one user's records in a collection:

```json
{ "query": { "collection": "comments", "index": { "name": "$user_id", "value": "<dave's user id>" } } }
```

It follows the basic rules: you get the ones you can read.

### One user's records that point to a record

Put a record `reference` and `$user_id` together to ask "what did this user attach to this record":

```json
{ "query": { "collection": "comments", "reference": "ART", "index": { "name": "$user_id", "value": "<dave's user id>" } } }
```

| You are ... | You get |
|---|---|
| that user | all of their records pointing to the target: every level, private ones included |
| the target's uploader, or it was shared with you | the same: every level, private ones included |
| anyone else | their records pointing to the target in the levels up to your own; `"**"` or a higher level is refused |

In the example: dave gets `C00`, `C50`, `CDP`; alice and carol get the same; bob gets `C00`. dave's comments that
point to something else, and his comments in other collections, never come back. dave's subscription comment
`CSUB` comes back when `collection.subscription` names dave, for dave himself, for alice and carol, and for his
subscribers.

This is one exact read too: DDBD keeps a key per referencing record by target, user, collection and level, in
upload order. Records come back in upload order (other sorts are not available here). A `tag` works, and is the
one condition DDBD checks on each record rather than on the key.

## Combining

| Query | Works | How it is read |
|---|---|---|
| collection, with or without `access_level`, `tag`, `index` | yes | exact ranges per level |
| + `reference` | yes | exact ranges per level, as the rule above decides |
| + `reference` + `$user_id` | yes | exact ranges per level, upload order only; a `tag` is checked per record |
| `record_id` / `unique_id` | yes | one lookup |

## Summary

- A collection is always required, except to read one record by `record_id` or `unique_id`.
- Without `access_level`, you get every level you can read; `tag`, `index` and `reference` combine with it.
- A reference read gives everything to the target's uploader and the users it was shared with, and the levels up to
  your own, with no private records, to anyone else.
- `$user_id` with a `reference` lists that user's records pointing to the target, by the same rule; your own
  records are always all yours.
- Subscription records need a subscription, except for their uploader and for whoever has access to the record they
  point to.
