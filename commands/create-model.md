---
description: Create a new SecuroDB model (schema) from a name and a natural-language list of fields.
argument-hint: 'model name; field:type pairs (e.g. Invoice; number:string! amount:decimal! paidAt:datetime)'
---

You are creating a new model in SecuroDB.

Use the `securodb` MCP server to do the work. Before calling anything, look at the
tools it exposes and pick the ones for listing, inspecting and creating models.
Follow their current schemas; do not invent endpoints or parameters.

## 1. Parse `$ARGUMENTS`

- **Model name**: the first token, or the text before the first `;`.
- **Fields**: `name:type` pairs. A trailing `!` means required (`email:string!`).
  Accept loose phrasing too ("an email that's required, and an optional age").
- **Options** if given: unique fields, indexes, defaults, relations
  (`customer:ref(Customer)`), descriptions.

If `$ARGUMENTS` is empty or has no fields, ask the user what the model is for and
propose a field list before creating anything.

## 2. Check what already exists

- List existing models. If one has the same or a very similar name, stop and ask
  whether to reuse it, extend it, or create a new one under another name.
- For every relation field, confirm the target model exists. If it doesn't, ask
  whether to create it first or drop the relation.

## 3. Map the field types

- Map each requested type to the closest type SecuroDB supports (for example
  `int` → integer, `text` → string, `bool` → boolean, `date`/`timestamp` → datetime).
- If a type has no clear match, ask rather than guess.
- Use the naming convention of existing models (camelCase vs snake_case,
  singular vs plural) when it's clear from them.
- Do not add fields the user didn't ask for, except ones SecuroDB requires
  (such as a primary key, if it isn't generated automatically).

## 4. Confirm, then create

Show the final definition in a short table (field, type, required, unique, default
or relation) and create the model once the user agrees. Skip this confirmation only
when the user already gave a complete, unambiguous definition and asked you to go
ahead.

## 5. Report

Confirm with:
- The model name and identifier returned by SecuroDB
- The fields as SecuroDB stored them, noting any type you had to map or any field
  that was changed or rejected
- A next step, such as `/securodb:create-model` for a related model, or the
  `post-api-transaction` skill for writing records to it

If the `securodb` MCP server isn't connected, say so plainly, and don't pretend the
model was created. Offer to write the model definition out as JSON so the user can
apply it later.
