# Project 1 : AI Gmail Assistant

An AI-powered chat assistant that can send emails on your behalf. You interact with it through n8n's built-in chat interface, give it natural language instructions, and it composes and sends Gmail messages using your connected Google account.

![Workflow overview](/.imgs/demo1.png)

---

## Workflow Overview

```
Chat Trigger → AI Agent → Gmail (Send Email)
```

---

## Nodes

### 1. Chat Trigger
**Type:** n8n built-in trigger  
Provides a chat interface directly inside n8n. When you send a message, it fires the workflow and passes your message as input to the AI Agent. No external service or webhook is needed — the chat UI is hosted by n8n itself.

### 2. AI Agent
**Type:** AI Agent (LangChain)  
The brain of the workflow. It receives your chat message, reasons about what to do, and decides which tools to call. It is configured with a **system prompt** that defines its role — for example, instructing it to act as a professional email assistant that writes and sends emails on your behalf.

The agent operates in a loop: it can call tools (like Gmail), observe the result, and continue reasoning until it has a final answer to return to the chat.

Key configuration:
- **Model** — connect an LLM (e.g., OpenAI GPT-4, Google Gemini) via a credential
- **System Prompt** — defines the assistant's behavior and constraints
- **Tools** — the Gmail tool is attached here, granting the agent the ability to send emails

### 3. Gmail — Send Email (Tool)
**Type:** Gmail node (used as an AI tool)  
Connected to the AI Agent as a callable tool. When the agent decides to send an email, it calls this node with the recipient, subject, and body it has composed. The node sends the email through the Gmail API using OAuth2 credentials.

---

## Setting Up Gmail API Credentials

n8n connects to Gmail via **OAuth2**, which requires creating a project in Google Cloud Console.

### Step 1 — Create a Google Cloud Project
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click **Select a project → New Project**, give it a name, and create it

### Step 2 — Enable the Gmail API
1. In your project, go to **APIs & Services → Library**
2. Search for **Gmail API** and click **Enable**

### Step 3 — Configure the OAuth Consent Screen
1. Go to **APIs & Services → OAuth consent screen**
2. Select **External**, fill in the app name and your email, and save
3. Under **Scopes**, add the Gmail scopes (at minimum `https://www.googleapis.com/auth/gmail.send`)
4. Under **Test users**, add the Google account you will use to send emails

### Step 4 — Create OAuth2 Credentials
1. Go to **APIs & Services → Credentials → Create Credentials → OAuth client ID**
2. Select **Web application**
3. Under **Authorized redirect URIs**, add your n8n OAuth callback URL:
   ```
   http://<your-n8n-host>:5678/rest/oauth2-credential/callback
   ```
4. Click **Create** and copy the **Client ID** and **Client Secret**

### Step 5 — Add the Credential in n8n
1. In n8n, go to **Credentials → New → Gmail OAuth2**
2. Paste your **Client ID** and **Client Secret**
3. Click **Connect** and complete the Google sign-in flow
4. The credential is now ready to attach to the Gmail node in your workflow
