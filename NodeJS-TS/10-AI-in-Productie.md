# 10 - AI în Producție

> Prompt versioning, hallucination prevention, scaling AI requests, tool-based agents,
> MCP, AI evaluation, AI instruction docs, HTTP timeout cu AI agent.
> Toate întrebările AI + Architecture din lista interviului.

---

## 1. Prompt Versioning și Evaluation

### De ce e o problemă

Prompturile sunt cod. Dacă schimbi un prompt și nu știi ce efect are, ești orb. Un prompt modificat la 2AM poate strica output-ul pentru toți utilizatorii.

```typescript
// ❌ Prompt hardcodat — untrackable, unversionable
const response = await openai.chat.completions.create({
  messages: [{ role: 'system', content: 'You are a helpful assistant.' }],
});

// ✅ Prompt versionat — tratat ca orice alt asset de cod
```

### Sistem de prompt versioning

```typescript
// prompts/extract-entities.v1.ts
export const EXTRACT_ENTITIES_V1 = {
  version: '1.0.0',
  name: 'extract-entities',
  template: (text: string) => `
Extract named entities (people, organizations, locations) from the following text.
Return a JSON object with keys: people, organizations, locations.
Each key maps to an array of strings.

Text: ${text}

Return ONLY valid JSON, no explanation.
  `.trim(),
};

// prompts/extract-entities.v2.ts — breaking change: output format schimbat
export const EXTRACT_ENTITIES_V2 = {
  version: '2.0.0',
  name: 'extract-entities',
  template: (text: string) => `
Extract named entities from the text below.
Return a JSON array where each item has: { text: string, type: 'PERSON'|'ORG'|'LOCATION' }

Text: ${text}

Return ONLY valid JSON array.
  `.trim(),
};

// Registry — care versiune e activă
const ACTIVE_PROMPTS = {
  'extract-entities': EXTRACT_ENTITIES_V2,
};

// Serviciu de prompts
class PromptService {
  getPrompt(name: keyof typeof ACTIVE_PROMPTS) {
    return ACTIVE_PROMPTS[name];
  }

  // AB testing: unii utilizatori iau v1, alții v2
  getPromptForUser(name: string, userId: string) {
    const isInExperiment = hashUser(userId) % 100 < 20; // 20% pe v2
    return isInExperiment ? EXTRACT_ENTITIES_V2 : EXTRACT_ENTITIES_V1;
  }
}
```

### Evaluare automată a prompturilor

```typescript
// Eval suite — rulezi înainte de a schimba un prompt în producție
interface EvalCase {
  input: string;
  expectedOutput: unknown;
  evaluator: (output: unknown, expected: unknown) => boolean;
}

const ENTITY_EXTRACTION_EVALS: EvalCase[] = [
  {
    input: 'Apple Inc was founded by Steve Jobs in Cupertino.',
    expectedOutput: {
      people: ['Steve Jobs'],
      organizations: ['Apple Inc'],
      locations: ['Cupertino'],
    },
    evaluator: (output, expected) => {
      const out = output as typeof expected;
      return (
        out.people.includes('Steve Jobs') &&
        out.organizations.includes('Apple Inc') &&
        out.locations.includes('Cupertino')
      );
    },
  },
  // ... mai multe cazuri
];

async function runEvals(prompt: typeof EXTRACT_ENTITIES_V1) {
  let passed = 0;
  const results = [];

  for (const evalCase of ENTITY_EXTRACTION_EVALS) {
    const rawOutput = await callAI(prompt.template(evalCase.input));
    const parsed = JSON.parse(rawOutput);
    const success = evalCase.evaluator(parsed, evalCase.expectedOutput);

    if (success) passed++;
    results.push({ input: evalCase.input, success, output: parsed });
  }

  return {
    passRate: passed / ENTITY_EXTRACTION_EVALS.length,
    results,
  };
}

// Rulat în CI — dacă passRate < 0.9, block deployment
const evalResult = await runEvals(EXTRACT_ENTITIES_V2);
if (evalResult.passRate < 0.9) {
  throw new Error(`Prompt eval failed: ${evalResult.passRate * 100}% pass rate`);
}
```

### Tools pentru prompt management la scară

- **Langfuse** — open source, logging + evaluare + prompt versioning
- **Braintrust** — eval platform cu CI integration
- **PromptLayer** — logging și versioning simplu
- **LangSmith** — ecosistem LangChain

---

## 2. Hallucination Prevention în Producție

### Strategii principale

```typescript
// STRATEGIE 1: Constrain output format (reduce hallucinations)
// ❌ Libertate totală = hallucinations
const badPrompt = `Tell me about the user's order status.`;

// ✅ Format strict + date concrete
const goodPrompt = (order: Order) => `
You are a customer support assistant. Answer ONLY based on the data below.
If the answer is not in the data, say "I don't have that information."
Do NOT invent information.

Order data:
- ID: ${order.id}
- Status: ${order.status}
- Created: ${order.createdAt}
- Items: ${order.items.map(i => i.name).join(', ')}

Customer question: What is my order status?
`;

// STRATEGIE 2: Structured output (JSON mode / Zod validation)
import { zodResponseFormat } from 'openai/helpers/zod';
import { z } from 'zod';

const OrderStatusSchema = z.object({
  status: z.enum(['processing', 'shipped', 'delivered', 'cancelled']),
  message: z.string().max(200),
  estimatedDelivery: z.string().nullable(),
});

const response = await openai.beta.chat.completions.parse({
  model: 'gpt-4o',
  messages: [/* ... */],
  response_format: zodResponseFormat(OrderStatusSchema, 'order_status'),
});

const parsed = response.choices[0].message.parsed; // type-safe, validated!

// STRATEGIE 3: RAG (Retrieval Augmented Generation)
// Nu lași AI-ul să "știe" din training — îi dai contextul explicit
async function answerWithRAG(question: string, userId: string) {
  // 1. Retrieval — găsești documentele relevante
  const relevantDocs = await vectorDb.similaritySearch(question, {
    filter: { userId }, // doar documentele utilizatorului
    topK: 5,
  });

  // 2. Augmentation — construiești contextul
  const context = relevantDocs.map(doc => doc.content).join('\n\n');

  // 3. Generation — cu context explicit
  const response = await ai.complete(`
    Answer the question using ONLY the context below.
    If the answer is not in the context, say "I don't know."

    Context:
    ${context}

    Question: ${question}
  `);

  return response;
}

// STRATEGIE 4: Self-consistency (multiple samples, vote)
async function reliableAnswer(question: string): Promise<string> {
  const [ans1, ans2, ans3] = await Promise.all([
    callAI(question),
    callAI(question),
    callAI(question),
  ]);

  // Dacă toate 3 sunt identice sau similare — confidence ridicat
  // Dacă diverge — flag pentru human review sau returnezi "uncertain"
  if (areConsistent([ans1, ans2, ans3])) return ans1;
  return 'I cannot provide a reliable answer for this question.';
}

// STRATEGIE 5: Validare output cu un al doilea model/prompt
async function validateOutput(question: string, answer: string): Promise<boolean> {
  const validationResponse = await ai.complete(`
    Is the following answer factually consistent with the question?
    Does it make any claims that cannot be verified?

    Question: ${question}
    Answer: ${answer}

    Respond with JSON: { "isValid": boolean, "issues": string[] }
  `);

  const { isValid } = JSON.parse(validationResponse);
  return isValid;
}
```

---

## 3. HTTP Timeout cu AI Agent — Răspunsul așteptat în interviu

**Aceasta e întrebarea cu cel mai specific răspuns așteptat. Știi-o pe de rost.**

### De ce apare problema

Cereri AI (LLM) pot dura 10-60+ secunde. HTTP timeout-ul standard e 30s. Soluția naivă (crești timeout-ul) nu scalează și blochează resurse.

### Soluțiile corecte

```
OPȚIUNEA A — Async Job + Polling (cea mai robustă)

Client → POST /api/ai/generate
Server → { jobId: "abc123" }   ← răspuns IMEDIAT (< 100ms)
         ↓ (background worker procesează)
Client → GET /api/ai/jobs/abc123 (polling la 2s)
Server → { status: "processing" } sau { status: "done", result: "..." }
```

```typescript
// Implementare Async Job + Polling

// Route: POST /api/ai/generate
export async function POST(req: Request) {
  const { prompt } = await req.json();

  // Creezi job în DB
  const job = await db.aiJob.create({
    data: {
      prompt,
      status: 'pending',
      userId: getCurrentUserId(req),
    },
  });

  // Adaugi în queue (Bull, BullMQ, sau simplu DB polling)
  await aiQueue.add('process-prompt', { jobId: job.id, prompt });

  // Răspuns IMEDIAT cu jobId — fără să aștepți AI-ul
  return Response.json({ jobId: job.id }, { status: 202 }); // 202 Accepted
}

// Route: GET /api/ai/jobs/[jobId]
export async function GET(req: Request, { params }: { params: { jobId: string } }) {
  const job = await db.aiJob.findUnique({ where: { id: params.jobId } });
  if (!job) return Response.json({ error: 'Job not found' }, { status: 404 });

  return Response.json({
    status: job.status,   // 'pending' | 'processing' | 'done' | 'failed'
    result: job.result,   // null dacă nu e done
    error: job.error,     // null dacă nu e failed
  });
}

// Worker (BullMQ)
aiWorker.process('process-prompt', async (job) => {
  await db.aiJob.update({ where: { id: job.data.jobId }, data: { status: 'processing' } });

  try {
    const result = await callLLM(job.data.prompt); // poate dura 30-60s
    await db.aiJob.update({
      where: { id: job.data.jobId },
      data: { status: 'done', result },
    });
  } catch (error) {
    await db.aiJob.update({
      where: { id: job.data.jobId },
      data: { status: 'failed', error: error.message },
    });
  }
});

// Client-side polling
async function pollForResult(jobId: string): Promise<string> {
  while (true) {
    const response = await fetch(`/api/ai/jobs/${jobId}`);
    const job = await response.json();

    if (job.status === 'done') return job.result;
    if (job.status === 'failed') throw new Error(job.error);

    await new Promise(resolve => setTimeout(resolve, 2000)); // poll la 2s
  }
}
```

```
OPȚIUNEA B — Server-Sent Events (SSE) / Streaming

Avantaj: utilizatorul vede răspunsul în timp real (token by token)
Util când: UI-ul trebuie să arate că "scrie" (ca ChatGPT)
```

```typescript
// Route: GET /api/ai/stream
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url);
  const prompt = searchParams.get('prompt')!;

  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      try {
        // OpenAI streaming
        const completion = await openai.chat.completions.create({
          model: 'gpt-4o',
          messages: [{ role: 'user', content: prompt }],
          stream: true, // ← streaming mode
        });

        for await (const chunk of completion) {
          const token = chunk.choices[0]?.delta?.content ?? '';
          if (token) {
            // SSE format: "data: {token}\n\n"
            controller.enqueue(encoder.encode(`data: ${JSON.stringify({ token })}\n\n`));
          }
        }

        controller.enqueue(encoder.encode('data: [DONE]\n\n'));
        controller.close();
      } catch (error) {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ error: 'AI failed' })}\n\n`));
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

// Client-side SSE consumption
function useAIStream(prompt: string) {
  const [text, setText] = useState('');
  const [done, setDone] = useState(false);

  useEffect(() => {
    const eventSource = new EventSource(`/api/ai/stream?prompt=${encodeURIComponent(prompt)}`);

    eventSource.onmessage = (event) => {
      if (event.data === '[DONE]') {
        setDone(true);
        eventSource.close();
        return;
      }
      const { token } = JSON.parse(event.data);
      setText(prev => prev + token);
    };

    return () => eventSource.close();
  }, [prompt]);

  return { text, done };
}
```

```
OPȚIUNEA C — Message Queue

Avantaj: decuplare completă, retry automat, durability
Bun pentru: sisteme distribuite, volum mare, reliability critic
Tools: BullMQ (Redis), RabbitMQ, AWS SQS, Google Pub/Sub
```

### Cum răspunzi în interviu

> "Prima regulă: nu bloca request lifecycle-ul. AI calls pot dura 60+ secunde — dacă
> aștepți sincron, timeout-ul îți va lovi utilizatorii.
>
> Opțiunea mea default e **Async Job + Polling**: clientul trimite request-ul, primește
> imediat un jobId cu status 202, un worker procesează în background, clientul
> poll-uiește la 2-3 secunde. E simplu, robust, funcționează cu orice client.
>
> Pentru UX mai bun, **SSE cu streaming**: clientul vede tokens în timp real, ca ChatGPT.
> Nu mai are timeout — conexiunea e keep-alive.
>
> Pentru sisteme distribuite la scară, **Message Queue** (BullMQ sau SQS): workers independenți,
> retry automat pe eroare, back-pressure când AI provider e lent."

---

## 4. Scaling AI Request Handling

```
Probleme la traffic spike:
1. Rate limit de la AI provider (OpenAI: 10k RPM tier 1)
2. Cost explozie (fiecare request = $$$)
3. Latență crescută sub load

Soluții în ordine de complexitate:

1. CACHING semantic (nu exact match)
   - Hash prompt → cache Redis cu TTL
   - Semantic similarity: dacă prompt-ul e similar cu unul cached, returnezi cached
   - Tools: Redis + embedding similarity (supabase pgvector, pinecone)

2. QUEUE cu rate limiting
   - BullMQ cu rate limiter: max N jobs/secundă per provider
   - Dacă queue e plin → răspund cu "sistem ocupat, încearcă în X minute"

3. RETRY cu exponential backoff + fallback provider
   - OpenAI down? → Anthropic Claude
   - Anthropic down? → Groq (mai ieftin, mai rapid)

4. CIRCUIT BREAKER pentru AI provider
   - Dacă X% din request-uri fail în Y secunde → open circuit
   - Returnezi răspuns degradat sau din cache

5. STREAMING reduces perceived latency
   - Utilizatorul vede output în 500ms chiar dacă generarea durează 10s

6. MODEL ROUTING bazat pe complexitate
   - Task simplu → GPT-4o-mini (ieftin, rapid)
   - Task complex → GPT-4o sau Claude Opus (scump, lent)
   - Detector de complexitate bazat pe prompt length/keywords
```

---

## 5. Tool-based Coding Agent — cum îl construiești

### Arhitectura unui agent cu tools

```typescript
// Un agent = LLM + loop + tools
// LLM decide CE tool să apeleze, tu execuți tool-ul și returnezi rezultatul

interface Tool {
  name: string;
  description: string;           // Ce face — LLM citește asta să decidă când să-l folosească
  parameters: JSONSchema;        // Schema parametrilor — LLM îi completează
  execute: (params: any) => Promise<string>; // Tu implementezi execuția
}

// Definire tools
const tools: Tool[] = [
  {
    name: 'read_file',
    description: 'Read the contents of a file at the given path',
    parameters: {
      type: 'object',
      properties: { path: { type: 'string', description: 'File path to read' } },
      required: ['path'],
    },
    execute: async ({ path }) => {
      return fs.readFileSync(path, 'utf-8');
    },
  },
  {
    name: 'write_file',
    description: 'Write content to a file',
    parameters: {
      type: 'object',
      properties: {
        path: { type: 'string' },
        content: { type: 'string' },
      },
      required: ['path', 'content'],
    },
    execute: async ({ path, content }) => {
      fs.writeFileSync(path, content);
      return `File written: ${path}`;
    },
  },
  {
    name: 'run_bash',
    description: 'Run a bash command and return output',
    parameters: {
      type: 'object',
      properties: { command: { type: 'string' } },
      required: ['command'],
    },
    execute: async ({ command }) => {
      const { stdout, stderr } = await exec(command);
      return stdout || stderr;
    },
  },
];

// Agent loop
async function runAgent(userMessage: string, maxIterations = 10) {
  const messages: Message[] = [
    { role: 'system', content: 'You are a coding assistant. Use tools to complete tasks.' },
    { role: 'user', content: userMessage },
  ];

  for (let i = 0; i < maxIterations; i++) {
    const response = await openai.chat.completions.create({
      model: 'gpt-4o',
      messages,
      tools: tools.map(t => ({
        type: 'function',
        function: { name: t.name, description: t.description, parameters: t.parameters },
      })),
      tool_choice: 'auto',
    });

    const choice = response.choices[0];

    // Dacă LLM nu mai apelează tools — a terminat
    if (choice.finish_reason === 'stop') {
      return choice.message.content;
    }

    // LLM a apelat un tool
    if (choice.finish_reason === 'tool_calls') {
      messages.push(choice.message); // adaugi mesajul asistentului cu tool_calls

      // Execuți TOATE tool calls din mesaj (pot fi mai multe în paralel)
      const toolResults = await Promise.all(
        choice.message.tool_calls!.map(async (toolCall) => {
          const tool = tools.find(t => t.name === toolCall.function.name);
          if (!tool) return { toolCallId: toolCall.id, result: 'Tool not found' };

          const params = JSON.parse(toolCall.function.arguments);
          const result = await tool.execute(params);

          return { toolCallId: toolCall.id, result };
        })
      );

      // Adaugi rezultatele tools în messages
      for (const { toolCallId, result } of toolResults) {
        messages.push({
          role: 'tool',
          tool_call_id: toolCallId,
          content: result,
        });
      }
      // Loop continuă — LLM vede rezultatele și decide ce urmează
    }
  }

  throw new Error('Max iterations reached');
}

// Folosire
const result = await runAgent(
  'Read the file src/users.ts and add TypeScript types for all functions that lack them'
);
```

---

## 6. MCP (Model Context Protocol) — ce e și când îl folosești

```
MCP = standard open pentru conectarea AI models la external tools/data sources
Creat de Anthropic, adoptat de ecosistem (Cursor, Claude, Zed, etc.)

Analogie: MCP e ca USB — un port standard.
Înainte de MCP: fiecare IDE/AI tool implementa custom integrări.
Cu MCP: o singură implementare funcționează oriunde.

Arhitectura:
┌─────────────────────────────────────────────────┐
│  AI Client (Claude, Cursor, custom agent)        │
│  Înțelege protocolul MCP, apelează tools        │
└──────────────────────┬──────────────────────────┘
                       │ JSON-RPC over stdio/HTTP
┌──────────────────────▼──────────────────────────┐
│  MCP Server                                      │
│  Expune: tools, resources, prompts              │
│  Exemple: filesystem, GitHub, Postgres, Slack   │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  External System (DB, API, filesystem, etc.)    │
└─────────────────────────────────────────────────┘

Când folosești MCP:
- Vrei să dai AI-ului acces la DB-ul tău (citire schema, query date)
- Vrei să AI-ul poate crea PRs pe GitHub din conversație
- Vrei să construiești un agent custom care să acceseze sisteme interne
- Vrei să standardizezi tool-urile AI în echipă (un MCP server = toate IDE-urile)
```

```typescript
// Exemplu simplu de MCP Server custom (SDK oficial Anthropic)
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new Server(
  { name: 'my-api-server', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

// Definești tools pe care AI-ul le poate apela
server.setRequestHandler('tools/list', async () => ({
  tools: [
    {
      name: 'get_user',
      description: 'Get user by ID from the database',
      inputSchema: {
        type: 'object',
        properties: { userId: { type: 'string' } },
        required: ['userId'],
      },
    },
  ],
}));

server.setRequestHandler('tools/call', async (request) => {
  if (request.params.name === 'get_user') {
    const user = await db.user.findUnique({
      where: { id: request.params.arguments.userId },
    });
    return { content: [{ type: 'text', text: JSON.stringify(user) }] };
  }
});

// Pornești serverul pe stdio (pentru Claude Desktop) sau HTTP
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 7. AI Instruction Markdown Docs — cum le menții

```
"How do you maintain AI instruction Markdown docs?"
= Fișiere ca CLAUDE.md, .cursorrules, .github/copilot-instructions.md

Scopul: AI-ul să cunoască convențiile proiectului fără să i le explici de fiecare dată.
```

```markdown
<!-- CLAUDE.md — exemplu pentru un proiect Node.js/Next.js -->

# Project Context

This is a Next.js 14 app (App Router) + Express API for [product name].

## Tech Stack
- Frontend: Next.js 14, TypeScript, Tailwind, shadcn/ui, Zustand, React Query
- Backend: Node.js, Express, Prisma, PostgreSQL, BullMQ
- Testing: Vitest, React Testing Library, MSW

## Conventions

### TypeScript
- Always use `interface` for object shapes, `type` for unions/utility types
- No `any` — use `unknown` and narrow
- All async functions must have explicit return types

### React / Next.js
- Server Components by default — only add 'use client' when needed
- Data fetching in Server Components, not useEffect
- All forms use react-hook-form + zod validation

### API / Backend
- All handlers wrapped in asyncHandler()
- Errors thrown as AppError(message, statusCode, code)
- Validation via Zod schemas in middleware
- No direct Prisma calls in route handlers — always through service layer

### File Structure
- Feature-based: src/features/users/ contains routes, service, repository, types
- UI components: src/components/ui/ (primitives) and src/components/features/ (domain)

## Current Focus
Working on: [current sprint goal]
Known issues: [any known technical debt]
```

```
Cum le menții:
1. Actualizezi când adaugi o convenție nouă (tratezi ca documentație)
2. Le revizuiești la sprint review — sunt ele actualizate?
3. Feedback loop: dacă AI-ul generează ceva greșit față de convenții, adaugi la docs
4. Versionezi în Git — schimbările în CLAUDE.md sunt la fel de importante ca schimbările în cod
```

---

## 8. Arhitectură adaptabilă — product direction schimbă săptămânal

```
"Strong signals": Modular architecture, Feature flags, Low coupling,
Avoid over-optimization, Refactor cadence
```

```typescript
// 1. FEATURE FLAGS — lansezi incremental, roll back fără deploy
// simplu: env var
const NEW_CHECKOUT_FLOW = process.env.NEXT_PUBLIC_NEW_CHECKOUT === 'true';

// mai bun: feature flag service (Unleash, LaunchDarkly, Flagsmith)
const { isEnabled } = useFlag('new-checkout-flow');

// 2. MODULAR ARCHITECTURE — fiecare feature e independentă
src/features/
├── checkout/          # tot ce tine de checkout
│   ├── checkout.service.ts
│   ├── checkout.routes.ts
│   └── index.ts       # public API — ce exporti din feature
├── payments/
└── notifications/

// 3. LOW COUPLING — features comunică prin interfețe, nu direct
// ❌ checkout importă direct din payments
import { stripeService } from '../payments/stripe.service';

// ✅ checkout depinde de abstracțiune
interface PaymentProcessor {
  charge(amount: number, customerId: string): Promise<PaymentResult>;
}
// Injectezi implementarea concretă din exterior

// 4. AVOID OVER-OPTIMIZATION — nu construi pentru scale pe care nu-l ai
// "We'll need this when we have 1M users" → nu, construiești acum pentru 100 users
// Refactorizezi când ai dovezi că e necesar, nu din presupunere

// 5. REFACTOR CADENCE — nu laș technical debt să se acumuleze
// 20% din sprint capacity pentru refactoring
// "Boy scout rule": leave code cleaner than you found it
// Architecture Decision Records (ADR) pentru decizii importante
```

---

## 9. Daily AI Coding Workflow — răspunsul tău

> "Dimineața, înainte să încep o sesiune, deschid CLAUDE.md (sau echivalentul) și mă asigur că
> AI-ul are contextul actualizat despre ce lucrăm. Apoi pentru orice task:
>
> Definesc problema și constrangerile eu — nu las AI-ul să ghicească contextul.
> Cer AI-ului să genereze structura sau codul inițial cu context specific din proiect.
> Revizuiesc: logică de business, securitate, edge cases, compatibilitate cu arhitectura.
> Iterez cu feedback specific, nu generic.
> Validez manual — rulez, testez, verific în browser.
>
> La Adobe am văzut cealaltă față: am construit UI-ul pentru AI image generation —
> știu ce înseamnă să livrezi un produs AI utilizatorilor, nu doar să folosești AI ca tool.
> Asta îmi dă o perspectivă duală care cred că e valoroasă."

---

*[← 09 - Next.js](./09-NextJS-Frontend-Avansat.md) | [11 - Întrebări Exacte →](./11-Intrebari-Exacte-Interviu.md)*
