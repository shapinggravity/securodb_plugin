---
description: Create a new SecuroDB model by posting an `add` transaction on `_Model`.
argument-hint: 'model name; attribute:type pairs (e.g. task; title:varchar! priority:int status:enum(open,done) company_fk:company)'
---

You are creating a new model (table) on a SecuroDB / BlitzData instance.

A model is a row of the system model `_Model`. You create one by posting a
single `add` transaction to `api/post.json`, the same endpoint and envelope the
`securodb:post-api-transaction` skill describes. Follow that skill for sending,
authentication and reading the result. This command covers what goes in `data`.

If the `securodb` MCP server is connected and has a tool for creating models or
posting transactions, use it and follow its schema. Otherwise post over HTTP as the
skill describes.

## 1. Parse `$ARGUMENTS`

- **Name**: the text before the first `;`. Use lowercase letters, digits and `_`.
- **Attributes**: `name:type` pairs. A trailing `!` means required. Also accept loose
  phrasing ("a required title and an optional priority number").
- If there are no attributes, ask what the model is for, propose a list, and wait
  for the user to agree.

## 2. Build the full model name

A model's name must start with `bdt` plus the creator's user hash. Any other name
is refused (`model_name_not_conforming`).

1. Get the caller's `userhash` from `GET api/ping.json`.
2. The full name is `bdt<userhash>_<name>`, e.g. `bdtv7f2e_task`.
3. Look for an existing `_Model` row with that name, or a very similar one. If
   one exists, ask whether to reuse it, edit it, or pick another name.

## 3. Map the attribute types

Follow the `securodb:data-model` skill for types, links and indexes. In short:

- Attribute names are lowercase `[a-z0-9_]`, and may not start with `_` or `@`.
- Each attribute is `{"type": …}`, plus `"label-int": {"en": "…"}`,
  `"required": 1`, `"default"` and `"index"` as needed.
- `"index": "index"` for anything lists filter or sort on. `"index": "none"`
  only for detail-only fields, which can't be filtered.
- Common types: `varchar`, `text`, `int`, `float`, `currency`, `boolean`,
  `date`, `datetime`, `enum` (+ `options`), `["varchar"]`, `tag` (+ `model`),
  `json`, `email`, `url`, `phone`, `image`, `file`, `location`.
- **Links:**
  - `<name>_fk` with `type` = the target's full model name (`bdt…_company`)
  - `owner_fk` → `0BAUsers`
  - `project_fk` → `_Project`
  - `<name>_mtm` with `type` = `["bdt…_target"]`
- Every link target must exist. If it doesn't, offer to create it first.
- Ask when a type has no clear match. Don't add attributes nobody asked for.
  System attributes (`_blitzID`, `_userID`, timestamps) are automatic.

## 4. Permissions and settings

These seven fields are required:

| Field | Default to propose |
|---|---|
| `_blitzID` | `bdt<userhash>_<name>` |
| `label` | `{"en": "<Human name>"}` |
| `attributes` | from step 3 |
| `modelpermission` | `30` |
| `_objectpermission` | `65` (anyone can add) or `64` (signed-in users only) |
| `objectpermissionoptions` | `[0, 30, 50]` |
| `defaultobjectpermission` | `30` (must be one of the options) |

Ask who should add objects and who should see them, unless the user already
said. The `securodb:data-model` skill's "three questions" decide these. Leave optional
fields out unless needed:
- `hasuserpermissions: 0` when every object is private (`objectpermissionoptions: [0]`);
- `hassort: 0` when order comes from an attribute;
- `modeltype: "analytical"` for append-only events.

`modelpermission` above 65 is refused.

## 5. Confirm, then post

Show the attribute table (name, type, required, notes) and the settings, then
post once the user agrees:

```json
[{
  "action": "add",
  "model": "_Model",
  "hash": "<unique hash>",
  "blitzstamp": <seconds since 2021-01-01>,
  "data": {
    "_blitzID": "bdt<userhash>_task",
    "label": {"en": "Tasks"},
    "attributes": {
      "title":    {"type": "varchar", "label-int": {"en": "Title"}, "required": 1, "index": "index"},
      "priority": {"type": "int", "label-int": {"en": "Priority"}},
      "status":   {"type": "enum", "options": ["open", "done"], "default": "open", "index": "index"},
      "owner_fk": {"type": "0BAUsers", "label-int": {"en": "Owner"}}
    },
    "modelpermission": 30,
    "_objectpermission": 65,
    "objectpermissionoptions": [0, 30, 50],
    "defaultobjectpermission": 30
  }
}]
```

## 6. Report

- **Success:** `results[<hash>].s` is set. Give the full model name, the
  attributes as created, and any type you mapped. Quote any notice in `n`:
  the server skips attributes it refuses (for example a name starting with `_`)
  and says so there. Suggest the
  `securodb:post-api-transaction` skill for adding rows.
- **Refused:** quote `results[<hash>].e`, explain it plainly (wrong name prefix,
  permission above 65, unknown `_fk` target, missing required field), and say that nothing was created.

To change an existing model later, edit its `_Model` row. Change one attribute at
a time with an edit by path on `attributes`, using `prev`/`new` as the skill
describes. Don't post a second `add`.
