# Schema Reference

Reference for every property of a `FormSchema` and its field types.

Sources of truth:

- `src/types/schema.ts`: TypeScript types
- `src/lib/schema-validation.ts`: zod validator applied when the builder saves a form
- `src/lib/schema-migration.ts`: version upgrades applied on load
- `src/lib/utils.ts`: condition evaluation
- `src/lib/prefill.ts`: query-string prefill

Forms are authored in the admin builder (`/admin`) and stored as JSONB in
the `forms` table. You rarely write this JSON by hand, but the builder's
JSON preview shows exactly this shape.

---

## FormSchema (schemaVersion 2)

```ts
interface FormSchemaV2 {
  schemaVersion: 2
  id: string                 // set to the form's UUID by the server
  title: string
  description?: string
  pages: FormPage[]          // at least one
  showPageCounter?: boolean  // default true: "Page N of M" + progress bar
  access?: 'public' | 'invitation' | 'authenticated'  // default 'public'
  captcha?: boolean          // default false: bot protection (see below)
  submission: FormSubmission // required object; may be {}
  successMessage?: string    // default "Your submission has been received. Thank you!"
  successRedirect?: string   // if set, the browser navigates here after submit
}
```

| Property | Validation (zod) | Notes |
|----------|------------------|-------|
| `id` | lowercase letters, digits, hyphens (`^[a-z0-9][a-z0-9-]*$`) | Replaced with the DB UUID on create and on every load. |
| `title` | non-empty | Also stored in the `forms.title` column. |
| `pages` | at least 1 | |
| `successRedirect` | string | Absolute or relative URL. Takes precedence over `successMessage`. In an SDK embed the host page is navigated instead of the iframe (absolute `http(s)` URLs only). The builder preview never redirects. |

### Access (`access`)

| Value | Who can fill it out | Drafts | Identity recorded |
|-------|---------------------|--------|-------------------|
| `public` (default) | Anyone with the link | Anonymous; resumed from a draft cookie (`form_draft_{id}`) | No |
| `invitation` | Holders of a valid invitation token (`?token=...`); otherwise "Invitation required" | The invitation's own draft | Recipient name/email from the invitation |
| `authenticated` | Signed-in members of the form's tenant (redirected to sign-in, then back) | Owned by the user: their latest unfinished draft is resumed, never a cookie or URL token (safe on shared PCs); "Start a new form" discards it | `respondent_user_id`, `respondent_name`, `respondent_email` from the session |

Rules shared by the form page, the embed, and `/api/submissions`
(`resolvePublicAccess` in `src/lib/form-loader.ts`):

- Only forms with status `published` are served.
- With an invitation token, expired and already-submitted tokens show an
  error page.
- `authenticated` forms: the page shows "Signed in as {name}. Not you?";
  they are listed on `/my-forms`; they cannot be embedded (third-party
  iframes can't hold a sign-in session) and don't use invitations (the
  invitation routes refuse them). On submit the respondent's identity comes
  from the session, never from the payload, and is included in the webhook
  `_meta.respondent`.

### Bot protection (`captcha`)

For forms anyone can fill out. With `captcha: true`:

- The page the respondent submits from shows an ALTCHA widget that solves
  a proof-of-work challenge in the background (self-hosted,
  `GET /api/captcha/challenge`; no image puzzles, no third-party requests)
  plus an off-screen honeypot field.
- **Which pages:** mark pages with `captcha: true` (builder: page settings,
  "Show bot protection on this page") to show it only there, typically the
  final page. With no page marked, whichever page ends the respondent's
  path shows it; with conditional pages that can be an early page (e.g.
  page 1 before an answer reveals the rest). A submission whose path ends
  on an unmarked page isn't asked for a CAPTCHA (the honeypot still
  applies), so mark every page a real submission can end on that needs
  protecting. Logic: `src/lib/captcha-pages.ts`.
- Final submit must carry the solved payload when its page carries the CAPTCHA. The server checks the
  signature and expiry (1 hour), that at least 3 seconds passed since the
  challenge was issued, and that the solution wasn't used before; a filled
  honeypot is rejected. Drafts are not checked.
- Validation errors are reported before the CAPTCHA is checked, so fixing
  them doesn't need a new verification; a rejected CAPTCHA resets the widget.
- Requires `CAPTCHA_HMAC_SECRET` and HTTPS (docs/DEPLOYMENT.md). The
  builder preview never shows the widget.

---

## FormPage

```ts
interface FormPage {
  key: string                // ^[a-z_][a-z0-9_]*$
  title: string
  description?: string
  condition?: FieldCondition // page is skipped when false
  showTitle?: boolean        // default true
  captcha?: boolean          // forms with captcha: this page carries it (see Bot protection)
  fields: FieldDef[]
}
```

- Pages are shown one at a time with Next/Previous. A single-page form
  shows only Submit.
- Page conditions are re-evaluated as values change; skipped pages are
  excluded from navigation and the page counter.
- Page structure does not affect the payload (see "Payload shape").

---

## FormSubmission

```ts
interface FormSubmission {
  targets?: WebhookTarget[]
}

interface WebhookTarget {
  type: 'webhook'
  url: string
  headers?: Record<string, string>
}
```

- The submission is always saved to the `submissions` table first.
  Webhooks are then called server-side, in parallel, fire-and-forget:
  failures are logged and do not affect the respondent.
- The webhook body is the payload plus a `_meta` object
  (`formId`, `submissionId`, `submittedAt`, `recipientName`, `recipientEmail`).
- `webhook` is the only target type. Legacy `api` targets from older
  schemas are removed when a form is loaded.

Targets are server-only: public pages receive the schema with
`submission` emptied (`toPublicSchema` in `src/lib/form-loader.ts`), so
`headers` may carry credentials. Each URL passes an SSRF check before it
is called (private/internal addresses are refused unless the host is in
`WEBHOOK_ALLOWED_HOSTS`), and redirects are not followed.

The webhook body is the **server-validated** payload: unknown keys and
values of hidden fields (field, section, or page condition not met) are
removed before storage and delivery. See `src/lib/submission-validation.ts`.

---

## Common field properties

All data-collecting fields (every type except `section` and `html`) share:

```ts
interface BaseField {
  key: string                // ^[a-z_][a-z0-9_]*$, unique across the whole form
  label: string
  description?: string       // helper text below the field
  placeholder?: string
  defaultValue?: unknown
  validation?: FieldValidation
  condition?: FieldCondition // hidden until the condition is met
  width?: 'full' | 'half'    // default 'full'; 'half' = two columns from the sm breakpoint up
  disabled?: boolean
}
```

Keys must be unique across all pages, because the payload is flat.

### FieldValidation

```ts
interface FieldValidation {
  required?: boolean | string          // true, or a custom error message
  minLength?: number
  maxLength?: number
  min?: number                         // number fields
  max?: number
  pattern?: { value: string; message: string } // value is a regex source string
}
```

- Validation runs in the browser (react-hook-form) when the respondent
  clicks Next or Submit. Hidden fields are not validated.
- The TypeScript type also declares `custom?: (value, values) => string | true`.
  Functions can't be stored as JSON, so the zod validator rejects it and
  it is unusable in stored forms.

The same rules run on the server (`src/lib/submission-validation.ts`),
which also drops unknown keys and, on submit, values of hidden fields
before storage. Keep the two in sync when changing field types.

---

## Field types

| Type | Extra properties | Value in payload |
|------|------------------|------------------|
| `text`, `email`, `phone` | none | string |
| `textarea` | `rows?: number` | string |
| `number` | none (use `validation.min/max`) | number |
| `select` | `options` (min 1) | string |
| `multiselect` | `options` (min 1) | string[] in option order (rendered as a checkbox group; required = at least one checked) |
| `radio` | `options` (min 1), `inline?: boolean` | string |
| `checkbox` | `checkboxLabel?: string` | boolean |
| `date` | `minDate?`, `maxDate?` (ISO `YYYY-MM-DD`) | string `YYYY-MM-DD` |
| `rating` | `options` (min 2) | string (value of the selected star) |
| `nps` | `options` (min 2; designed for 3) | string |
| `repeatable` | see below | array of objects |
| `checklist` | see below | flat keys per item: answer string + optional comment |
| `section` | see below | none |
| `html` | see below | none |

### Options

`options` accepts plain strings or `{ label, value }` objects, mixed
freely. A plain string is used as both label and value.

```json
"options": ["Yes", "No", { "label": "Not sure", "value": "unsure" }]
```

### rating

A row of star icons that behaves like a radio group. One star per option,
in order; selecting a star fills it and all before it. The selected
option's label is shown below the stars.

### nps

A row of sentiment faces (unhappy to happy) that behaves like a radio
group. Colours: red for the first option, amber for the middle, green for
the last. Intended for exactly 3 options.

### section

```ts
interface SectionField {
  type: 'section'
  key: string
  label: string              // rendered as a heading
  description?: string
  condition?: FieldCondition
}
```

A section's `condition` **cascades** to every field after it on the same
page until the next section. A field below a hidden section is hidden
regardless of its own condition; when the section is visible, the field's
own condition still applies.

### html

```ts
interface HtmlField {
  type: 'html'
  key: string
  label: string              // shown only in the builder
  content: string            // HTML, sanitized at render time
  condition?: FieldCondition
}
```

Static content (instructions, notices). Script tags, event-handler
attributes and `javascript:` URLs are stripped by `src/lib/sanitize-html.ts`.

### repeatable

```ts
interface RepeatableField extends BaseField {
  type: 'repeatable'
  childFields: ChildFieldDef[] // min 1; any type except 'repeatable'
  summaryColumns: string[]     // min 1; child keys shown as grid columns
  addLabel?: string            // button text, e.g. "Add location"
  minRows?: number
  maxRows?: number
}
```

- Each row is entered in a modal child form, then shown in a summary grid
  (edit and delete per row).
- All child values are stored, not just the summary columns.
- Nesting a `repeatable` inside another is not allowed.
- Conditions on child fields reference other child fields in the same row.
- Payload: `"locations": [{ "name": "...", "city": "..." }, ...]`.
- Repeatables cannot be prefilled from the query string.

### checklist

A list of criteria, each answered from the same choices, with an optional
comment per item. Built for inspection/QA checklists.

```ts
interface ChecklistField {
  type: 'checklist'
  key: string                  // identifies the field; NOT a payload key
  label: string                // shown as the group legend
  items: Array<{
    key: string                // payload key for the answer; unique across the form
    label: string              // the criterion
    group?: string             // sub-heading, e.g. "Fit"
    allowNA?: boolean          // offer the N/A choice on this item
  }>                           // at least 1
  choices?: Array<{ value: string; label: string }>  // default Pass / Fail
  naChoice?: { value: string; label: string }        // default { value: 'na', label: 'N/A' }
  comments?: 'none' | 'optional'                     // default 'optional'
  commentRequiredOn?: string[]                       // e.g. ['fail']
  validation?: { required?: boolean | string }       // required = every item must be answered
  // plus description, condition (applies to the whole checklist)
}
```

Payload (flat, like every other field):

```json
{ "form_tissue": "fail", "form_tissue_comment": "Too bulbous at 14", "function_screw": "na" }
```

- The answer is stored under `item.key`, the comment under
  `item.key + '_comment'`. These keys must not collide with any other
  field or item key in the form (the validator rejects duplicates).
- `commentRequiredOn` answers make the comment mandatory; those rows are
  flagged in the form and in the submission view (icon and text, not only
  color). Values must be among the choices (including N/A).
- A hidden checklist (its condition or its section's condition not met)
  contributes no keys on submit.
- In the builder, "Add items" accepts one criterion per line with an
  optional `Group: ` prefix and generates unique keys; criteria containing
  "if" (e.g. "if applicable") get N/A automatically.
- CSV export has one column per item and one per comment. Checklists can't
  be used inside repeatables, prefilled from the query string, or targeted
  by visibility conditions (conditions can reference item keys directly).

---

## Conditions

A condition controls whether a field, section or page is shown.

### Simple

```ts
interface SimpleCondition {
  field: string                       // key of another field
  equals?: string | boolean | number
  notEquals?: string | boolean | number
  oneOf?: (string | number)[]
  hasValue?: boolean
}
```

Only the first operator present is evaluated, in this order: `equals`,
`notEquals`, `oneOf`, `hasValue`. A condition with no operator is true.

- `equals`: strict equality (`===`). Checkbox values are booleans, so use
  `"equals": true`, not `"true"`.
- `notEquals`: false while the referenced field is empty, so a field
  doesn't appear before the respondent has answered.
- `oneOf`: true when the value is in the list.
- `hasValue: true`: the value is not empty (`undefined`, `null`, `''`).
  `hasValue: false` is the inverse.

### Compound (AND / OR)

```ts
interface CompoundCondition {
  and?: FieldCondition[]   // all must be true
  or?: FieldCondition[]    // at least one must be true
}
```

Compound conditions nest arbitrarily. If both `and` and `or` are present,
both must hold. The builder's condition editor supports simple
conditions and one level of AND/OR grouping.

```json
{ "and": [
  { "field": "country", "equals": "usa" },
  { "or": [ { "field": "plan", "equals": "pro" }, { "field": "seats", "hasValue": true } ] }
] }
```

Conditions are evaluated against the values of the whole form (all
pages). A condition referencing a key that doesn't exist evaluates
against `undefined`, so `equals` conditions stay hidden.

---

## Payload shape

The submission payload is a flat `Record<string, unknown>` keyed by field
key, regardless of pages. `section` and `html` fields contribute nothing;
repeatables contribute arrays of objects.

```json
{
  "full_name": "Jane Doe",
  "email": "jane@example.com",
  "subscribe": true,
  "locations": [{ "name": "Main", "city": "Springfield" }],
  "dynamic_values": { "campaign": "fall" }
}
```

- `dynamic_values` holds query-string parameters that don't match a field
  (see Prefill).
- `_page` is used internally for draft resume and is stripped on submit.
- Values of fields hidden by a condition are currently kept in the
  payload if they were entered before the field was hidden.

---

## Prefill (query string)

`/forms/{id}?key=value&...` prefills fields by key. Values are checked
against the field and silently ignored if invalid:

| Field type | Accepted |
|------------|----------|
| `select`, `radio`, `rating`, `nps` | an existing option value |
| `multiselect` | comma-separated option values (invalid ones dropped) |
| `number` | numeric, within `validation.min/max` |
| `date` | a parseable date, normalised to `YYYY-MM-DD`, within `minDate/maxDate` |
| `checkbox` | `true` or `false` |
| `text`, `email`, `phone`, `textarea` | any string |
| `repeatable`, `section`, `html` | ignored |

- `token` is reserved (invitation token). The embed route also reserves
  `theme`, `cardBg` and `textColor`.
- Parameters that don't match a field are stored in `dynamic_values`, so
  context such as a reference ID survives drafts and appears in the
  submission.
- Saved draft values take precedence over prefill values.
- Prefilled fields remain editable, and URL values are not signed, so
  treat them as user input.

---

## Versioning and migration

`schemaVersion` identifies the schema shape. `migrateSchema()` upgrades
any stored schema to `LATEST_VERSION` (currently **2**) on every load
path (admin, public page, submissions API), and the upgraded schema is
written back. Published snapshots in `form_versions` are migrated in memory
only (they are immutable records).

| From | To | Change |
|------|----|--------|
| 0 (no `schemaVersion`) | 1 | Flat `fields[]` wrapped in a single page `page_1` titled with the form title. |
| 1 | 2 | `allowAnonymous: false` becomes `access: 'invitation'`; anything else `access: 'public'`. `allowAnonymous` is removed. |

To make a breaking schema change, increment `LATEST_VERSION`, add
`migrate_N_to_N+1`, and register it in the `migrations` map. Never require
manual data migration.

Publishing a form (see `docs/ADDING_A_FORM.md`) snapshots the schema into
`form_versions` with an incrementing version number.

---

## Known issues

None currently. (Fixed: form IDs starting with a digit failed validation,
Phase 1; the embed route ignored the access mode and `token`, Phase 2; the
public lookup did not run schema migrations, Phase 4.)
