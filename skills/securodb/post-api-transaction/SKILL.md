---
name: securodb-post-api-transaction
description: Writes records to SecuroDB through a POST API transaction, so several creates, updates or deletes succeed or fail together. Use when the user wants to insert or change data in SecuroDB models, especially multi-record or multi-model writes that must stay consistent (an order with its line items, a transfer between two accounts). Not for creating or changing models; use the create-model command for that.
---

# POST API transaction

Write data to SecuroDB as one atomic transaction: build the operations, check them
against the models, send them in a single POST, and report exactly what was
committed.

## Example requests

- "Create an order for customer 42 with these three line items."
- "Move 150 from account A to account B."
- "Import these rows into Products, and roll back if any of them fail."

## Workflow

1. **Find the tools.** Use the `securodb` MCP server. Look at its current tool list
   and schemas for: listing models, reading a model's fields, and submitting a
   transaction (a POST that accepts a list of operations). Use those schemas
   exactly. Do not make up endpoints, field names or payload shapes. If the server
   isn't connected, stop and tell the user. Don't simulate a response.

2. **Understand the write.** Work out which models are involved and what each
   operation is: `create`, `update` or `delete`. For update and delete, find out how
   the target record is identified (ID or a unique field). If one record depends on
   another in the same transaction (a line item pointing at a new order), note the
   order and how SecuroDB references an earlier operation's result.

3. **Validate before sending.** Fetch each involved model's definition and check:
   - Required fields are present, and the values fit the field types.
   - Unique fields aren't obviously duplicated within the batch.
   - Relation targets exist, or are created earlier in the same transaction.
   - Update and delete targets exist. Read them first when that's cheap.
   Ask the user about anything missing or ambiguous instead of filling in guesses.

4. **Build the payload.** Put every operation into one transaction request, in
   dependency order. Keep the user's values exactly as given apart from type
   coercion that is clearly safe (`"42"` → `42` for an integer field). Send
   secrets and credentials only in whatever auth the MCP server already handles.
   Never put them in the payload or show them in chat.

5. **Confirm destructive or large writes.** Before sending, show a short summary
   (model, operation, key fields, count) and wait for a yes when the transaction:
   - deletes records,
   - overwrites existing values, or
   - touches more than about 20 records.
   Small, clearly requested creates can go straight through.

6. **Send once.** Submit the transaction in a single call. If SecuroDB supports an
   idempotency key, include one, so a retry can't apply the write twice. Don't
   retry automatically after a timeout. Check whether the write landed first.

7. **Report the outcome.**
   - **Committed:** list each operation with the IDs SecuroDB returned.
   - **Rolled back or rejected:** quote the error SecuroDB returned, name the
     operation that caused it, explain the fix in plain words, and say clearly
     that nothing was written.
   - **Unknown (timeout, dropped connection):** say the state is unknown, then
     read the affected records to find out before doing anything else.

## Rules

- One user request is one transaction. Split it only if SecuroDB's limits require
  it, and tell the user when the parts are no longer atomic.
- Never report a write as done unless SecuroDB's response confirms it.
- Don't change model definitions here. If a write needs a field that doesn't exist,
  stop and suggest `/securodb:create-model`, or extending the model first.
