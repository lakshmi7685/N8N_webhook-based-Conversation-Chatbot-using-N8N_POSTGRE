# 🤖 Webhook-Based Conversational Chatbot using N8N & PostgreSQL

A fully functional AI Chatbot API built with **n8n**, **OpenAI**, and **PostgreSQL** — triggered via Webhook and capable of maintaining conversation history across sessions.

---

## 🚀 Features

- ✅ Accepts POST requests via Webhook
- ✅ Auto-generates or reuses Session IDs
- ✅ AI responses powered by OpenAI GPT
- ✅ Persistent conversation memory using PostgreSQL
- ✅ Stores full chat history per session
- ✅ Returns structured JSON response to caller

---

## 🏗️ Workflow Architecture

```
Webhook (POST)
    ↓
Session Manager (Code Node)
    ↓
AI Agent
  ├── OpenAI Chat Model (GPT)
  └── Postgres Chat Memory
    ↓
Execute SQL Query (Save chat history)
    ↓
Respond to Webhook
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation |
| **OpenAI GPT** | AI chat responses |
| **PostgreSQL** | Conversation memory + chat history |
| **Webhook** | API trigger |
| **Thunder Client / Postman** | API testing |

---

## ⚙️ Prerequisites

- [n8n](https://n8n.io/) installed locally or on cloud
- PostgreSQL database running
- OpenAI API key
- Node.js (for n8n)

---

## 🗄️ Database Setup

Run the following SQL in your PostgreSQL database:

### 1. Create Chat Sessions Table

```sql
CREATE TABLE chat_sessions (
  id SERIAL PRIMARY KEY,
  session_id VARCHAR(255) UNIQUE,
  conversation JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### 2. Add Unique Constraint (Required for ON CONFLICT)

```sql
ALTER TABLE chat_sessions 
ADD CONSTRAINT chat_sessions_session_id_unique 
UNIQUE (session_id);
```

---

## 📦 N8N Node Configuration

### 1. Webhook Node
| Setting | Value |
|---|---|
| HTTP Method | `POST` |
| Path | `chatbot` |
| Respond | `Using Respond to Webhook Node` |

### 2. Session Manager (Code Node)

```javascript
const body = $input.first().json.body || $input.first().json;
const incoming = body.sessionId || '';
const message = (body.message || body.chatInput || '').trim();

function generateSessionId() {
  const random = Math.random().toString(36).slice(2, 8);
  return `session_${Date.now()}_${random}`;
}

let sessionId;
if (message === '__new_session__' || !incoming) {
  sessionId = generateSessionId();
} else {
  sessionId = incoming;
}

return [{ json: { sessionId, chatInput: message } }];
```

### 3. AI Agent Node
- **Agent Type:** Conversational Agent
- **Chat Model:** OpenAI (connect to OpenAI Chat Model sub-node)
- **Memory:** Postgres Chat Memory (connect to PostgreSQL sub-node)

### 4. Execute SQL Query Node

```sql
INSERT INTO chat_sessions (session_id, conversation)
VALUES (
  '{{ $('Session Manager').item.json.sessionId }}',
  jsonb_build_array(
    jsonb_build_object(
      'turn', 1,
      'user', '{{ $('Session Manager').item.json.chatInput.replace(/'/g, "''") }}',
      'assistant', '{{ $('AI Agent').item.json.output.replace(/'/g, "''") }}',
      'timestamp', NOW()
    )
  )
)
ON CONFLICT (session_id)
DO UPDATE SET
  conversation = chat_sessions.conversation || jsonb_build_array(
    jsonb_build_object(
      'turn', jsonb_array_length(chat_sessions.conversation) + 1,
      'user', '{{ $('Session Manager').item.json.chatInput.replace(/'/g, "''") }}',
      'assistant', '{{ $('AI Agent').item.json.output.replace(/'/g, "''") }}',
      'timestamp', NOW()
    )
  ),
  updated_at = NOW();
```

### 5. Respond to Webhook Node
- **Respond With:** `First Incoming Item`

---

## 🧪 Testing the API

Use **Thunder Client**, **Postman**, or **curl**:

### New Session
```json
POST http://localhost:5678/webhook/chatbot
Content-Type: application/json

{
  "sessionId": "",
  "message": "Hello, who are you?"
}
```

### Continue Existing Session
```json
POST http://localhost:5678/webhook/chatbot
Content-Type: application/json

{
  "sessionId": "session_001",
  "message": "What did we discuss earlier?"
}
```

### Force New Session
```json
{
  "sessionId": "session_001",
  "message": "__new_session__"
}
```

### Using curl
```bash
curl -X POST http://localhost:5678/webhook/chatbot \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "", "message": "Hello!"}'
```

---

## 📋 Session ID Behavior

| Input | Result |
|---|---|
| Empty `sessionId` | Auto-generates new session |
| Existing `sessionId` | Continues that conversation |
| `__new_session__` message | Forces a new session |

---

## 🔧 Troubleshooting

| Error | Fix |
|---|---|
| `404 - Webhook not registered` | Click **Execute Workflow** or **Publish** the workflow |
| `No Respond to Webhook node found` | Add and connect **Respond to Webhook** node at the end |
| `500 - Internal Server Error` | Check JSON syntax in Respond to Webhook node |
| Duplicate session rows | Add UNIQUE constraint on `session_id` column |
| `Webhook (Deactivated)` | Right-click node → Enable, then Publish workflow |

---

## 📁 Project Structure

```
├── README.md
├── workflow/
│   └── Conversation-Chatbot-API.json   # n8n workflow export
└── sql/
    └── setup.sql                        # Database setup scripts
```

---

## 🔮 Future Improvements

- [ ] Add authentication to Webhook
- [ ] Build a frontend chat UI
- [ ] Deploy n8n to cloud (Railway / Render / AWS)
- [ ] Add rate limiting
- [ ] Support multiple AI models

---

## 🤝 Contributing

Pull requests are welcome! For major changes, please open an issue first.

---

## 📄 License

MIT License

---

## 👩‍💻 Author

**Lakshmi** — [GitHub](https://github.com/lakshmi7685)

---

⭐ If you found this helpful, please give it a star!
