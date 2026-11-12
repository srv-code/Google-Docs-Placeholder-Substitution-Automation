# Google Docs Placeholder Substitution Automation

## Project Context, Requirements, Architecture & LLM Handoff

**Purpose:** This document is the durable handoff brief for continuing
development in a new ChatGPT, Claude, or other LLM conversation. Upload
it together with the latest `Code.gs` and `Index.html`.

**Source of truth:** The latest uploaded source files are authoritative
for exact implementation details. This handoff records requirements,
decisions, constraints, and known implementation details. If the source
and this document disagree, identify the discrepancy rather than
silently assuming either one is correct.

---

## 1. Project overview

This Google Apps Script automation opens a custom modal in Google Docs.
It reads the currently active tab, dynamically renders a form for
predefined and additional placeholders, substitutes submitted values in
that tab, renames the tab, and clears its dedicated tab emoji icon.

Project files:

- `Code.gs` --- server-side configuration, tab eligibility checks,
  placeholder discovery, validation, replacement, and tab-title
  generation.
- `Index.html` --- modal UI, dynamic field rendering, client-side
  validation, loading/status messages, and tab-title preview.
- `README.md` --- concise repository overview and setup guide.

The Apps Script project uses `DocumentApp`, `HtmlService`, `Utilities`,
and the Advanced Google Docs API (`Docs.Documents.get()` and
`Docs.Documents.batchUpdate()`). The Advanced Google Docs API service
must be enabled.

## 2. Non-negotiable development preferences

1.  **For code changes, provide the complete contents of every modified
    file.** Do not provide fragmented snippets, partial functions,
    diffs, or patch instructions unless explicitly requested.
2.  State which files changed and which did not.
3.  Keep explanations concise but explain important behavior,
    assumptions, and risks.
4.  Do not remove or alter existing functionality as a side effect of an
    unrelated change.
5.  Preserve the architecture: central configuration in `Code.gs`,
    data-driven HTML, and server-side revalidation.
6.  Never duplicate the predefined placeholder list in `Index.html`; the
    server configuration is the source of truth.
7.  Inspect the latest uploaded source files before changing code. Do
    not assume an earlier chat's source snapshot is still current.
8.  Do not claim the code was tested in the user's Apps Script
    environment unless it actually was.

## 3. Menu and modal

- Custom menu: `Automation`
- Menu item: `Substitute Values`
- Handler: `showAutomationDialog`
- HTML file: `Index`
- Dialog title: `Value Substitution`
- Dialog dimensions: 560 × 760

On initialization, the HTML displays `Loading document information...`
with a spinner and calls `getInitialData()`.

## 4. Active-tab targeting and safety

This is a critical requirement.

- Only the currently active tab is eligible.
- The active tab must be a **direct child** of a tab titled exactly
  `Generated`.
- Root tabs and tabs under any other parent are ineligible.
- The active tab ID is captured during initialization and passed back
  to `substitutePlaceholders()`.
- The server rechecks that the active tab ID has not changed, that the
  target tab still exists, and that its direct parent is still
  `Generated`.
- Every `replaceAllText` request must be scoped to the captured tab ID
  using `tabsCriteria`.
- Never broaden replacement to the entire document or other tabs.
- If the active tab changes while the modal is open, reject the
  operation and ask the user to reopen the modal from the intended
  tab.

## 5. Central configuration and current settings

`PLACEHOLDER_CONFIG` in `Code.gs` is the single source of truth. It is
frozen using `Object.freeze()`.

Current important settings, verified against the latest uploaded
`Code.gs`:

```javascript
settings: {
  menuTitle: 'Automation',
  menuItemTitle: 'Substitute Values',
  htmlFile: 'Index',
  dialogTitle: 'Value Substitution',
  dialogWidth: 560,
  dialogHeight: 760,
  requiredParentTab: 'Generated',
  timeZone: 'Asia/Kolkata',
  dateFormat: 'dd MMMM yyyy',
  tabTitleDateFormat: 'dd/MMM/yyyy',
  maxTabTitleLength: 100,
  titleEllipsis: '...',
  additionalSectionTitle: 'ADDITIONAL PLACEHOLDERS',
  predefinedSectionTitle: 'PLACEHOLDERS'
}
```

### Date formats --- keep them separate

- `dateFormat: 'dd MMMM yyyy'` controls the default value of the
  `{{DATE}}` placeholder. Example: `11 November 2026`.
- `tabTitleDateFormat: 'dd/MMM/yyyy'` controls only the generated
  Google Docs tab-title date. Example: `11/Nov/2026`.
- The user explicitly requested changing the `{{DATE}}` placeholder
  format, not the tab-title date format. Do not change
  `tabTitleDateFormat` when changing `dateFormat`.
- Both use the configured `Asia/Kolkata` time zone.

### Document title configuration

```javascript
documentTitle: {
  companyFieldKey: 'companyName',
  separator: ' - '
}
```

The generated title is `<COMPANY_NAME> - <SCRIPT_GENERATED_DATE>`.

## 6. Current predefined placeholder configuration

The latest uploaded `Code.gs` defines these six predefined fields:

---

Key Placeholder Display title Required? Default / behavior

---

`date` `{{DATE}}` Date Yes `defaultValueType: 'today'`;
generated using `dateFormat`
and `Asia/Kolkata`

`companyName` `{{COMPANY_NAME}}` Company Name Yes `your company`

`companyLocation` `{{COMPANY_LOCATION}}` Company Yes `Bengaluru, Karnataka`
Location

`jobTitle` `{{JOB_TITLE}}` Job Title Yes `Lead React Native engineer`

`jobReferralSource` `{{JOB_REFERRAL_SOURCE}}` Job Referral Yes `LinkedIn`
Source

`jobId` `{{JOB_ID}}` Job ID No Empty string; special
replacement format

---

These are the only predefined fields. The first five are the **only
mandatory fields**. `{{JOB_ID}}` is the only optional predefined field.

Every field definition must have an explicit boolean `optional`
property.

### Exact optional Job ID behavior

The latest uploaded `Code.gs` has:

```javascript
Object.freeze({
  key: 'jobId',
  placeholder: '{{JOB_ID}}',
  title: 'Job ID',
  defaultValue: '',
  optional: true,
  replacementFormat: '(Job ID: {value})',
});
```

When the user supplies a Job ID, replace `{{JOB_ID}}` with
`(Job ID: USER_INPUT_VALUE)`. Note carefully: **the current format has
no leading hyphen**. Do not reintroduce `- (Job ID: VALUE)` unless the
user explicitly requests it.

When the user leaves the field blank, replace `{{JOB_ID}}` with an empty
string so the marker is removed from the document.

### Optional does not mean unconfigured --- important regression fix

This distinction previously caused a duplicate-field bug:

- **Configured placeholders** includes every entry in
  `PLACEHOLDER_CONFIG.fields`, whether optional or mandatory.
- **Mandatory placeholders** includes only entries where
  `optional === false`.

The latest `getConfiguredPlaceholders_()` correctly returns every
configured token:

```javascript
function getConfiguredPlaceholders_() {
  return PLACEHOLDER_CONFIG.fields.map(field => field.placeholder);
}
```

Mandatory validation is performed separately by filtering
`PLACEHOLDER_CONFIG.fields` for `!field.optional`.

Do not implement configured-token detection by filtering out optional
fields. Doing so causes `{{JOB_ID}}` to be rediscovered and displayed a
second time under `ADDITIONAL PLACEHOLDERS`.

The user previously observed `{{JOB_ID}}` twice in the modal. The root
cause was that the configured-token helper excluded optional fields. The
latest source has corrected this. Preserve it and keep it in regression
tests.

## 7. Additional placeholder discovery

The server scans the active tab's text for tokens matching:

```regex
\{\{[^{}\r\n]+\}\}
```

It treats as additional only tokens that are not present in the full set
of configured placeholders.

For example, `{{HIRING_MANAGER}}` or `{{RECRUITER_NAME}}` may be
rendered under `ADDITIONAL PLACEHOLDERS`.

Behavior:

- The additional section is hidden if there are no additional tokens.
- The field label strips the braces; the hint displays the full token.
- Additional fields start with an empty value.
- Additional fields are optional: empty values replace their token
  markers with an empty string.
- Additional fields must not be confused with configured optional
  fields such as `{{JOB_ID}}`.
- The server rediscovers additional placeholders at substitution time
  and validates submitted data against the latest document content.

## 8. Form and validation behavior

### Required fields

Only these five fields are mandatory:

- `{{DATE}}`
- `{{COMPANY_NAME}}`
- `{{COMPANY_LOCATION}}`
- `{{JOB_TITLE}}`
- `{{JOB_REFERRAL_SOURCE}}`

All five must have non-empty values after trimming.

### Optional fields

- `{{JOB_ID}}` is visibly marked `(Optional)` in the HTML form.
- Any additional discovered placeholder is also visibly marked
  `(Optional)`.
- Optional fields default to an empty string.
- Empty optional values remove the placeholder token from the
  document.
- A non-empty `{{JOB_ID}}` value is wrapped in `(Job ID: VALUE)` with
  no leading hyphen.

The HTML uses the `optional` property supplied by the server to display
the optional marker. Do not hardcode Job ID's optional status in HTML.

### Validation

- Client-side validation checks only mandatory predefined fields.
- Optional predefined fields and additional fields may be empty.
- Server-side validation remains authoritative.
- Current validation rejects replacement values containing another
  double-curly placeholder token (`{{...}}`).
- Recoverable client-side validation messages keep the form visible so
  the user can correct values.

## 9. Unexpected placeholder warnings

After substitution, unresolved or alternative-format token-like strings
are warnings, not substitution errors.

Detection includes: - `{{TOKEN}}` - `{TOKEN}` - `[[TOKEN]]` -
`[TOKEN]` - `<<TOKEN>>` - `<TOKEN>` - `%%TOKEN%%` - `%TOKEN%` -
`$TOKEN$` - `##TOKEN##` - `@TOKEN@`

Alternative patterns are intended for uppercase token-like names
containing letters, digits, and underscores. Collect all detected tokens
together; do not stop after the first match.

The warning scan must use freshly fetched post-substitution content from
`Docs.Documents.get()`, not stale pre-substitution text.

If substitution succeeds but warnings remain: - Return success plus a
warning. - Hide the form, disable buttons, and show only the warning. -
Do not claim that document changes were rolled back.

If substitution succeeds without warnings, close the modal.

## 10. Generated tab-title behavior

Title format:

```text
<COMPANY_NAME> - <SCRIPT_GENERATED_DATE>
```

Example:

```text
Google - 11/Nov/2026
```

Rules:

- Company name comes from `companyName`.
- Date is generated by Apps Script using `Asia/Kolkata` and
  `tabTitleDateFormat: 'dd/MMM/yyyy'`.
- The title date is not taken from the `{{DATE}}` form field.
- Editing the `DATE` input does not affect the tab-title preview.
- Editing the company name updates the preview.
- The server generates the title date again at substitution time. If
  the modal remains open across midnight in the configured time zone,
  preview and final title may differ.
- Existing tab titles are compared case-insensitively.
- Duplicate titles receive suffixes such as `(2)`, `(3)`, appended
  after the date.
- Maximum tab-title length is 100 characters; ellipsis is `...`.
- When the script updates the title, it also clears the tab's dedicated
  Google Docs emoji icon (the separate icon shown beside the tab title).
  This does not strip emoji characters from the title text or document
  body. The operation applies only to the active target tab.
- Current server and HTML preview truncate the complete title and
  append ellipsis if it exceeds the limit. This may truncate
  date/suffix for unusually long company names. Do not change this
  silently; if improving it, preserve the date and suffix and keep
  preview/server algorithms consistent.

### Clearing the tab's dedicated emoji icon

The Google Docs tab has a dedicated emoji/icon property separate from its
title text. The current substitution flow clears that icon as part of the
same tab-properties update that renames the tab:

```javascript
updateDocumentTabProperties: {
  tabProperties: {
    tabId: tabId,
    title: newTitle,
    iconEmoji: '',
  },
  fields: 'title,iconEmoji',
}
```

Important distinctions:

- `iconEmoji: ''` clears the dedicated tab icon.
- `fields: 'title,iconEmoji'` updates both properties in the request.
- The request targets only the active eligible tab via its `tabId`.
- It does not remove emoji characters that are part of the title string
  or document content.
- Do not replace this with title-string emoji sanitization; that would
  change a different feature.

## 11. UI behavior

The modal contains: - Document name - Document title preview -
`PLACEHOLDERS` section - Optional `ADDITIONAL PLACEHOLDERS` section -
Reset and Substitute buttons - Status-message container

Fields are rendered dynamically from server-provided data.

Status rules: - Loading/info message: show only the message and spinner;
hide the form. - Initialization warning/error: show only the message;
hide the form. - Post-substitution warning: show only the warning; hide
the form and disable buttons. - Server-side error: show only the error;
hide the form. - Client-side required-field validation error: keep the
form visible. - Standalone messages have no divider. - Reset restores
predefined defaults, empties additional inputs, clears the message, and
refreshes the title preview. - Optional fields show `(Optional)` beside
the field title; the placeholder hint remains aligned to the right.

## 12. Server-side flow and important helpers

The latest `Code.gs` contains these principal functions:

- `onOpen()` --- creates the custom menu.
- `showAutomationDialog()` --- opens the HTML modal.
- `getConfiguredDefaults_()` --- computes defaults; fields marked
  `defaultValueType: 'today'` use `dateFormat`.
- `getConfiguredPlaceholders_()` --- returns **all** configured
  placeholder tokens, including optional ones.
- `getTabTitleDate_()` --- computes the title date using
  `tabTitleDateFormat`.
- `getInitialData()` --- validates active-tab eligibility, checks
  mandatory tokens only, discovers additional tokens, and returns
  field configuration and defaults.
- `getTabInformation_()` --- rechecks tab and parent information.
- `flattenTabs_()` --- recursively flattens nested tabs.
- `findAdditionalCurlyPlaceholders_()` --- discovers unknown
  double-curly tokens after excluding all configured tokens.
- `getPlaceholderLabel_()` --- strips braces from an additional token
  for its field label.
- `findUnexpectedPlaceholders_()` --- finds unresolved/alternative
  tokens.
- `generateUniqueDocumentTitle_()` and `truncateTabTitle_()` ---
  generate a unique title.
- `validateInputs_()` --- validates required fields and replacement
  values.
- `extractTextFromContent_()` --- extracts text from Docs API content.
- `substitutePlaceholders(tabId, values)` --- revalidates the target
  tab and placeholders, creates tab-scoped replacements, renames the
  tab and clears its dedicated emoji icon, verifies replacement counts,
  refetches updated content, and returns success/warnings.

The latest `Index.html` dynamically renders server-provided fields,
collects values, resets fields, previews the tab title, submits
substitution, handles status messages, shows loading state, and handles
errors. It displays the optional marker based on each field's `optional`
property.

## 13. Latest source snapshot reviewed

The user supplied these files as the latest source versions for this
update:

- `Code (1).gs` --- 741 lines; use as the current `Code.gs` baseline.
- `Index(2).html` --- 781 lines; use as the current `Index.html`
  baseline.

The latest `Code.gs` confirms: - `dateFormat` is `dd MMMM yyyy`. -
`tabTitleDateFormat` is `dd/MMM/yyyy`. - Time zone is `Asia/Kolkata`. -
`{{COMPANY_LOCATION}}` default is `Bengaluru, Karnataka`. -
`{{JOB_TITLE}}` default is `Lead React Native engineer`. -
`{{JOB_REFERRAL_SOURCE}}` default is `LinkedIn`. - `{{JOB_ID}}`
replacement format is `(Job ID: {value})`, without a leading hyphen. -
`{{JOB_ID}}` is optional and defaults to an empty string. - All six
predefined placeholders are returned by
`getConfiguredPlaceholders_()`. - Mandatory placeholder checks
independently filter for `optional: false`. - Additional placeholders
are discovered only if they are not in the complete
configured-placeholder set. - Replacement requests are scoped to the
target tab using `tabsCriteria`. - Tab title uniqueness is checked
case-insensitively. - The post-substitution warning scan fetches updated
document content again.

The latest `Index.html` confirms: - The form is dynamically rendered
from server-provided fields. - Optional markers are displayed using
`field.optional`. - Additional fields are displayed separately and
dynamically. - Only mandatory predefined fields are validated
client-side. - Validation errors can preserve the form; other messages
hide it. - A loading spinner is implemented. - The title preview is
updated when the company-name field changes. - The title preview date is
independent of the `DATE` input. - Successful substitution without
warnings closes the modal. - Successful substitution with warnings
leaves the modal open and displays only the warning.

No further functional changes were identified in the latest HTML beyond
the current dynamic form/UI implementation. The source files, not this
summary, remain authoritative.

## 14. Known implementation considerations

Review these when relevant; do not change them silently:

1.  **Configured optional placeholder regression:** `{{JOB_ID}}` must be
    excluded from additional-field discovery because it is already
    configured. All configured placeholders, including optional ones,
    must be returned by `getConfiguredPlaceholders_()`.
2.  **Title truncation:** Current truncation can truncate the date or
    duplicate suffix for an unusually long company name. Any improvement
    should preserve date and suffix and keep preview/server behavior
    aligned.
3.  **Date boundary:** Preview date is computed at initialization; final
    title date is computed again during substitution.
4.  **Document changes while modal is open:** Additional placeholders
    are rediscovered server-side. If the document changes and a new
    placeholder appears after initialization, the server must fail
    clearly or otherwise handle it deliberately; never silently leave a
    new token untreated.
5.  **Replacement counts:** A replacement-count check occurs after the
    batch update. If it fails after the batch has executed, the document
    may already have changed. Consider this in error handling.
6.  **Text extraction:** Verify recursive extraction for any new Docs
    API content structures.
7.  **Alternative token detection:** `<TOKEN>` can match uppercase
    angle-bracket text; this is intentional current warning behavior.
8.  **Case sensitivity:** Replacement searches use `matchCase: true`;
    preserve unless requested otherwise.
9.  **No automatic GitHub archive/deployment:** The script modifies
    Google Docs; it does not create GitHub commits or deploy itself.

## 15. Testing checklist

### Eligibility and safety

- Root tab is rejected.
- Child tab under a parent other than `Generated` is rejected.
- Direct child of `Generated` is accepted.
- Switching active tab while modal is open causes substitution to be
  rejected.
- Deleted/renamed target or parent is handled safely.
- Replacements affect only the target tab.

### Predefined fields

- Exactly the six predefined fields listed in Section 6 appear under
  `PLACEHOLDERS`.
- `{{JOB_ID}}` appears once only, under `PLACEHOLDERS`.
- `{{JOB_ID}}` never appears under `ADDITIONAL PLACEHOLDERS`.
- Only the first five fields are mandatory.
- `{{COMPANY_LOCATION}}` defaults to `Bengaluru, Karnataka`.
- `{{JOB_TITLE}}` defaults to `Lead React Native engineer`.
- `{{JOB_REFERRAL_SOURCE}}` defaults to `LinkedIn`.
- `{{DATE}}` default displays a date like `11 November 2026`.
- `tabTitleDateFormat` remains `dd/MMM/yyyy`.

### Optional behavior

- Job ID input displays `(Optional)`.
- Blank Job ID removes `{{JOB_ID}}` from the document.
- Supplied Job ID becomes `(Job ID: VALUE)` with no leading hyphen.
- Empty additional placeholder values remove their markers.
- Additional fields display `(Optional)` and are not required.

### Additional placeholders and warnings

- Unknown double-curly placeholders appear in the additional section.
- Configured tokens, including optional tokens, are not rediscovered
  as additional.
- Multiple additional placeholders are handled.
- Unresolved and alternative-format tokens are collected in one
  warning.
- Warning scan uses freshly fetched post-substitution content.

### UI

- Loading spinner appears during initialization and substitution.
- Standalone messages hide the form.
- Required-field validation keeps the form visible.
- Reset restores defaults and clears additional inputs.
- Successful substitution without warnings closes the modal.
- Successful substitution with warnings shows only the warning and
  disables buttons.

### Tab title and dedicated tab emoji

- Company name appears before the script-generated date.
- Title date uses `dd/MMM/yyyy`, independently of the `{{DATE}}`
  field.
- Changing `DATE` does not affect title preview.
- Changing company name updates title preview.
- Duplicate title suffixes are generated and compared
  case-insensitively.
- Maximum length is respected.
- If the target tab has a dedicated emoji icon assigned, substitution
  clears it when updating the tab title.
- If the target tab has no emoji icon assigned, the title update still
  succeeds and leaves the tab with the default/no custom emoji state.
- Emoji characters in the title text and document content are not
  removed.
- Other tabs' emoji icons are not modified.

## 16. Instructions to the next LLM

You are continuing an existing project, not starting from scratch.

1.  Read this document in full.
2.  Read the latest `Code.gs` and `Index.html` before making code
    changes.
3.  Treat the source files as the exact implementation baseline.
4.  Preserve the requirements and preferences documented here.
5.  Return the complete contents of every modified file, ready to copy
    and paste.
6.  Do not return partial snippets or diffs unless explicitly requested.
7.  State which files changed and which did not.
8.  Preserve `dateFormat: 'dd MMMM yyyy'` and
    `tabTitleDateFormat: 'dd/MMM/yyyy'` as separate settings.
9.  Preserve the five mandatory fields and the optional `{{JOB_ID}}`
    behavior.
10. Preserve the exact Job ID replacement format `(Job ID: VALUE)`
    without a leading hyphen.
11. Ensure optional configured placeholders are included in
    configured-token detection so they are not duplicated in the
    additional section.
12. Keep active-tab safeguards and tab-scoped replacement intact.
13. Preserve the tab-emoji behavior: when updating the target tab title,
    also clear its dedicated tab icon by setting `iconEmoji: ''` and
    including `iconEmoji` in the `updateDocumentTabProperties` field mask.
    Do not remove emoji characters from the title text or document body,
    and do not modify other tabs' icons.
14. Do not claim Apps Script environment testing unless actually
    performed.
