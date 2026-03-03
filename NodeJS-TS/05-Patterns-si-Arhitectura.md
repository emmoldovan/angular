# 05 - Design Patterns & Arhitectură Backend

> Design patterns relevante pentru Node.js + Express, arhitectura unui API scalabil
> și decizii de design explicate cu trade-offs.

---

## 1. Layered Architecture (cel mai comun în Express)

```
┌─────────────────────────────┐
│         Routes/Controllers  │  → Primesc HTTP request, returnează response
│         (HTTP layer)        │  → Validare input (Zod)
├─────────────────────────────┤
│         Services            │  → Business logic
│         (Business layer)    │  → Orchestrează repositories
├─────────────────────────────┤
│         Repositories        │  → Acces la date (DB, cache, external APIs)
│         (Data layer)        │  → Abstractizează ORM/driver
├─────────────────────────────┤
│         Database / External │
└─────────────────────────────┘
```

```typescript
// Controller — NU conține business logic
export const userController = {
  create: asyncHandler(async (req, res) => {
    const user = await userService.create(req.body); // delegă la service
    res.status(201).json({ data: user });
  }),

  findById: asyncHandler(async (req, res) => {
    const user = await userService.findById(req.params.id);
    if (!user) throw new NotFoundError('User', req.params.id);
    res.json({ data: user });
  }),
};

// Service — conține business logic
export class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly emailService: EmailService,
  ) {}

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) throw new AppError('Email already in use', 409, 'EMAIL_TAKEN');

    const passwordHash = await bcrypt.hash(dto.password, 12);
    const user = await this.userRepo.create({ ...dto, passwordHash });

    await this.emailService.sendWelcome(user.email); // business rule

    return user;
  }
}

// Repository — abstractizează data access
export class UserRepository {
  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({ data });
  }
}

// Beneficii: testabilitate (mock repo în service tests), separarea responsabilităților
```

---

## 2. Repository Pattern — abstractizare date

```typescript
// Interface — definit în business layer, implementat în data layer
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findAll(filter?: UserFilter): Promise<PaginatedResult<User>>;
  create(data: CreateUserData): Promise<User>;
  update(id: string, data: UpdateUserData): Promise<User | null>;
  softDelete(id: string): Promise<boolean>;
}

// Implementare cu Prisma
class PrismaUserRepository implements UserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id, deletedAt: null },
    });
  }

  async findAll({ page = 1, limit = 20, role, search }: UserFilter): Promise<PaginatedResult<User>> {
    const where: Prisma.UserWhereInput = {
      deletedAt: null,
      ...(role && { role }),
      ...(search && {
        OR: [
          { name: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } },
        ]
      }),
    };

    const [total, items] = await Promise.all([
      this.prisma.user.count({ where }),
      this.prisma.user.findMany({
        where,
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { createdAt: 'desc' },
      }),
    ]);

    return { items, total, page, limit, totalPages: Math.ceil(total / limit) };
  }
}

// Beneficiu: poți schimba DB (Prisma → TypeORM → MongoDB) fără să atingi business logic
// Bonus: testezi service-urile cu un InMemoryUserRepository
class InMemoryUserRepository implements UserRepository {
  private users: User[] = [];

  async findById(id: string): Promise<User | null> {
    return this.users.find(u => u.id === id) ?? null;
  }
  // ...
}
```

---

## 3. Dependency Injection — fără framework

```typescript
// Composition Root — un singur loc unde construiești dependency tree
// src/container.ts

import { PrismaClient } from '@prisma/client';
import { UserRepository } from './users/users.repository';
import { UserService } from './users/users.service';
import { EmailService } from './email/email.service';

const prisma = new PrismaClient();

// Bottom-up construction
const userRepository = new PrismaUserRepository(prisma);
const emailService = new EmailService(process.env.SMTP_URL);
const userService = new UserService(userRepository, emailService);

export const container = {
  userService,
  // ... alte servicii
};

// Folosire în routes
import { container } from '../container';

router.post('/users', validate(createUserSchema), asyncHandler(async (req, res) => {
  const user = await container.userService.create(req.body);
  res.status(201).json({ data: user });
}));

// Alternativă: framework-uri DI — tsyringe, inversify, awilix
// Awilix e popular pentru Node.js (fără decoratori, dependency-uri auto-detect)
```

---

## 4. Observer / Event-Driven — decuplarea side effects

```typescript
// Problema: UserService face prea multe lucruri (email, audit log, analytics)
class UserServiceCoupled {
  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.create(dto);
    await this.emailService.sendWelcome(user.email);   // coupling!
    await this.auditService.log('user.created', user); // coupling!
    await this.analyticsService.track('signup', user); // coupling!
    return user;
  }
}

// Soluție: Event-driven (decuplat)
import EventEmitter from 'events';

class DomainEvents extends EventEmitter {
  emit<K extends keyof EventMap>(event: K, payload: EventMap[K]): boolean {
    return super.emit(event, payload);
  }
  on<K extends keyof EventMap>(event: K, listener: (payload: EventMap[K]) => void): this {
    return super.on(event, listener);
  }
}

type EventMap = {
  'user.created': User;
  'user.deleted': { userId: string };
  'order.completed': { orderId: string; userId: string; total: number };
};

export const events = new DomainEvents();

// Handlers — înregistrați în startup
events.on('user.created', async (user) => emailService.sendWelcome(user.email));
events.on('user.created', async (user) => auditService.log('user.created', user));
events.on('user.created', async (user) => analyticsService.track('signup', user));

// UserService — simplu și decuplat
class UserService {
  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.create(dto);
    events.emit('user.created', user); // fire & forget (sau await cu Promise.allSettled)
    return user;
  }
}
```

---

## 5. Strategy Pattern — algoritmi interșanjabili

```typescript
// Exemplu: payment providers
interface PaymentStrategy {
  charge(amount: number, currency: string, customerId: string): Promise<PaymentResult>;
  refund(paymentId: string, amount: number): Promise<RefundResult>;
}

class StripeStrategy implements PaymentStrategy {
  async charge(amount, currency, customerId) {
    return stripe.paymentIntents.create({ amount, currency, customer: customerId });
  }
  async refund(paymentId, amount) {
    return stripe.refunds.create({ payment_intent: paymentId, amount });
  }
}

class PayPalStrategy implements PaymentStrategy {
  async charge(amount, currency, customerId) {
    return paypal.orders.create({ amount, currency, customerId });
  }
  async refund(paymentId, amount) {
    return paypal.captures.refund(paymentId, { amount });
  }
}

class PaymentService {
  constructor(private strategy: PaymentStrategy) {}

  setStrategy(strategy: PaymentStrategy) {
    this.strategy = strategy;
  }

  async processPayment(amount: number, currency: string, customerId: string) {
    return this.strategy.charge(amount, currency, customerId);
  }
}

// Runtime switch bazat pe user preference
const strategy = user.preferredPayment === 'paypal'
  ? new PayPalStrategy()
  : new StripeStrategy();
const paymentService = new PaymentService(strategy);
```

---

## 6. Factory Pattern

```typescript
// Factory pentru loggers diferite în funcție de environment
interface Logger {
  info(message: string, meta?: object): void;
  error(message: string, error?: Error, meta?: object): void;
  debug(message: string, meta?: object): void;
}

class ConsoleLogger implements Logger {
  info(message, meta) { console.log({ level: 'info', message, ...meta }); }
  error(message, error, meta) { console.error({ level: 'error', message, error: error?.message, ...meta }); }
  debug(message, meta) { console.debug({ level: 'debug', message, ...meta }); }
}

class WinstonLogger implements Logger {
  private winston = require('winston').createLogger({ /* config */ });
  info(message, meta) { this.winston.info(message, meta); }
  error(message, error, meta) { this.winston.error(message, { error, ...meta }); }
  debug(message, meta) { this.winston.debug(message, meta); }
}

// Factory function
function createLogger(env: 'development' | 'production' | 'test'): Logger {
  if (env === 'test') return new SilentLogger();
  if (env === 'production') return new WinstonLogger();
  return new ConsoleLogger();
}

export const logger = createLogger(process.env.NODE_ENV as any ?? 'development');
```

---

## 7. Circuit Breaker — resilience pattern

```typescript
// Previne cascade failures când un serviciu downstream e down
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

class CircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime?: Date;

  constructor(
    private readonly threshold: number = 5,  // failures înainte să se deschidă
    private readonly timeout: number = 60000, // ms înainte de HALF_OPEN
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      const timeSinceFailure = Date.now() - (this.lastFailureTime?.getTime() ?? 0);
      if (timeSinceFailure < this.timeout) {
        throw new Error('Circuit breaker is OPEN — service unavailable');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = new Date();
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}

// Folosire
const paymentBreaker = new CircuitBreaker(5, 30000);

async function chargeCustomer(amount: number, customerId: string) {
  return paymentBreaker.execute(() =>
    externalPaymentService.charge(amount, customerId)
  );
}
```

---

## 8. Clean Code în TypeScript — principii

```typescript
// 1. Funcții mici, o singură responsabilitate
// ❌ Prea mult într-o funcție
async function processOrder(orderId: string) {
  const order = await db.order.findUnique({ where: { id: orderId } });
  const user = await db.user.findUnique({ where: { id: order.userId } });
  const payment = await stripe.charge({ amount: order.total, customer: user.stripeId });
  await db.order.update({ where: { id: orderId }, data: { status: 'paid', paymentId: payment.id } });
  await sendEmail(user.email, 'Order confirmed');
  await db.inventory.decrementMany(order.items);
}

// ✅ Separare clară
async function processOrder(orderId: string) {
  const { order, user } = await getOrderWithUser(orderId);
  const payment = await chargeUser(user, order.total);
  await Promise.all([
    markOrderAsPaid(order.id, payment.id),
    notifyUser(user.email, order),
    updateInventory(order.items),
  ]);
}

// 2. Fail fast cu guard clauses
// ❌ Deeply nested
async function getDiscount(userId: string) {
  const user = await findUser(userId);
  if (user) {
    if (user.isPremium) {
      if (user.loyaltyYears > 2) {
        return 0.20;
      } else {
        return 0.10;
      }
    } else {
      return 0;
    }
  }
}

// ✅ Guard clauses
async function getDiscount(userId: string): Promise<number> {
  const user = await findUser(userId);
  if (!user) throw new NotFoundError('User', userId);
  if (!user.isPremium) return 0;
  if (user.loyaltyYears <= 2) return 0.10;
  return 0.20;
}

// 3. Naming — intent-revealing
// ❌
const d = new Date();
const u = users.filter(x => x.a === true);

// ✅
const today = new Date();
const activeUsers = users.filter(user => user.isActive);

// 4. Magic numbers / strings → constants sau enums
// ❌
if (user.role === 3) { /* ... */ }
setTimeout(cleanup, 86400000);

// ✅
const MAX_SESSION_DURATION_MS = 24 * 60 * 60 * 1000;
setTimeout(cleanup, MAX_SESSION_DURATION_MS);

enum UserRole {
  Admin = 'admin',
  Editor = 'editor',
  Viewer = 'viewer',
}
if (user.role === UserRole.Admin) { /* ... */ }
```

---

## Întrebări de interviu — Patterns & Arhitectură

**Q: Cum structurezi un proiect Express mare?**
A: Layered architecture: Routes → Controllers → Services → Repositories. Separare pe feature (users/, orders/) nu pe tip (controllers/, services/). Un composition root pentru DI. Middleware-uri globale în app.ts, specifice în router. Config validate cu Zod la startup.

**Q: Ce e Dependency Injection și de ce e important?**
A: DI înseamnă că o clasă primește dependențele din exterior în loc să le creeze singură. Beneficii: testabilitate (injectezi mocks), flexibilitate (schimbi implementarea fără să atingi codul), separation of concerns. Fără DI: `new PrismaClient()` direct în service → imposibil de testat fără DB real.

**Q: Când folosești Repository Pattern?**
A: Când vrei să abstractizezi accesul la date astfel încât business logic-ul să nu știe de unde vin datele (DB, cache, API). Permite schimbarea ORM/DB fără să atingi service-urile și face testarea service-urilor simplă (InMemory repository). Nu e necesar pentru proiecte mici cu un singur DB simplu.

**Q: Explică Circuit Breaker pattern.**
A: Previne cascade failures: când un serviciu downstream dă erori repetate (threshold), circuitul se "deschide" și returnezi eroare imediat fără să mai apelezi serviciul. După un timeout, treci în HALF_OPEN — lași un request să treacă. Dacă reușește, închizi circuitul. Dacă nu, rămâi OPEN.

---

*[← 04 - Agentic Coding](./04-Agentic-Coding-AI.md) | [06 - Gândire →](./06-Gandire-si-Problem-Solving.md)*
