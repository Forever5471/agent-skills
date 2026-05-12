---
name: agent-intake-form-link
description: Use when a user starts a task in natural language and the agent may need additional structured information before proceeding. This skill guides the agent to decide whether intake is needed, generate an external self-contained form link/page, ask the user to fill it, parse the copied JSON submission, and continue the task. Works without native frontend card rendering or a backend by using copy-back form submissions.
metadata:
  short-description: Gather missing task info through external form links
---

# Agent Intake Form Link

Use this skill when a user starts a task in natural language and you need to decide whether more structured information is required.

If the task has enough information, proceed normally.

If information is missing, generate an external form page for the missing fields. The user opens the page, fills the form, clicks submit, copies the generated JSON, and pastes it back into the chat. Then parse the JSON and continue the task.

This skill does not require a native frontend card renderer in the agent platform.

## Core Flow

1. Read the user's task.
2. Decide whether the task is actionable as-is.
3. If not, identify only the missing information needed for the next useful step.
4. Create a form schema for those fields.
5. Generate or reference a self-contained external form page.
6. Give the user the link or local file path.
7. Ask the user to submit the form and paste the generated JSON back.
8. Stop and wait.
9. When the next user message arrives, parse the JSON submission.
10. Continue the original task using the structured values.

## Decision Rule

Do not create a form for every task.

Create a form only when at least one of these is true:

- Required details are missing.
- The task needs multiple structured fields.
- Free-form chat would likely be ambiguous.
- The agent needs typed values such as dates, amounts, emails, URLs, file references, or options.
- The workflow benefits from required/optional flags.

Ask a normal chat question instead when only one small clarification is needed.

## Form Mode

Default to **copy-back mode**:

- No backend is required.
- The external HTML page renders the form.
- On submit, the page generates JSON.
- The user copies that JSON back into chat.

Use backend mode only if the user or environment explicitly provides a submission endpoint/result API.

## Form Schema

Use this logical schema:

```json
{
  "type": "external_form",
  "schemaVersion": "2.0",
  "interactionId": "form_task_001",
  "title": "Form title",
  "purpose": "Why the form is needed.",
  "returnMode": "copy_back",
  "fields": [],
  "submitLabel": "Generate submission JSON"
}
```

Supported field kinds:

- `single`
- `multi`
- `text`
- `textarea`
- `email`
- `url`
- `number`
- `date`
- `datetime`
- `file`

Every field must include:

- `kind`
- `id`
- `header`
- `label`

For `single` and `multi`, every option must include:

- `label`
- `value`
- `description`

Reason over option `value`, not `label`.

## Interaction Ids

Generate a stable unique `interactionId` for every form.

Examples:

- `form_expense_001`
- `form_project_intake_001`
- `form_deploy_config_001`

If this is a later step, include `parentInteractionId` and `step`.

## Missing Field Design

When designing fields:

- Ask only for information needed to continue.
- Prefer typed fields over regex.
- Use `single` or `multi` when the answer space is known.
- Use `textarea` for optional context.
- Keep the first form short, usually 3-8 fields.
- Avoid collecting sensitive data unless necessary.

## Generating The External Page

If local file creation is available, create a standalone HTML file using `assets/form-template.html` as the starting point.

Replace the placeholder:

```text
__FORM_SCHEMA_JSON__
```

with the JSON form schema.

Save generated pages under an appropriate workspace/output directory, for example:

```text
generated-forms/<interactionId>.html
```

Then give the user the local file path or hosted URL.

If file creation is not available, output the form schema and ask the host system to create a form page from it.

## User Message After Generating The Form

Use this pattern:

```text
I need a few structured details before I can continue. I generated an external form for this step.

Form: <title>
interactionId: <interactionId>
Link: <path or URL>

Please open the form, fill it out, click submit, then paste the generated JSON back here.
```

Use the user's language where appropriate.

After sending this message, stop and wait for the next user message.

## Submission JSON

The form page should generate this JSON:

```json
{
  "type": "form_submission",
  "source": "external_form_link",
  "interactionId": "form_task_001",
  "status": "submitted",
  "values": {
    "field_id": "value"
  },
  "notes": "",
  "submittedAt": "2026-05-12T10:00:00.000Z"
}
```

Supported statuses:

- `submitted`
- `cancelled`

## Parsing The Returned Submission

When the next user message arrives:

1. Parse JSON first.
2. Confirm `type` is `form_submission` when present.
3. Confirm `interactionId` matches the expected form.
4. Check `status`.
5. Use `values` as authoritative.
6. Use `notes` as optional context.

If the user pasted plain text instead of JSON, parse lines in this format:

```text
field_id -> value
field_id → value
```

For `multi`, split values by English comma.

## Validation After Submission

Validate lightly after receiving the submission:

- Required values are present.
- `single` values match allowed option values.
- `multi` values all match allowed option values.
- `email` looks like an email.
- `url` starts with `http://` or `https://` unless otherwise allowed.
- `number` parses as a number and respects min/max when specified.
- `date` uses `YYYY-MM-DD` when possible.

If validation fails, ask for only the missing or invalid fields. You may either:

- Ask in chat, or
- Generate a smaller follow-up form.

## File Fields

In copy-back mode, file fields cannot upload raw files unless the page or platform supports uploads.

Use file fields for:

- File names
- Local paths
- URLs
- Platform file references

Do not assume you can read file content unless the runtime provides access.

## Example: Expense Reimbursement Intake

User:

```text
I want to submit an expense reimbursement.
```

The task needs structured details, so create a form schema:

```json
{
  "type": "external_form",
  "schemaVersion": "2.0",
  "interactionId": "form_expense_001",
  "title": "Expense Reimbursement",
  "purpose": "Collect reimbursement type, amount, purpose, and invoice status.",
  "returnMode": "copy_back",
  "fields": [
    {
      "kind": "single",
      "id": "expense_type",
      "header": "Type",
      "label": "Select the reimbursement type",
      "required": true,
      "options": [
        {
          "label": "Travel",
          "value": "travel",
          "description": "Transportation, hotel, meals, and other travel costs"
        },
        {
          "label": "Office",
          "value": "office",
          "description": "Office supplies, equipment, and consumables"
        },
        {
          "label": "Entertainment",
          "value": "entertainment",
          "description": "Client hospitality or business meals"
        },
        {
          "label": "Communication",
          "value": "communication",
          "description": "Phone, network, or communication costs"
        },
        {
          "label": "Other",
          "value": "other",
          "description": "Other expense types"
        }
      ]
    },
    {
      "kind": "number",
      "id": "amount",
      "header": "Amount",
      "label": "Enter the reimbursement amount",
      "required": true,
      "min": 0.01,
      "step": 0.01
    },
    {
      "kind": "text",
      "id": "purpose",
      "header": "Purpose",
      "label": "Briefly describe the expense purpose",
      "required": true,
      "maxLength": 120
    },
    {
      "kind": "single",
      "id": "invoice_status",
      "header": "Invoice",
      "label": "Do you have an invoice?",
      "required": true,
      "options": [
        {
          "label": "Has invoice",
          "value": "has_invoice",
          "description": "A valid invoice is available"
        },
        {
          "label": "No invoice",
          "value": "no_invoice",
          "description": "No invoice is currently available"
        }
      ]
    },
    {
      "kind": "textarea",
      "id": "notes",
      "header": "Notes",
      "label": "Additional notes (optional)",
      "required": false,
      "maxLength": 500
    }
  ],
  "submitLabel": "Generate submission JSON"
}
```

Then generate the external HTML page and tell the user to paste the generated JSON back.

## Completion Behavior

After receiving a valid submission:

1. Briefly confirm receipt.
2. Summarize the structured values.
3. Continue the original task.
4. Do not ask for information already collected.

## Fallback

If you cannot create or link an external form page, fall back to a chat-based structured form:

```text
field_id -> value
```

Use the same fields and parsing rules.
