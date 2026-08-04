---
name: gdrive-pdf-extract
description: Extract text from view-only, copy-protected PDFs in Google Drive using Claude in Chrome MCP, by jumping the virtualized viewer to force lazy-loaded pages to render. Use when you need to read PDFs shared on Google Drive where download and copy are disabled.
---

# gdrive-pdf-extract

Extract text from view-only PDFs in Google Drive using Claude in Chrome MCP.
Use when you need to read PDFs shared on Google Drive where download/copy is disabled.

## When to use

- User shares a Google Drive folder URL with restricted PDFs
- PDFs are view-only (no download, no copy)
- Need to feed content to Claude for processing (CLAUDE.md, docs, skills, etc.)

## Prerequisites

- Chrome must be open with the Claude in Chrome extension active
- User must be logged into the correct Google account in that Chrome instance
- The Drive folder/file must already be accessible (viewer permission at minimum)

## Workflow

### 1. Connect to Chrome

list_connected_browsers, then select_browser (pick the local one).

### 2. Navigate to the Drive folder

navigate to the Google Drive folder URL.
Take a screenshot to confirm folder contents are visible.

### 3. For each PDF file

Open it by double-clicking its icon (computer tool, double_click action).
Wait for the PDF viewer to load (check page counter in top bar shows "X / N").

### 4. Extract all pages (virtualized viewer workaround)

The Google Drive PDF viewer lazy-loads pages. Only nearby pages are in the DOM.
Strategy: jump to specific pages to force rendering, then call get_page_text each time.

Step A: get pages 1-5 (already loaded on open)
  get_page_text, save result.

Step B: navigate to middle of document
  triple_click page-number field (coords approx 44, 39)
  type the middle page number
  press Return, wait 1 second
  get_page_text, save result.

Step C: navigate to last page using End key
  left_click on PDF content area to give keyboard focus
  press End key, wait 2 seconds
  get_page_text, save result (gets last pages plus previously cached pages).

Step D: fill remaining gaps for long docs (20+ pages)
  Also jump to pages at roughly 25%, 50%, 75% of total page count.
  Each time: triple_click field, type page number, Return, get_page_text.

### 5. Deduplicate and combine

Each get_page_text call returns "Page X of N" markers.
Combine all calls, deduplicate by page number, sort in order, write to output file.

### 6. Navigate back for next file

Navigate back to the Drive folder URL, then open the next PDF.

## Output

Save combined text to a permanent location (NOT scratchpad, it is session-specific):
- Personal docs: a scratch/temp directory of your choosing (e.g. `C:\temp\descriptive_name.md` on Windows, `/tmp/descriptive_name.md` on macOS/Linux)
- Project docs: inside the repo under docs/ or Docs/

## Key gotchas

- get_page_text reads the DOM, not a screenshot. It works even with copy-protection.
- Page navigation field needs triple-click to select before typing a new number.
- Keyboard focus must be in the PDF content area (not the sidebar) for End/Home to work.
- The viewer renders about 5 pages around current position. Overlap your page jumps.
- Tables and code blocks come out as plain text with no markdown formatting.
- Images in the PDF are skipped. Text only.

## Example

User says: "Can you read those PDFs from the Drive link and save them?"
Run this skill with the provided URL.
Save output to a scratch file, e.g. C:\temp\project_docs_extracted.md
Report back how many files and pages were captured.
