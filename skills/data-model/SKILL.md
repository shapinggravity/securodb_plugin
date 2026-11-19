---
name: data-model
description: Teaches how SecuroDB / BlitzData structures data (models, attributes, objects, links, permissions) so an app's schema is right the first time. Use when someone designs or reviews the data model of an app built on SecuroDB, chooses attribute types, models relationships between entities, decides who can read or add records, or asks how models, attributes and objects fit together. Use the `/securodb:create-model` command to create the model and the `securodb:post-api-transaction` skill to write records.
---

# SecuroDB data model

SecuroDB runs on the BlitzData protocol. Three ideas cover everything:

| Concept | What it is | Where it lives |
|---|---|---|
| **Model** | A table: its attributes, permissions and options | One row of the system model `_Model` |
| **Attribute** | A field of a model: a name, a `type` and options | The model's `attributes` JSON |
| **Object** | One record of a model | One row of the model's table |

Everything is created and changed through transactions posted to
`api/post.json`, and read through `api/list` and `api/get`. That includes
models, since a model is just an object of `_Model`. There is no `CREATE TABLE`
or migration file. A model is created by an `add` on `_Model`, and changed by
an `edit` of its `_Model` row. The protocol is the only way in or out: it
applies permissions, the audit log, encryption and hooks.

## 1. Models

### Naming

- A model's id (`_blitzID` of its `_Model` row) is `bdt<creator's user hash>_<name>`,
  for example `bdt246wahvt_industry`. Any other name is refused. Get the user
  hash from `GET api/ping.json` (`userhash`).
- Use one lowercase, singular noun per entity: `contact`, `deal`, `invoice_line`.
- System models start with `_` or a digit: `_Model`, `_Project`, `0BAUsers`
  (users). Don't create names like these.

### Required fields of a model

| Field | Meaning |
|---|---|
| `_blitzID` | The model's name, as above |
| `label` | i18n display name: `{"en": "Industries", "de": "Branchen"}` |
| `attributes` | The field definitions (section 2) |
| `modelpermission` | Floor level everyone gets on objects whose `_objectpermission` is 30 or higher |
| `_objectpermission` | Permission on the model itself, which decides who can **add** objects (65 = anyone, 64 = signed-in users) |
| `objectpermissionoptions` | The `_objectpermission` values an object may take, e.g. `[0, 30, 50]` |
| `defaultobjectpermission` | The value a new object gets. Must be in the options |

### Optional fields

Leave these out unless you have a reason to change them. The defaults are the
recommended setup.

| Field | Default | Change it when |
|---|---|---|
| `hasuserpermissions` | `1` | Set `0` on a model whose objects are always private (`objectpermissionoptions: [0]`). Set `2` for receiver mode (section 4). `0↔1` can't be changed later. |
| `haslogs` | `1` | Keep it. Every change is logged and deleted objects are kept in `{model}~deleted`. |
| `hassort` | `1` | Adds `_sort`, one hand-ordered sequence for the **whole table**. For an order inside a parent (a playlist's tracks), use an `int` attribute with `customSort` and set `hassort: 0`. |
| `allowedit` / `allowdelete` | `1` | `0` to forbid edits or deletes of objects |
| `hasexpiration` / `haspublishingdate` | `0` | `1` adds `_expiration` / `_publishingdate` to every object |
| `hasgroups`, `cache` | `0` | Rarely needed |
| `modeltype` | `transactional` | `analytical` for append-only event data. Can't be edited later. |

## 2. Attributes

```json
"attributes": {
  "title":    { "type": "varchar", "label-int": {"en": "Title"}, "required": 1, "index": "index" },
  "status":   { "type": "enum", "options": ["open", "won", "lost"], "default": "open" },
  "notes":    { "type": "text", "index": "none" },
  "owner_fk": { "type": "0BAUsers" }
}
```

### Rules for names

- Use lowercase letters, digits and `_`. Names are stored lowercase, and other
  characters are cleaned out.
- A name may not start with `_` (reserved for system attributes) or `@`.
- `*_fk` and `*_mtm` suffixes mean links (section 3). Use them only for links.
- `project_fk` is reserved for the link to a project.

### Options valid on every type

| Key | Meaning |
|---|---|
| `type` | Required. A type name, or a list such as `["varchar"]` for multiple values |
| `label-int` | i18n label, e.g. `{"en": "Title", "de": "Titel"}`. Prefer it to `label`, and don't set both. |
| `required` | `1` to refuse an `add` without it |
| `default` | Value used when none is given |
| `index` | `index` makes it fast to filter and sort; `unique` makes it unique across the **whole table, every user's rows included**; `none` is an "additional attribute" (see below); omitted gives a plain column |
| `encrypted` | `1` encrypts the value. It can be decrypted only in the browser session that holds the key. |

**`index: "none"`** stores the value in a shared JSON column, so adding it
needs no schema change. **It can't be filtered, sorted or searched.** Use it
for things only a detail view shows (notes, contact details, logos). Anything
a list filters, sorts or searches needs a real column.

### Choosing a type

| Data | Type |
|---|---|
| Short text: names, titles, codes | `varchar` (`length`, default 255) |
| Long text | `text`. For formatted text use `htmlText` or `markdown`. |
| Text in several languages | `texti18n` |
| Whole number | `int`. Use `tinyint` for small ranges. |
| Decimal | `float` / `double`. Money is `currency`, percent is `percentage`. |
| Yes / no | `boolean` (always `0` or `1`, never null) |
| Date / date and time | `date` / `datetime` |
| One of a fixed list that changes only with the schema | `enum` + `options`. `["enum"]` allows several. |
| Labels from a list that grows with the data | `tag` / `["tag"]` + `model` (+ `attribute`, default `name`) |
| Free words with no shared list | `["varchar"]` |
| Structured data with no fixed shape | `json` |
| Email, URL, phone | `email`, `url`, `phone` |
| Place | `location` |
| Files | `image`, `file`, `pdf`, `video`. `access: "signed"` for private files. |
| Source code | `code` (+ `language`) |
| A record of another model | a link, section 3 |

How to choose between enum, tag, a list of words and a link:

- **`enum`**: the values change only when the schema does (status, priority).
- **`tag`**: users pick from a list that grows, and the row stores the text. Renaming a tag doesn't update the rows that already carry it.
- **`["varchar"]`**: free words with no shared list.
- **A link (`_mtm`)**: the other thing is a real object with its own fields, renamed in one place.

### Per-user attributes

A value per object **per user**, such as a favourite flag or a personal counter,
goes under `userattributes` beside `attributes`, not inside it:

```json
"userattributes": { "isfavorite@u": { "type": "int", "default": 0 } }
```

Names ending in `@u` can be written by any reader, and unmarked names need
ADMIN. Write them with `editUserField`, and read them as `user.isfavorite@u`.
These need `hasuserpermissions: 1`.

## 3. Links between models

| Relationship | How |
|---|---|
| Each object points to **one** object of model X | attribute `<name>_fk`, `type` = X's full model name: `"company_fk": {"type": "bdt…_company"}` |
| Points to a **user** | `"owner_fk": {"type": "0BAUsers"}`. The creator is already `_userID`, so don't duplicate it. |
| Belongs to a **project** | `"project_fk": {"type": "_Project"}`. This name only. It enables project permissions. |
| **Many** to many | attribute `<name>_mtm`, `type` = a list with X's full model name: `"tags_mtm": {"type": ["bdt…_tag"]}` |
| A tree | a `parent_fk` that points to the model itself |

- **Direction:** put the `_fk` on the "many" side. An `invoice_line` has an
  `invoice_fk`; an invoice doesn't keep a list of its lines.
- **Deleting the target:** if the object an `_fk` points to is deleted, the
  `_fk` becomes empty and the row stays. Many-to-many link rows are deleted
  along with it. If children should go with their parent, delete them with
  transactions first.
- **Permission to link:** writing an `_fk` or `_mtm` needs permission to
  *link* to the target. That is above 40 by default (the target model's
  `mindefaultlinkingpermission`). A user who can't read the target can't
  link to it.
- **Writing links:** an `_fk` takes the target's `_blitzID`. An `_mtm` is edited
  with `{"add": "<blitzID>"}` or `{"remove": "<blitzID>"}`, not `prev`/`new`.

## 4. Permissions

Levels run from 0 to 100: `50` READ, `60` LINK, `64`/`65` ADD, `70` EDIT,
`80` DELETE, `90` ADMIN, `100` creator. The creator (`_userID`) always has 100.

An object's **`_objectpermission`** decides which sources of permission it accepts:

| Value | Who gets access |
|---|---|
| `0` | Creator only |
| `10` | + individually granted users (`addObjectUserPermission`), or the `receiver_fk` user in mode 2 |
| `20` | + members of the object's `project_fk` |
| `30` | + the model's `modelpermission` as a floor for everyone (the usual default) |
| `≥ 50` | Everyone gets that level (`50` = public read) |

Design the permissions in this order:

1. **Who can add objects?** Anyone: `_objectpermission: 65`. Signed-in users
   only: `64`. Members of one project: `20` plus `project_fk`. Only you: `0`.
2. **What can others do with each object?** Fix one option with
   `objectpermissionoptions: [0]` (private) or `[50]` (public), or list
   several and choose `defaultobjectpermission`.
3. **Is there a floor for everyone?** Set `modelpermission` (`0` none, `30`
   default, `50` public read). It applies only to objects at 30 or higher.

**Receiver mode** (`hasuserpermissions: 2`): for messages, assignments and
anything addressed to one person. Declare
`"receiver_fk": {"type": "0BAUsers"}`. That user gets READ on objects at 10 or
higher.

## 5. Objects

Every object carries system attributes. Don't declare them:

| Attribute | Meaning |
|---|---|
| `_blitzID` | The object's id: `[A-Za-z0-9_-]`, at most 50 characters, unique per model across all users |
| `_localID` | Internal auto-increment number |
| `_userID` | Creator |
| `_blitzstamp` | Creation time, in seconds since 2021-01-01 |
| `_modified` | Last change |
| `_objectpermission` | The object's permission tier (section 4) |
| `_sort`, `_projectsort` | Ordering (with `hassort`) |
| `_expiration`, `_publishingdate` | With `hasexpiration` / `haspublishingdate` |

- **Ids:** let the server assign `_blitzID` (it becomes the `add`'s hash).
  Set your own only for a real natural key (`IT`, `EUR`). Keep outside ids
  (emails, URNs) in an ordinary attribute, because two users importing the
  same one would collide.
- **Empty values:** on `add`, leave the attribute out. On `edit`, use `"@NULL"`.
  On reads, ask with `["attr", "IS NULL"]`.
- **Edits:** one attribute per `edit`, compare-and-set with `prev`/`new`.
  There is no upsert. Arrays and `_mtm` use `add`/`remove`, and JSON attributes
  take a `path`. See the `securodb:post-api-transaction` skill.
- System attributes can't be edited, except `_objectpermission`, `_sort`,
  `_projectsort`, `_expiration` and `_publishingdate`.

## 6. Reading

- **List:** `GET api/list/<model>.json?conditions=[...]&limit=50&customSort=<attr>&customSortDirection=ASC`
- **One object:** `GET api/list/<model>/_blitzID/<id>.json` (or `api/get/<model>/<id>.json`)
- **Conditions:** a JSON list, all of which must match: `["status", "=", "open"]`,
  `["amount", ">=", 100]`, `["name", "LIKE", "acme%"]`, `["status", "IN", ["open","won"]]`,
  `["tags", "CONTAINS", "vip"]` (multi-value), `["company_fk", "IS NULL"]`,
  `["_userID", "=", "@me"]` (the caller's own objects)
- **Paging:** `limit` (default 200), then `pagination=<last value of the sort attribute>`
- **Projection:** `var=["title","status"]` selects attributes; `fks=object` or
  `label` expands `_fk` links; `manyToMany=blitzID|label|object` does the same
  for `_mtm`
- Reads return only objects the caller can read (40 or higher). Nobody sees a
  row their permissions don't reach, so the model's permissions shape every list.

## 7. Changing a model later

Edit its `_Model` row (`model: "_Model"`, `blitzID: <model name>`), one field
per transaction:

- **Add an attribute:** an `edit` of `attributes` by `path` with no `prev`:
  `{"attributes": {"path": "priority", "new": {"type": "int"}}}`.
- **Change an attribute:** the same with `prev` set to its current definition.
- **Can't be changed:** `modeltype`, `hasuserpermissions` between 0 and 1,
  `allowdelete`. Decide these at creation.
- **Changing type or meaning:** make a new attribute, copy the values over
  with transactions, then stop using the old one. Don't reuse an old name for
  a new meaning.

## Design checklist

Before creating models for an app, check that:

- [ ] Each entity is one model, named `bdt<hash>_<singular noun>`
- [ ] Every attribute a list filters, sorts or searches is a real column with `index`, not `index: none`
- [ ] `unique` is used only when the value must be unique across **all** users
- [ ] Links use `_fk` on the many side with the target's **full** model name, and `_mtm` for many-to-many
- [ ] Fixed choices use `enum`, a growing shared list uses `tag`, and real entities use links
- [ ] Permissions answer the three questions, and `objectpermissionoptions` contains `defaultobjectpermission`
- [ ] Private-only models set `hasuserpermissions: 0`
- [ ] Per-user state (favourites, read flags) is in `userattributes`, not a separate model
- [ ] Parent models are created before the models that link to them
