# Project 2: Meeting Notes Tracker

When a meeting ends, you save notes as a plain **`.txt`** file and upload it to a dedicated **Google Drive** folder. That upload is the **trigger**: n8n downloads the file, pulls out the text, sends it to an **LLM** to infer what was discussed, important **action items**, and **due dates** when mentioned. The workflow then **updates a Google Sheet** (your team’s running log) and **emails** the team lead or a fixed list of teammates so nothing falls through the cracks.

![Workflow overview](/.imgs/demo2.png)

---

## Workflow Overview

```
Google Drive (file uploaded to watch folder)
  → Download file
  → Extract text from file
  → AI Agent (summarize, tasks, dates)
  → Google Sheets (append / update log)
  → Gmail (notify team lead or distribution list)
```

You can implement the last two steps either as **tools on the same AI Agent** (similar to how Project 1 attaches Gmail) or as **separate nodes** after an LLM node that returns structured JSON — both patterns work; the agent + tools layout stays closer to [Project 1](../project1/readme.md).

---

## Nodes

### Reused from Project 1 (details there)

| Piece | Where it’s explained |
|--------|----------------------|
| **AI Agent** (model, system prompt, tool loop) | [Project 1 — AI Agent](../project1/readme.md#2-ai-agent) |
| **Gmail** (sending mail, OAuth2) | [Project 1 — Gmail tool](../project1/readme.md#3-gmail--send-email-tool) and [Gmail credential steps](../project1/readme.md#setting-up-gmail-api-credentials) |

Use a **system prompt** tailored to meetings: ask for a short summary, a bullet list of action items with owner if stated, due dates in ISO form when present, and a one-line “parking lot” for unclear items.

---

### 1. Google Drive — Trigger

**Type:** Google Drive (trigger)  
Fires when a **new file** appears (or is updated, depending on the event you choose) inside a **specific folder** — the inbox where people upload meeting notes.

Configure:

- **Folder** to watch (shared drive folder is fine if the credential has access)
- **Event** — e.g. file created, or file updated if you use “save again” uploads

You need **Google Drive API** enabled and **OAuth2** credentials in Google Cloud (same project as Gmail/Sheets is convenient). In n8n, create a **Google Drive OAuth2** credential and attach it to this trigger.

---

### 2. Google Drive — Download

**Type:** Google Drive  
Takes the file ID from the trigger output and **downloads the file** (binary) so the next node can read it. Use the same Drive credential as the trigger.

---

### 3. Extract from File

**Type:** Extract From File (or equivalent in your n8n version)  
Turns the downloaded **`.txt`** (or other supported format) into **text** the LLM can consume. If the file is always UTF-8 plain text, this node should output a string field you map into the AI Agent’s user message (e.g. “Meeting notes below:\n…”).

---

### 4. Google Sheets — Append or Update Row

**Type:** Google Sheets  
Writes structured results — for example **one row per meeting** with columns like `Date`, `Source file`, `Summary`, `Action items`, `Due dates`, `Raw notes link`.

Enable the **Google Sheets API** on the same Google Cloud project, add the Sheets scope to your OAuth consent screen if needed, and use a **Google Sheets OAuth2** credential in n8n. You will need the **spreadsheet ID** (from the URL) and the **sheet name** or gid.

If you use the **AI Agent with a Sheets tool**, configure the tool so the model can only touch that spreadsheet. If you use a **linear flow**, add a node that parses the LLM output (or use structured output / JSON mode) before mapping fields into **Append Row**.

---

### 5. Gmail — Notify team

**Type:** Gmail  
Sends a concise email to the **team lead** or a **static list** of addresses (BCC if you want privacy). Body can repeat the summary and link to the Drive file or Sheet row.

Credential setup is the same as in Project 1 (OAuth2). You can reuse the existing Gmail credential if the same Google account is allowed to send to those recipients.

---

## Google Cloud checklist (this project)

In addition to Gmail (Project 1), enable:

| API | Purpose |
|-----|--------|
| **Google Drive API** | Trigger + download |
| **Google Sheets API** | Log updates |

Use one OAuth client or separate credentials per service, depending on how you prefer to scope permissions in n8n.

---

## Operational tips

- **Folder discipline** — Only meeting notes go in the watched folder (or use a filename prefix / subfolder rule if you add a filter node).
- **Idempotency** — Drive may emit more than one event for a single upload; consider a **dedupe** strategy (e.g. process file ID once via a small DB or sheet lookup) if you see duplicates.
- **PII** — Notes may contain sensitive topics; align retention and Sheet access with your org policy.
