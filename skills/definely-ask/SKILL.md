---
name: definely-ask
description: "Answer legal questions about an uploaded contract or agreement using Definely's suite of document intelligence tools. Use this skill whenever the user asks a question about the content, language, structure, or coverage of a contract — for example: 'does this agreement contain X?', 'confirm there is language about Y', 'what does clause Z say?', 'is there a provision covering...?', 'check whether the sellers have rights to...', 'summarise what the document says about...', or any natural-language question directed at a legal document. Trigger even if the question is phrased casually or without specifying a clause number. Always use this skill when a .docx contract is uploaded and the user wants to interrogate, analyse, or verify its contents."
---
 
# Definely Ask — Natural-Language Contract Q&A
 
Answer any natural-language question about a contract using Definely's document intelligence tools. The workflow is: upload → analyse structure → retrieve relevant context → use retrieved context to generate an answer → present in a branded widget.
 
---
 
## Stage 1 — Upload & prepare (skip if already done this conversation)
 
1. Call `definely_upload_document` with the uploaded filename.
2. Call `definely_prepare_document` with `file_id`, `document_name`, and
   `spell_check_option: "en-GB"`.
3. Remember the `documentUuid` (= `file_id`) for all subsequent calls.

---
 
## Stage 2 — Map the document structure
 
Call definely tools to retrtieve the table of contents.
 
Parse the returned table of contents to identify:
- Which top-level clauses are most likely to contain the answer.
- Use the clause names to determine where relevant language is likely to live and fetch the text of those clauses.

Do not skip this step — the TOC is the navigation layer for all subsequent definely tool calls.
 
---
 
## Stage 3 — Retrieve relevant clauses
 
Use one or more of the following tools depending on what the question requires:

- Start with definely tools to fetch the most directly relevant clause(s).
- If a retrieved clause contains cross-references to other clauses (e.g. "subject to clause X"), use definely tools to pull those through.
- If the question asks whether language is "standard" or "market", run a Vault search.
- For defined terms, follow through to definitions to understand exact scope.
- Retrieve as many clauses as needed to give a complete, accurate answer. Do not stop at the first clause found.

---
 
## Stage 4 — Analyse and present
 
### Analysis framework
 
For each question, your answer must address:
 
1. **What the document says** — quote the exact operative language (kept short, under 15 words per quote), then paraphrase the full provision.
2. **What is covered** — confirm which aspects of the question are addressed and by which clause(s).
3. **What is not covered or ambiguous** — identify any gaps, sweep-up language that may or may not cover the situation, or provisions that need further confirmation.
4. **Overall verdict** — a clear yes/no/partial answer with a brief rationale.

### Output structure
 
Present the answer as a `show_widget` HTML widget using the **Definely brand design system** (see Brand & Design Tokens below). The widget is the primary deliverable; keep any prose in the chat brief.
 
Widget structure:
1. **Summary bar** — verdict badge (Confirmed / Partial / Not Found) + one-sentence summary
2. **One card per relevant clause** — clause reference, clause title, quoted/paraphrased operative text, brief analysis
3. **Gap alert** (amber, if applicable) — flag anything missing or uncertain
 
### Multi-clause questions
 
If the question touches multiple topics (e.g. "confirm sellers have no rights to block the sale AND have waived all governance rights"), handle each topic as a separate card in the widget, with its own verdict badge and clause reference.
 
---
 
## Error handling
 
- If definely tool to return the table of contents returns no results, report to the user that the document may not have prepared correctly and retry Stage 1.
- If a item search returns no match, try a variant of the clause name or fetch the parent clause.
- If no relevant language is found after exhausting all reasonable search strategies, render a "Not Found" verdict card and suggest the user may wish to add a provision.
- Never speculate about clause content — only report what the tools return.

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