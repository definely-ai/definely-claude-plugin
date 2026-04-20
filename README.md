# definely-claude-plugin

Contract intelligence for Claude. Upload a `.docx` agreement and ask anything about it — Claude reads, analyses, and proof-reads using [Definely's](https://definely.com) document intelligence tools, then presents results in Definely's branded UI.

---

## What it does

| Skill | What you get |
|---|---|
| **definely-ask** | Ask questions about a contract: "What do the indemnities in this MSA cover, and are there any carve-outs?|
| **definely-ask-vault** | Search your precedent library: "Find the closest precedent in Definely Vault to this SPA and flag the key differences |
| **definely-proofread** | Proofread this document and give me a list of issues to work through |
| **definely-issues-list** | Turn the changes in this marked-up contract into an issues list in Word with columns for item, counterparty position, and client response |

---

## Prerequisites

- A Definely account with full suite licence (contact Definely team to set this up if you don't have one)
- Claude.ai or Claude Desktop with the Definely MCP server connected
- A contract document to upload and work with (in `.docx` format for best results, but other formats accepted by Definely will also work)
---

## Authentication

The Definely MCP server uses session-based authentication via Claude.ai's connector flow. Connect your Definely account once via **Claude Settings → Connectors → Definely**, and all skills will use it automatically.

---

## Skills

### `definely-ask` — Contract Q&A

Answer any natural-language question about a contract.

**Triggers:** "Does this agreement contain X?", "What does clause Y say?", "Confirm there is language about Z", "Check whether the sellers have rights to…", "Summarise what it says about…"

**Workflow:**

1. Upload the document to Definely and prepare it
2. Map the document's clause structure via definely tools
3. Retrieve relevant provisions based on the question, using definely tools
4. Present the answer in a structured format with clause references and context snippets
---

### `definely-ask-vault` — Vault Search & Comparison

Search your firm's precedent library and compare provisions against the current document.

**Triggers:** "Compare this clause against precedent", "How did we draft X in Project Zeus?", "Find a similar limitation of liability clause in the vault", "What's the closest precedent to this SPA?"

**Workflow:**
1. Upload the  document to Definely and prepare it
2. Map the document's clause structure via definely tools
3. Search the vault for similar clauses or agreements using definely tools
4. Retrieve the top matches and compare them against the current document, highlighting key differences in language and flagging any potential issues or deviations from precedent
---

### `definely-proofread` — Interactive Proofread Report

Run a full proofread and action issues directly in the document.

**Triggers:** "Proofread this", "Run Definely on this", "Check the doc", "Fix the capitalisation issues", "Review defined terms"

**Workflow:**
1. Upload the document to Definely and prepare it
2. Proofread the contract for a wide range of potential proofreading issues using definely tools
3. Get the issue counts grouped by type using definely proofread summary tool
4. In the **Clause selector**, click any clause to drill into its issues using definely clause-level proofread tool
5. Get every issue with type, context snippet, resolution badge, and a **Fix in doc ↗** button
6. Click the **Fix in doc ↗** button to apply the change as a tracked change (or insert a review comment) in the `.docx` using Claude's docx skill

---

### `definely-issues-list` — Issues List → Word

Generate a formatted Issues List as a branded `.docx` file from a marked-up contract. Issues list is generated based on tracked changes in the document, with deletions, insertions, modifications and comments extracted into an issues table.

**Triggers:** "Prepare an issues list", "Create an issues list in Word", "Generate a negotiation tracker", "Summarise the counterparty mark-up into a table"

**Workflow:**
1. Upload the document to Definely and prepare it
2. Extract the issues list using Definely's issues list tool
3. Build a branded Word file with the issues list, rendering tracked changes with red strikethrough for deletions and blue text for insertions


---

## Example prompts

```
Does this SPA contain a leakage definition, and does it cover deemed distributions?
```

```
Compare our limitation of liability cap against the Project Athena precedent.
```

```
Proof read clause 10 and fix any capitalisation errors.
```

```
Prepare an issues list from this marked-up agreement with columns: Clause, Issue, Buyer Position, Seller Position, Status.
```

---

## Directory structure

```
definely/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── definely-ask/
│   │   └── SKILL.md
│   ├── definely-ask-vault/
│   │   └── SKILL.md
│   ├── definely-proofread/
│   │   └── SKILL.md
│   └── definely-issues-list/
│       └── SKILL.md
└── README.md
```

---

## Notes

- All skills require a document input. The user can upload a document in the conversation, or specify a previously uploaded document by name. If multiple documents are uploaded in the same conversation, the skill will ask which one to use.
- Documents are uploaded to Definely's servers for processing. See [Definely's privacy policy](https://definely.com/privacy) for data handling details.
- The proofread and issues-list skills use the `docx` skill as part of their workflow — ensure the docx skill is available in the environment.

---

## License

See [LICENSE](./LICENSE).
