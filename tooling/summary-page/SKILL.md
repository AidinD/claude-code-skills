---
name: summary-page
description: Render the end-of-task summary as a polished local HTML page (sections, status badges, code examples, review checklist, optional diagrams/mocks) instead of a wall of chat text. Use automatically right after finishing a medium-to-large task, or whenever the user asks to see the summary visually / "visa som html".
---

# Summary Page

Turns the end-of-task summary into a self-contained HTML file, overwritten in place each time, and opens it in the default browser. No files are added to the project - the page lives outside any git repo, so there is nothing to clean up or gitignore.

## When to use

- Automatically, right after wrapping up a task big enough to have a real "Changed / Remaining / Decisions" summary (a handful of files or more, a multi-step feature, a bug fix with verification). Skip it for trivial one-line answers or pure Q&A.
- On request: "visa som html", "gör en summary-page", `/summary-page`.

This is *in addition to* the normal chat summary, not a replacement - always give the short chat summary too (one line on the outcome, then Changed / Remaining / Decisions needed), then also produce the page.

## Where the file goes

Always the same fixed path, overwritten every time - never create a new filename per run:

- Windows: `%TEMP%\claude-summary.html`

Fixed path outside the project directory is the point: it can never end up in git, so there's no cleanup step and no `.gitignore` entry to maintain.

## Steps

1. Build the HTML from the template below. Fill in only the sections that apply - omit empty ones entirely rather than showing "none".
2. Write it to the fixed path with the Write tool.
3. Open it: `Start-Process "$env:TEMP\claude-summary.html"` (PowerShell) - this launches the OS-default browser.
4. Still give the normal short chat summary in the same reply. The page is a visual companion, not a replacement.

## Content rules

- Same content discipline as the chat summary: what changed, what it means, what's left, what needs a decision. No re-narration of steps already visible in commits/diffs.
- Never introduce unexplained jargon/error codes/acronyms without a plain-language gloss (same rule as the chat channel).
- Status badges: green = done & verified, yellow = done but unverified / needs a look, red = blocked / needs a decision.
- Code examples: only when a snippet is genuinely the fastest way to show what changed (e.g. a config value, a command to run) - not full diffs, those belong in git.
- Review checklist: concrete, checkable items ("Open the app, confirm X shows Y") - not vague "review the changes".
- Diagrams/mocks: prefer a real screenshot or mock file already on disk (`<img src="file:///...">`) over inventing a diagram. For simple proportional data (e.g. before/after counts), an inline SVG bar is fine; don't reach for a charting library.

## HTML template

Self-contained: no external CSS/JS/fonts (the file is opened via `file://`, so no network calls will resolve). Copy this structure and fill in the placeholders; delete any `<section>` whose content would be empty.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Summary - {{TASK_TITLE}}</title>
<style>
  :root {
    --bg: #12141a; --panel: #1b1e27; --text: #e6e8ef; --muted: #9aa0b0;
    --green: #3ddc84; --yellow: #f4c95d; --red: #ff6b6b; --accent: #7aa2ff;
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--bg); color: var(--text); font: 15px/1.5 -apple-system, Segoe UI, Roboto, sans-serif; padding: 32px; }
  .wrap { max-width: 780px; margin: 0 auto; }
  h1 { font-size: 22px; margin-bottom: 4px; }
  .meta { color: var(--muted); font-size: 13px; margin-bottom: 28px; }
  section { background: var(--panel); border-radius: 10px; padding: 18px 22px; margin-bottom: 18px; }
  section h2 { font-size: 14px; text-transform: uppercase; letter-spacing: .04em; color: var(--muted); margin: 0 0 12px; }
  ul { margin: 0; padding-left: 20px; }
  li { margin-bottom: 8px; }
  .badge { display: inline-block; width: 9px; height: 9px; border-radius: 50%; margin-right: 8px; }
  .badge.green { background: var(--green); } .badge.yellow { background: var(--yellow); } .badge.red { background: var(--red); }
  code, pre { background: #0d0f14; border-radius: 6px; padding: 2px 6px; font-family: Consolas, monospace; font-size: 13px; }
  pre { padding: 12px 14px; overflow-x: auto; }
  .checklist li { list-style: none; margin-left: -20px; display: flex; align-items: flex-start; gap: 8px; cursor: pointer; }
  .checklist input { accent-color: var(--accent); width: 16px; height: 16px; margin-top: 2px; cursor: pointer; }
  .checklist li.done span { color: var(--muted); text-decoration: line-through; }
  img { max-width: 100%; border-radius: 8px; margin-top: 8px; }
  svg { margin-top: 8px; }
</style>
</head>
<body>
<div class="wrap">
  <h1>{{TASK_TITLE}}</h1>
  <div class="meta">{{ONE_LINE_OUTCOME}}</div>

  <section>
    <h2>Changed</h2>
    <ul>
      <li><span class="badge green"></span>{{item}}</li>
    </ul>
  </section>

  <section>
    <h2>Remaining</h2>
    <ul>
      <li><span class="badge yellow"></span>{{item}}</li>
    </ul>
  </section>

  <section>
    <h2>Decisions needed</h2>
    <ul>
      <li><span class="badge red"></span>{{item}}</li>
    </ul>
  </section>

  <section>
    <h2>Example</h2>
    <pre><code>{{snippet}}</code></pre>
  </section>

  <section>
    <h2>Review checklist</h2>
    <ul class="checklist" id="checklist">
      <li><label><input type="checkbox"><span>{{concrete checkable step}}</span></label></li>
    </ul>
  </section>

  <section>
    <h2>Mock / diagram</h2>
    <img src="file:///{{absolute path to screenshot or mock}}" alt="">
  </section>
</div>
<script>
document.querySelectorAll('#checklist input[type=checkbox]').forEach(function (box) {
  box.addEventListener('change', function () {
    box.closest('li').classList.toggle('done', box.checked);
  });
});
</script>
</body>
</html>
```

Note: `<label>` wraps the checkbox and text so clicking either toggles it; the script just adds the strikethrough on check. State resets each time the file is overwritten - that's fine, it's meant to be ticked during that one review pass, not persisted.
