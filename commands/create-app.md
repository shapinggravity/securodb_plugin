---
description: Design and create a SecuroDB app, its data model and optionally a web front end, from a plain description.
argument-hint: 'what the app does and who uses it (e.g. "a CRM for a 5-person sales team: companies, contacts, deals")'
---

You are building an app on SecuroDB (a BlitzData instance) from the user's
description in `$ARGUMENTS`. Work through the steps in order and **stop for the
user's OK at the end of step 2 and before step 5**. Never skip the design
review: the data model is the part that is hard to change once data exists.

Use the plugin's other pieces rather than repeating them:
- `securodb:data-model` (skill) for every modeling decision;
- `/securodb:create-model` (command) for the rules on creating each model;
- `securodb:post-api-transaction` (skill) for every write and for reading results.

If the `securodb` MCP server is connected and has tools for these calls, use
them. Otherwise use HTTP against the instance, as the skills describe. If you
can reach neither, still do steps 1–2 and write the design to a file, and say
plainly that nothing was created.

## 1. Understand the app

From `$ARGUMENTS`, and short questions only where the answer changes the model,
find out:
- **Entities**: the things the app keeps (companies, contacts, deals…) and the
  facts about each.
- **Relationships**: what belongs to what, and what links to many.
- **People and access**: who signs in, who adds what, who may see or change
  whose records (private, shared with a team or project, public).
- **Views**: the lists people need, and what they filter or sort by. This
  decides the indexes.
- **Instance**: the base URL (default `https://securodb.com`) and confirmation
  that the user is signed in. `GET api/ping.json` should return a `userhash`;
  you need it for model names.

Ask at most a few questions at once, and propose sensible defaults instead of
asking about routine choices.

## 2. Design the data model and get approval

Apply the `securodb:data-model` skill and its design checklist. Show the whole
design at once:

1. **Models** as a table: model (`bdt<userhash>_<name>`), purpose, who can add,
   and `objectpermissionoptions` / `defaultobjectpermission` / `modelpermission`.
2. **Attributes** for each model: name, type, required, index, notes. Include
   every link (`*_fk` with the target's full model name, `*_mtm`, `owner_fk` →
   `0BAUsers`, `project_fk` → `_Project`).
3. **Creation order**: parents before the models that link to them.
4. **Choices worth a second look**: anything you inferred (a `tag` rather than
   an `enum`, a private model with `hasuserpermissions: 0`, `index: none`
   fields that can't be filtered).

Write the same design to `securodb-app/DESIGN.md` in the working directory, so
it survives the conversation. Then **wait for the user's OK**. Apply changes
and show the design again until they agree.

## 3. Create the models

For each model, in creation order, follow `/securodb:create-model` steps 2–6
without asking again (the design is already approved):
- Check whether the model already exists. If it does, reuse it as is, or stop
  and ask if its definition differs from the design.
- Post one `add` on `_Model` per model, with a fresh hash for each.
- Read `results[hash]`. On `e`, **stop**: report what was created, what failed,
  and why. Don't continue to models that link to the failed one.
- Quote notices (`n`) about skipped attributes.

Keep a running table in `securodb-app/DESIGN.md` of each model's full name and
status (created / existing / failed).

## 4. Sample data (optional)

Ask whether to add a few sample records. If yes, add 3–5 realistic records per
model through `securodb:post-api-transaction`, parents first. Set each parent's
`_blitzID` yourself so children can link to it in the same batch. Report every
hash's result. Skip this step if the user declines.

## 5. Web front end (optional)

Ask: **"Backend only, or also a web app?"** For backend only, go to step 6.

For a web app, generate a small, framework-free browser app in `securodb-app/web/`
using the BlitzData SDK (`@blitzdata.ts/core`). If the user asks for React,
Vue or similar, use that instead, with the same SDK calls.

### Loading the SDK

The instance serves the SDK, so no npm access is needed. Find the version it
serves (`https://securodb.com/assets/js/blitzdata/1.17.0/blitzdata.js`
answered 200 on 2026-11-19; confirm with a request before pinning it) and import
the ES module from there:

```js
import { BlitzData, BDModel, OAuthClient } from 'https://securodb.com/assets/js/blitzdata/1.17.0/blitzdata.js';
```

Download `queue-worker.js` from the same folder into `web/` and set
`queue.workerPath` to it. The worker must come from the app's own origin.
(With npm access to `shapinggravity/blitzdata.ts`, `npm install` from Git works
too, and the worker ships in `dist/`.)

### Files

| File | Contents |
|---|---|
| `web/index.html` | Shell, nav between the models' screens, `<script type="module" src="app.js">` |
| `web/config.js` | `INSTANCE` URL, SDK version, and the full model names from step 3 |
| `web/app.js` | Sign-in, `BlitzData.initialize`, routing |
| `web/<model>.js` | One screen per model: list, add, edit, delete |
| `web/queue-worker.js` | Downloaded from the instance |
| `web/README.md` | How to run it and the instance settings it needs |

### The SDK calls to use (and no others)

```js
// Sign in (cross-host): OAuth code + PKCE; the SDK attaches the Bearer token
const auth = new OAuthClient({ baseUrl: INSTANCE });
if (!await auth.init()) { /* show a "Sign in" button */ auth.loginWithRedirect(); }

// Boot once
await BlitzData.initialize({
  clusters: { main: { readURL: INSTANCE, addURL: INSTANCE } },
  sync: { level: 'cache', ttl: 30000 },
  queue: { workerPath: './queue-worker.js' },
});

const model = await BDModel.get(MODELS.deal);

// Read + live updates; returns an unsubscribe function
const stop = model.subscribeToList(
  { conditions: [['status', '=', 'open']], customSort: 'amount', customSortDirection: 'DESC', limit: 50 },
  ({ items }) => render(items));

// Create, edit, delete: applied locally first, then delivered by the queue
await model.add({ title: 'New deal', amount: 1000 });
await obj.title.edit('Renamed');        // obj.title = … throws
await obj.delete();

// Values and save-state
obj.title.value;                         // undefined = not fetched, null = empty
obj.title.syncStatus(({ status }) => …); // pending / completed / failed / conflict
```

- Only list and filter on attributes that have a real column (not `index: none`).
- Show each model's attributes with inputs suited to the type (text, number,
  checkbox, date, a `<select>` for `enum` options).
- For `_fk` fields, a `<select>` filled from the target model's `list()`.
- Show the save-state, and a conflict as "changed elsewhere, reload".
- Keep it plain: no build step, no framework, one small CSS block.

### Instance settings the user must check

Signing in from another host needs the instance to allow the app's origin.
Write this into `web/README.md` and the final report:
- `CORS_ALLOWED_ORIGINS` and `OAUTH_ALLOWED_REDIRECT_URIS` must include the app's
  origin (for example `http://localhost:8080`, and the production URL).
- For securodb.com, that's a request to its operators. The app can't change it.

Running locally: `cd securodb-app/web && python3 -m http.server 8080`, then open
`http://localhost:8080`.

## 6. Report

Finish with:
- The models: full names, status, and the creation order used.
- Sample data: counts per model, and any refused records.
- The web app (if made): files, how to run it, and the instance settings to check.
- What's next: for example adding a model later with `/securodb:create-model`,
  changing an attribute (see `securodb:data-model`, "Changing a model later"),
  or adding realtime updates (`realtime: { models: [...] }` in `initialize`).

Never report a model or record as created unless the server's result confirms it.
