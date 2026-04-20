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
Using the docx skill, <action description> in clause <N> of the document at mounted directory of the file (e.g. /mnt/user-data/uploads/<filename>). Apply as a tracked change authored "Claude". <specific context and fix>.
```
 
After a button is clicked, it turns to "Sent ✓" (green) and is disabled.
 
---
 
## Stage 5 — Actioning an issue (docx skill)
 
When a "Fix in doc ↗" button fires a prompt, follow the docx skill workflow exactly:
 
### Setup (first fix in a conversation)
```bash
cp -r /mnt/skills/public/docx/scripts /home/claude/scripts
cp /mnt/user-data/uploads/<filename> /home/claude/spa.docx
python /home/claude/scripts/office/unpack.py /home/claude/spa.docx /home/claude/unpacked/
```
 
### For subsequent fixes
**Do not re-unpack.** The unpacked folder persists. Edit `/home/claude/unpacked/word/document.xml` directly with `str_replace`. All changes accumulate in the same unpacked folder.
 
### Finding the text
```bash
grep -n "<search_phrase>" /home/claude/unpacked/word/document.xml | head -10
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
 
Copy the `<w:rPr>` from the original run into both del and ins runs. Use IDs starting at 9001 and incrementing for each change (check existing IDs first with `grep 'w:id=' unpacked/word/document.xml | tail -5`).
 
### Inserting a Word comment (reviewable/manual issues)
Use the `comment.py` script:
```bash
python /home/claude/scripts/comment.py /home/claude/unpacked/ <id> "<Comment text>" --author "Claude"
```
Then add `<w:commentRangeStart>`, `<w:commentRangeEnd>`, and `<w:commentReference>` markers as siblings of `<w:r>` in the relevant paragraph (never inside `<w:r>`).
 
### Repacking after every fix
```bash
python /home/claude/scripts/office/pack.py /home/claude/unpacked/ /home/claude/spa_fixed.docx --original /home/claude/spa.docx
cp /home/claude/spa_fixed.docx /mnt/user-data/outputs/<original_filename>
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
 
These are the **authoritative values** for every colour, font, size, and spacing used across all widgets. Always apply exactly — never deviate.
 
### Colour palette
 
| Token name | Hex | Usage |
|---|---|---|
| Navy | `#061131` | Widget header bar bg, table `<thead>` bg, section separator text, clause selector header bg |
| Light Blue | `#00A3DA` | Accent colour: item number text, count bubbles bg, selected/loading button border, section-count pill bg |
| Blue Tint 1 | `#F2FAFD` | Alternating even rows bg, section separator row bg, smart-search card bg, selector section-label bg |
| Blue Tint 2 | `#E5F6FB` | Loading button bg, widget outer tint where used |
| White | `#FFFFFF` | Odd row bg, header text colour, count-pill text |
| Body text | `#1A2540` | All table cell text, button default text, clause name text |
| Muted text | `#5A6880` | Issue description sub-line, context snippet text, selector section-label text, smart-search item label |
| Border | `#C8E8F5` | All table cell borders (0.5px), card borders, button borders (default state) |
| Fixable green bg | `#EAF3DE` | Fixable badge background, Sent button background |
| Fixable green text | `#27500A` | Fixable badge text, Sent button text |
| Fixable green border | `#C0DD97` | Sent button border |
| Reviewable blue bg | `#E6F1FB` | Reviewable badge background |
| Reviewable blue text | `#0C447C` | Reviewable badge text |
| Manual amber bg | `#FAEEDA` | Manual-edit badge background |
| Manual amber text | `#633806` | Manual-edit badge text |
| Orange | `#F76B07` | Sparingly: key issue highlights only |
| Navy muted | `#9fb8d8` | Doc-header sub-title text (on navy bg) |
 
### Typography
 
Always load both fonts at the top of every widget HTML:
```html
<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@500;600&family=Inter:wght@400;500&display=swap" rel="stylesheet">
```
 
| Element | Font | Weight | Size | Colour | Other |
|---|---|---|---|---|---|
| Widget doc-header title | Poppins | 600 | 15px | `#FFFFFF` | — |
| Widget doc-header sub-title | Inter | 400 | 12px | `#9fb8d8` | — |
| Section card heading (inside navy bar) | Poppins | 600 | 12px | `#FFFFFF` | `letter-spacing: 0.05em; text-transform: uppercase` |
| Section total count pill (inside navy bar) | Inter | 500 | 11px | `#FFFFFF` | Pill bg `#00A3DA`, `padding: 2px 9px`, `border-radius: 20px` |
| Table `<th>` | Poppins | 600 | 10px | `#FFFFFF` | `letter-spacing: 0.05em; text-transform: uppercase; padding: 8px 10px` |
| Table `<td>` — all cells | Inter | 400 | 12px | `#1A2540` | `padding: 8px 10px; vertical-align: top` |
| Issue name (primary line in type cell) | Inter | 500 | 12.5px | `#1A2540` | — |
| Issue description (sub-line) | Inter | 400 | 11px | `#5A6880` | `margin-top: 2px` |
| Context snippet text | Inter | 400 | 11px | `#5A6880` | `font-style: italic` |
| Context highlighted word | Inter | 500 | 11px | `#061131` | `font-style: normal; background: #FFF3C4; padding: 0 2px; border-radius: 2px` |
| Row number cell | Inter | 600 | 11px | `#00A3DA` | — |
| Section separator row text | Inter | 600 | 10px | `#061131` | `letter-spacing: 0.07em; text-transform: uppercase; padding: 6px 10px` |
| Resolution badge | Inter | 500 | 10px | (see badge colours) | `padding: 2px 8px; border-radius: 20px; white-space: nowrap` |
| Action button (default) | Inter | 400 | 11px | `#1A2540` | `padding: 4px 8px; border: 0.5px solid #C8E8F5; border-radius: 5px; background: transparent` |
| Action button (sent) | Inter | 400 | 11px | `#27500A` | `background: #EAF3DE; border-color: #C0DD97; cursor: default` |
| Selector section-label | Inter | 600 | 10px | `#5A6880` | `letter-spacing: 0.08em; text-transform: uppercase; padding: 10px 0 5px; border-top: 0.5px solid #C8E8F5` |
| Selector button clause number | Inter | 400 | 11px | `#5A6880` | `min-width: 26px` |
| Selector button clause name | Inter | 400 | 13px | `#1A2540` | `flex: 1; margin-left: 8px` |
| Selector button arrow | Inter | 400 | 11px | `#00A3DA` | — |
| Selector header title | Poppins | 600 | 13px | `#061131` | `letter-spacing: 0.04em; text-transform: uppercase; margin-bottom: 12px` |
| Smart-search card title | Poppins | 600 | 12px | `#061131` | `letter-spacing: 0.05em; text-transform: uppercase; margin-bottom: 10px` |
| Smart-search item label | Inter | 400 | 11px | `#5A6880` | — |
| Smart-search item count | Inter | 600 | 16px | `#00A3DA` | — |
 
### Spacing & layout
 
| Element | Rule |
|---|---|
| Widget outer padding | `padding: 12px 0 24px` — no background colour on outer container |
| Doc-header bar | `background: #061131; border-radius: 10px; padding: 14px 18px; margin-bottom: 18px` |
| Section card | `border: 0.5px solid #C8E8F5; border-radius: 10px; overflow: hidden; margin-bottom: 16px` |
| Section card navy header | `background: #061131; padding: 9px 16px; display: flex; justify-content: space-between; align-items: center` |
| Issue row (odd) | `background: #FFFFFF` |
| Issue row (even) | `background: #F9FEFF` — very subtle, not the full Blue Tint 1 |
| Issue row hover | `background: #E5F6FB` |
| Section separator row | `background: #F2FAFD; border-bottom: 0.5px solid #C8E8F5` |
| Table cell border | `border-bottom: 0.5px solid #E5F6FB` (lighter than card border) |
| Table | `width: 100%; border-collapse: collapse; table-layout: fixed` |
| Column widths (issues table) | `#`: 32px · `Type`: 130px · `Issue text`: 130px · `Context`: auto · `Resolution`: 90px · `Action`: 90px |
| Selector button | `display: flex; align-items: center; justify-content: space-between; width: 100%; background: #FFFFFF; border: 0.5px solid #C8E8F5; border-radius: 7px; padding: 9px 13px; margin-bottom: 5px` |
| Selector button hover | `background: #F2FAFD; border-color: #00A3DA` |
| Selector button loading | `background: #E5F6FB; border-color: #00A3DA; color: #0C447C; cursor: default` |
| Selector button sent/done | `background: #EAF3DE; border-color: #C0DD97; color: #27500A; cursor: default` |
| Smart-search card | `background: #F2FAFD; border: 0.5px solid #C8E8F5; border-radius: 10px; padding: 12px 16px; margin-bottom: 16px` |
| Smart-search grid | `display: grid; grid-template-columns: 1fr 1fr; gap: 8px` |
| Smart-search item | `background: #FFFFFF; border: 0.5px solid #C8E8F5; border-radius: 7px; padding: 8px 12px` |
| Legend row | `display: flex; gap: 12px; margin-bottom: 14px; flex-wrap: wrap` |
 
### Resolution badges (copy-paste CSS)
 
```css
.badge          { display: inline-block; padding: 2px 8px; border-radius: 20px; font-family: 'Inter', sans-serif; font-size: 10px; font-weight: 500; white-space: nowrap; }
.badge-fixable  { background: #EAF3DE; color: #27500A; }
.badge-review   { background: #E6F1FB; color: #0C447C; }
.badge-manual   { background: #FAEEDA; color: #633806; }
```
 
### Action button (copy-paste CSS)
 
```css
.fix-btn        { background: transparent; border: 0.5px solid #C8E8F5; border-radius: 5px; padding: 4px 8px; font-family: 'Inter', sans-serif; font-size: 11px; color: #1A2540; cursor: pointer; white-space: nowrap; }
.fix-btn:hover  { background: #F2FAFD; border-color: #00A3DA; }
.fix-btn.sent   { background: #EAF3DE; border-color: #C0DD97; color: #27500A; cursor: default; }
```
 
### Full CSS reset block (include at top of every widget `<style>`)
 
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body, div, td, th, span, p, button { font-family: 'Inter', sans-serif; }
h1, h2, h3, .heading { font-family: 'Poppins', sans-serif; }
```
 
---
 
## Error handling
 
- If Definely upload fails (404 on prepare), re-upload and retry once before reporting the error.
- If definely tool to get the proofread issues in a section returns 0 issues, confirm to the user that this clause is clean.
- If the docx unpack/pack fails, check that the file path is correct and the docx scripts are available at `/mnt/skills/public/docx/scripts/`.
- If a `str_replace` fails (string not found), use `grep` to find the actual text and adjust the search string.
- If `w:id` conflicts occur on repack, find the highest existing ID in the document and increment from there.
 