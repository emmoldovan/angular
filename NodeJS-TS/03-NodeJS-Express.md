# 03 - Node.js & Express

> Node.js internals, Express patterns, API design, autentificare și best practices
> pentru backend development cu TypeScript.

---

## 1. Node.js — Ce îl face special

### Single-threaded, non-blocking I/O

```
Node.js process
├── Main Thread (V8 JavaScript engine)
│   └── Executa codul JS (single thread)
├── libuv Thread Pool (default: 4 threads)
│   └── DNS, File I/O, Crypto, zlib (blocking ops)
└── Event Loop
    └── Coordonează I/O callbacks, timers, etc.
```

**De ce e important pentru interviu:** Node.js nu e bun pentru CPU-intensive tasks (face thread-ul principal să fie blocant). E excelent pentru I/O-bound operations (HTTP requests, DB queries, file reads) datorită modelului async non-blocking.

```typescript
// CPU-intensive BLOCHEAZĂ event loop-ul
app.get('/compute', (req, res) => {
  // GREȘIT: blochează 5 secunde toate request-urile
  const result = fibonacci(45); // sincron, CPU-intensive
  res.json({ result });
});

// Fix 1: Worker Threads pentru CPU-intensive
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

app.get('/compute', (req, res) => {
  const worker = new Worker('./fibonacci.worker.js', {
    workerData: { n: 45 }
  });

  worker.on('message', result => res.json({ result }));
  worker.on('error', err => res.status(500).json({ error: err.message }));
});

// Fix 2: Microservii sau job queues (Bull, BullMQ)
```

---

## 2. Streams — procesare date mari

```typescript
import { createReadStream, createWriteStream } from 'fs';
import { Transform } from 'stream';
import { pipeline } from 'stream/promises';
import { createGzip } from 'zlib';

// Problema fără streams: fișier de 1GB → OutOfMemoryError
app.get('/download-bad', async (req, res) => {
  const data = await fs.readFile('huge-file.csv'); // 1GB în RAM!
  res.send(data);
});

// Cu streams: procesează în chunks, memorie constantă
app.get('/download-good', async (req, res) => {
  res.setHeader('Content-Encoding', 'gzip');
  await pipeline(
    createReadStream('huge-file.csv'),
    createGzip(),
    res
  );
});

// Transform stream custom
class CSVParser extends Transform {
  private buffer = '';

  _transform(chunk: Buffer, encoding: string, callback: Function) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop() ?? ''; // ultimul rând poate fi incomplet

    for (const line of lines) {
      if (line.trim()) {
        this.push(JSON.stringify(line.split(',')) + '\n');
      }
    }
    callback();
  }

  _flush(callback: Function) {
    if (this.buffer) {
      this.push(JSON.stringify(this.buffer.split(',')) + '\n');
    }
    callback();
  }
}

// Streaming response pentru DB cursor
app.get('/users/export', async (req, res) => {
  res.setHeader('Content-Type', 'application/x-ndjson'); // newline-delimited JSON

  const cursor = await db.collection('users').find({}).cursor();
  for await (const user of cursor) {
    res.write(JSON.stringify(user) + '\n');
  }
  res.end();
});
```

---

## 3. Module System — CommonJS vs ESM

```typescript
// CommonJS (tradițional Node.js)
const express = require('express');
module.exports = { handler };

// ESM (modern, standard)
import express from 'express';
export { handler };

// Diferențe cheie:
// CJS: `require()` e sincron, dinamic (poți require condiționat)
// ESM: `import` e static (tree-shakeable), async dynamic import

// Dynamic import (ESM) — util pentru lazy loading
const { handler } = await import('./handlers/user.js');

// package.json config
{
  "type": "module",    // toate .js fișierele sunt tratate ca ESM
  // sau
  "type": "commonjs"  // default (toate .js = CJS)
}
// .mjs forțează ESM, .cjs forțează CJS indiferent de "type"

// tsconfig.json pentru Node.js modern cu ESM
{
  "compilerOptions": {
    "module": "NodeNext",       // sau "Node16"
    "moduleResolution": "NodeNext",
    "target": "ES2022"
  }
}
```

---

## 4. Express — Arhitectura unui API bine structurat

```typescript
// Structura proiect
src/
├── app.ts              // Express app (fără server.listen)
├── server.ts           // Entry point (listen + graceful shutdown)
├── config/
│   └── env.ts          // Environment variables validate cu zod
├── routes/
│   ├── index.ts        // Router aggregator
│   └── users/
│       ├── users.router.ts
│       ├── users.controller.ts
│       ├── users.service.ts
│       └── users.schema.ts    // Zod schemas
├── middleware/
│   ├── auth.ts
│   ├── error.ts
│   └── validate.ts
└── types/
    └── express.d.ts    // Module augmentation

// app.ts
import express from 'express';
import { router } from './routes';
import { errorHandler } from './middleware/error';
import { notFound } from './middleware/notFound';

export function createApp() {
  const app = express();

  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));

  // Routes
  app.use('/api/v1', router);

  // Error handling (ULTIMUL middleware)
  app.use(notFound);
  app.use(errorHandler);

  return app;
}
```

---

## 5. Middleware Pattern

```typescript
// Middleware = funcție (req, res, next)
// Ordinea contează! Express e un pipeline liniar.

// Middleware de logging
const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  const correlationId = crypto.randomUUID();
  req.correlationId = correlationId;

  res.on('finish', () => {
    console.log({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration: Date.now() - start,
      correlationId,
    });
  });

  next();
};

// Middleware de validare cu Zod
import { z, ZodSchema } from 'zod';

export const validate = (schema: ZodSchema) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten(),
      });
    }
    req.body = result.data; // date validate și parsate
    next();
  };
};

// Middleware de autentificare JWT
export const authenticate = async (req: Request, res: Response, next: NextFunction) => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing or invalid token' });
  }

  const token = authHeader.split(' ')[1];
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET) as JwtPayload;
    req.user = await userService.findById(payload.sub);
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
};

// Middleware de autorizare (după authenticate)
export const authorize = (...roles: Role[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};

// Folosire în router
router.delete(
  '/users/:id',
  authenticate,          // 1. Verifică token
  authorize('admin'),    // 2. Verifică rol
  validate(paramsSchema),// 3. Validează params
  userController.delete  // 4. Handler
);
```

---

## 6. Error Handling — Pattern corect

```typescript
// Custom error classes
export class AppError extends Error {
  constructor(
    public readonly message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly isOperational = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id '${id}' not found`, 404, 'NOT_FOUND');
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

// Global error handler (trebuie să aibă 4 parametri!)
export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof AppError && err.isOperational) {
    // Erori operaționale: trimite răspuns la client
    return res.status(err.statusCode).json({
      error: err.code,
      message: err.message,
      correlationId: req.correlationId,
    });
  }

  // Erori programatice (bug-uri): log și 500
  console.error('Unexpected error:', err);

  // Nu expune detalii interne în producție!
  return res.status(500).json({
    error: 'INTERNAL_SERVER_ERROR',
    message: process.env.NODE_ENV === 'production'
      ? 'An unexpected error occurred'
      : err.message,
    correlationId: req.correlationId,
  });
};

// Wrapper pentru async handlers (evită try/catch în fiecare handler)
export const asyncHandler = (fn: AsyncRequestHandler) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// Sau cu Express 5 (async error handling nativ):
// express 5 propagă automat erorile din async handlers la next(err)

// Graceful shutdown — critică pentru producție
process.on('SIGTERM', async () => {
  console.log('SIGTERM received. Graceful shutdown...');
  await server.close();   // Stop accepting new connections
  await db.close();       // Close DB connections
  process.exit(0);
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled Promise rejection:', reason);
  process.exit(1); // Let process manager (PM2, K8s) restart
});
```

---

## 7. Authentication — JWT + Refresh Tokens

```typescript
// Pattern corect: Access Token (scurt) + Refresh Token (lung, httpOnly cookie)
class AuthService {
  private readonly ACCESS_TOKEN_EXPIRY = '15m';
  private readonly REFRESH_TOKEN_EXPIRY = '7d';

  async login(email: string, password: string): Promise<AuthTokens> {
    const user = await this.userRepo.findByEmail(email);
    if (!user || !await bcrypt.compare(password, user.passwordHash)) {
      throw new AppError('Invalid credentials', 401, 'INVALID_CREDENTIALS');
    }

    return this.generateTokens(user);
  }

  generateTokens(user: User): AuthTokens {
    const payload = { sub: user.id, email: user.email, role: user.role };

    const accessToken = jwt.sign(payload, process.env.JWT_SECRET, {
      expiresIn: this.ACCESS_TOKEN_EXPIRY,
    });

    const refreshToken = jwt.sign(
      { sub: user.id },
      process.env.REFRESH_TOKEN_SECRET,
      { expiresIn: this.REFRESH_TOKEN_EXPIRY }
    );

    // Stochează refresh token hash în DB (pentru revocare)
    await this.tokenRepo.create({
      userId: user.id,
      tokenHash: await bcrypt.hash(refreshToken, 10),
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });

    return { accessToken, refreshToken };
  }

  async refresh(refreshToken: string): Promise<AuthTokens> {
    const payload = jwt.verify(refreshToken, process.env.REFRESH_TOKEN_SECRET);
    const userId = (payload as any).sub;

    // Verifică că refresh token-ul nu a fost revocat
    const storedTokens = await this.tokenRepo.findByUser(userId);
    const isValid = await Promise.any(
      storedTokens.map(t => bcrypt.compare(refreshToken, t.tokenHash))
    );

    if (!isValid) throw new AppError('Token revoked', 401, 'TOKEN_REVOKED');

    // Rotație token: invalidează vechiul, emite nou
    await this.tokenRepo.deleteByUser(userId);
    const user = await this.userRepo.findById(userId);
    return this.generateTokens(user);
  }
}

// Route setup
router.post('/auth/login', validate(loginSchema), asyncHandler(async (req, res) => {
  const tokens = await authService.login(req.body.email, req.body.password);

  // Refresh token în httpOnly cookie (nu accesibil JS)
  res.cookie('refreshToken', tokens.refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000,
  });

  res.json({ accessToken: tokens.accessToken });
}));
```

---

## 8. API Design Best Practices

```typescript
// RESTful resource naming
GET    /api/v1/users           // List (cu pagination, filter)
POST   /api/v1/users           // Create
GET    /api/v1/users/:id       // Get one
PATCH  /api/v1/users/:id       // Partial update (PATCH > PUT)
DELETE /api/v1/users/:id       // Delete
GET    /api/v1/users/:id/orders // Relație nested (shallow nesting)

// Response format consistent
interface ApiResponse<T> {
  data: T;
  meta?: {
    page?: number;
    limit?: number;
    total?: number;
  };
}

interface ApiError {
  error: string;      // Error code (machine-readable)
  message: string;    // Human-readable
  details?: unknown;  // Validation errors, etc.
  correlationId: string;
}

// Pagination
GET /api/v1/users?page=2&limit=20&sortBy=createdAt&order=desc&filter[role]=admin

// Versioning în URL (simplu, explicit)
/api/v1/users
/api/v2/users  // cu breaking changes
```

---

## 9. Performance în Node.js/Express

```typescript
// 1. Compression
import compression from 'compression';
app.use(compression());

// 2. Rate limiting
import rateLimit from 'express-rate-limit';
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minute
  max: 100,                   // max 100 requests / window
  standardHeaders: true,
  message: { error: 'Too many requests' }
});
app.use('/api/', limiter);

// 3. Caching cu Redis
import { createClient } from 'redis';
const redis = createClient({ url: process.env.REDIS_URL });

const cache = (ttlSeconds: number) => async (req: Request, res: Response, next: NextFunction) => {
  const key = `cache:${req.url}`;
  const cached = await redis.get(key);
  if (cached) {
    return res.json(JSON.parse(cached));
  }

  const originalJson = res.json.bind(res);
  res.json = (data) => {
    redis.setEx(key, ttlSeconds, JSON.stringify(data));
    return originalJson(data);
  };

  next();
};

// 4. Database connection pooling (nu creezi o conexiune per request!)
import { Pool } from 'pg';
const pool = new Pool({
  max: 20,          // max connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

---

## Întrebări de interviu — Node.js & Express

**Q: De ce Node.js e single-threaded dar poate gestiona mii de conexiuni?**
A: Node.js folosește I/O non-blocking. Când face o operație I/O (DB query, file read), nu blochează thread-ul — deleghează la libuv (thread pool sau OS-level async I/O) și continuă cu alte request-uri. Callback-ul rulează când I/O e gata. Un singur thread poate coordona mii de operații I/O în paralel.

**Q: Când NU ar trebui să folosești Node.js?**
A: CPU-intensive tasks (image processing, video encoding, ML inference, criptografie grea). Acestea blochează event loop-ul și scad performanța pentru toate celelalte request-uri. Soluții: Worker Threads, microservii specializate (Python pentru ML), job queues.

**Q: Care e ordinea middlewarelor și de ce contează?**
A: Express procesează middlewarele în ordinea în care le definești. Ordinea standard: (1) Security headers (helmet), (2) CORS, (3) Body parsing, (4) Logging, (5) Rate limiting, (6) Routes, (7) 404 handler, (8) Error handler. Error handler TREBUIE să fie ultimul și să aibă 4 parametri `(err, req, res, next)`.

**Q: Cum gestionezi erorile async în Express?**
A: Express 4 nu prinde automat Promise rejections — trebuie să apelezi `next(err)` manual sau să folosești un `asyncHandler` wrapper. Express 5 (în beta) prinde automat. Pattern-ul meu: un wrapper `asyncHandler(fn)` care wrappează orice async handler și propagă erorile la `next`.

**Q: JWT vs session-based authentication — ce alegi și de ce?**
A: JWT pentru API-uri stateless, distributed systems, microservii — nu ai nevoie de shared session store. Session pentru aplicații server-rendered, când vrei revocare imediată a sesiunilor. JWT-urile nu pot fi "revocate" fără un blacklist (adică DB lookup), ceea ce elimină avantajul stateless. Pattern hibrid: Access Token JWT scurt (15 min) + Refresh Token în DB (revocabil).

---

*[← 02 - JavaScript Core](./02-JavaScript-Core-si-Event-Loop.md) | [04 - Agentic Coding →](./04-Agentic-Coding-AI.md)*
