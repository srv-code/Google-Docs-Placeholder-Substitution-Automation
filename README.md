# Google Docs Placeholder Substitution Automation

A Google Apps Script project that helps populate predefined and custom
placeholders in a selected Google Docs tab.

## Repository contents

-   `Code.gs` --- Server-side Apps Script logic, configuration,
    validation, placeholder replacement, and tab-title generation.
-   `Index.html` --- Modal interface for entering replacement values and
    reviewing status messages.
-   `PROJECT_CONTEXT.md` --- Project requirements, design decisions,
    development preferences, known considerations, and testing
    checklist.

## Key behavior

-   Adds an **Automation → Substitute Values** menu to Google Docs.
-   Operates only on the active tab when it is a direct child of the
    `Generated` tab.
-   Builds the form dynamically from the central configuration in
    `Code.gs`.
-   Detects additional `{{PLACEHOLDER}}` tokens and provides inputs for
    them.
-   Replaces text only in the eligible target tab.
-   Generates a unique tab title in the format
    `<COMPANY_NAME> - <SCRIPT_GENERATED_DATE>`.
-   Uses `Asia/Kolkata` and the configured `dd/MMM/yyyy` date format for
    the tab-title date.
-   Reports unresolved or unexpected placeholder-like tokens as
    warnings.

## Setup

1.  Create or open a Google Apps Script project bound to the target
    Google Doc.
2.  Add the contents of `Code.gs` and `Index.html` to the project, using
    the same filenames.
3.  Enable the **Google Docs API** advanced service in Apps Script.
4.  Ensure the document has a parent tab named `Generated` and that the
    intended working tab is its direct child.
5.  Save and reload the Google Doc. Use **Automation → Substitute
    Values**.

## Development notes

Read `PROJECT_CONTEXT.md` before making changes. It documents the
intended behavior and important implementation constraints.

Keep `PLACEHOLDER_CONFIG` in `Code.gs` as the central source of truth
for predefined fields and settings. The HTML form is designed to render
from server-provided configuration rather than maintaining a separate
hardcoded placeholder list.

This repository is an archive of the project source and its development
context. No automated deployment or testing workflow is included.
