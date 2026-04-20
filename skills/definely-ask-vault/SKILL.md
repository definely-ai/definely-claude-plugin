---
name: definely-ask-vault
description: >
  Search the firm's Definely Vault to compare clauses, definitions, or provisions in a current contract against precedent documents. Use this skill whenever the user asks to: compare a clause or provision against a precedent, look up how the firm has previously drafted a specific term, find a precedent document by name, check how a definition or obligation was worded in a past deal, or asks anything like "how did we do this in [Project X]", "what does our precedent say about Y", "find a similar clause in the vault", or "compare this against [document name]". Also can be used when a document is already uploaded and the user wants to benchmark it against vault content — even if they just say "check against precedent" or "look this up".
---
 
# Definely Ask Vault — Search & Compare Workflow
 
This skill covers searching the Definely Vault for precedent clauses, definitions, and documents, and comparing them against the current document or a user query. It also covers the case where a document is uploaded and needs to be prepared before vault comparison can occur.
 
---
 
## Stage 1 — Upload & prepare (skip if already done this conversation)
  
1. Call `definely_upload_document` with the uploaded filename.
2. Call `definely_prepare_document` with `file_id`, `document_name`, and
   `spell_check_option: "en-GB"`.
3. Remember the `documentUuid` (= `file_id`) for all subsequent calls.

---
 
## Stage 2 — Identify the search target
 
Determine what the user wants to find in the vault and use the appropriate tool(s) to search:

- **Clause, term, or concept** (e.g., "Leakage", "how we cap liability on warranties")
- **Named precedent document** (e.g., "Project X SPA")
- **Closest precedent document** (e.g., "most similar SPA in the vault")
When the user names a specific precedent document, pass its name in `documentNames` on subsequent searches to scope results to that document only.

---
 
## Stage 3 — Extract the current document's version (if applicable)
 
If the user wants a comparison (not just a vault lookup), also extract the relevant
provision from the uploaded document:
 
- Use definely tools to fetch provisions by passing the current document file id and the clause/term name.
- If the clause sits within a larger schedule or part, get the current document's table of contents to locate the correct provisions, then use definely to retrieve their full text.
- Run these in parallel with the vault search where possible.

---
 
## Stage 4 — Render the results as a comparison widget
 
Use `show_widget` (HTML, not React) to present the findings. Always apply the brand design system below exactly. Render all comparison output as a widget — do not reproduce large blocks of clause text in plain prose.
 
### Output structure
 
A comparison widget should always include:
 
1. **Doc-header bar** — navy, showing the document name(s) being compared and the vault document found.
2. **Summary note card** — 2–4 bullet points in plain English summarising the key differences or findings.
3. **Comparison table** — one row per clause head or provision being compared.
   Columns: `Head / provision`, `Current document`, `Vault precedent`, `Assessment`
   (badge). All cells must wrap text (see Tables rule above).
4. **Suggested language section** (where applicable) — individual cards per suggested change, with a short label and the proposed alternative wording.
Keep all explanatory prose **outside** the widget in your normal response text.
The widget contains only the visual comparison, not paragraphs of explanation.

---
 
## Brand Design System
  
All widgets use **Definely's visual identity**. Fonts, colours, spacing, and component patterns are defined here and must be applied exactly in every `show_widget` call.
 
> **Note for the user:** Any visual output produced by this skill — cards, badges, tables, summary bars — uses Definely's proprietary fonts (Poppins + Inter), colour palette, and component style.
 
### Fonts
 
Always load both fonts at the top of every widget HTML:
```html
<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@500;600&family=Inter:wght@400;500&display=swap" rel="stylesheet">
```
 
### Colour palette
 
| Token | Hex | Usage |
|-------|-----|-------|
| Navy | `#061131` | Widget header bar, `<thead>`, section separator |
| Light Blue | `#00A3DA` | Accent: item numbers, count pills, selected borders |
| Blue Tint 1 | `#F2FAFD` | Alternating even rows, smart-search card bg |
| Blue Tint 2 | `#E5F6FB` | Loading state bg |
| White | `#FFFFFF` | Odd rows, header text |
| Body text | `#1A2540` | All cell text, button default |
| Muted text | `#5A6880` | Sub-lines, context snippets, section labels |
| Border | `#C8E8F5` | All cell borders (0.5px), card borders |
| Green bg | `#EAF3DE` | "Confirmed" badge bg, Sent button |
| Green text | `#27500A` | "Confirmed" badge text |
| Green border | `#C0DD97` | Sent button border |
| Blue bg | `#E6F1FB` | "Supporting" badge bg |
| Blue text | `#0C447C` | "Supporting" badge text |
| Amber bg | `#FAEEDA` | Gap/warning alert bg, "Partial" badge bg |
| Amber text | `#633806` | Gap/warning alert text, "Partial" badge text |
| Red bg | `#FCEBEB` | "Not Found" badge bg |
| Red text | `#A32D2D` | "Not Found" badge text |
| Navy muted | `#9fb8d8` | Sub-title text on navy backgrounds |
 
### Typography
 
| Element | Font | Weight | Size | Colour |
|---------|------|--------|------|--------|
| Widget doc-header title | Poppins | 600 | 15px | `#FFFFFF` |
| Widget doc-header sub-title | Inter | 400 | 12px | `#9fb8d8` |
| Section card heading (navy bar) | Poppins | 600 | 12px | `#FFFFFF` |
| Table `<th>` | Poppins | 600 | 10px | `#FFFFFF` |
| Table `<td>` | Inter | 400 | 12px | `#1A2540` |
| Issue name (primary line) | Inter | 500 | 12.5px | `#1A2540` |
| Issue description (sub-line) | Inter | 400 | 11px | `#5A6880` |
| Context snippet | Inter | 400 | 11px | `#5A6880` (italic) |
| Row number | Inter | 600 | 11px | `#00A3DA` |
| Badge | Inter | 500 | 10px | (see badge colours) |
| Action button | Inter | 400 | 11px | `#1A2540` |
 
### Spacing & layout
 
| Element | Rule |
|---------|------|
| Widget outer | `padding: 12px 0 24px` — no background colour |
| Doc-header bar | `background: #061131; border-radius: 10px; padding: 14px 18px; margin-bottom: 18px` |
| Section card | `border: 0.5px solid #C8E8F5; border-radius: 10px; overflow: hidden; margin-bottom: 16px` |
| Section card navy header | `background: #061131; padding: 9px 16px; display: flex; justify-content: space-between; align-items: center` |
| Issue row (odd) | `background: #FFFFFF` |
| Issue row (even) | `background: #F9FEFF` |
| Issue row hover | `background: #E5F6FB` |
| Table | `width: 100%; border-collapse: collapse; table-layout: fixed` |
 
### Component CSS (copy-paste into every widget)
 
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body, div, td, th, span, p, button { font-family: 'Inter', sans-serif; }
h1, h2, h3, .heading { font-family: 'Poppins', sans-serif; }
 
.badge { display: inline-block; padding: 2px 8px; border-radius: 20px; font-family: 'Inter', sans-serif; font-size: 10px; font-weight: 500; white-space: nowrap; }
.badge-confirmed  { background: #EAF3DE; color: #27500A; }
.badge-partial    { background: #FAEEDA; color: #633806; }
.badge-supporting { background: #E6F1FB; color: #0C447C; }
.badge-notfound   { background: #FCEBEB; color: #A32D2D; }
 
.gap-alert { background: #FAEEDA; border: 0.5px solid #FAC775; border-radius: 8px; padding: .75rem 1rem; display: flex; gap: 10px; align-items: flex-start; margin-top: 1rem; }
.gap-text { font-size: 13px; color: #633806; line-height: 1.6; }
 
.summary-bar { background: #F2FAFD; border: 0.5px solid #C8E8F5; border-radius: 8px; padding: .85rem 1.1rem; display: flex; align-items: center; gap: 12px; margin-bottom: 1.25rem; }
.clause-text { font-size: 13px; color: #1A2540; line-height: 1.65; border-left: 2px solid #C8E8F5; padding-left: 12px; margin-bottom: .75rem; }
.analysis { font-size: 13px; color: #5A6880; line-height: 1.6; }
.highlight { color: #1A2540; font-weight: 500; }
```
 ### Tables — critical rule
 
**All table cells must use `word-wrap: break-word; overflow-wrap: break-word;
white-space: normal`** so that clause text wraps within its cell rather than
overflowing. Set this on every `<td>` and `<th>`. Use `table-layout: fixed` on
every `<table>` and set explicit `width` on each `<col>` or `<th>` so the browser
respects the fixed layout. Never allow text to overflow or extend the table beyond
its container.
---
 
## Verdict badges
 
Use these badges in the summary bar and per-clause card headers:
 
| Situation | Badge class | Label |
|-----------|-------------|-------|
| Language clearly present and unambiguous | `badge-confirmed` | Confirmed ✓ |
| Language present but with caveats or gaps | `badge-partial` | Partial |
| Related/supporting provision found | `badge-supporting` | Supporting |
| No language found | `badge-notfound` | Not Found |
---
