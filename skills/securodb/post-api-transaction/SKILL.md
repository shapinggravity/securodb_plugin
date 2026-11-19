---
name: securodb-post-api-transaction
description: Writes rows to a SecuroDB / BlitzData instance by posting transactions (add, edit, delete) to `api/post.json`, and reads each transaction's result. Use when the user wants to insert, change or delete records in SecuroDB models, or debug a post that was refused or conflicted. For creating a model, use the create-model command, which posts on `_Model` through this same endpoint.
---

# POST API transactions

Every write to SecuroDB is a **transaction**: a small JSON envelope posted to
`api/post.json`. One POST can carry several transactions. **Each one is committed
or rolled back on its own.** A batch is not all-or-nothing: some can succeed while
others are refused, and the response gives one result per transaction.

## The request

`POST <instance>/api/post.json`. The body is a JSON **array** of transactions:

```json
[{
  "action": "add",
  "model": "bdt<userhash>_tasks",
  "hash": "a1b2c3-x9y8z7",
  "blitzstamp": 189734512,
  "data": { "title": "Write spec", "priority": 2 }
}]
```

| Field | Meaning |
|---|---|
| `action` | `add`, `edit`, `delete`. Less common: `addObjectUserPermission` (grant a user a level on a row), `editUserField`, `addSort`, `removeUser`. |
| `model` | The model's full name (`bdt…_name`), or a system model such as `_Model`. |
| `hash` | Unique per transaction. The response is keyed by it, and the server treats a repost of the same hash as a duplicate, not a second write. Make it fresh for every new transaction and reuse it only to retry that same transaction. |
| `blitzstamp` | Seconds since 2021-01-01T00:00:00Z: `floor(Date.now()/1000) - 1609459200`. |
| `blitzID` | The target row for `edit` and `delete`. |
| `data` | Attribute values (see below). |

### Per action

- **add**: `data` holds the attribute values. The new row's id is
  `data._blitzID` if given, otherwise the `hash`. Required attributes must be
  present. Keys the model doesn't declare are skipped with a notice, or refused on
  older servers, so send only declared attributes.
- **edit**: a compare-and-set on one attribute. The change is written only if the
  row still holds `prev`:
  ```json
  { "action": "edit", "model": "bdt…_tasks", "blitzID": "<row id>", "hash": "…", "blitzstamp": …,
    "data": { "title": { "prev": "Draft", "new": "Final" } } }
  ```
  Read the row first to get `prev`. A missing `prev` means "the row holds nothing
  here", so it fails on any row that holds a value. Use one transaction per changed
  attribute. For a key inside a JSON attribute, add `"path": "key.subkey"`.
  `"@NULL"` stands for null in `prev` and `new`.
- **delete**: `{ "action": "delete", "model": "…", "blitzID": "<row id>", "hash": "…", "blitzstamp": … }`.

Attribute names match without regard to case and are stored lowercase. A `*_fk`
attribute takes the blitzID of a row in the model it points to. A many-to-many
attribute takes a list of blitzIDs.

## Authentication

Use whatever credential is already configured. Never put credentials in a
transaction or show them in chat.
- **OAuth Bearer token**: `Authorization: Bearer <access token>` (the normal case
  for apps on another host).
- **Development key**: headers `U: <userID>` and `K: <userKey>` from
  `api/login.json`. For scripts only.
- **Same host as the instance**: session cookies.

`GET api/ping.json` returns the current `userID` and `userhash`. Use it to check
the credential works before writing.

## The response

```json
{ "results": { "<hash>": { "s": 1, "n": "…" } }, "executionTime": 12, "logTime": 3 }
```

For each transaction, read `results[<its hash>]`:

| Shape | Meaning |
|---|---|
| `s` set and truthy (also `s0`/`s1`), possibly with notices in `n` | Committed |
| `conflict: { "<attr>": <current value> }` | An edit's `prev` didn't match. Nothing was written. |
| `e: "<message>"` | Refused (permission, missing required attribute, invalid value…). Nothing was written for this transaction. |
| `[]` | `addSort` succeeded |

If there's no `results` and only `error`/`errors`, the whole request failed (auth,
malformed body) and nothing was written.

## Workflow

1. **Connect.** If the `securodb` MCP server is connected and has a post or
   transaction tool, use it and follow its schema. Otherwise use HTTP as above. If
   neither is available, say so. Never report a write you didn't make.
2. **Know the model.** Read the model's `_Model` row (`attributes`, required
   attributes, `_fk` targets) before building `data`. Ask about missing required
   values; don't invent them.
3. **Order dependencies.** If a row refers to a row created in the same request,
   set the parent's `_blitzID` yourself, use it in the child's `_fk`, and put the
   parent first.
4. **Decide how to handle partial failure.** Since a batch isn't atomic:
   - For independent rows (imports), post them together, then report which hashes
     failed.
   - For rows that must all exist together (an order and its lines), post the
     parent first and stop if it fails. If a later step fails, tell the user exactly
     what was written and offer to delete it (compensating deletes). Don't call the
     result "rolled back".
5. **Confirm before risky writes.** Show a summary (model, action, key values,
   count) and wait for a yes before any delete, any edit that overwrites data, or
   more than about 20 transactions.
6. **Send, then report every hash.** List each transaction as committed (row
   blitzID), conflicted (current value, with an offer to retry with it as `prev`),
   or refused (the `e` text, explained plainly).
7. **Retry safely.** After a timeout or dropped connection, resend the *same*
   transactions with the *same* hashes. Duplicates aren't applied twice. Never
   regenerate hashes for a retry.
