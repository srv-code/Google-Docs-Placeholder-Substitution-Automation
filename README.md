# Google Docs Placeholder Substitution Automation

A Google Apps Script project that dynamically substitutes predefined and
custom placeholders in an eligible Google Docs tab.

## Repository contents

- `Code.gs` --- Server-side configuration, tab eligibility checks,
  validation, placeholder discovery and replacement, and tab-title
  generation.
- `Index.html` --- Dynamic modal form for entering placeholder values
  and viewing status messages.
- `PROJECT_CONTEXT.md` --- Detailed project requirements, current
  configuration, implementation decisions, known considerations, and
  LLM handoff instructions.

## Key features

- Adds an **Automation → Substitute Values** menu to Google Docs.
- Modifies only the active tab when it is a direct child of the
  `Generated` tab.
- Builds predefined fields dynamically from the central
  `PLACEHOLDER_CONFIG` in `Code.gs`.
- Supports required and optional predefined fields.
- Discovers unknown `{{PLACEHOLDER}}` tokens and displays them in a
  separate additional-fields section.
- Replaces placeholders only in the eligible target tab.
- Generates a unique tab title in the format
  `<COMPANY_NAME> - <SCRIPT_GENERATED_DATE>`.
- Clears the active tab's dedicated Google Docs emoji icon when the tab
  title is updated. This affects the tab icon, not emoji characters in
  the title or document content.
- Reports unresolved or unexpected placeholder-like tokens as
  warnings.

## Predefined placeholders

---

Placeholder Required? Default / behavior

---

`{{DATE}}` Yes Current date in `dd MMMM yyyy`
format
(e.g. `11 November 2026`)

`{{COMPANY_NAME}}` Yes `your company`

`{{COMPANY_LOCATION}}` Yes `Bengaluru, Karnataka`

`{{JOB_TITLE}}` Yes `Lead React Native engineer`

`{{JOB_REFERRAL_SOURCE}}` Yes `LinkedIn`

`{{JOB_ID}}` No Empty by default; when
supplied, replaced as
`(Job ID: VALUE)`

---

The first five placeholders are the only mandatory fields. `{{JOB_ID}}`
is the only optional predefined field. It must appear only once in the
predefined fields section, not again as an additional placeholder.

Additional placeholders discovered in the document are also optional. If
left blank, their placeholder markers are removed.

## Document tab emoji

When substitution successfully updates the eligible tab's title, the
script also clears that tab's dedicated Google Docs tab emoji by setting
its `iconEmoji` property to an empty string in the same
`updateDocumentTabProperties` request. This is separate from the tab title:
it does not remove emoji characters from the title or from document content.
Only the active target tab is affected; other tabs are left unchanged.

## Date formats

- `dateFormat: 'dd MMMM yyyy'` controls the `{{DATE}}` placeholder.
- `tabTitleDateFormat: 'dd/MMM/yyyy'` independently controls the
  generated tab-title date.
- Both use the `Asia/Kolkata` time zone.

## Setup

1.  Create or open a Google Apps Script project bound to the target
    Google Doc.
2.  Add `Code.gs` and `Index.html` to the project.
3.  Enable the **Google Docs API** advanced service in Apps Script.
4.  Ensure the document contains a parent tab named `Generated` and that
    the intended working tab is its direct child.
5.  Save and reload the Google Doc.
6.  Open the intended child tab and select **Automation → Substitute
    Values**.

## Development notes

Read `PROJECT_CONTEXT.md` before making changes. Keep
`PLACEHOLDER_CONFIG` in `Code.gs` as the single source of truth;
`Index.html` must remain data-driven.

Optional fields must still be included in the configured-placeholder
set. Optional means that the user is not required to enter a value; it
does not mean that the token is unknown. This distinction prevents
`{{JOB_ID}}` from appearing in both the predefined and additional
sections.

This repository is intended for source and project-context archiving. It
does not currently include an automated deployment or test workflow.
