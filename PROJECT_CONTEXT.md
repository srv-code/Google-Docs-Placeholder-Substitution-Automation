# Google Docs Placeholder Substitution Automation
## Project Context, Requirements, Architecture & LLM Handoff

**Document purpose:** This is the project handoff brief for continuing development in a new ChatGPT, Claude, or other LLM conversation. Upload this document together with the latest `Code.gs` and `Index.html`.

**Source of truth:** The uploaded `Code.gs` and `Index.html` are the authoritative implementation snapshot. This brief records the intended behavior, user decisions, constraints, and known implementation details. If a discrepancy exists, do not silently rewrite the code to match this document. Identify the discrepancy and ask or explain before changing behavior.

---

## 1. Project overview

This is a Google Apps Script automation for Google Docs. It opens a custom modal form, dynamically discovers placeholder tokens in the currently active Google Docs tab, collects replacement values, replaces those tokens in that tab, and renames that tab using a generated title.

The project consists of two files:

- `Code.gs` — server-side Apps Script logic, configuration, Google Docs API operations, validation, placeholder discovery, replacement, and tab renaming.
- `Index.html` — client-side modal UI, dynamic form rendering, input collection, validation, status messages, loading state, and document-title preview.

The Apps Script project uses:
- `DocumentApp`
- `HtmlService`
- `Utilities`
- The **Advanced Google Docs API service**, accessed as `Docs.Documents.get()` and `Docs.Documents.batchUpdate()`.

The current implementation assumes the Advanced Google Docs API service is enabled in the Apps Script project.

## 2. Important development instructions and user preferences

These preferences are important and should be followed in every future development turn:

1. **When code changes are requested, provide the complete contents of every modified file.** Do not provide fragmented snippets, isolated functions, diffs, or instructions such as “replace this section.” The user wants copy-paste-ready full files.
2. Clearly identify which files have changed and which have not changed.
3. Keep explanations concise, but include enough context to explain meaningful behavior changes, risks, or assumptions.
4. Do not silently remove existing functionality or alter established behavior while implementing an unrelated request.
5. Preserve the current project architecture unless a change is justified:
   - central configuration in `Code.gs`;
   - dynamic UI generation in `Index.html`;
   - server-side revalidation before making document changes.
6. Do not hardcode the predefined placeholder list independently in `Index.html`. The server-side central configuration must remain the source of truth.
7. If a requested change has an ambiguity that materially affects behavior, ask one focused clarification rather than guessing.
8. The user is a software engineer and is comfortable with technical explanations.
9. For future code review, inspect the uploaded current files rather than assuming an earlier chat snapshot is still current.

## 3. Current UI and menu behavior

### Google Docs custom menu

The custom menu is configured in `PLACEHOLDER_CONFIG.settings`:

- Menu title: `Automation`
- Menu item: `Substitute Values`
- Menu handler: `showAutomationDialog`

The modal dialog:
- HTML file: `Index`
- Dialog title: `Value Substitution`
- Width: `560`
- Height: `760`

### Modal initialization

When the modal loads:
1. It displays `Loading document information...` with a spinner.
2. It calls the server-side `getInitialData()`.
3. The server identifies the active tab, checks its parent, reads its content, validates required predefined placeholders, and discovers additional double-curly placeholders.
4. If eligible, the server returns the data and configuration required by the HTML form.
5. If ineligible or initialization fails, the form remains hidden and only a warning/error message is shown.

### Form layout

The form includes:
- Document name
- Document title preview
- A predefined placeholder section titled `PLACEHOLDERS`
- An optional additional placeholder section titled `ADDITIONAL PLACEHOLDERS`
- `Reset` and `Substitute` buttons

Fields are dynamically rendered from data returned by `getInitialData()`.

### Status messages and form visibility

The intended UI behavior is:

- Loading/informational message: show only the message; hide the form.
- Initialization warning: show only the warning; hide the form.
- Initialization error: show only the error; hide the form.
- Post-substitution warning: show only the warning; hide the form and disable buttons.
- Server-side substitution error: show only the error; hide the form.
- **Exception — client-side form validation errors:** keep the form visible so the user can correct missing inputs; show the validation message below/alongside the form.
- Standalone messages do not have a divider.
- The loading spinner is shown during initialization and substitution processing.

The current HTML implements this using `showMessage(message, type, loading = false, preserveForm = false)`. The `preserveForm` argument is used for recoverable form-validation errors.

### Reset behavior

- Predefined fields reset to their configured defaults returned by the server.
- Additional placeholder fields reset to empty strings.
- The document-title preview is refreshed after reset.
- Reset does not change the generated title date.

## 4. Active-tab targeting and safety rules

This is a critical functional constraint.

### Only the active child tab under `Generated` may be modified

- The target is the currently active Google Docs tab when the modal is initialized.
- The active tab must be a **direct child** of a tab titled exactly `Generated`.
- A root tab is not eligible.
- A tab under any other parent is not eligible.
- The server captures the active tab ID and returns it to the HTML as `tabId`.
- The HTML stores it as `ACTIVE_TAB_ID`.
- On substitution, the client sends that captured tab ID to `substitutePlaceholders()`.
- The server checks that the current active tab still has that ID.
- The server also re-fetches the document and revalidates that the target tab still exists and that its direct parent is still `Generated`.

If the active tab changes after the modal opens, substitution must be rejected with a clear message instructing the user to close the form and reopen it from the intended tab.

### Do not modify other tabs

All placeholder replacement requests must be scoped to the target tab using the Google Docs API `tabsCriteria` with the captured `tabId`.

Do not broaden replacement to the entire document or other tabs.

## 5. Central configuration in `Code.gs`

The central configuration object is named `PLACEHOLDER_CONFIG` and is frozen using `Object.freeze()`.

The current uploaded `Code.gs` defines these settings:

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
  dateFormat: 'dd/MMM/yyyy',
  tabTitleDateFormat: 'dd/MMM/yyyy',
  maxTabTitleLength: 100,
  titleEllipsis: '...',
  additionalSectionTitle: 'ADDITIONAL PLACEHOLDERS',
  predefinedSectionTitle: 'PLACEHOLDERS'
}
```

**Important date-format decision:** The user explicitly changed the tab-title date format to:

```javascript
tabTitleDateFormat: 'dd/MMM/yyyy',
```

Preserve this exact format in future changes unless the user explicitly changes it again. It produces a date such as `07/Nov/2026` (depending on the date).

The regular `dateFormat` is also currently `dd/MMM/yyyy`. It is used for the default value of the `{{DATE}}` form field. `tabTitleDateFormat` is used separately for the generated tab title date. Do not conflate these settings.

### Document title configuration

```javascript
documentTitle: {
  companyFieldKey: 'companyName',
  separator: ' - '
}
```

The company name comes from the `companyName` field, and the separator between company name and date is ` - `.

### Predefined placeholder fields

The current configuration contains these fields:

| Key | Placeholder | Display title | Default |
|---|---|---|---|
| `date` | `{{DATE}}` | Date | Generated using `Utilities.formatDate()` with `dateFormat` and `Asia/Kolkata` |
| `companyName` | `{{COMPANY_NAME}}` | Company Name | `your company` |
| `companyCity` | `{{COMPANY_CITY}}` | Company City | `Bengaluru, Karnataka` |
| `jobTitle` | `{{JOB_TITLE}}` | Job Title | `Senior React Native Engineer` |
| `jobReferralSource` | `{{JOB_REFERRAL_SOURCE}}` | Job Referral Source | `Linkedin` |
| `keyRequirement1` | `{{KEY_REQUIREMENT_1}}` | Key Requirement 1 | `React Native` |
| `keyRequirement2` | `{{KEY_REQUIREMENT_2}}` | Key Requirement 2 | `Redux` |
| `keyRequirement3` | `{{KEY_REQUIREMENT_3}}` | Key Requirement 3 | `JavaScript/TypeScript` |

The `DATE` default uses `defaultValueType: 'today'`; the server computes it at initialization. Other fields use their configured `defaultValue`.

### Configuration principle

When adding, removing, or renaming a predefined placeholder:
- Change `PLACEHOLDER_CONFIG.fields` in `Code.gs`.
- Keep the HTML form fully data-driven.
- Ensure server-side validation and replacement continue to derive from the same configuration.
- Do not duplicate the placeholder definitions in the HTML.

## 6. Additional placeholder discovery

The server scans the active tab's text for double-curly tokens matching:

```regex
\{\{[^{}\r\n]+\}\}
```

It excludes tokens already defined in `PLACEHOLDER_CONFIG.fields`.

Every other matching token is treated as an additional placeholder.

For example, if the tab contains:
- `{{HIRING_MANAGER}}`
- `{{RECRUITER_NAME}}`

then the modal dynamically creates two additional inputs.

Additional-field behavior:
- The section appears only if at least one additional placeholder exists.
- The displayed field label strips the braces (e.g. `HIRING_MANAGER`).
- The token hint retains the complete placeholder (e.g. `{{HIRING_MANAGER}}`).
- Additional fields start empty.
- All additional fields are required before substitution.
- Additional placeholder values are submitted along with predefined values.
- The server independently rediscovers additional placeholders from the latest tab content at substitution time and validates the submitted values against that fresh discovery.

This server-side rediscovery is intentional. It prevents relying only on stale client-side information if the document changes after the modal opens.

## 7. Placeholder validation and warning behavior

### Required predefined placeholders

Before displaying the form, the active tab must contain every configured predefined placeholder. If any are missing:
- Do not display the form.
- Show one warning listing all missing predefined tokens.

### Input validation

Before substitution:
- Every predefined input must have a non-empty value after trimming.
- Every additional input must have a non-empty value after trimming.
- Current server validation rejects replacement values containing another double-curly placeholder token (`{{...}}`).
- Client-side validation keeps the form visible so the user can fix empty fields.
- Server-side validation remains authoritative.

### Unresolved or alternative-format tokens after substitution

Unexpected tokens are warnings, not substitution errors.

The current server-side detector checks:
- Any remaining `{{...}}` token.
- `{TOKEN}`
- `[[TOKEN]]`
- `[TOKEN]`
- `<<TOKEN>>`
- `<TOKEN>`
- `%%TOKEN%%`
- `%TOKEN%`
- `$TOKEN$`
- `##TOKEN##`
- `@TOKEN@`

The alternative patterns are intended for uppercase token-like names containing letters, digits, and underscores.

All detected token strings are collected into a set and returned together in one warning message. The warning should not stop the scan after the first token or first pattern.

After replacement, the server fetches the document again using `Docs.Documents.get()` and checks the updated tab content. This is important: warning detection must use freshly fetched post-substitution content, not the original text or a stale in-memory string.

### Success with warning

If all requested substitutions succeed but token-like content remains:
- The server returns `success: true` and a warning.
- The HTML hides the form, disables buttons, and displays the warning.
- The document changes are not rolled back.

If substitution succeeds and there are no warnings, the modal closes.

## 8. Generated tab-title behavior

The generated tab title has this structure:

```text
<COMPANY_NAME> - <SCRIPT_GENERATED_DATE>
```

For example, with a company value of `Google` and the configured date format, it could be:

```text
Google - 07/Nov/2026
```

If the same title already exists, append a numeric suffix after the date:

```text
Google - 07/Nov/2026 (2)
Google - 07/Nov/2026 (3)
```

Rules:
- Company name appears before the date.
- Date is generated by Apps Script using `Utilities.formatDate()` and `Asia/Kolkata`.
- The title date is **not** taken from the `DATE` form field.
- The `DATE` field remains an ordinary placeholder replacement value for `{{DATE}}` in document content.
- The modal receives a title-date preview from `getTabTitleDate_()`.
- Editing the `DATE` input does not update the title preview.
- Editing the company-name input updates the title preview.
- The server generates the title date again at substitution time, so the final title uses the script's current date at execution time. If the modal stays open across midnight in the configured time zone, the preview could differ from the final title.
- Existing tab titles are compared case-insensitively when determining uniqueness.
- The configured maximum tab-title length is 100 characters.
- The configured ellipsis is `...`.

**Current implementation detail to be aware of:** Both the HTML preview and server-side `truncateTabTitle_()` currently truncate the complete generated title from the right and append `...` when it exceeds the maximum. This can truncate part or all of the date/suffix when the company name is extremely long. Do not change this silently; if improving it, a sensible approach is to preserve the date and duplicate suffix and truncate only the company-name portion, while respecting the 100-character maximum. Keep the preview and server algorithm consistent.

## 9. Current server-side execution flow

The current `Code.gs` follows this general flow:

### `onOpen()`
Creates the `Automation` menu and its `Substitute Values` item.

### `showAutomationDialog()`
Loads `Index.html` and opens the modal using configured dimensions and title.

### Configuration helpers
- `getConfiguredDefaults_()` — builds the defaults object; computes today's date for fields marked `defaultValueType: 'today'`.
- `getConfiguredPlaceholders_()` — returns configured placeholder strings.
- `getTabTitleDate_()` — generates the tab-title date using the configured time zone and `tabTitleDateFormat`.

### `getInitialData()`
- Gets active document and active tab.
- Fetches all tabs through `Docs.Documents.get(..., {includeTabsContent: true})`.
- Flattens nested tabs using `flattenTabs_()`.
- Finds active tab by ID.
- Checks that its direct parent title is `Generated`.
- Extracts text from the active tab body.
- Checks every predefined placeholder exists.
- Discovers additional double-curly placeholders.
- Creates dynamic field definitions and defaults.
- Collects other tab titles.
- Generates the title preview date and initial title.
- Returns the data required by the HTML.
- Returns a warning or error state without showing the form when initialization cannot proceed.

### Tab helpers
- `getTabInformation_(documentId, tabId)` — re-fetches tabs and determines parent and eligibility.
- `flattenTabs_(tabs, result = [])` — recursively flattens nested tabs.

### Placeholder helpers
- `findAdditionalCurlyPlaceholders_(documentText)`
- `getPlaceholderLabel_(placeholder)`
- `findUnexpectedPlaceholders_(documentText)`

### Title helpers
- `generateUniqueDocumentTitle_(values, existingTitles, tabTitleDate)`
- `truncateTabTitle_(title, maxLength, ellipsis)`

### Validation and text extraction
- `validateInputs_(values, additionalFields)`
- `extractTextFromContent_(content)` — recursively extracts text-run content from document structures.

### `substitutePlaceholders(tabId, values)`
- Checks that the active tab ID still matches the ID captured at initialization.
- Fetches latest document state.
- Confirms target tab still exists.
- Revalidates that the target is a direct child of `Generated`.
- Extracts latest tab text.
- Rechecks all configured predefined placeholders.
- Rediscovers additional placeholders.
- Validates submitted values.
- Builds replacement operations for predefined and additional placeholders.
- Generates a fresh script date and a unique tab title.
- Builds `replaceAllText` requests scoped to `tabId`.
- Adds `updateDocumentTabProperties` request to rename the tab.
- Executes all requests using `Docs.Documents.batchUpdate()`.
- Checks replacement occurrence counts.
- Fetches the document again and scans the updated target tab for unresolved tokens.
- Returns success and optional warning.

## 10. Current HTML implementation details

`Index.html` is a single HTML file containing:
- CSS styles for the modal.
- A hidden form container (`#formContent`).
- Document metadata rows.
- Dynamic predefined and additional field containers.
- Reset and Substitute buttons.
- A message container (`#message`).
- Client-side JavaScript.

Important client-side state variables:
- `FIELD_CONFIG`
- `ADDITIONAL_FIELDS`
- `DEFAULTS`
- `TITLE_CONFIG`
- `EXISTING_TAB_TITLES`
- `ACTIVE_TAB_ID`
- `IS_ELIGIBLE`
- `SECTION_SETTINGS`
- `TAB_TITLE_DATE`

Important functions:
- `initialize()`
- `handleInitialization(data)`
- `handleInitializationFailure(error)`
- `createFieldElement(field, index, section)`
- `renderPredefinedFields()`
- `renderAdditionalFields()`
- `collectFieldValues()`
- `resetForm()`
- `generateDocumentTitle()`
- `updateDocumentTitle()`
- `substitute()`
- `handleSubstitutionSuccess(result)`
- `showMessage(message, type, loading = false, preserveForm = false)`
- `clearMessage()`
- `enableButtons()`
- `disableButtons()`
- `setLoading(loading)`
- `handleFailure(error)`

The title preview uses the company-name field and `TAB_TITLE_DATE`. The title date is received from the server and is independent of the `DATE` input.

## 11. Current code snapshot facts

The files accompanying this handoff were uploaded as the latest versions at the time this document was generated:

- `Code.gs` — 729 lines.
- `Index.html` — 779 lines.

The latest uploaded `Code.gs` confirms:
- `PLACEHOLDER_CONFIG` is the central configuration object.
- `dateFormat` is `dd/MMM/yyyy`.
- `tabTitleDateFormat` is `dd/MMM/yyyy`.
- `timeZone` is `Asia/Kolkata`.
- Parent tab requirement is `Generated`.
- The configured predefined placeholders are the eight fields listed above.
- Additional placeholder discovery and unresolved-token warning detection are implemented server-side.
- Substitution is scoped to the active target tab using `tabsCriteria`.
- Tab title uniqueness is checked case-insensitively.
- The title is updated using `updateDocumentTabProperties`.
- The post-substitution warning scan fetches the updated document again.

The latest uploaded `Index.html` confirms:
- The form is dynamically rendered from server-provided field data.
- Additional placeholders are displayed separately.
- Validation errors can preserve the form.
- Other messages hide the form.
- A loading spinner is implemented.
- The title preview is updated when the company-name field changes.
- The title preview date is independent of the `DATE` input.
- Successful substitution without warnings closes the modal.
- Successful substitution with warnings leaves the modal open and displays only the warning.

## 12. Known implementation considerations for future work

These are not necessarily bugs; review them when relevant to a requested change.

1. **Title truncation:** Current truncation can truncate the date or duplicate suffix for an unusually long company name. A future enhancement could truncate only the company name, preserving the date and suffix.
2. **Date boundary:** Preview date is computed during initialization; final date is computed again during substitution. This is intentional, but can produce a difference if execution crosses midnight in `Asia/Kolkata`.
3. **Document changes while modal is open:** The server rediscovers placeholders and revalidates the tab, but the HTML may have an outdated list of additional fields if the user edits the document while the modal is open. The server should fail clearly if a newly discovered placeholder has no submitted value.
4. **Replacement counts:** The current code checks that each replacement request changed at least one occurrence. If a request fails this check after the batch update has already executed, the document may already have been modified. Consider this carefully if redesigning transactional/error handling.
5. **Text extraction:** `extractTextFromContent_()` recursively traverses document structures and collects `textRun.content`. If future content types or tab structures are added, verify that extraction is complete and does not accidentally include irrelevant nested data.
6. **Alternative token detection:** The angle-bracket patterns can flag uppercase angle-bracket text such as `<TOKEN>`. This is part of the current requested warning behavior; do not remove it without discussion.
7. **Case sensitivity:** Replacement searches use `matchCase: true`; preserve this unless the user requests case-insensitive replacement.
8. **No automatic save/archive behavior:** The current scripts modify the Google Docs tab. They do not independently create GitHub commits, upload source files, or archive project history.

## 13. Testing checklist for future changes

Before considering a change complete, test the relevant cases in a copy of a Google Doc:

### Eligibility
- Active root tab — form hidden, warning displayed.
- Active child of a parent other than `Generated` — form hidden, warning displayed.
- Active direct child of `Generated` — eligible.
- Switch active tab after opening modal — substitution rejected.
- Delete or rename parent/target tab while modal is open — substitution rejected safely.

### Required placeholders
- All configured placeholders present — form loads.
- One configured placeholder missing — form hidden and all missing tokens listed.
- Multiple configured placeholders missing — all listed together.

### Additional placeholders
- No additional placeholders — additional section hidden.
- One additional placeholder — one input displayed.
- Multiple additional placeholders — all displayed.
- Additional fields start empty and are required.
- Additional values are substituted correctly.
- A new additional placeholder added after modal initialization — server-side validation fails clearly rather than silently leaving it untreated.

### Replacement and warnings
- All expected placeholders replaced.
- Replacement values containing `{{OTHER_TOKEN}}` rejected.
- Unresolved `{{TOKEN}}` detected after substitution.
- Alternative token forms detected.
- Multiple token patterns collected in a single warning.
- Warning scan uses freshly fetched updated document content.
- Successful substitution with warning returns success plus warning.
- Successful substitution without warning closes the modal.

### UI
- Spinner appears during initialization.
- Spinner appears during substitution.
- Initialization messages hide the form.
- Validation errors preserve the form.
- Server errors hide the form.
- Reset restores defaults and clears additional inputs.
- Buttons are disabled while processing and after post-substitution warning.

### Tab title
- Company name appears before date.
- Title date uses `dd/MMM/yyyy` in `Asia/Kolkata`.
- Changing `DATE` input does not change title preview.
- Changing company name updates title preview.
- Duplicate titles receive `(2)`, `(3)`, etc.
- Duplicate comparison is case-insensitive.
- Maximum length is respected.
- Final title uses server-generated execution date, not the form's DATE value.

## 14. Instructions to the next LLM

You are continuing an existing Google Apps Script project, not starting a new one.

1. Read this handoff document in full.
2. Read both uploaded source files (`Code.gs` and `Index.html`) before making changes.
3. Treat the uploaded files as the current implementation baseline.
4. Preserve all established requirements and preferences above.
5. If the requested task requires code changes, return the **entire contents** of each modified file, ready to copy and paste.
6. Do not return partial code fragments or diffs unless the user explicitly asks for them.
7. State clearly which files changed.
8. Do not change `tabTitleDateFormat: 'dd/MMM/yyyy'` unless explicitly requested.
9. Keep the server-side validation and active-tab safeguards intact.
10. Do not claim that code has been tested in the user's Google Apps Script environment unless it actually has been.
