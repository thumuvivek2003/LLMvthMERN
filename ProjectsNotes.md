Awesome—let’s build a tiny, **streaming chat UI** that talks to an LLM **through your own Node/Express proxy**. I’ll give you a copy-pasteable starter that works end-to-end, plus quick notes on why each piece exists so you learn the concepts as you go.

---

# 0) What you’ll get

* **Server**: `/api/chat` endpoint (Express) that:

  * Keeps your API key secret (`.env`)
  * Calls an LLM provider (OpenAI shown, but you can swap providers later)
  * **Streams** tokens back to the browser (SSE-style chunked text)
* **Client (React)**:

  * Minimal chat box
  * Shows messages and **streams** the assistant’s reply token-by-token
  * Clean, beginner-friendly state management

---

# 1) Project layout

```
hello-llm/
  server/
    package.json
    index.js
    .env               # create this
  client/
    package.json
    vite.config.js
    src/
      main.jsx
      App.jsx
      ChatMessage.jsx
      api.js
```

---

# 2) Server (Node + Express)

## 2.1 Install

```bash
mkdir -p hello-llm/server && cd hello-llm/server
npm init -y
npm i express cors dotenv
# optional: node-fetch if you’re on older Node; Node 18+ has fetch built-in
```

## 2.2 `.env`

Create `server/.env`:

```
OPENAI_API_KEY=sk-...your key...
PORT=5050
```

## 2.3 `index.js`

```js
import 'dotenv/config';
import express from 'express';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json());

// 👉 Health check
app.get('/health', (_req, res) => res.json({ ok: true }));

/**
 * POST /api/chat
 * Body: { messages: [{role: "user"|"assistant"|"system", content: string}] }
 * Streams text tokens back to the client in a single HTTP response.
 */
app.post('/api/chat', async (req, res) => {
  try {
    const { messages = [] } = req.body || {};
    if (!process.env.OPENAI_API_KEY) {
      res.status(500).json({ error: 'Missing OPENAI_API_KEY' });
      return;
    }

    // Prepare upstream request (OpenAI Chat Completions w/ SSE)
    const upstream = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: 'gpt-4o-mini',         // good, fast, cheap; change anytime
        stream: true,                 // <— request streaming
        temperature: 0.2,
        messages: [
          { role: 'system', content: 'You are a concise, friendly assistant.' },
          ...messages
        ],
      })
    });

    if (!upstream.ok || !upstream.body) {
      const text = await upstream.text().catch(() => '(no body)');
      res.status(500).json({ error: 'Upstream error', detail: text });
      return;
    }

    // Tell the browser we’re about to stream text chunks
    res.setHeader('Content-Type', 'text/event-stream; charset=utf-8');
    res.setHeader('Cache-Control', 'no-cache, no-transform');
    res.setHeader('Connection', 'keep-alive');

    // Parse OpenAI’s SSE and forward delta text chunks to client
    const reader = upstream.body.getReader();
    const decoder = new TextDecoder();

    let assistantTextSoFar = '';

    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      const chunk = decoder.decode(value, { stream: true });

      for (const line of chunk.split('\n')) {
        const trimmed = line.trim();
        if (!trimmed.startsWith('data:')) continue;
        const data = trimmed.slice(5).trim(); // after "data:"

        if (data === '[DONE]') {
          // Signal end of stream
          res.write(`event: done\ndata: ${JSON.stringify({ done: true })}\n\n`);
          break;
        }

        try {
          const json = JSON.parse(data);
          const delta = json.choices?.[0]?.delta?.content || '';
          if (delta) {
            assistantTextSoFar += delta;
            // Send just the text chunk to client
            res.write(`data: ${JSON.stringify({ delta })}\n\n`);
          }
        } catch {
          // ignore parse errors on keep-alive lines
        }
      }
    }

    res.end();
  } catch (err) {
    console.error(err);
    if (!res.headersSent) {
      res.status(500).json({ error: 'Server error', detail: String(err) });
    }
  }
});

const PORT = process.env.PORT || 5050;
app.listen(PORT, () => {
  console.log(`Server listening on http://localhost:${PORT}`);
});
```

> ✅ Why this design?
>
> * **Security**: The browser talks to **your** server; the API key never leaves the backend.
> * **Streaming**: We request `stream: true` and forward each token chunk as it arrives.
> * **SSE-like**: We set `Content-Type: text/event-stream` and write lines; the client reads them with `ReadableStream` via `fetch`.

---

# 3) Client (React + Vite)

## 3.1 Create app & install

```bash
cd ../
npm create vite@latest client -- --template react
cd client
npm i
```

If you want pretty fonts / minimal styles you can add later—keeping it bare-bones for learning.

## 3.2 `src/api.js`

A tiny helper to call the proxy and **read the stream**:

```js
export async function streamChat(messages, onDelta) {
  const resp = await fetch('http://localhost:5050/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages }),
  });

  if (!resp.ok || !resp.body) {
    const text = await resp.text().catch(() => '(no body)');
    throw new Error(`Proxy error: ${text}`);
  }

  const reader = resp.body.getReader();
  const decoder = new TextDecoder();
  let buf = '';

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    buf += decoder.decode(value, { stream: true });

    // Our server sends lines like: `data: {"delta":"..."}` + \n\n
    let sep;
    while ((sep = buf.indexOf('\n\n')) !== -1) {
      const packet = buf.slice(0, sep);
      buf = buf.slice(sep + 2);

      if (packet.startsWith('data:')) {
        const payload = packet.slice(5).trim();
        if (payload) {
          try {
            const { delta } = JSON.parse(payload);
            if (delta) onDelta(delta);
          } catch {
            // ignore bad lines
          }
        }
      }
    }
  }
}
```

## 3.3 `src/ChatMessage.jsx`

```jsx
export default function ChatMessage({ role, content }) {
  const isUser = role === 'user';
  return (
    <div style={{
      alignSelf: isUser ? 'flex-end' : 'flex-start',
      background: isUser ? '#e8f0fe' : '#f5f5f5',
      borderRadius: 12,
      padding: '10px 12px',
      margin: '6px 0',
      maxWidth: 560,
      whiteSpace: 'pre-wrap',
      lineHeight: 1.5
    }}>
      <strong style={{ opacity: 0.7 }}>{isUser ? 'You' : 'Assistant'}</strong>
      <div>{content}</div>
    </div>
  );
}
```

## 3.4 `src/App.jsx`

```jsx
import { useState, useRef } from 'react';
import ChatMessage from './ChatMessage.jsx';
import { streamChat } from './api.js';

export default function App() {
  const [messages, setMessages] = useState([
    { role: 'assistant', content: 'Hi! Ask me anything 🤖' }
  ]);
  const [input, setInput] = useState('');
  const [isStreaming, setIsStreaming] = useState(false);
  const streamAbortRef = useRef(null);

  async function handleSend(e) {
    e.preventDefault();
    const userText = input.trim();
    if (!userText || isStreaming) return;

    const next = [...messages, { role: 'user', content: userText }];
    setMessages(next);
    setInput('');
    setIsStreaming(true);

    // placeholder assistant message we’ll stream into
    const idx = next.length;
    setMessages(prev => [...prev, { role: 'assistant', content: '' }]);

    let assistantText = '';

    try {
      await streamChat(next, (delta) => {
        assistantText += delta;
        // Append streamed delta into the last assistant message
        setMessages(prev => {
          const copy = prev.slice();
          copy[idx] = { role: 'assistant', content: assistantText };
          return copy;
        });
      });
    } catch (err) {
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: `⚠️ Error: ${String(err.message || err)}`
      }]);
    } finally {
      setIsStreaming(false);
    }
  }

  return (
    <div style={{
      display: 'flex',
      flexDirection: 'column',
      height: '100vh',
      fontFamily: 'system-ui, sans-serif'
    }}>
      <header style={{
        padding: '12px 16px',
        borderBottom: '1px solid #eee',
        fontWeight: 600
      }}>
        Chat UI — Hello LLM
      </header>

      <main style={{
        flex: 1,
        overflow: 'auto',
        display: 'flex',
        flexDirection: 'column',
        gap: 6,
        padding: 16
      }}>
        {messages.map((m, i) => (
          <ChatMessage key={i} role={m.role} content={m.content} />
        ))}
      </main>

      <form onSubmit={handleSend} style={{
        display: 'flex',
        gap: 8,
        padding: 12,
        borderTop: '1px solid #eee'
      }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder={isStreaming ? 'Generating…' : 'Type a message'}
          disabled={isStreaming}
          style={{
            flex: 1, padding: '10px 12px', borderRadius: 10,
            border: '1px solid #ddd', outline: 'none'
          }}
        />
        <button
          type="submit"
          disabled={isStreaming || !input.trim()}
          style={{
            padding: '10px 14px', borderRadius: 10,
            border: '1px solid #ddd', background: '#111', color: 'white'
          }}
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

## 3.5 `src/main.jsx`

*(Vite’s default is fine—just ensure it renders `<App />`)*

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

## 3.6 Run it

In one terminal:

```bash
cd hello-llm/server
node index.js
```

In another:

```bash
cd hello-llm/client
npm run dev
```

Open the shown localhost URL and chat away.

---

# 4) Quick test (no React)

From a terminal you can sanity-check the server streaming:

```bash
curl -N -X POST http://localhost:5050/api/chat \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Say hello in 5 words."}]}'
```

You should see `data: {"delta":"..."}` lines arriving until the stream ends.

---

# 5) Understanding the moving parts

* **Why a proxy?** Browsers must not hold your secret keys. The proxy also lets you add auth, rate limits, logging, and swap providers later without touching the frontend.
* **Streaming**:

  * We ask the provider for `stream: true`.
  * They send **SSE** packets (lines prefixed by `data:`). We forward those to the browser as they arrive.
  * The React app **accumulates** `delta` text into the latest assistant message for a smooth typing effect.
* **Prompt design (minimal)**:

  * Start with a short **system message** (tone, role).
  * Add each user turn as `{role:'user', content:'...'}`.
  * If you want to “ground” the model (e.g., instructions or app rules), inject it once in the system message.

**Example system prompt** (already included):

> “You are a concise, friendly assistant.”

Later, you can enrich with: formatting rules, refusal policies, or app-specific knowledge.

---

# 6) Common pitfalls & fixes

* **CORS errors**: Ensure `app.use(cors())` on the server. If hosting, set allowed origins explicitly.
* **No stream on client**: Double-check:

  * Server sets `Content-Type: text/event-stream`
  * You **don’t** call `await resp.text()` before reading the stream
  * Use `resp.body.getReader()` and parse `\n\n` boundaries
* **Long responses cut off**: Most often a hosting/proxy timeout. For local dev you’re fine; in prod, keep-alive and timeouts must be generous.
* **Model/provider swapping**: Keep your `/api/chat` shape the same; only change the upstream fetch.

---

# 7) Your next tiny improvements (pick any)

* “**Enter to send**”, “Shift+Enter for newline”
* “**Stop generating**” button (abort controller, cancel the read loop)
* **Message history** in localStorage
* **System prompt** textbox in the UI
* **Role badges** & timestamps
* **Loading dots** while first tokens arrive

---

If you want, I can tailor this to **MERN** hosting (Render/Heroku + Netlify/Vercel), add **Abort** support, or switch providers (Groq, Together, etc.). Want me to add a “Stop” button and a system-prompt editor next?



# Prompt Playground (MERN) — Starter Kit

A minimal, copy‑pasteable Prompt Playground that lets you tweak **system/user prompts**, **model**, **temperature/top_p**, and **save/load presets** in **MongoDB**. Includes an **Express proxy** to call an LLM (OpenAI shown) with optional **streaming**.

---

## 0) What you’ll build

* **Backend (Node/Express + MongoDB/Mongoose)**

  * `/api/presets` CRUD
  * `/api/run` to call the LLM (non‑stream or `?stream=1` for SSE streaming)
* **Frontend (React + Vite)**

  * Form for system/user prompts, model, temperature, top_p
  * "Run" to see model output
  * Save, update, delete presets

---

## 1) Folder layout

```
prompt-playground/
  server/
    package.json
    .env
    server.js
    models/Preset.js
    routes/presets.js
    routes/run.js
  client/
    package.json
    vite.config.js
    index.html
    src/
      main.jsx
      App.jsx
      api.js
      components/
        PresetForm.jsx
        PresetList.jsx
        Playground.jsx
        Slider.jsx
        TextArea.jsx
```

---

## 2) Backend (Express + MongoDB)

### 2.1 Install

```bash
mkdir -p prompt-playground/server && cd prompt-playground/server
npm init -y
npm i express cors dotenv mongoose
```

### 2.2 `.env`

```
PORT=5051
MONGO_URL=mongodb://localhost:27017/prompt_playground
OPENAI_API_KEY=sk-...your key...
```

### 2.3 `package.json`

```json
{
  "name": "prompt-playground-server",
  "version": "1.0.0",
  "type": "module",
  "main": "server.js",
  "scripts": {
    "dev": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "mongoose": "^8.4.0"
  }
}
```

### 2.4 `server.js`

```js
import 'dotenv/config';
import express from 'express';
import cors from 'cors';
import mongoose from 'mongoose';
import presetsRouter from './routes/presets.js';
import runRouter from './routes/run.js';

const app = express();
app.use(cors());
app.use(express.json({ limit: '1mb' }));

app.get('/health', (_req, res) => res.json({ ok: true }));
app.use('/api/presets', presetsRouter);
app.use('/api/run', runRouter);

const PORT = process.env.PORT || 5051;

try {
  await mongoose.connect(process.env.MONGO_URL, { dbName: 'prompt_playground' });
  console.log('MongoDB connected');
  app.listen(PORT, () => console.log(`Server listening on http://localhost:${PORT}`));
} catch (err) {
  console.error('Mongo connect error', err);
  process.exit(1);
}
```

### 2.5 `models/Preset.js`

```js
import mongoose from 'mongoose';

const PresetSchema = new mongoose.Schema(
  {
    name: { type: String, required: true, index: true },
    systemPrompt: { type: String, default: '' },
    userPrompt: { type: String, default: '' },
    model: { type: String, default: 'gpt-4o-mini' },
    temperature: { type: Number, default: 0.7, min: 0, max: 2 },
    top_p: { type: Number, default: 1, min: 0, max: 1 },
    tags: { type: [String], default: [] },
    notes: { type: String, default: '' }
  },
  { timestamps: true }
);

PresetSchema.index({ name: 1, updatedAt: -1 });

export default mongoose.model('Preset', PresetSchema);
```

### 2.6 `routes/presets.js`

```js
import { Router } from 'express';
import Preset from '../models/Preset.js';

const r = Router();

// List with optional search by name/tag
r.get('/', async (req, res) => {
  const { q } = req.query;
  const filter = q
    ? { $or: [ { name: new RegExp(q, 'i') }, { tags: q } ] }
    : {};
  const items = await Preset.find(filter).sort({ updatedAt: -1 }).limit(200);
  res.json(items);
});

// Read one
r.get('/:id', async (req, res) => {
  const item = await Preset.findById(req.params.id);
  if (!item) return res.status(404).json({ error: 'Not found' });
  res.json(item);
});

// Create
r.post('/', async (req, res) => {
  const body = req.body || {};
  const item = await Preset.create({
    name: body.name?.trim() || 'Untitled',
    systemPrompt: body.systemPrompt || '',
    userPrompt: body.userPrompt || '',
    model: body.model || 'gpt-4o-mini',
    temperature: Number(body.temperature ?? 0.7),
    top_p: Number(body.top_p ?? 1),
    tags: Array.isArray(body.tags) ? body.tags : [],
    notes: body.notes || ''
  });
  res.status(201).json(item);
});

// Update
r.put('/:id', async (req, res) => {
  const body = req.body || {};
  const item = await Preset.findByIdAndUpdate(
    req.params.id,
    {
      $set: {
        name: body.name,
        systemPrompt: body.systemPrompt,
        userPrompt: body.userPrompt,
        model: body.model,
        temperature: Number(body.temperature),
        top_p: Number(body.top_p),
        tags: body.tags,
        notes: body.notes
      }
    },
    { new: true }
  );
  if (!item) return res.status(404).json({ error: 'Not found' });
  res.json(item);
});

// Delete
r.delete('/:id', async (req, res) => {
  const out = await Preset.findByIdAndDelete(req.params.id);
  if (!out) return res.status(404).json({ error: 'Not found' });
  res.json({ ok: true });
});

export default r;
```

### 2.7 `routes/run.js`

```js
import { Router } from 'express';

const r = Router();

// Helper to call OpenAI Chat Completions
async function openaiChat({ model, systemPrompt, userPrompt, temperature, top_p, stream }) {
  const headers = {
    'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
    'Content-Type': 'application/json'
  };
  const body = {
    model: model || 'gpt-4o-mini',
    temperature: typeof temperature === 'number' ? temperature : 0.7,
    top_p: typeof top_p === 'number' ? top_p : 1,
    messages: [
      ...(systemPrompt ? [{ role: 'system', content: systemPrompt }] : []),
      { role: 'user', content: userPrompt || '' }
    ],
    stream: !!stream
  };

  const resp = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST', headers, body: JSON.stringify(body)
  });
  return resp;
}

// POST /api/run
// Body: { model, systemPrompt, userPrompt, temperature, top_p }
// Optional: ?stream=1 for SSE
r.post('/', async (req, res) => {
  const { model, systemPrompt, userPrompt, temperature, top_p } = req.body || {};
  const wantStream = String(req.query.stream || '') === '1';

  if (!process.env.OPENAI_API_KEY) {
    return res.status(500).json({ error: 'Missing OPENAI_API_KEY' });
  }

  const upstream = await openaiChat({ model, systemPrompt, userPrompt, temperature, top_p, stream: wantStream });

  if (!upstream.ok) {
    const text = await upstream.text().catch(() => '(no body)');
    return res.status(500).json({ error: 'Upstream error', detail: text });
  }

  if (!wantStream) {
    const json = await upstream.json();
    const content = json.choices?.[0]?.message?.content || '';
    return res.json({ content, raw: json });
  }

  // Streaming (SSE-like)
  res.setHeader('Content-Type', 'text/event-stream; charset=utf-8');
  res.setHeader('Cache-Control', 'no-cache, no-transform');
  res.setHeader('Connection', 'keep-alive');

  if (!upstream.body) {
    res.write(`event: error\ndata: ${JSON.stringify({ error: 'No body' })}\n\n`);
    return res.end();
  }

  const reader = upstream.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    const chunk = decoder.decode(value, { stream: true });
    for (const line of chunk.split('\n')) {
      const trimmed = line.trim();
      if (!trimmed.startsWith('data:')) continue;
      const data = trimmed.slice(5).trim();
      if (data === '[DONE]') {
        res.write(`event: done\ndata: {"done":true}\n\n`);
        break;
      }
      try {
        const json = JSON.parse(data);
        const delta = json.choices?.[0]?.delta?.content || '';
        if (delta) res.write(`data: ${JSON.stringify({ delta })}\n\n`);
      } catch {
        // ignore
      }
    }
  }

  res.end();
});

export default r;
```

---

## 3) Frontend (React + Vite)

### 3.1 Create app & install

```bash
cd ../
npm create vite@latest client -- --template react
cd client
npm i
```

### 3.2 `package.json`

```json
{
  "name": "prompt-playground-client",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "vite": "^5.4.0"
  }
}
```

### 3.3 `src/api.js`

```js
const BASE = 'http://localhost:5051';

export async function listPresets(q = '') {
  const url = q ? `${BASE}/api/presets?q=${encodeURIComponent(q)}` : `${BASE}/api/presets`;
  const r = await fetch(url);
  if (!r.ok) throw new Error('Failed to list presets');
  return r.json();
}

export async function createPreset(data) {
  const r = await fetch(`${BASE}/api/presets`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(data)
  });
  if (!r.ok) throw new Error('Failed to create');
  return r.json();
}

export async function updatePreset(id, data) {
  const r = await fetch(`${BASE}/api/presets/${id}`, {
    method: 'PUT', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(data)
  });
  if (!r.ok) throw new Error('Failed to update');
  return r.json();
}

export async function deletePreset(id) {
  const r = await fetch(`${BASE}/api/presets/${id}`, { method: 'DELETE' });
  if (!r.ok) throw new Error('Failed to delete');
  return r.json();
}

export async function runOnce(payload) {
  const r = await fetch(`${BASE}/api/run`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload)
  });
  if (!r.ok) throw new Error('Run failed');
  return r.json();
}

export async function runStream(payload, onDelta) {
  const r = await fetch(`${BASE}/api/run?stream=1`, {
    method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload)
  });
  if (!r.ok || !r.body) throw new Error('Stream failed');

  const reader = r.body.getReader();
  const decoder = new TextDecoder();
  let buf = '';

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buf += decoder.decode(value, { stream: true });
    let sep;
    while ((sep = buf.indexOf('\n\n')) !== -1) {
      const packet = buf.slice(0, sep);
      buf = buf.slice(sep + 2);
      if (packet.startsWith('data:')) {
        const payload = packet.slice(5).trim();
        if (!payload) continue;
        try {
          const { delta } = JSON.parse(payload);
          if (delta) onDelta(delta);
        } catch {}
      }
    }
  }
}
```

### 3.4 `src/components/Slider.jsx`

```jsx
export default function Slider({ label, min, max, step=0.1, value, onChange }) {
  return (
    <label style={{ display: 'block', marginBottom: 8 }}>
      <div style={{ fontSize: 12, opacity: 0.7 }}>{label}: <b>{value}</b></div>
      <input type="range" min={min} max={max} step={step}
        value={value} onChange={e => onChange(Number(e.target.value))}
        style={{ width: '100%' }} />
    </label>
  );
}
```

### 3.5 `src/components/TextArea.jsx`

```jsx
export default function TextArea({ label, value, onChange, rows=6, placeholder }) {
  return (
    <label style={{ display: 'block', marginBottom: 10 }}>
      <div style={{ fontSize: 12, opacity: 0.7, marginBottom: 4 }}>{label}</div>
      <textarea value={value} onChange={e => onChange(e.target.value)} rows={rows}
        placeholder={placeholder}
        style={{ width: '100%', padding: 8, borderRadius: 8, border: '1px solid #ddd', fontFamily: 'monospace' }} />
    </label>
  );
}
```

### 3.6 `src/components/PresetForm.jsx`

```jsx
import Slider from './Slider.jsx';
import TextArea from './TextArea.jsx';

export default function PresetForm({ preset, setPreset, onRun, running }) {
  return (
    <div style={{ display:'grid', gap: 12 }}>
      <label>
        <div style={{ fontSize: 12, opacity: 0.7 }}>Name</div>
        <input value={preset.name} onChange={e=>setPreset({...preset, name: e.target.value})}
          placeholder="My preset"
          style={{ width:'100%', padding:8, border:'1px solid #ddd', borderRadius:8 }} />
      </label>

      <label>
        <div style={{ fontSize: 12, opacity: 0.7 }}>Model</div>
        <input value={preset.model} onChange={e=>setPreset({...preset, model: e.target.value})}
          placeholder="gpt-4o-mini"
          style={{ width:'100%', padding:8, border:'1px solid #ddd', borderRadius:8 }} />
      </label>

      <Slider label="Temperature" min={0} max={2} step={0.1}
        value={preset.temperature}
        onChange={v=>setPreset({...preset, temperature: v})} />

      <Slider label="top_p" min={0} max={1} step={0.05}
        value={preset.top_p}
        onChange={v=>setPreset({...preset, top_p: v})} />

      <TextArea label="System Prompt" rows={4}
        value={preset.systemPrompt}
        placeholder="You are a helpful assistant..."
        onChange={v=>setPreset({...preset, systemPrompt: v})} />

      <TextArea label="User Prompt" rows={8}
        value={preset.userPrompt}
        placeholder="Explain temperature vs top_p to a beginner."
        onChange={v=>setPreset({...preset, userPrompt: v})} />

      <div style={{ display:'flex', gap:8 }}>
        <button onClick={() => onRun(false)} disabled={running}
          style={{ padding:'10px 12px', borderRadius:8, border:'1px solid #ddd', background:'#111', color:'#fff' }}>
          Run (no stream)
        </button>
        <button onClick={() => onRun(true)} disabled={running}
          style={{ padding:'10px 12px', borderRadius:8, border:'1px solid #ddd' }}>
          Run (stream)
        </button>
      </div>
    </div>
  );
}
```

### 3.7 `src/components/PresetList.jsx`

```jsx
export default function PresetList({ items, onSelect, onDelete }) {
  return (
    <div style={{ display:'grid', gap:6 }}>
      {items.map(p => (
        <div key={p._id} style={{ border:'1px solid #eee', borderRadius:10, padding:8 }}>
          <div style={{ fontWeight:600 }}>{p.name}</div>
          <div style={{ fontSize:12, opacity:0.7 }}>{p.model} · t={p.temperature} · p={p.top_p}</div>
          <div style={{ display:'flex', gap:8, marginTop:6 }}>
            <button onClick={() => onSelect(p)} style={{ padding:'6px 10px', borderRadius:8, border:'1px solid #ddd' }}>Load</button>
            <button onClick={() => onDelete(p._id)} style={{ padding:'6px 10px', borderRadius:8, border:'1px solid #f3c'}}>
              Delete
            </button>
          </div>
        </div>
      ))}
      {items.length === 0 && <div style={{ opacity:0.6 }}>No presets yet</div>}
    </div>
  );
}
```

### 3.8 `src/components/Playground.jsx`

```jsx
export default function Playground({ output, streaming }) {
  return (
    <div style={{ border:'1px solid #eee', borderRadius:10, padding:10, minHeight:200, whiteSpace:'pre-wrap' }}>
      <div style={{ fontSize:12, opacity:0.7, marginBottom:6 }}>Output</div>
      <div>{output}</div>
      {streaming && <div style={{ opacity:0.6, marginTop:8 }}>…streaming</div>}
    </div>
  );
}
```

### 3.9 `src/main.jsx`

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### 3.10 `src/App.jsx`

```jsx
import { useEffect, useMemo, useState } from 'react';
import { listPresets, createPreset, updatePreset, deletePreset, runOnce, runStream } from './api.js';
import PresetForm from './components/PresetForm.jsx';
import PresetList from './components/PresetList.jsx';
import Playground from './components/Playground.jsx';

const EMPTY = { name:'New preset', model:'gpt-4o-mini', temperature:0.7, top_p:1, systemPrompt:'', userPrompt:'', tags:[], notes:'' };

export default function App() {
  const [presets, setPresets] = useState([]);
  const [preset, setPreset] = useState(EMPTY);
  const [output, setOutput] = useState('');
  const [running, setRunning] = useState(false);

  async function refresh(q='') {
    const data = await listPresets(q).catch(()=>[]);
    setPresets(data);
  }

  useEffect(() => { refresh(); }, []);

  async function save() {
    if (preset._id) {
      const updated = await updatePreset(preset._id, preset);
      setPreset(updated);
      await refresh();
    } else {
      const created = await createPreset(preset);
      setPreset(created);
      await refresh();
    }
  }

  async function remove(id) {
    await deletePreset(id);
    if (preset._id === id) setPreset(EMPTY);
    await refresh();
  }

  async function onRun(stream) {
    setRunning(true);
    setOutput('');
    const payload = {
      model: preset.model,
      systemPrompt: preset.systemPrompt,
      userPrompt: preset.userPrompt,
      temperature: preset.temperature,
      top_p: preset.top_p
    };

    try {
      if (!stream) {
        const res = await runOnce(payload);
        setOutput(res.content || '');
      } else {
        let acc = '';
        await runStream(payload, (delta) => {
          acc += delta;
          setOutput(acc);
        });
      }
    } catch (e) {
      setOutput('⚠️ ' + (e?.message || String(e)));
    } finally {
      setRunning(false);
    }
  }

  return (
    <div style={{ display:'grid', gridTemplateColumns:'280px 1fr 1fr', gap:16, height:'100vh', padding:16, boxSizing:'border-box', fontFamily:'system-ui, sans-serif' }}>
      <div style={{ display:'grid', gridTemplateRows:'auto 1fr auto', gap:12 }}>
        <div style={{ fontWeight:700 }}>Presets</div>
        <div style={{ overflow:'auto' }}>
          <PresetList items={presets} onSelect={setPreset} onDelete={remove} />
        </div>
        <div style={{ display:'flex', gap:8 }}>
          <button onClick={() => setPreset(EMPTY)} style={{ flex:1, padding:'8px 10px', border:'1px solid #ddd', borderRadius:8 }}>New</button>
          <button onClick={save} style={{ flex:1, padding:'8px 10px', border:'1px solid #ddd', borderRadius:8, background:'#111', color:'#fff' }}>Save</button>
        </div>
      </div>

      <div style={{ overflow:'auto' }}>
        <div style={{ fontWeight:700, marginBottom:8 }}>Editor</div>
        <PresetForm preset={preset} setPreset={setPreset} onRun={onRun} running={running} />
      </div>

      <div style={{ display:'grid', gridTemplateRows:'auto 1fr', gap:8 }}>
        <div style={{ fontWeight:700 }}>Playground</div>
        <Playground output={output} streaming={running} />
      </div>
    </div>
  );
}
```

### 3.11 `index.html`

```html
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Prompt Playground</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

---

## 4) Run locally

In one terminal (MongoDB must be running):

```bash
cd prompt-playground/server
node server.js
```

In another terminal:

```bash
cd prompt-playground/client
npm run dev
```

Open the shown localhost URL.

---

## 5) Learning checkpoints

* **Prompt engineering basics**: Edit **system** vs **user** prompts; try higher **temperature** for creative, lower for deterministic; **top_p** to narrow/widen sampling.
* **Schema for experiments**: The `Preset` model captures all knobs so you can version experiments.
* **Form handling**: Controlled inputs + small components (Slider/TextArea) keep it simple.
* **Persistence**: CRUD to Mongo lets you share & revisit presets.
* **Proxying**: `/api/run` keeps keys secret and lets you toggle streaming.

---

## 6) Next steps (optional improvements)

* Add **tags + search box** in UI
* Add **export/import JSON** for presets
* Store **last output** with each preset for quick recall
* Add **Abort** button for streamed runs (AbortController)
* Add **model dropdown** and validation on server
* Multi‑message runs (history array) instead of single user prompt
* Auth (GitHub/Google) if you deploy



Amazing—great project choice. I’ve dropped a full **MERN Prompt Playground starter** into the canvas with copy-paste code (backend + frontend), plus run instructions and “next steps.” It covers:

* Mongo model + Express routes for **/api/presets** (CRUD)
* **/api/run** endpoint that calls the LLM (OpenAI), with **non-stream** or **SSE streaming** output
* React UI to edit **system/user prompts**, **temperature/top_p**, choose model, **save/load presets**, and **Run** (stream or not)

### How to use it now

1. Start MongoDB locally.
2. In `server/.env`, set:

```
PORT=5051
MONGO_URL=mongodb://localhost:27017/prompt_playground
OPENAI_API_KEY=sk-...your key...
```

3. Run backend:

```
cd prompt-playground/server
npm i
node server.js
```

4. Run frontend:

```
cd prompt-playground/client
npm i
npm run dev
```

Open the shown localhost URL. Create a preset, tweak prompts/temperature/top_p, and click **Run (stream)** to watch tokens arrive.

If you want, I can add an **Abort** button, **model dropdown** with safe defaults, or **export/import presets** next.



# Webpage Summarizer (MERN) — Starter

A copy‑pasteable MERN mini‑app: **paste a URL → server fetches HTML, cleans text, LLM summarizes with bullet options**. You’ll practice **server‑side scraping**, **text cleaning**, **chunking/token budgeting**, and **concise prompting**.

---

## 0) What you’ll build

* **Backend (Express)**

  * `/api/summarize` → POST `{ url, bullets, length, includeQuotes, model }`
  * Fetches HTML, extracts main content (Readability), chunks, map‑reduce summarizes via OpenAI.
* **Frontend (React + Vite)**

  * Form to paste URL + options, shows clean summary.

---

## 1) Folder layout

```
web-summarizer/
  server/
    package.json
    .env
    server.js
    routes/summarize.js
    utils/fetchAndExtract.js
    utils/chunk.js
    utils/summarizeLLM.js
  client/
    package.json
    vite.config.js
    index.html
    src/
      main.jsx
      App.jsx
      api.js
```

---

## 2) Backend (Express)

### 2.1 Install

```bash
mkdir -p web-summarizer/server && cd web-summarizer/server
npm init -y
npm i express cors dotenv jsdom @mozilla/readability html-entities
```

> Node 18+ has `fetch` built‑in. If you’re on older Node: `npm i node-fetch` and import it.

### 2.2 `.env`

```
PORT=5052
OPENAI_API_KEY=sk-...your key...
```

### 2.3 `package.json`

```json
{
  "name": "web-summarizer-server",
  "version": "1.0.0",
  "type": "module",
  "main": "server.js",
  "scripts": {
    "dev": "node server.js"
  },
  "dependencies": {
    "@mozilla/readability": "^0.5.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "html-entities": "^2.5.2",
    "jsdom": "^24.0.0"
  }
}
```

### 2.4 `server.js`

```js
import 'dotenv/config';
import express from 'express';
import cors from 'cors';
import summarizeRouter from './routes/summarize.js';

const app = express();
app.use(cors());
app.use(express.json({ limit: '1mb' }));

app.get('/health', (_req, res) => res.json({ ok: true }));
app.use('/api/summarize', summarizeRouter);

const PORT = process.env.PORT || 5052;
app.listen(PORT, () => console.log(`Server listening on http://localhost:${PORT}`));
```

### 2.5 `utils/fetchAndExtract.js`

```js
import { JSDOM } from 'jsdom';
import { Readability } from '@mozilla/readability';
import { AllHtmlEntities } from 'html-entities';
const entities = new AllHtmlEntities();

export async function fetchAndExtract(url) {
  const resp = await fetch(url, { headers: { 'User-Agent': 'WebSummarizer/1.0' } });
  if (!resp.ok) throw new Error(`Fetch failed: ${resp.status}`);
  const html = await resp.text();

  // Basic protection: bail on extremely small pages
  if (!html || html.length < 200) return { title: url, text: '' };

  const dom = new JSDOM(html, { url });
  const document = dom.window.document;

  // Remove noisy nodes before Readability
  const selectors = ['script', 'style', 'noscript', 'iframe', 'header nav', 'footer', 'form', 'aside', '.advert', '.ad', '.promo'];
  selectors.forEach(sel => document.querySelectorAll(sel).forEach(n => n.remove()));

  const reader = new Readability(document);
  const article = reader.parse();

  let title = article?.title || document.title || url;
  let content = article?.textContent || document.body?.textContent || '';

  // Normalize whitespace & entities
  content = entities.decode(content)
    .replace(/\u00a0/g, ' ')
    .replace(/[\t\r]+/g, ' ')
    .replace(/\n{3,}/g, '\n\n')
    .trim();

  // Very long pages sometimes include cookie / nav leftovers
  // Keep only lines with some letters and at least 30 chars
  const lines = content.split('\n').map(l => l.trim());
  const filtered = lines.filter(l => /[A-Za-z]/.test(l) && l.length >= 30);
  const text = filtered.join('\n\n');

  return { title, text };
}
```

### 2.6 `utils/chunk.js`

```js
// Rough token budgeting: ~4 chars ≈ 1 token (very approximate)
const MAX_CHARS_PER_CHUNK = 4000 * 4; // ~4k tokens

export function chunkText(text) {
  if (!text) return [];
  if (text.length <= MAX_CHARS_PER_CHUNK) return [text];

  // Split on paragraph boundaries first
  const paras = text.split(/\n\n+/);
  const chunks = [];
  let buf = '';

  for (const p of paras) {
    if ((buf + '\n\n' + p).length > MAX_CHARS_PER_CHUNK) {
      if (buf) chunks.push(buf.trim());
      if (p.length > MAX_CHARS_PER_CHUNK) {
        // Hard split very long paragraphs
        for (let i = 0; i < p.length; i += MAX_CHARS_PER_CHUNK) {
          chunks.push(p.slice(i, i + MAX_CHARS_PER_CHUNK));
        }
        buf = '';
      } else {
        buf = p;
      }
    } else {
      buf = buf ? buf + '\n\n' + p : p;
    }
  }
  if (buf) chunks.push(buf.trim());
  return chunks;
}
```

### 2.7 `utils/summarizeLLM.js`

```js
const OPENAI_URL = 'https://api.openai.com/v1/chat/completions';

async function callOpenAI({ model, messages, temperature = 0.2, top_p = 1 }) {
  const r = await fetch(OPENAI_URL, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ model, messages, temperature, top_p, stream: false })
  });
  if (!r.ok) {
    const t = await r.text().catch(()=>'(no body)');
    throw new Error('OpenAI error: ' + t);
  }
  const json = await r.json();
  return json.choices?.[0]?.message?.content?.trim() || '';
}

export async function summarizeChunks({ chunks, bullets, length, includeQuotes, model = 'gpt-4o-mini' }) {
  // 1) Map: summarize each chunk
  const partials = [];
  for (let i = 0; i < chunks.length; i++) {
    const content = chunks[i];
    const summary = await callOpenAI({
      model,
      messages: [
        { role: 'system', content: 'You are a precise summarizer. Be faithful to the text, avoid speculation, keep it concise.' },
        { role: 'user', content: [
          `Summarize the following article chunk into up to ${bullets} bullet points.`,
          `Length focus: ${length} (short=very brief; medium=balanced; long=more detail).`,
          `${includeQuotes ? 'If there are short, verbatim key quotes (<=20 words), include 1–2 inline in quotes.' : 'Do not include verbatim quotes.'}`,
          'Avoid repeating the title or already-stated bullets. Use neutral tone.',
          '\n--- CHUNK START ---\n',
          content,
          '\n--- CHUNK END ---'
        ].join('\n') }
      ],
      temperature: 0.2
    });
    partials.push(summary);
  }

  // 2) Reduce: combine partials into a final concise set
  const combined = await callOpenAI({
    model,
    messages: [
      { role: 'system', content: 'You combine multiple chunk summaries into one concise, non-redundant summary.' },
      { role: 'user', content: [
        `Merge the following chunk summaries into a single summary with up to ${bullets} bullets.`,
        `Length focus: ${length}.`,
        'Eliminate duplicates, keep facts, and preserve important numbers, names, dates.',
        includeQuotes ? 'Preserve at most 1–2 short quotes if essential.' : 'Remove any quotes.',
        '\n--- SUMMARIES START ---\n',
        partials.join('\n\n'),
        '\n--- SUMMARIES END ---'
      ].join('\n') }
    ],
    temperature: 0.2
  });

  return { partials, final: combined };
}
```

### 2.8 `routes/summarize.js`

```js
import { Router } from 'express';
import { fetchAndExtract } from '../utils/fetchAndExtract.js';
import { chunkText } from '../utils/chunk.js';
import { summarizeChunks } from '../utils/summarizeLLM.js';

const r = Router();

r.post('/', async (req, res) => {
  try {
    const { url, bullets = 7, length = 'medium', includeQuotes = false, model = 'gpt-4o-mini' } = req.body || {};
    if (!process.env.OPENAI_API_KEY) return res.status(500).json({ error: 'Missing OPENAI_API_KEY' });
    if (!url || !/^https?:\/\//i.test(url)) return res.status(400).json({ error: 'Valid url required' });

    const { title, text } = await fetchAndExtract(url);
    if (!text) return res.json({ title, url, summary: 'No main content found.', meta: { chunks: 0, words: 0 } });

    const chunks = chunkText(text);
    const { final, partials } = await summarizeChunks({ chunks, bullets, length, includeQuotes, model });

    const words = text.split(/\s+/).filter(Boolean).length;

    res.json({
      title,
      url,
      summary: final,
      meta: { chunks: chunks.length, words, bullets, length, includeQuotes, model }
    });
  } catch (e) {
    res.status(500).json({ error: String(e.message || e) });
  }
});

export default r;
```

---

## 3) Frontend (React + Vite)

### 3.1 Create app & install

```bash
cd ../
npm create vite@latest client -- --template react
cd client
npm i
```

### 3.2 `package.json`

```json
{
  "name": "web-summarizer-client",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "vite": "^5.4.0"
  }
}
```

### 3.3 `src/api.js`

```js
const BASE = 'http://localhost:5052';

export async function summarize(url, options) {
  const r = await fetch(`${BASE}/api/summarize`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url, ...options })
  });
  if (!r.ok) throw new Error(await r.text());
  return r.json();
}
```

### 3.4 `src/App.jsx`

```jsx
import { useState } from 'react';
import { summarize } from './api.js';

export default function App() {
  const [url, setUrl] = useState('');
  const [bullets, setBullets] = useState(7);
  const [length, setLength] = useState('medium');
  const [includeQuotes, setIncludeQuotes] = useState(false);
  const [model, setModel] = useState('gpt-4o-mini');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  const [error, setError] = useState('');

  async function onSubmit(e) {
    e.preventDefault();
    setError('');
    setResult(null);
    setLoading(true);
    try {
      const data = await summarize(url, { bullets, length, includeQuotes, model });
      setResult(data);
    } catch (e) {
      setError(String(e.message || e));
    } finally {
      setLoading(false);
    }
  }

  return (
    <div style={{ maxWidth:900, margin:'0 auto', padding:16, fontFamily:'system-ui, sans-serif' }}>
      <h2>Summarize Webpage</h2>
      <form onSubmit={onSubmit} style={{ display:'grid', gap:12, marginBottom:16 }}>
        <input
          value={url}
          onChange={e=>setUrl(e.target.value)}
          placeholder="https://example.com/article"
          style={{ padding:10, border:'1px solid #ddd', borderRadius:8 }}
        />
        <div style={{ display:'grid', gridTemplateColumns:'1fr 1fr 1fr 1fr', gap:8 }}>
          <label>Bullets
            <input type="number" min={3} max={15} value={bullets}
              onChange={e=>setBullets(Number(e.target.value))}
              style={{ width:'100%', padding:8, border:'1px solid #ddd', borderRadius:8 }} />
          </label>
          <label>Length
            <select value={length} onChange={e=>setLength(e.target.value)}
              style={{ width:'100%', padding:8, border:'1px solid #ddd', borderRadius:8 }}>
              <option value="short">short</option>
              <option value="medium">medium</option>
              <option value="long">long</option>
            </select>
          </label>
          <label>Include quotes?
            <input type="checkbox" checked={includeQuotes} onChange={e=>setIncludeQuotes(e.target.checked)} />
          </label>
          <label>Model
            <input value={model} onChange={e=>setModel(e.target.value)}
              style={{ width:'100%', padding:8, border:'1px solid #ddd', borderRadius:8 }} />
          </label>
        </div>
        <button type="submit" disabled={loading || !url}
          style={{ padding:'10px 12px', borderRadius:8, border:'1px solid #ddd', background:'#111', color:'#fff' }}>
          {loading ? 'Summarizing…' : 'Summarize'}
        </button>
      </form>

      {error && <div style={{ color:'#b00020' }}>⚠️ {error}</div>}

      {result && (
        <div style={{ border:'1px solid #eee', borderRadius:10, padding:12 }}>
          <div style={{ fontSize:12, opacity:0.7, marginBottom:4 }}>{result.url}</div>
          <h3 style={{ marginTop:0 }}>{result.title}</h3>
          <pre style={{ whiteSpace:'pre-wrap' }}>{result.summary}</pre>
          <div style={{ fontSize:12, opacity:0.7, marginTop:8 }}>
            {result.meta.chunks} chunk(s) · ~{result.meta.words} words · bullets={result.meta.bullets} · {result.meta.length}
          </div>
        </div>
      )}
    </div>
  );
}
```

### 3.5 `src/main.jsx`

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### 3.6 `index.html`

```html
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Web Summarizer</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

---

## 4) Run locally

In one terminal:

```bash
cd web-summarizer/server
npm i
node server.js
```

In another terminal:

```bash
cd web-summarizer/client
npm i
npm run dev
```

Open the shown localhost URL.

---

## 5) How it works (token budgeting + map‑reduce)

* **Extract main text** with Mozilla Readability (via JSDOM), then normalize and filter junk lines.
* **Chunk** to ~4k tokens (rough char heuristic) to avoid context overflow.
* **Map step**: summarize each chunk with a strict prompt (bullets/quotes/length knobs).
* **Reduce step**: merge chunk summaries; remove duplicates; keep facts and numbers.

---

## 6) Nice next steps (pick any)

* SSE **streaming** of the final combine step (send chunks as they arrive)
* Respect **robots.txt** and add an **allowlist** for hosts
* Cache extracted text by URL (e.g., in Mongo) with ETag/Last‑Modified headers
* Add **language detection** and request summaries in that language
* Output as **Markdown**; add a Copy button / Download .md
* Add **"key takeaways"** vs **"news timeline"** modes (different prompts)
* Switch providers or add a **provider dropdown**


You got it! I’ve put a complete **Webpage Summarizer (MERN) starter** in the canvas with copy-paste code:

* **Express** API: `/api/summarize` fetches the URL, extracts clean text (Readability), chunks it (token budgeting), and runs a **map-reduce** summary with an LLM.
* **React** UI: paste a URL, choose **bullets / length / include quotes / model**, and get a clean summary.

### Quick start

1. Backend

```
cd web-summarizer/server
npm i
# add OPENAI_API_KEY in server/.env
node server.js
```

2. Frontend

```
cd web-summarizer/client
npm i
npm run dev
```

### What you’ll learn here

* **Server-side scraping** with jsdom + Readability
* **Text cleaning** and de-junking
* **Chunking & token budgeting** (~4 chars ≈ 1 token heuristic)
* **Concise prompting** (bullets, quotes, length)
* **Map → Reduce** summarization pattern

Want me to add **SSE streaming**, a **Markdown export button**, or a **Mongo cache** for previously summarized URLs next?
