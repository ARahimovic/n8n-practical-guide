# Project 4: Gmail AI Assistant — Upgraded (Read, Summarize & Log to Google Sheets)

An upgraded version of the [Gmail AI Assistant (Project 1)](../project1/readme.md). The agent can now **read emails** in addition to sending them, and it is connected to a **Google Sheets tool** so it can log structured data directly into a spreadsheet. A typical command: *"Get my latest 50 emails and fill the sheet with the date, sender, subject, and a short summary for each one."*

![Workflow overview](/.imgs/demo1_2.png)

---

## Workflow Overview

```
Chat Trigger
  → AI Agent
      ├── Gmail (Read Emails)
      ├── Gmail (Send Email)
      └── Google Sheets (Append / Update Rows)
```

---

## What's New vs Project 1

| Capability | Project 1 | Project 4 |
|-----------|-----------|-----------|
| Send emails | ✅ | ✅ |
| Read / search inbox | ❌ | ✅ |
| Log to Google Sheets | ❌ | ✅ |

---

## Nodes

### 1. Chat Trigger

**Type:** n8n built-in trigger  
Same as Project 1 — provides the chat UI inside n8n. You type a natural-language instruction (e.g. *"Fetch the 50 most recent emails and log them to the sheet"*) and the agent takes it from there.

---

### 2. AI Agent

**Type:** AI Agent (LangChain)  
The reasoning core. It receives your message and decides which tools to call, in what order, and with what parameters. For the batch-log use case it will:

1. Call **Gmail – Get Many Messages** to retrieve the latest emails
2. For each email, extract the date, sender, subject, and generate a short summary
3. Call **Google Sheets – Append Row** once per email (or in a batch) to write the results

Key configuration:
- **Model** — connect an LLM (e.g., OpenAI GPT-4o, Google Gemini) via a credential
- **System Prompt** — instruct the agent to act as an email management assistant; tell it to populate every sheet column and to keep summaries to one or two sentences
- **Tools** — all three tools below are attached here

---

### 3. Gmail — Get Many Messages (Tool)

**Type:** Gmail node (used as an AI tool)  
Grants the agent the ability to **read** the inbox. Configure:

- **Operation** — `Get Many`
- **Max Results** — leave flexible so the agent can pass the count you specify in chat (e.g. 50)
- **Filters** — optional: unread only, label, date range

Uses the same **Gmail OAuth2** credential from Project 1. If you haven't set it up yet, follow the [Gmail credential steps in Project 1](../project1/readme.md#setting-up-gmail-api-credentials) — the only difference is that you also need the `gmail.readonly` scope (or `gmail.modify`) on your OAuth consent screen in addition to `gmail.send`.

---

### 4. Gmail — Send Email (Tool)

**Type:** Gmail node (used as an AI tool)  
Unchanged from Project 1 — lets the agent compose and send emails when asked. Attach the same credential.

---

### 5. Google Sheets — Append Row (Tool)

**Type:** Google Sheets node (used as an AI tool)  
Writes one row per email to your tracking spreadsheet. The expected columns are:

| Column | Content |
|--------|---------|
| **Date** | Date the email was received (ISO format) |
| **Sender** | From address |
| **Subject** | Email subject line |
| **Summary** | One-to-two sentence AI-generated summary of the email body |

Configure:
- **Spreadsheet ID** — paste from your Google Sheet URL
- **Sheet name** — name of the tab to write to
- **Operation** — `Append Row`

---

## Google Cloud Checklist

| API | Purpose |
|-----|---------|
| **Gmail API** | Read and send emails (add `gmail.readonly` scope) |
| **Google Sheets API** | Append rows to the log sheet |

If you already completed the Google Cloud setup for Project 1, just:
1. Go to **APIs & Services → Library** and enable the **Google Sheets API**
2. Go to **OAuth consent screen → Scopes** and add `https://www.googleapis.com/auth/spreadsheets`
3. Create a **Google Sheets OAuth2** credential in n8n (or extend the existing one if you use a combined credential)

---

## Example Prompts

```
Get my 50 latest emails and log each one to the sheet with date, sender, subject, and a short summary.
```

```
Find all unread emails from this week and fill the Google Sheet.
```

```
Summarize the email from alice@example.com about the budget meeting and send a reply confirming I'll attend.
```

---

## Operational Tips

- **Rate limits** — Gmail's API allows up to 250 quota units per second per user; processing 50 emails in one agent run is well within limits.
- **Token length** — For very long emails the agent will summarize the first portion. Instruct the model in the system prompt to summarize only the email body, not quoted threads.
- **Deduplication** — If you run the same prompt twice you'll get duplicate rows. Add a filter step or a sheet lookup tool to skip already-logged message IDs.
- **Privacy** — Email bodies pass through the LLM; make sure your model provider's data policy aligns with your organization's requirements.
