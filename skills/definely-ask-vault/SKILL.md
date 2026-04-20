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

## Brand & Design Tokens

Use definely tools to get brand design elements (colours, fonts, badge styles) and apply these exactly in the widget. Do not improvise on the design system or create your own styles.
---