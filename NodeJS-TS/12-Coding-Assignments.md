# 12 - Coding Assignments — Pregătire Completă

> Interviul are 75 minute. 30-40 minute (60%+ din timp) sunt coding assignment.
> **Ei observă PROCESUL tău, nu doar output-ul. Screen share ON.**
> Pregătește-te pentru toate 4 variante — nu știi care va fi aleasă.

---

## Ce observă intervievatorul în timp ce codezi

Din Research Sheet (Canva, Meta, HackerRank):

```
NU SE UITĂ LA:          SE UITĂ LA:
❌ Dacă merge           ✅ Cum descompui problema înainte să scrii cod
❌ Cât cod ai scris     ✅ Cum promptezi AI-ul
❌ Sintaxa perfectă     ✅ Cum revizuiești și validezi output-ul AI
❌ Viteza               ✅ Când respingi sugestii AI și de ce
                        ✅ Cum gestionezi când AI greșește
                        ✅ Planifici înainte de a prompta?
```

### Primele 2-3 minute sunt critice

```
Ce faci ÎNAINTE să deschizi editorul:
1. Citești brief-ul integral
2. Pui întrebări de clarificare (semn de maturitate, nu de nesiguranță)
   - "Pot folosi orice AI tool doresc?"
   - "E ok să folosesc un starter sau construiesc de la zero?"
   - "Care e criteriul de succes — funcționează end-to-end sau architecture?"
3. Schițezi mental/verbal arhitectura: "Voi structura asta în X și Y"
4. APOI deschizi editorul și AI-ul
```

---

## ASSIGNMENT 1 (RECOMANDAT) — Browser Extension + AI Vision Analysis

**Brief:** Extension Chrome care capturează screenshot full-page → trimite la Node.js backend → backend trimite la AI vision model → afișează analiza UX/accessibility în popup.

### Arhitectura înainte să scrii cod

```
┌─────────────────────────────────────────────────────────┐
│  Chrome Extension (Manifest V3)                         │
│  ├── manifest.json    — permissions, background, popup  │
│  ├── popup.html/js    — UI cu buton "Capture & Analyze" │
│  └── background.js    — chrome.tabs.captureVisibleTab() │
└──────────────────────────┬──────────────────────────────┘
                           │ POST /analyze (base64 image)
┌──────────────────────────▼──────────────────────────────┐
│  Node.js Backend (Express)                              │
│  ├── POST /analyze    — validare + AI call              │
│  ├── services/ai.ts   — client OpenAI/Anthropic vision  │
│  └── middleware/      — validation, error handling      │
└──────────────────────────┬──────────────────────────────┘
                           │ API call cu image
┌──────────────────────────▼──────────────────────────────┐
│  AI Vision API (OpenAI GPT-4o / Anthropic Claude)       │
│  Prompt: "Analyze this webpage for UX issues,           │
│  accessibility problems, and design improvements"       │
└─────────────────────────────────────────────────────────┘
```

### Phase 1 — Browser Extension (15 min)

```javascript
// manifest.json — Manifest V3
{
  "manifest_version": 3,
  "name": "UX Analyzer",
  "version": "1.0.0",
  "permissions": ["activeTab", "scripting"],
  "action": {
    "default_popup": "popup.html"
  },
  "background": {
    "service_worker": "background.js"
  }
}

// popup.html
<!DOCTYPE html>
<html>
<head><style>
  body { width: 320px; padding: 16px; font-family: sans-serif; }
  button { width: 100%; padding: 12px; background: #4F46E5; color: white; border: none; border-radius: 6px; cursor: pointer; }
  #result { margin-top: 16px; font-size: 13px; }
  .loading { color: #666; }
  .error { color: #ef4444; }
</style></head>
<body>
  <h3>UX Analyzer</h3>
  <button id="captureBtn">Capture & Analyze</button>
  <div id="status"></div>
  <div id="result"></div>
  <script src="popup.js"></script>
</body>
</html>

// popup.js
document.getElementById('captureBtn').addEventListener('click', async () => {
  const status = document.getElementById('status');
  const result = document.getElementById('result');

  try {
    status.innerHTML = '<p class="loading">Capturing screenshot...</p>';
    result.innerHTML = '';

    // Trimite mesaj la background service worker
    const response = await chrome.runtime.sendMessage({ action: 'captureAndAnalyze' });

    if (response.error) {
      status.innerHTML = `<p class="error">Error: ${response.error}</p>`;
      return;
    }

    status.innerHTML = '';
    result.innerHTML = formatAnalysis(response.analysis);
  } catch (err) {
    status.innerHTML = `<p class="error">${err.message}</p>`;
  }
});

function formatAnalysis(analysis) {
  // analysis e JSON structurat de la AI
  return `
    <h4>UX Issues (${analysis.issues?.length ?? 0})</h4>
    ${(analysis.issues ?? []).map(i => `<p>• ${i}</p>`).join('')}
    <h4>Accessibility</h4>
    ${(analysis.accessibility ?? []).map(a => `<p>• ${a}</p>`).join('')}
    <h4>Recommendations</h4>
    ${(analysis.recommendations ?? []).map(r => `<p>• ${r}</p>`).join('')}
  `;
}

// background.js — Service Worker (nu are acces la DOM, doar chrome APIs)
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === 'captureAndAnalyze') {
    handleCapture().then(sendResponse).catch(err => sendResponse({ error: err.message }));
    return true; // ← CRITIC: pentru async response
  }
});

async function handleCapture() {
  // Capturează tab-ul activ
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });

  // chrome.tabs.captureVisibleTab returnează data URL (base64 PNG)
  const dataUrl = await chrome.tabs.captureVisibleTab(tab.windowId, { format: 'png' });
  const base64Image = dataUrl.split(',')[1]; // elimini "data:image/png;base64,"

  // Trimite la backend
  const response = await fetch('http://localhost:3001/analyze', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ image: base64Image, url: tab.url }),
  });

  if (!response.ok) {
    const err = await response.json();
    throw new Error(err.message ?? 'Backend error');
  }

  return response.json();
}
```

### Phase 2 — Node.js Backend (15 min)

```typescript
// src/app.ts
import express from 'express';
import cors from 'cors';
import { analyzeRouter } from './routes/analyze';
import { errorHandler } from './middleware/error';

const app = express();
app.use(cors({ origin: '*' })); // extensia Chrome trimite din chrome-extension://...
app.use(express.json({ limit: '10mb' })); // imaginile base64 pot fi mari

app.use('/', analyzeRouter);
app.use(errorHandler);

export default app;

// src/routes/analyze.ts
import { Router } from 'express';
import { z } from 'zod';
import { analyzeImage } from '../services/ai';
import { asyncHandler } from '../middleware/asyncHandler';

export const analyzeRouter = Router();

const AnalyzeSchema = z.object({
  image: z.string().min(100, 'Image too small'), // base64
  url: z.string().url().optional(),
});

analyzeRouter.post('/analyze', asyncHandler(async (req, res) => {
  const { image, url } = AnalyzeSchema.parse(req.body);

  const analysis = await analyzeImage(image, url);
  res.json(analysis);
}));

// src/services/ai.ts
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function analyzeImage(base64Image: string, url?: string) {
  const message = await client.messages.create({
    model: 'claude-opus-4-6',
    max_tokens: 1024,
    messages: [
      {
        role: 'user',
        content: [
          {
            type: 'image',
            source: {
              type: 'base64',
              media_type: 'image/png',
              data: base64Image,
            },
          },
          {
            type: 'text',
            text: `Analyze this webpage screenshot for UX issues, accessibility problems, and design improvements.
${url ? `Page URL: ${url}` : ''}

Return a JSON object with this exact structure:
{
  "issues": ["issue 1", "issue 2"],
  "accessibility": ["accessibility concern 1"],
  "recommendations": ["recommendation 1"]
}

Return ONLY the JSON, no other text.`,
          },
        ],
      },
    ],
  });

  const text = message.content[0].type === 'text' ? message.content[0].text : '';

  try {
    return JSON.parse(text);
  } catch {
    // AI-ul nu a returnat JSON valid — returnăm structura cu textul brut
    return {
      issues: [text],
      accessibility: [],
      recommendations: [],
    };
  }
}
```

### Phase 3 — Extension OAuth Google (dacă rămâne timp)

```javascript
// Adaugi în manifest.json:
"oauth2": {
  "client_id": "YOUR_CLIENT_ID.apps.googleusercontent.com",
  "scopes": ["https://www.googleapis.com/auth/userinfo.email"]
},
"permissions": ["activeTab", "scripting", "identity"]

// În popup.js, înainte de capture:
async function getAuthToken() {
  return new Promise((resolve, reject) => {
    chrome.identity.getAuthToken({ interactive: true }, (token) => {
      if (chrome.runtime.lastError) reject(chrome.runtime.lastError);
      else resolve(token);
    });
  });
}

// Trimiți token-ul în header:
const response = await fetch('http://localhost:3001/analyze', {
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  ...
});

// Backend validează token-ul Google:
import { OAuth2Client } from 'google-auth-library';
const oauthClient = new OAuth2Client();

async function verifyGoogleToken(token: string) {
  const ticket = await oauthClient.getTokenInfo(token);
  return ticket; // contains email, user_id
}
```

### Ce spui la code walkthrough

> "Am structurat în 3 straturi: extension pentru capture, backend Express ca proxy și
> validator, service AI separat. Am separat concerns-urile — route-ul nu știe de AI,
> serviciul AI nu știe de HTTP. Dacă switch la OpenAI, schimb doar ai.ts.
>
> Ce aș îmbunătăți cu mai mult timp: streaming al rezultatelor (acum aștept tot),
> caching bazat pe URL (aceeași pagină → același rezultat), history în extension storage,
> rate limiting pe backend per extension instance."

---

## ASSIGNMENT 2 — Chat-with-Any-URL

**Brief:** Next.js app, user paste URL → backend scrape + extract text → chat cu AI, streaming token-by-token.

### Arhitectura

```
Frontend (Next.js):
- URL input → POST /scrape → get content
- Chat interface → POST /chat (cu SSE streaming)
- Token-by-token rendering

Backend (Route Handlers sau Express):
- POST /scrape: fetch URL → cheerio extract text → cache în Map
- POST /chat: scraped content + history → OpenAI stream → SSE response
```

### Phase 1 — Scraping Backend

```typescript
// app/api/scrape/route.ts
import * as cheerio from 'cheerio';

const cache = new Map<string, { content: string; cachedAt: number }>();
const CACHE_TTL = 10 * 60 * 1000; // 10 minute

export async function POST(req: Request) {
  const { url } = await req.json();

  // Validare URL
  try { new URL(url); } catch {
    return Response.json({ error: 'Invalid URL' }, { status: 400 });
  }

  // Cache hit
  const cached = cache.get(url);
  if (cached && Date.now() - cached.cachedAt < CACHE_TTL) {
    return Response.json({ content: cached.content, fromCache: true });
  }

  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000); // 10s timeout

    const res = await fetch(url, {
      signal: controller.signal,
      headers: { 'User-Agent': 'Mozilla/5.0 (compatible; URLAnalyzer/1.0)' },
    });
    clearTimeout(timeout);

    if (!res.ok) return Response.json({ error: `HTTP ${res.status}` }, { status: 400 });

    const html = await res.text();
    const $ = cheerio.load(html);

    // Elimini noise: scripts, styles, nav, footer
    $('script, style, nav, footer, header, [class*="cookie"], [class*="banner"]').remove();

    const title = $('title').text();
    const content = $('body').text()
      .replace(/\s+/g, ' ')
      .trim()
      .slice(0, 15000); // limită context window

    const result = { title, content };
    cache.set(url, { content: JSON.stringify(result), cachedAt: Date.now() });

    return Response.json(result);
  } catch (err: any) {
    if (err.name === 'AbortError') return Response.json({ error: 'Timeout' }, { status: 408 });
    return Response.json({ error: err.message }, { status: 500 });
  }
}
```

### Phase 2 — AI Chat cu Streaming SSE

```typescript
// app/api/chat/route.ts
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function POST(req: Request) {
  const { pageContent, message, history } = await req.json();

  const stream = new ReadableStream({
    async start(controller) {
      const enc = new TextEncoder();

      try {
        const completion = await openai.chat.completions.create({
          model: 'gpt-4o-mini',
          stream: true,
          messages: [
            {
              role: 'system',
              content: `Answer questions about the following webpage content.
If the answer is not in the content, say so clearly.

WEBPAGE CONTENT:
${pageContent.slice(0, 12000)}`,
            },
            ...history,
            { role: 'user', content: message },
          ],
        });

        for await (const chunk of completion) {
          const token = chunk.choices[0]?.delta?.content ?? '';
          if (token) {
            controller.enqueue(enc.encode(`data: ${JSON.stringify({ token })}\n\n`));
          }
        }

        controller.enqueue(enc.encode('data: [DONE]\n\n'));
        controller.close();
      } catch (err: any) {
        controller.enqueue(enc.encode(`data: ${JSON.stringify({ error: err.message })}\n\n`));
        controller.close();
      }
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}

// Frontend hook pentru SSE
function useStreamingChat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [streaming, setStreaming] = useState(false);

  const sendMessage = async (pageContent: string, userMessage: string) => {
    const userMsg = { role: 'user' as const, content: userMessage };
    setMessages(prev => [...prev, userMsg]);
    setStreaming(true);

    let assistantText = '';
    const assistantMsg = { role: 'assistant' as const, content: '' };
    setMessages(prev => [...prev, assistantMsg]);

    const response = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        pageContent,
        message: userMessage,
        history: messages,
      }),
    });

    const reader = response.body!.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const lines = decoder.decode(value).split('\n');
      for (const line of lines) {
        if (!line.startsWith('data: ')) continue;
        const data = line.slice(6);
        if (data === '[DONE]') { setStreaming(false); break; }

        const { token } = JSON.parse(data);
        assistantText += token;
        setMessages(prev => [
          ...prev.slice(0, -1),
          { role: 'assistant', content: assistantText },
        ]);
      }
    }
  };

  return { messages, streaming, sendMessage };
}
```

---

## ASSIGNMENT 3 — AI PR Review Webhook Service

**Brief:** Webhook endpoint GitHub PR events → fetch diff → AI code review → dashboard.

### Phase 1 — Webhook Parsing

```typescript
// Payload GitHub PR event (ce vine în POST /webhook/github):
{
  action: "opened" | "synchronize" | "closed",
  pull_request: {
    number: 42,
    title: "Add user authentication",
    diff_url: "https://github.com/owner/repo/pull/42.diff",
    head: { sha: "abc123" }
  },
  repository: { full_name: "owner/repo" }
}

// Route handler:
app.post('/webhook/github', async (req, res) => {
  const event = req.headers['x-github-event'];
  if (event !== 'pull_request') return res.status(200).send('ignored');

  const { action, pull_request, repository } = req.body;
  if (!['opened', 'synchronize'].includes(action)) return res.status(200).send('ignored');

  // Fetch diff
  const diffRes = await fetch(pull_request.diff_url, {
    headers: { Authorization: `token ${process.env.GITHUB_TOKEN}` },
  });
  const diff = await diffRes.text();

  // Queue pentru procesare async (nu blochezi webhook response)
  await reviewQueue.add({ prNumber: pull_request.number, repo: repository.full_name, diff });

  res.status(202).send('queued');
});

// Phase 3 — HMAC signature verification:
import crypto from 'crypto';

function verifyGitHubSignature(payload: string, signature: string): boolean {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET!)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```

### Phase 2 — AI Review Engine

```typescript
async function reviewDiff(diff: string): Promise<ReviewResult[]> {
  // Chunking: diff-ul poate fi enorm
  const chunks = chunkDiff(diff, 4000); // max chars per chunk
  const reviews = await Promise.all(chunks.map(chunk => reviewChunk(chunk)));
  return reviews.flat();
}

async function reviewChunk(diffChunk: string): Promise<ReviewResult[]> {
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    response_format: { type: 'json_object' },
    messages: [{
      role: 'user',
      content: `Review this code diff and return a JSON object:
{
  "reviews": [
    { "file": "path/to/file.ts", "line": 42, "severity": "error|warning|info", "comment": "..." }
  ]
}

Rules: Focus on bugs, security issues, performance problems. Be specific and actionable.
Return ONLY valid JSON.

DIFF:
${diffChunk}`,
    }],
  });

  const { reviews } = JSON.parse(response.choices[0].message.content!);
  return reviews ?? [];
}
```

---

## ASSIGNMENT 4 — AI Form Builder

**Brief:** User descrie form în natural language → AI generează JSON schema → render live → submit + validate.

### Phase 1 — AI Form Generation

```typescript
const FormFieldSchema = z.object({
  type: z.enum(['text', 'email', 'textarea', 'select', 'checkbox', 'file']),
  label: z.string(),
  name: z.string(),
  required: z.boolean().default(false),
  options: z.array(z.string()).optional(), // pentru select
  validation: z.object({
    minLength: z.number().optional(),
    maxLength: z.number().optional(),
    pattern: z.string().optional(),
  }).optional(),
});

const FormSchema = z.object({
  title: z.string(),
  fields: z.array(FormFieldSchema),
});

// API route:
export async function POST(req: Request) {
  const { description } = await req.json();

  const completion = await openai.chat.completions.create({
    model: 'gpt-4o',
    response_format: { type: 'json_object' },
    messages: [{
      role: 'user',
      content: `Generate a form schema based on this description: "${description}"

Return JSON matching exactly this TypeScript type:
{
  title: string,
  fields: Array<{
    type: "text" | "email" | "textarea" | "select" | "checkbox" | "file",
    label: string,
    name: string (camelCase, no spaces),
    required: boolean,
    options?: string[] (only for select),
    validation?: { minLength?: number, maxLength?: number }
  }>
}`,
    }],
  });

  const raw = JSON.parse(completion.choices[0].message.content!);

  // Validare cu Zod — dacă AI returnează ceva invalid, prindem eroarea
  const parsed = FormSchema.safeParse(raw);
  if (!parsed.success) {
    return Response.json({ error: 'AI generated invalid schema', details: parsed.error.flatten() }, { status: 422 });
  }

  return Response.json(parsed.data);
}
```

### Phase 2 — Dynamic Form Renderer

```tsx
'use client';

interface FormField {
  type: 'text' | 'email' | 'textarea' | 'select' | 'checkbox' | 'file';
  label: string;
  name: string;
  required: boolean;
  options?: string[];
}

function DynamicForm({ schema }: { schema: { title: string; fields: FormField[] } }) {
  const [values, setValues] = useState<Record<string, any>>({});
  const [errors, setErrors] = useState<Record<string, string>>({});

  function renderField(field: FormField) {
    const commonProps = {
      id: field.name,
      name: field.name,
      required: field.required,
      'aria-label': field.label,
      onChange: (e: any) => setValues(prev => ({
        ...prev,
        [field.name]: field.type === 'checkbox' ? e.target.checked : e.target.value,
      })),
    };

    switch (field.type) {
      case 'textarea': return <textarea {...commonProps} rows={4} className="w-full border rounded p-2" />;
      case 'select': return (
        <select {...commonProps} className="w-full border rounded p-2">
          <option value="">Select...</option>
          {field.options?.map(opt => <option key={opt} value={opt}>{opt}</option>)}
        </select>
      );
      case 'checkbox': return <input type="checkbox" {...commonProps} />;
      case 'file': return <input type="file" {...commonProps} />;
      default: return <input type={field.type} {...commonProps} className="w-full border rounded p-2" />;
    }
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <h2 className="text-xl font-bold mb-4">{schema.title}</h2>
      {schema.fields.map(field => (
        <div key={field.name} className="mb-4">
          <label htmlFor={field.name} className="block text-sm font-medium mb-1">
            {field.label}{field.required && <span className="text-red-500 ml-1">*</span>}
          </label>
          {renderField(field)}
          {errors[field.name] && <p className="text-red-500 text-sm mt-1">{errors[field.name]}</p>}
        </div>
      ))}
      <button type="submit" className="bg-blue-600 text-white px-4 py-2 rounded">Submit</button>
    </form>
  );
}
```

---

## Strategie generală pentru coding assignment

### Cum folosești AI în timpul assignment-ului (ei observă asta!)

```
1. PLANIFICI ÎNAINTE (2-3 min fără AI):
   - Gândești cu voce tare: "Voi face X, Y, Z"
   - Identifici unknown-urile
   - Pui întrebări de clarificare

2. PROMPTEZI CU CONTEXT (nu "generează o extensie Chrome"):
   Bun: "Generate a Chrome Manifest V3 extension with:
   - activeTab and scripting permissions
   - popup.html with a single button
   - background service worker that handles messages
   Follow the official Chrome Extension docs structure."

3. REVIZUIEȘTI CRITIC (ei văd dacă citești output-ul):
   - Citești codul generat
   - Comentezi ce observi: "Asta e corect, asta trebuie ajustat..."
   - Nu accepți blind

4. ITEREZI SPECIFIC:
   Nu: "fix the bug"
   Da: "The sendResponse in chrome.runtime.onMessage needs return true for async response,
   otherwise the message channel closes before the async operation completes"

5. RULEZI și VERIFICI:
   - Încarci extensia în Chrome
   - Testezi manual
   - Verifici în DevTools
```

### Ce spui la code walkthrough (ultimele 5 min)

```
Template de răspuns:
"Am structurat aplicația în [X straturi/module] pentru [motiv].
[Decizia arhitecturală 1] am ales-o pentru că [trade-off].
[Decizia arhitecturală 2] aș schimba-o cu mai mult timp — acum [ce face],
  în producție aș [ce ar trebui să facă].
AI-ul m-a ajutat cu [boilerplate specific], dar am ajustat [ce ai schimbat] pentru că [motiv].
Ce aș adăuga: [2-3 lucruri concrete]."
```

---

*[← 11 - Întrebări Exacte](./11-Intrebari-Exacte-Interviu.md) | [13 - Scoring & Red Flags →](./13-Scoring-si-Red-Flags.md)*
