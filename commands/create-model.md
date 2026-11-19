---
description: Create a new SecuroDB model by posting an `add` transaction on `_Model`.
argument-hint: 'model name; attribute:type pairs (e.g. tasks; title:varchar! priority:int status:enum(open,done) project_fk:<model>)'
---

You are creating a new model (table) on a SecuroDB / BlitzData instance.

A model is a row of the system model `_Model`. You create one by posting a
single `add` transaction to `api/post.json`, the same endpoint and envelope the
`securodb-post-api-transaction` skill describes. Follow that skill for sending,
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
2. The full name is `bdt<userhash>_<name>`, e.g. `bdtv7f2e_tasks`.
3. Look for an existing `_Model` row with that name, or a very similar one. If
   one exists, ask whether to reuse it, edit it, or pick another name.

## 3. Map the attribute types

Store attribute names in lowercase. The server lowercases them anyway, and keeps
only the first of two names that differ only in case.

| User says | `type` |
|---|---|
| string, short text, title | `varchar` (add `"length": 255` when a limit is wanted) |
| long text, notes | `text` |
| rich text / html | `htmlText` |
| int, integer, count | `int` |
| small int, flag 0-9 | `tinyint` |
| decimal, float | `float`, or `double` for high precision |
| money | `currency` |
| percent | `percentage` |
| bool, yes/no | `boolean` |
| date / datetime, timestamp | `date` / `datetime` |
| choice of values | `enum` with `"options": ["a", "b"]` |
| json, object | `json` |
| email, url, phone | `email`, `url`, `phone` |
| image, file, video, pdf | `image`, `file`, `video`, `pdf` |
| place, coordinates | `location` |
| code snippet | `code` (optionally `"language": "js"`) |
| link to one row of model X | attribute named `<name>_fk`, `type` = X's full model name |
| link to many rows of model X | `type` = `["<X's full model name>"]` (many-to-many) |
| list of values | `type` = `["varchar"]` (multi-value) |

- Required: `"required": 1`. Label: `"label": "<Human name>"`.
- Ask when a type has no clear match. Don't guess.
- For every `_fk` or many-to-many target, check that the target model exists.
  If it doesn't, ask whether to create it first.
- Don't add attributes the user didn't ask for. `_blitzID`, `_userID` and
  timestamps are system attributes; the server adds them.

## 4. Model settings

Use these defaults unless the user asks otherwise, and mention them in the
summary:

```json
"label": {"en": "<Human name>"},
"modelpermission": 30,
"_objectpermission": 65,
"defaultobjectpermission": 30,
"hasuserpermissions": 1,
"haslogs": 1
```

Other settings, used only when asked: `allowdelete`, `allowedit` (0 to forbid),
`hassort`, `hasexpiration`, `haspublishingdate`, `hasgroups`, `hasprojects`,
`cache`, `mobilecache`. `modelpermission` above 65 is refused
(`invalid_model_permission`). `_objectpermission` 65 lets everyone add rows; use 64
to restrict adding.

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
    "_blitzID": "bdt<userhash>_tasks",
    "label": {"en": "Tasks"},
    "attributes": {
      "title":    {"type": "varchar", "length": 255, "required": 1},
      "priority": {"type": "int"},
      "status":   {"type": "enum", "options": ["open", "done"]}
    },
    "modelpermission": 30,
    "_objectpermission": 65,
    "defaultobjectpermission": 30,
    "hasuserpermissions": 1,
    "haslogs": 1
  }
}]
```

## 6. Report

- **Success:** `results[<hash>].s` is set. Give the full model name, the
  attributes as created, and any type you mapped. Suggest the
  `securodb-post-api-transaction` skill for adding rows.
- **Refused:** quote `results[<hash>].e`, explain it plainly (wrong name prefix,
  permission above 65, unknown `_fk` target), and say that nothing was created.

To change an existing model later, edit its `_Model` row. Change one attribute at
a time with an edit by path on `attributes`, using `prev`/`new` as the skill
describes. Don't post a second `add`.
