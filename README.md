# <img src="assets/icon.png" alt="" width="32" height="32"> SecuroDB plugin for Claude Code

Work with SecuroDB from Claude Code: create models and write records through
SecuroDB's post API, in plain language.

The plugin adds:

- **`/securodb:create-app`**, a command that turns a description of an app into
  a reviewed data model, creates the models, and optionally adds sample data and
  a web front end.
- **`/securodb:create-model`**, a command that creates a new model (table).
- **`securodb:data-model`**, a skill that teaches how models, attributes,
  links, permissions and objects fit together, so an app's schema is right the
  first time.
- **`securodb:post-api-transaction`**, a skill Claude uses whenever you ask it to
  add, edit or delete records.
- **The `securodb` MCP server** (`https://securodb.com/mcp`), connected
  automatically when the plugin is enabled.

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add shapinggravity/securodb_plugin
```

```
/plugin install securodb@securodb-plugin-marketplace
```

Restart Claude Code if the command doesn't appear right away. Run `/mcp` to check
that the `securodb` server is connected.

## Usage

### Create an app

```
/securodb:create-app a CRM for a 5-person sales team: companies, contacts, deals
```

Claude asks a few questions about the app and designs the models. It shows the
design for your OK, saving it to `securodb-app/DESIGN.md`, then creates the
models in order. Two optional steps follow:
- **Sample data:** a few records per model.
- **Web front end:** a small, framework-free app in `securodb-app/web/` that
  signs in with OAuth and lists, adds, edits and deletes records. It loads the
  SDK from the instance, so no build step or npm access is needed.

Choose "backend only" to stop after the models.

### Create a model

```
/securodb:create-model task; title:varchar! priority:int status:enum(open,done) company_fk:company
```

- `name:type` pairs. A trailing `!` marks the attribute as required.
- Plain phrasing also works: `/securodb:create-model contacts with a required name, an email and a phone`.
- Claude adds the required `bdt<your user hash>_` prefix to the name, maps your
  types to SecuroDB types, shows you the definition, and creates the model once
  you agree.

Supported types include `varchar`, `text`, `htmlText`, `int`, `tinyint`, `float`,
`double`, `currency`, `percentage`, `boolean`, `date`, `datetime`, `enum`,
`json`, `email`, `url`, `phone`, `image`, `file`, `video`, `pdf`, `location` and
`code`.

Links:
- **To one object:** an attribute named `*_fk`, typed with the target model's full name.
- **To many:** an attribute named `*_mtm`, typed `["<model>"]`.
- **To users and projects:** `owner_fk: 0BAUsers` and `project_fk: _Project`.

### Design a data model

Describe the app and ask Claude to design its models, for example "Design the
SecuroDB models for a small CRM with companies, contacts and deals." The
`securodb:data-model` skill guides the result:
- one model per entity, with attribute types chosen for how the data is used;
- indexes on what lists filter and sort by;
- `_fk` and `_mtm` links in the right direction;
- permissions that answer who adds objects and who sees them.

Then create each model with `/securodb:create-model`, parent models first.

### Write records

No command needed. Ask in your own words and Claude uses the skill:

- "Add three tasks to `bdt…_tasks`: write spec, review, ship."
- "Change the title of task `1xk7o7-1zajzo90` from Draft to Final."
- "Create an order for customer 42 with these line items."
- "Delete every done task from last month." (Claude asks first.)

Claude reads the model first, posts the transactions, and reports each one as
saved, conflicted or refused.

## How SecuroDB writes work

Every write is a **transaction** posted to `<instance>/api/post.json`. The body
is a JSON array:

```json
[{
  "action": "add",
  "model": "bdt<userhash>_tasks",
  "hash": "a1b2c3-x9y8z7",
  "blitzstamp": 189734512,
  "data": { "title": "Write spec", "priority": 2 }
}]
```

- **Actions**: mainly `add`, `edit` and `delete`. `edit` and `delete` name the
  row with `blitzID`.
- **Edits are compare-and-set.** `data` names the value the row held when you
  read it and the new one: `{"title": {"prev": "Draft", "new": "Final"}}`. If the
  row changed in the meantime, the edit is refused with the current value.
- **`hash`** identifies the transaction. The response is keyed by it, and posting
  the same hash again is treated as a duplicate, so retries are safe.
- **`blitzstamp`** is seconds since 2021-01-01 UTC.
- **A batch is not all-or-nothing.** Each transaction commits on its own. Check
  every result: `s` means saved, `e` means refused, and `conflict` means an edit
  found a different value.
- **Models** are rows of the system model `_Model`. A model's name must start
  with `bdt` plus its creator's user hash, which `api/ping.json` returns.

## Authentication

Claude uses whatever credential is already set up for your instance:

- **OAuth**: `Authorization: Bearer <token>`. This is the normal setup for apps
  and tools on another host.
- **Development keys**: headers `U: <userID>` and `K: <userKey>` from
  `api/login.json`. Use these only for local scripts and never share them.

Credentials never go inside a transaction.

## Repository layout

```
.claude-plugin/
  plugin.json        Plugin manifest (name: securodb, icon)
  marketplace.json   Marketplace entry pointing at this repository
.mcp.json            The securodb MCP server
assets/
  icon.png           Plugin icon
commands/
  create-app.md      /securodb:create-app
  create-model.md    /securodb:create-model
skills/
  data-model/SKILL.md
  post-api-transaction/SKILL.md
```

## Contributing

- **Commands** are Markdown files in `commands/`. The file name becomes the
  command name, and the front matter sets `description` and `argument-hint`.
- **Skills** go in `skills/<skill-name>/SKILL.md`, with `name` and
  `description` in the front matter. Claude chooses a skill based on its
  `description`, so say there when the skill applies.
- **Testing**: try a change locally before opening a pull request:
  ```
  claude --plugin-dir /path/to/securodb_plugin
  ```

## Links

- Website: <https://www.securodb.com>
