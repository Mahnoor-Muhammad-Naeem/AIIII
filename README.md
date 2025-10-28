# WhatsApp Bot (n8n) – README

This repository contains an **n8n** workflow that powers a **WhatsApp Q&A bot** using an **AI Agent** (LangChain Agent node) with **Google Gemini Chat** as the language model and a **Buffer Window Memory** for short‑term conversation context.

> **What you get**
> - A minimal, production‑ready n8n flow for WhatsApp → AI → WhatsApp.
> - Clear node‑by‑node configuration notes.
> - Example **input & output**.
> - Linked **screenshots** of every step.

---

## 1) Architecture

**Trigger → Agent → WhatsApp reply** with memory:

```
WhatsApp Trigger  ──▶  AI Agent (LangChain)
                          ├─ Language Model: Google Gemini Chat
                          └─ Memory: Buffer Window (session by wa_id)
                   ──▶  WhatsApp: Send message
```

- **WhatsApp Trigger** receives inbound user messages via Meta’s Cloud API and forwards text to the Agent.
- **AI Agent** receives the raw text prompt and calls **Google Gemini Chat**.  
- **Simple Memory (Buffer Window)** stores the last few exchanges per user (`wa_id`) so the agent can answer follow‑ups naturally.
- **Send message** replies back to the original WhatsApp user with the model’s answer.

---

## 2) Screenshots

The following screenshots document the full setup and an example run.

- **Full Canvas (alt view)**  
  ![Full Canvas](sandbox:/mnt/data/WhatsApp%20Bot.png)

- **Workflow Canvas**  
  ![Workflow Canvas](sandbox:/mnt/data/whatsapp_bot.png)

- **Chat Received (Trigger)**  
  ![WhatsApp Trigger](sandbox:/mnt/data/chat%20received.png)

- **AI Agent (LangChain)**  
  ![AI Agent Node](sandbox:/mnt/data/ai%20agent.png)

- **Google Gemini Chat Model**  
  ![Gemini Model Node](sandbox:/mnt/data/google%20gemini.png)

- **Simple Memory (Buffer Window)**  
  ![Memory Node](sandbox:/mnt/data/simple%20memory.png)

- **Send Message (WhatsApp)**  
  ![Send Message Node](sandbox:/mnt/data/send%20message.png)

- **Agent Output (example)**  
  ![Agent Output JSON](sandbox:/mnt/data/ai%20agent%20output.png)

- **WhatsApp Chat (example)**  
  ![WhatsApp Chat](sandbox:/mnt/data/chat.png)

> If your viewer doesn’t inline images from `sandbox:`, open each link directly.

---

## 3) Node‑by‑Node Setup

### A. **WhatsApp Trigger** (`Chat Received1`)
- **Credential**: *WhatsApp OAuth account* (Meta Cloud API).
- **Trigger On**: `Messages`.
- The node outputs a structure with:
  - `contacts[0].wa_id`: The WhatsApp user ID.
  - `messages[0].text.body`: The inbound message text.

### B. **AI Agent** (`AI Agent`)
- **Prompt Source**: *Define below*  
  **Text**: `={{ $json.messages[0].text.body }}`
- **Language Model input**: Connected from **Google Gemini Chat Model**.
- **Memory**: Connected from **Simple Memory (Buffer Window)**.
- **What it does**: Forwards the user’s text to Gemini and returns the model’s answer as `output`.

### C. **Google Gemini Chat Model** (`Google Gemini Chat Model1`)
- **Model**: `models/gemini-2.5-flash` (fast, general chat model).
- **Credential**: *Google Gemini (PaLM) API account*.

### D. **Simple Memory (Buffer Window)** (`Simple Memory1`)
- **Session ID Type**: *Custom Key*  
  **Session Key**: `={{ $('Chat Received1').item.json.contacts[0].wa_id }}`  
  (This ensures per‑user conversation memory by WhatsApp `wa_id`.)
- Keeps a small rolling window of recent turns to make follow‑ups coherent.

### E. **Send Message** (`Send message`)
- **Operation**: `Send`
- **Sender Phone Number (ID)**: Your Meta Cloud **Phone Number ID**.
- **Recipient Phone Number**: From your test chat during development; in production, map it dynamically:
  ```
  ={{ $json.contacts[0].wa_id }}
  ```
- **Text Body**:
  ```
  =success {{ $json.output }}
  ```
  (Prefixes successful AI responses with `success ` for quick smoke‑testing.)

---

## 4) Example: Input & Output

### Example **Input** (from WhatsApp)
```text
Explain what APIs are in simple terms?
```

### Example **Output** (AI Agent → WhatsApp)
The agent returns a friendly explanation (abbreviated for readability):

```text
success Imagine you're at a restaurant.
1. You (your app/software) want some information/service.
2. You don't go into the kitchen and cook it yourself.
3. You look at the menu (what you can order and how).
4. You tell the waiter (the API) what you want from the menu.
5. The waiter takes your order to the kitchen (the server/other software).
6. The kitchen prepares your “food” (data/service).
7. The waiter brings your result back to you.

So, an API is like the “waiter + menu”: a standard way for software to request and receive data or actions from another system without knowing how the kitchen works inside.
```

---

## 5) Importing the Workflow into n8n

1. Open your n8n instance.
2. Create a new workflow and **Import from JSON**.
3. Paste your exported JSON (or use the block below as a reference).
4. Connect credentials for:
   - **WhatsApp OAuth account** (Trigger).
   - **WhatsApp account** (Send message).
   - **Google Gemini (PaLM) API account** (Model).
5. Click **Activate** and send a WhatsApp message to your connected number.

> **Tip:** During development, hardcode a test recipient in **Send message**.  
> In production, map to `{{$json.contacts[0].wa_id}}` so you always reply to the sender.

---

## 6) Workflow JSON (reference)

> **Note:** This is the *structure* you shared. Replace credential IDs with your own.  
> The important parts are the **prompt mapping**, **session key**, and **send message body**.

```json
{
  "name": "WhatsApp Bot",
  "nodes": [
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ $json.messages[0].text.body }}",
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 2.2,
      "position": [-192, -208],
      "id": "dd590067-db55-4c67-b1da-9deadd2fbc49",
      "name": "AI Agent"
    },
    {
      "parameters": {
        "operation": "send",
        "phoneNumberId": "=863930600134898",
        "recipientPhoneNumber": "923153154540",
        "textBody": "=success {{ $json.output }}",
        "additionalFields": {}
      },
      "type": "n8n-nodes-base.whatsApp",
      "typeVersion": 1,
      "position": [192, -208],
      "id": "46ad7a32-d70e-4254-b06d-deeffd0150e2",
      "name": "Send message",
      "webhookId": "5acd846a-51e0-40a0-8f9f-5d5d5f89b1d7"
    },
    {
      "parameters": {
        "updates": ["messages"],
        "options": {}
      },
      "type": "n8n-nodes-base.whatsAppTrigger",
      "typeVersion": 1,
      "position": [-416, -208],
      "id": "e9588a6d-9b5a-4e78-a0c2-70493bbb733f",
      "name": "Chat Received1",
      "webhookId": "86fd7fc1-d7e5-4fc4-a41c-69fad7c3b81f"
    },
    {
      "parameters": { "options": {} },
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [-272, 16],
      "id": "2ff1d9be-a758-457b-8845-19f2da932e90",
      "name": "Google Gemini Chat Model1"
    },
    {
      "parameters": {
        "sessionIdType": "customKey",
        "sessionKey": "={{ $('Chat Received1').item.json.contacts[0].wa_id }}"
      },
      "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
      "typeVersion": 1.3,
      "position": [-112, 48],
      "id": "b9416477-9a37-4859-8ed5-a21b674a8aca",
      "name": "Simple Memory1",
      "notesInFlow": false
    }
  ],
  "connections": {
    "AI Agent": { "main": [[{ "node": "Send message", "type": "main", "index": 0 }]] },
    "Chat Received1": { "main": [[{ "node": "AI Agent", "type": "main", "index": 0 }]] },
    "Google Gemini Chat Model1": { "ai_languageModel": [[{ "node": "AI Agent", "type": "ai_languageModel", "index": 0 }]] },
    "Simple Memory1": { "ai_memory": [[{ "node": "AI Agent", "type": "ai_memory", "index": 0 }]] }
  },
  "active": false,
  "settings": { "executionOrder": "v1" }
}
```

---

## 7) Troubleshooting

- **No reply on WhatsApp**  
  - Verify **phoneNumberId**, recipient mapping, and that the workflow is **active**.
  - Check Meta Cloud **webhook subscriptions** (the Trigger’s credential) and your n8n **webhook URL** is reachable.
- **The agent says “I have no name…”**  
  - That was just the model’s response to a specific input. The flow is working.
- **Memory doesn’t persist**  
  - Confirm `sessionKey` uses `contacts[0].wa_id]` and that **Simple Memory** is connected to the Agent.

---

## 8) License

MIT — use freely in your own projects.

---

**Happy building!**  
If you want a quick production template with environment variables and dynamic recipient mapping, drop this README into your repo and share the workflow JSON alongside it.
