# Project 3: Jenkins Log Analyzer (LLM + RAG)

When a **Jenkins pipeline fails**, your pipeline (or a shared library) sends an **HTTP POST** to an n8n **Webhook** URL with the **console log** (and useful metadata). That starts an **AI Agent** configured as a **log-analysis expert** for your stack. The agent uses a **Simple Vector Store** tool backed by your **internal knowledge base** (runbooks, past incidents, architecture notes) so answers are **RAG-grounded**. It produces a **root-cause style analysis and proposed fix**, then uses the **Gmail tool** to email a **formatted report** to the **team lead** or to the **person who triggered the build** (when that address is present in the webhook payload).

A **second path** keeps the knowledge base fresh: either an **admin Webhook** you call after publishing new docs, or a **Google Drive trigger** when files are uploaded to a watched folder. Those paths **chunk text, run the same embedding model** as the retrieval tool, and **upsert** into the same vector store the analyzer queries.


![Workflow overview](/.imgs/demo3.png)

---

## Workflow Overview

### A — Failure analysis (Jenkins → Webhook)

```
Jenkins (POST on failure: log + metadata)
  → Webhook
  → Set (normalize / map payload for the LLM)
  → AI Agent (log expert system prompt + Simple Vector Store tool)
  → Gmail (team lead and/or build initiator from payload)
```

### B — Knowledge base refresh (same embedding model as retrieval)

Either:

```
Webhook (manual / CI “reindex”)
  → … prepare documents …
  → Embeddings
  → Simple Vector Store (insert / upsert)
```

or:

```
Google Drive (file uploaded to KB folder)
  → Download file
  → Extract text
  → (optional chunk / split)
  → Embeddings
  → Simple Vector Store (insert / upsert)
```

Use **one embedding model ID / credential** for both **ingestion (B)** and the **Vector Store tool attached to the agent (A)** so query vectors live in the same space as document vectors.

Implementation detail: in n8n these are often **separate workflows** (one webhook URL for failures, one for indexing, one for Drive) — the diagram in [`.imgs/demo3.png`](../../.imgs/demo3.png) may show how yours are split.

---

## Nodes

### Reused from earlier projects (details there)

| Piece | Where it’s explained |
|--------|----------------------|
| **AI Agent** (model, system prompt, tool loop) | [Project 1 — AI Agent](../project1/readme.md#2-ai-agent) |
| **Gmail** (formatted email, OAuth2) | [Project 1 — Gmail tool](../project1/readme.md#3-gmail--send-email-tool) and [Gmail credentials](../project1/readme.md#setting-up-gmail-api-credentials) |
| **Google Drive** (trigger, download, extract) | [Project 2 — Drive + file pipeline](../project2/readme.md#1-google-drive--trigger) through [Extract from File](../project2/readme.md#3-extract-from-file) |

Configure the agent’s **system prompt** for your domain (e.g. Java/K8s/Docker at your org): require **sections** such as *Summary*, *Likely root cause*, *Evidence from log*, *Suggested fix*, *Queries for the vector store* when needed, and *Severity / next owner*.

---

### 1. Webhook — Jenkins failure

**Type:** Webhook (trigger)  
Exposes a **POST** URL. Jenkins (HTTP Request step, `curl`, or a notifier plugin) sends JSON or raw text containing at least the **log tail or full log**, **job name**, **build number**, **build URL**, and ideally **`startedBy` / `userEmail`** for the person who initiated the build.

**Security:** protect the URL (secret header, IP allowlist, or VPN-only n8n) so arbitrary callers cannot spam your LLM or inbox.

---

### 2. Set

**Type:** Set  
Maps the webhook body into **stable field names** the AI Agent expects (e.g. `logText`, `jobName`, `buildUrl`, `recipientEmail`). Truncate or split very large logs here if your model context window requires it, and attach a pointer to full logs (S3/Jenkins artifact URL) in the email body when you truncate.

---

### 3. Simple Vector Store (RAG tool)

**Type:** Simple Vector Store (LangChain / AI tool)  
Attached to the **AI Agent** as a tool the model can call to **retrieve** snippets from your knowledge base. The store must be populated by workflow **B** (see above).

**Critical:** use the **same embeddings configuration** (provider + model name + dimensions) for **ingestion** and **retrieval**, or similarity search will be meaningless.

---

### 4. Embeddings (ingestion path)

**Type:** Embeddings (e.g. OpenAI, Google, Cohere — per your n8n version)  
Converts each text chunk into a vector before writing to the Simple Vector Store. Wire the **identical** embedding node (or sub-workflow) used when configuring the agent’s vector tool.

---

### 5. Webhook — Reindex / vector store update

**Type:** Webhook (second workflow or second path)  
Lets CI or an admin **POST** new or replacement documents (or URLs to fetch) without using Drive. Often returns `202` quickly and processes async if reindexing is heavy.

---

### 6. Google Drive — KB uploads (optional second trigger)

**Type:** Google Drive trigger + Download + Extract  
Same pattern as [Project 2](../project2/readme.md): uploads to a **KB-only** folder automatically flow into chunk → embed → vector store. Keep this folder separate from unrelated Drive automation.

---

## Jenkins side (sketch)

From a pipeline `post { failure { ... } }` (or equivalent), call your n8n webhook with a JSON body, for example:

```json
{
  "jobName": "my-service",
  "buildNumber": 42,
  "buildUrl": "https://jenkins.example/job/my-service/42/",
  "logText": "... last N KB of console output ...",
  "startedByEmail": "developer@example.com"
}
```

Map `startedByEmail` (or a username resolved via Jenkins API) in the **Set** node so the **Gmail** tool can target the **build initiator**; fall back to a static **team lead** if the field is missing.

---

## Operational tips

- **Log size** — Full logs can exceed context limits; summarize in **Set** or a dedicated “compress log” LLM step only if needed, and always keep **build URL** in the email.
- **Vector hygiene** — Version or namespace documents (e.g. by `source_path`) so you can **delete/replace** stale chunks when runbooks change.
- **Cost & rate limits** — Every failure triggers an LLM + possible multiple vector queries; add **deduplication** (same `buildUrl` within minutes) if jobs are flaky and noisy.
- **Secrets** — Strip tokens and credentials from logs in **Set** before sending to the model or storing in email history.

---

## Next steps

1. Create the **Simple Vector Store** and run **workflow B** once with seed documents.  
2. Wire **Jenkins POST** to the failure **Webhook** and validate **Set → Agent → Gmail** with a forced failure.  

