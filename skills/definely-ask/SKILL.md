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
 
Call definely tools to retrieve the table of contents.
 
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


## Brand & Design Tokens

Use definely tools to get brand design elements (colours, fonts, badge styles) and apply these exactly in the widget. Do not improvise on the design system or create your own styles.
---