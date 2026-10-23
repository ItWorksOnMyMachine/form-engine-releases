# Adding a New Form

Forms are created and maintained in the admin builder and stored in the
database. There are no static form files, no form registry, and no code
changes or deploys involved in adding a form.

For every property mentioned below, see `docs/SCHEMA_REFERENCE.md`.

---

## Prerequisites

- You can sign in to `/admin` and have selected a tenant.
- To create forms you need the `creator` or `admin` role (from the IdP).
  The creator automatically receives the `owner` grant on the new form.

| Permission | Can |
|------------|-----|
| `admin` role | Everything on every form in the tenant |
| `owner` grant | Publish, unpublish, delete; plus editor rights |
| `editor` grant | Edit and save; send invitations |
| `reader` grant | View submissions, metrics, export (in `/dashboard`) |

---

## Step 1: Create the form

1. Go to `/admin` and choose **New Form**.
2. Enter a title. The form is created as a **draft** with one empty page
   and you land in the builder at `/admin/forms/{formId}`. The form's ID
   is a UUID and is used in all URLs.

## Step 2: Settings

In the form settings panel:

- **Title / description**: shown at the top of the form.
- **Show page counter & progress bar** (`showPageCounter`).
- **Who can fill out this form** (`access`): *Anyone with the link*
  (public), *Invited people only* (invitation), or *Signed-in members*
  (authenticated: sign-in required, submitter recorded, listed on
  `/my-forms`). See Step 7.
- **Success message** or **success redirect URL**.

## Step 3: Pages

Add pages for multi-step forms. Each page has a key, title, optional
description, an option to hide the title, and an optional condition that
skips the page. Fields can be moved between pages. Page layout does not
change the submission payload.

## Step 4: Fields

Add fields from the field type picker: section, HTML content, text,
email, phone, number, textarea, select, multi-select, radio, checkbox,
date, rating, NPS, and repeatable. For each field set:

- **Key**: lowercase with underscores, unique across the whole form. It
  becomes the payload key and the CSV column, so choose it deliberately
  and avoid renaming after the form is live.
- **Label, description, placeholder, width** (full or half).
- **Options** for select, multi-select, radio, rating and NPS.
- **Allowed dates** (date fields): earliest and latest date, fixed or
  relative to the day the form is filled out (e.g. no future dates, or up
  to a year ahead). The **?** next to it explains the options.
- **Validation**: required (optionally with a custom message), min/max
  length, min/max value, regex pattern with message.
- **Condition**: show the field only when another field matches (equals,
  not equals, one of, has value), optionally grouped with AND/OR.

Tips:

- Use **sections** to group related fields. A section's condition hides
  every field under it until the next section.
- Use **HTML content** for instructions or notices. It is sanitized.
- For "list of things" data (locations, contacts), use a **repeatable**
  field: define the child fields, pick 2 to 4 summary columns, set the
  add-button label and min/max rows.
- Prefix keys when they could collide: `contact_email`, `billing_email`.
- Test conditions: a condition that references a nonexistent key never
  shows its field.

The **JSON preview** shows the exact schema that will be saved.

## Step 5: Submission targets (optional)

Every submission is stored in the database. To also forward it (for
example to an n8n workflow), add a **webhook target** with a URL and
optional headers. Webhooks are called server-side after the submission is
saved; failures are logged, not shown to the respondent.

Targets are server-only: public pages never receive their URLs or
headers, so headers may carry credentials (see `docs/SCHEMA_REFERENCE.md`).

## Step 6: Save and publish

- **Save Draft** stores your changes. Drafts are not reachable publicly.
- **Publish** saves, sets the status to published, and records a snapshot
  in the version history (`v1`, `v2`, ...). Publish again after later
  edits to create a new version.
- **Unpublish** returns the form to draft; the public URL then returns
  404.
- Publishing, unpublishing and deleting require the `owner` grant (or the
  `admin` role).

## Step 7: Share it

**Public forms**: copy the form URL shown in the builder's top bar,
`https://forms.example.com/forms/{formId}`. Respondents can leave and
return; progress resumes from a draft cookie.

**Signed-in forms**: share the same URL, or point people to
`/my-forms`, which lists every signed-in form in their tenant and marks
the ones they have in progress. People sign in, each person's draft is
their own, and the submission records who submitted it. These forms can't
be embedded or sent as invitations.

**Invitation-only forms**: send invitations from
the form's **Send** action in `/admin` (one recipient, emailed), or
generate invitation links through the admin invitations API. Each
invitation is a single-use link with a token
(`/forms/{formId}?token=...`), optionally expiring. Campaign-driven sends
are orchestrated by n8n through the integration API.

**Embedding** in another site: use the JS SDK
(`packages/form-engine-sdk`, served at `/form-engine-sdk.js`) for inline,
dialog, or popup embeds. See `packages/form-engine-sdk/README.md`. The
embed route (`/forms/{formId}/embed`) accepts `theme`, `cardBg` and
`textColor` colour overrides.

**Prefill**: append field keys as query parameters, for example
`/forms/{formId}?country=usa&ref=abc123`. Invalid values are ignored, and
unknown parameters are stored in `dynamic_values`. Details in
`docs/SCHEMA_REFERENCE.md`.


## Step 8: Grant access and view results

- **Manage Access** in the builder opens the form's grants in the
  dashboard: give colleagues `owner`, `editor` or `reader`.
- Submissions, metrics and CSV export are under `/dashboard`.

### Finding and exporting submissions

On a form's submissions page in `/dashboard`:

- **Search responses** matches any answer value (not field names) and the
  submitter's name or email.
- **Submitted by** filters by respondent (signed-in forms) or invitation
  recipient name/email.
- **Filter by field** adds exact, case-insensitive filters on one answer,
  e.g. a case number, a disposition, or a single checklist item; fields with
  fixed options offer a dropdown. Filters combine.
- **From / To** dates are inclusive.
- **Export CSV** exports exactly what the filters show. Columns cover every
  form version present in the export (the latest version's order first).

Viewing a submission and exporting are recorded in the audit log.

**Retiring a form:** once a form has submitted records it can't be
deleted; unpublish it instead. Its records stay available in the
dashboard.

## Step 9: Test it

Before sharing, open the published URL in a private window and check:

- all pages and fields render, and page conditions skip correctly
- conditional fields show and hide as expected
- required and pattern validation trigger on Next/Submit
- repeatable rows add, edit and delete
- checklists: every item answers, N/A appears only where intended, and a
  comment is demanded for failing answers
- the submission appears in `/dashboard` and reaches any webhook target

---

## Moving forms between environments

On the builder's **Schema JSON** tab:

- **Download JSON** saves the form definition as `<title>.form.json`. Keep
  these files in your own source control if you want a history of form
  changes outside the database.
- **Import JSON** loads a downloaded definition into the form you have
  open (create an empty form first in the target environment). The file is
  migrated to the current schema version and validated with the same rules
  as Save; you confirm before it replaces the editor contents, and nothing
  is stored until you click **Save Draft** (then **Publish**).

Imported webhook targets come along unchanged; point them at the target
environment's endpoints before publishing.
