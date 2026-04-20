---
name: definely-issues-list
description: "Generate a formatted Issues List as a Microsoft Word (.docx) file from a contract or agreement, using Definely to extract tracked changes and mark-up. Use this skill whenever the user asks to: prepare an issues list, create an issues list in Word, generate a negotiation tracker, summarise seller/buyer/counterparty mark-up into a table, or produce a Word document from a Definely issues report. Can be used even if the user just says \"issues list\", \"prepare the issues list\", \"generate issues list from this document\", or \"Word issues list\" — with or without specifying column names or party labels of the table. Always use this skill when a .docx issues list output is requested and a document upload is present."
---
 
# Definely Issues List → Word (.docx)
 
You are generating a branded Microsoft Word issues list from a contract document. The workflow has three stages: upload to Definely, extract issues, then build the Word file.
 
---

## Stage 1 — Upload & prepare (skip if already done this conversation)
  
1. Call `definely_upload_document` with the uploaded filename.
2. Call `definely_prepare_document` with `file_id`, `document_name`, and
   `spell_check_option: "en-GB"`.
3. Remember the `documentUuid` (= `file_id`) for all subsequent calls.
 
---
 
## Stage 2 — Extract the issues list
 
Call definely tool to get the issues list for the document.
 
Read the header row of the returned markdown table — this defines the columns. Read every data row exactly as returned. Do not summarise, paraphrase, or rewrite any cell content at this stage unless explicitly instructed by the user to do so. The raw content of each cell will be used to populate the Word file in Stage 3.
 
---
 
## Stage 3 — Build the Word file
 
Use `npm` (`docx` library, already installed globally) to generate the file.
Follow the docx skill rules exactly: dual column widths in DXA, ShadingType.CLEAR, no unicode bullets, separate Paragraph elements (never `\n`), A4 landscape with explicit page size.
 
### Column structure
 
There are two cases:
 
**Case A — user specifies columns in their prompt**
Use exactly the column names and order the user specifies. Map the Definely output into those columns as instructed.
 
**Case B — user gives no column instructions**
Use the column names exactly as they appear in the header row of the definely's issues list tool response — do not rename, reorder, reinterpret, or default to anything. The tool's header row is the sole source of truth.
Reproduce every column in the same order, with identical capitalisation and spacing as the tool returned.
 
In both cases: populate each cell with the raw content from the corresponding Definely column. Do not paraphrase or rewrite cell content. Leave any columns that are blank in the tool output blank in the Word file.
 
### Markup text colouring
 
The Markup Text column returned by definely's issues list tool uses **two different formats** for insertions depending on the type of change. When building the Word file, **always** render these inline tokens with colour — do not strip them or render them as plain text:
 
| Token in Definely output | Word rendering |
|--------------------------|----------------|
| `<ins>inserted text</ins>` | Blue (`#0055CC`) — explicit insertion tag (used for standalone insertions with no adjacent deletion) |
| `~~deleted text~~`         | Red (`#CC0000`) strikethrough — the deletion |
| Text immediately following `~~...~~` with no whitespace gap | Blue (`#0055CC`) — the insertion that replaces the deletion |
| `**bold text**`            | Bold, body colour — defined terms and cross-references |
 
**Critical:** Both `<ins>...</ins>` HTML tags and the implicit post-deletion
blue pattern can appear in the same document. The parser must handle both.
 
**Parsing algorithm** (implement in the `parseMarkup` helper function):
 
Use a single regex that matches all four token types in priority order:
 
```javascript
const regex = /<ins>([\s\S]*?)<\/ins>|~~(.*?)~~|\*\*(.*?)\*\*|([^~*<]+|<(?!\/?ins)[^>]*>|[~*<])/gs;
```
 
Tokenise into `{ type, text }` objects:
- `match[1]` → `{ type: 'ins' }` — explicit `<ins>` tag
- `match[2]` → `{ type: 'del' }` — `~~deletion~~`
- `match[3]` → `{ type: 'bold' }` — `**bold**`
- `match[4]` → `{ type: 'plain' }` — everything else
Then emit `TextRun` elements:
1. `ins` token → `color: '#0055CC'` (blue, no strike)
2. `del` token → `strike: true, color: '#CC0000'` (red strikethrough); set `prevWasDel = true`
3. `bold` token → `bold: true`, body colour; reset `prevWasDel = false`
4. `plain` token — if `prevWasDel` is true and the text begins with a non-whitespace
   character, split at the first whitespace boundary: emit the leading chunk as
   `color: '#0055CC'` (the implicit insertion), remainder as normal body colour.
   Always reset `prevWasDel = false` after processing a plain token.
Add a legend line to the title block instructions paragraph:
`"Deletions shown in red strikethrough. Additions shown in blue."`
 
### Key issue flagging
 
Only apply if the user explicitly asks for it. If they do, add a short line
in orange beneath the relevant cell:
`★  Key issue — requires client instruction`
Do not add flags unless asked. Use orange sparingly.
 
---
 
## Brand & design
 
Apply these settings exactly — do not deviate:
 
| Token | Value |
|-------|-------|
| Navy | `#061131` — document title, column headers background, clause text |
| Light Blue | `#00A3DA` — accent column header, item numbers, subtitle, page numbers |
| Orange | `#F76B07` — key issue flag only (sparingly) |
| Blue Tint 1 | `#F2FAFD` — alternating row background |
| Blue Tint 2 | `#E5F6FB` — header/footer rule colour |
| Heading font | Poppins — title, column headers, clause names, document header |
| Body font | Inter — all body copy, cell content, footer |
| White | `#FFFFFF` — non-alternating row background, header text |
| Body text | `#1A2540` — main content colour |
| Muted text | `#5A6880` — subtitle, footer, instructions line |
| Border | `#C8E8F5` — all table cell borders |
 
**Page:** A4 landscape. Margins: 720 DXA (0.5 in) all sides.
 
### Typography — fixed sizes (do not deviate)
 
In the `docx` library the `size` property is in **half-points** (1 pt = 2 units).
The table below is the single source of truth for every text element. Always use
the exact `size` value listed — never guess or adjust.
 
| Element | Font | pt | `size` (half-pt) | Weight | Colour |
|---------|------|----|-----------------|--------|--------|
| Page header — document name | Poppins | 11 | 22 | Bold | Navy `#061131` |
| Page header — date | Inter | 11 | 22 | Regular | Muted `#5A6880` |
| Page footer — confidentiality text | Inter | 9 | 18 | Italic | Muted `#5A6880` |
| Page footer — page number | Inter | 9 | 18 | Regular | Light Blue `#00A3DA` |
| Title block — matter name heading | Poppins | 40 | 80 | Bold | Navy `#061131` |
| Title block — subtitle (agreement + date) | Poppins | 22 | 44 | Regular | Light Blue `#00A3DA` |
| Title block — instructions line | Inter | 9 | 17 | Italic | Muted `#5A6880` |
| Table header row text | Poppins | 10 | 20 | Bold | White `#FFFFFF` |
| Table data — item number | Poppins | 9 | 18 | Bold | Light Blue `#00A3DA` |
| Table data — clause reference | Inter | 9 | 18 | Regular | Navy `#061131` |
| Table data — body / markup text | Inter | 9 | 18 | Regular | Body `#1A2540` |
| Table data — insertion (blue) | Inter | 9 | 18 | Regular | `#0055CC` |
| Table data — deletion (red strike) | Inter | 9 | 18 | Regular | `#CC0000` |
| Table data — bold defined term | Inter | 9 | 18 | Bold | Body `#1A2540` |
| Key issue flag (if requested) | Inter | 9 | 18 | Regular | Orange `#F76B07` |
 
**Header (every page):** Document name left-aligned (Poppins 11pt Bold Navy),
date right-aligned (Inter 11pt Muted). Separated from body by a Light Blue
bottom border.
 
**Footer (every page):** "Strictly Confidential – Legally Privileged" left
(Inter 9pt Italic Muted), page number right (Inter 9pt Light Blue).
Separated from body by a Blue Tint 2 top border.
 
**Title block (first page only):**
- Heading: document/matter name — Poppins Bold 40pt (`size: 80`) Navy
- Subtitle: agreement name + date — Poppins Regular 22pt (`size: 44`) Light Blue
- Instructions line: 1-sentence description — Inter Italic 9pt (`size: 17`) Muted
**Table header row:** Navy background, White Poppins Bold 10pt (`size: 20`) text.
 
**Data rows:** alternating White / Blue Tint 1. Body copy Inter 9pt (`size: 18`) `#1A2540`.
 
---
 
## Output
 
Save the file as `[Matter]_Issues_List.docx` (derive the matter name from
the document filename or the user's prompt).
 
Copy to `/mnt/user-data/outputs/` and call `present_files`.
 
After presenting, give the user a brief summary: total issue count and
clause areas covered. Keep it to 2–3 sentences.
 
---
 
## Error handling
 
- If Definely returns zero issues, tell the user and ask them to confirm the
  document contains tracked changes before retrying.
- If pagination is needed and a page call fails, include what was retrieved
  and note the incomplete retrieval.
- If the docx build fails validation, unpack → fix XML → repack using the
  docx skill scripts before presenting.
