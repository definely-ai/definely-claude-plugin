---
name: definely-proofread
description: "Run a full Definely proofread on a contract or agreement, present an interactive clause-by-clause proof report, and action individual issues directly in the document using tracked changes. Use this skill whenever the user asks to: proofread a contract, run a proof report, check a document for drafting issues, fix capitalisation or cross-references in a contract, review defined terms, or action Definely proof issues. Trigger even if the user just says 'proofread this', 'check the doc', 'run Definely on this', or 'fix the issues' — with or without specifying which clause or issue type. Always use this skill when a contract is uploaded and the user wants any kind of proofreading, issue review, or drafting fix."
---
 
# Definely Proofread — Full Workflow
 
This skill covers the complete end-to-end proofreading workflow:
1. Upload the document to Definely and get a summary of all issues
2. Present an interactive clause selector so the user can drill into any section
3. Show a detailed, actionable issues table per clause
4. For docx documents, fix individual issues in the .docx using tracked changes (docx skill)
---
 
## Stage 1 — Upload & prepare (skip if already done this conversation)
  
1. Call `definely_upload_document` with the uploaded filename.
2. Call `definely_prepare_document` with `file_id`, `document_name`, and
   `spell_check_option: "en-GB"`.
3. Remember the `documentUuid` (= `file_id`) for all subsequent calls.
 
---
 
## Stage 2 — Summary report
 
Call definely tools to get the proofread summary for the document.
 
Parse the response and render a **summary widget** using `show_widget` (HTML, not React).
 
### Summary widget design
 
Group issues by the category returned by the tool
Add a Smart Search section (informational).
 
For each issue type show: issue name, description, total count, and three resolution badges.
 
After the summary widget, present the **clause selector widget** (Stage 3) immediately below in the same response.
 
---
 
## Stage 3 — Interactive clause selector
 
Get document table of contents using definely tools to get all clause names and hashes.
 
Render an **HTML widget** (using `show_widget`) with a clickable button list of every top-level clause and schedule. Each button calls `sendPrompt()` on click with the message:
 
```
Pull the detailed proofread issues for: <clause label> (hash: <hash>)
```
 
### Selector widget rules
- Render ALL clauses as a flat, fully visible list — no scroll containers (the iframe auto-expands, inner scroll does not work)
- Mark already-reviewed clauses with a green "done" badge
- Group boilerplate clauses (13+) under a section label
- Group schedules under a "Schedules" section label
- Show a "loading..." badge on the button immediately after click (before `sendPrompt` fires)
- Buttons use the brand colour scheme (navy border, light blue selected state)
---
 
## Stage 4 — Clause detail: proofread issues table
 
When the user selects a clause (via the selector or by typing a clause name/hash):
 
1. Call definely tools to get proofread issues in the section.
2. Parse all issues from the response.
3. Render a **fully expanded flat HTML table** (using `show_widget`) — all rows visible, no scroll container.
### Issues table columns
| # | Type | Issue text | Context | Resolution | Action |
 
### Action buttons
Every row gets a **"Fix in doc ↗"** button. On click it calls `sendPrompt()` with a precise instruction:
 
- **Fixable issues** (capitalisation, spacing, punctuation): instruct Claude to apply the fix as a **tracked change** (del + ins) in the document using the docx skill.
- **Reviewable issues** (missing x-refs, undefined terms, duplicate paragraphs): instruct Claude to insert a **Word comment** at the relevant location flagging the issue for review, using the docx skill.
- **Manual edit issues**: instruct Claude to insert a **Word comment** summarising the problem and suggesting what needs to be done.
Prompt format for each button:
```
Using the docx skill, <action description> in clause <N> of the document at its uploaded path. Apply as a tracked change authored "Claude". <specific context and fix>.
```
 
After a button is clicked, it turns to "Sent ✓" (green) and is disabled.
 
---
 
## Stage 5 — Actioning an issue (docx skill)
 
When a "Fix in doc ↗" button fires a prompt, follow the docx skill workflow exactly:
 
### Setup (first fix in a conversation)
```bash
cp -r <docx_scripts_dir> <workdir>/scripts
cp <uploads_dir>/<filename> <workdir>/document.docx
python <workdir>/scripts/office/unpack.py <workdir>/document.docx <workdir>/unpacked/
```
 
### For subsequent fixes
**Do not re-unpack.** The unpacked folder persists. Edit `<workdir>/unpacked/word/document.xml` directly with `str_replace`. All changes accumulate in the same unpacked folder.
 
### Finding the text
```bash
grep -n "<search_phrase>" <workdir>/unpacked/word/document.xml | head -10
```
Use 4–6 distinctive words from the context snippet to locate the right paragraph.
 
### Applying a capitalisation fix (tracked change)
Split the run at the word boundary, then wrap the old word in `<w:del>` and the new word in `<w:ins>`:
 
```xml
<w:del w:id="<unique_id>" w:author="Claude" w:date="2026-04-17T00:00:00Z">
  <w:r><w:rPr>...</w:rPr><w:delText>oldword</w:delText></w:r>
</w:del>
<w:ins w:id="<unique_id+1>" w:author="Claude" w:date="2026-04-17T00:00:00Z">
  <w:r><w:rPr>...</w:rPr><w:t>Newword</w:t></w:r>
</w:ins>
```
 
Copy the `<w:rPr>` from the original run into both del and ins runs. Use IDs starting at 9001 and incrementing for each change (check existing IDs first with `grep 'w:id=' <workdir>/unpacked/word/document.xml | tail -5`).
 
### Inserting a Word comment (reviewable/manual issues)
Use the `comment.py` script:
```bash
python <workdir>/scripts/comment.py <workdir>/unpacked/ <id> "<Comment text>" --author "Claude"
```
Then add `<w:commentRangeStart>`, `<w:commentRangeEnd>`, and `<w:commentReference>` markers as siblings of `<w:r>` in the relevant paragraph (never inside `<w:r>`).
 
### Repacking after every fix
```bash
python <workdir>/scripts/office/pack.py <workdir>/unpacked/ <workdir>/document_fixed.docx --original <workdir>/document.docx
cp <workdir>/document_fixed.docx <outputs_dir>/<original_filename>
```
 
Then call `present_files` with the output path and confirm to the user which issue was fixed and that it appears as a tracked change in the document.
 
### Important rules
- Always copy `<w:rPr>` from the original run into tracked change runs to preserve formatting
- Never re-unpack if the unpacked folder already exists — edits accumulate across multiple fixes
- Use `grep` on the document XML to find the exact text before editing — never guess at line numbers
- IDs in `w:id` attributes must be unique across the document — increment from the last used ID
- Use `str_replace` directly; do not write Python scripts for editing
---

## Brand & Design Tokens

Use definely tools to get brand design elements (colours, fonts, badge styles) and apply these exactly in the widget. Do not improvise on the design system or create your own styles.
---

## Error handling
 
- If Definely upload fails (404 on prepare), re-upload and retry once before reporting the error.
- If definely tool to get the proofread issues in a section returns 0 issues, confirm to the user that this clause is clean.
- If the docx unpack/pack fails, check that the file path is correct and the docx scripts are available at `<docx_scripts_dir>`.
- If a `str_replace` fails (string not found), use `grep` to find the actual text and adjust the search string.
- If `w:id` conflicts occur on repack, find the highest existing ID in the document and increment from there.
 