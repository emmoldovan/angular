# 01 - TypeScript Avansat

> Focus: concepte avansate de TypeScript relevante pentru un proiect Node.js + Express + React.
> Nivelul de cunoștințe așteptat: **Senior / Principal Engineer**.

---

## 1. Generics avansate

### De ce contează
Generics sunt fundamentale pentru a scrie cod refolosibil și type-safe. La nivel senior, nu ajunge să știi `Array<T>` — trebuie să știi să constrângi, să inferezi și să compui.

### Constraints (`extends`)

```typescript
// Constraint: T trebuie să aibă o proprietate `id`
function findById<T extends { id: string | number }>(items: T[], id: T['id']): T | undefined {
  return items.find(item => item.id === id);
}

// Constraint: T trebuie să fie cheie din U
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Exemplu real: repository generic pentru Express
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id'>): Promise<T>;
  update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T | null>;
  delete(id: string): Promise<boolean>;
}
```

### Inference cu `infer`

```typescript
// Extrage tipul de return al unei funcții
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

// Extrage tipul din Promise
type Awaited<T> = T extends Promise<infer R> ? R : T;

// Extrage primul parametru
type FirstParam<T extends (...args: any[]) => any> =
  T extends (first: infer F, ...rest: any[]) => any ? F : never;

// Exemplu real: extrage tipul din array
type ArrayItem<T> = T extends Array<infer Item> ? Item : never;

type UserArray = User[];
type SingleUser = ArrayItem<UserArray>; // User
```

### Generic cu default type

```typescript
// Default type — dacă nu specifici T, folosește `string`
interface ApiResponse<T = unknown, E = Error> {
  data: T;
  error: E | null;
  status: number;
}

// Folosire
type UserResponse = ApiResponse<User>;      // ApiResponse<User, Error>
type GenericResponse = ApiResponse;         // ApiResponse<unknown, Error>
```

---

## 2. Conditional Types

```typescript
// Baza: T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type Test1 = IsString<string>;   // true
type Test2 = IsString<number>;   // false

// Distributive conditional types — se aplică pe union members
type ToArray<T> = T extends any ? T[] : never;

type StringOrNumberArray = ToArray<string | number>;
// = string[] | number[]   (distribuit pe fiecare union member)

// Exclude distribuie și el:
type MyExclude<T, U> = T extends U ? never : T;
type WithoutString = MyExclude<string | number | boolean, string>;
// = number | boolean

// Non-distributive (wrapping în tuple)
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type Test = ToArrayNonDist<string | number>; // (string | number)[]
```

### Conditional types cu `infer` — pattern real

```typescript
// Extrage tipul handler-ului dintr-un Express route
type ExtractHandler<T> = T extends (req: Request, res: Response, next: NextFunction) => infer R
  ? R
  : never;

// Transformă un obiect de handlers în tipuri de return
type HandlerReturns<T extends Record<string, (...args: any[]) => any>> = {
  [K in keyof T]: ReturnType<T[K]>;
};
```

---

## 3. Mapped Types

```typescript
// Baza: transformă cheile unui tip
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

type Partial<T> = {
  [K in keyof T]?: T[K];
};

// Mapped type cu re-mapping (TypeScript 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
}

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; }

// Filtrare chei cu conditional type
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

type StringProps = PickByValue<{ a: string; b: number; c: string }, string>;
// { a: string; c: string }
```

---

## 4. Template Literal Types

```typescript
// Type-safe event names
type EventNames = 'click' | 'focus' | 'blur';
type HandlerNames = `on${Capitalize<EventNames>}`;
// = 'onClick' | 'onFocus' | 'onBlur'

// Type-safe REST endpoints
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
type Endpoint = `/api/${string}`;
type Route = `${HttpMethod} ${Endpoint}`;

// Exemplu real: type-safe event emitter
type EventMap = {
  'user:created': { userId: string };
  'user:deleted': { userId: string };
  'order:completed': { orderId: string; total: number };
};

type EventKey = keyof EventMap;
// = 'user:created' | 'user:deleted' | 'order:completed'

class TypedEventEmitter {
  emit<K extends EventKey>(event: K, payload: EventMap[K]): void {
    // ...
  }

  on<K extends EventKey>(event: K, handler: (payload: EventMap[K]) => void): void {
    // ...
  }
}

const emitter = new TypedEventEmitter();
emitter.emit('user:created', { userId: '123' });   // OK
emitter.emit('user:created', { orderId: 'abc' });  // ERROR ✓
```

---

## 5. Discriminated Unions & Exhaustive Checks

```typescript
// Discriminated union cu `type` discriminator
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string; code: number };

function processResult<T>(result: Result<T>): T | null {
  if (result.success) {
    return result.data; // TypeScript știe că e { success: true; data: T }
  }
  console.error(`Error ${result.code}: ${result.error}`);
  return null;
}

// Exhaustive check cu never
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
    case 'triangle':
      return (shape.base * shape.height) / 2;
    default:
      // Dacă adaugi un shape nou și uiți să-l tratezi, TypeScript va da eroare
      const _exhaustive: never = shape;
      throw new Error(`Unhandled shape: ${_exhaustive}`);
  }
}
```

---

## 6. Type Guards & Narrowing

```typescript
// Type guard cu `is`
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    typeof (value as any).name === 'string'
  );
}

// Type guard cu `instanceof`
function handleError(error: unknown): string {
  if (error instanceof Error) {
    return error.message; // TypeScript știe că e Error
  }
  if (typeof error === 'string') {
    return error;
  }
  return 'Unknown error';
}

// Assertion function (TypeScript 3.7+)
function assert(condition: boolean, message: string): asserts condition {
  if (!condition) throw new Error(message);
}

function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new TypeError('Expected string');
  }
}

// Narrowing cu satisfies (TypeScript 4.9+)
const config = {
  port: 3000,
  host: 'localhost',
  db: { url: 'postgres://...' }
} satisfies Record<string, unknown>;
// config.port este number (nu unknown), dar TypeScript verifică structura
```

---

## 7. Utility Types (built-in)

```typescript
// Partial<T> — toate proprietățile opționale
type UpdateUserDto = Partial<User>;

// Required<T> — toate proprietățile obligatorii
type StrictConfig = Required<Config>;

// Readonly<T> — toate readonly
type ImmutableUser = Readonly<User>;

// Pick<T, K> — subset de chei
type UserPreview = Pick<User, 'id' | 'name' | 'email'>;

// Omit<T, K> — exclude chei
type CreateUserDto = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;

// Record<K, V> — dictionar type-safe
type UserMap = Record<string, User>;
type StatusMap = Record<'active' | 'inactive' | 'banned', User[]>;

// Exclude<T, U> — exclude din union
type NonNullable<T> = Exclude<T, null | undefined>;

// Extract<T, U> — extrage din union
type StringOrNumber = Extract<string | number | boolean, string | number>;

// ReturnType<T> — tipul returnat de funcție
type HandlerResult = ReturnType<typeof myHandler>;

// Parameters<T> — tipul parametrilor
type HandlerParams = Parameters<typeof myHandler>;

// ConstructorParameters<T> — parametrii constructorului
type ServiceParams = ConstructorParameters<typeof UserService>;

// InstanceType<T> — tipul instanței
type ServiceInstance = InstanceType<typeof UserService>;

// Awaited<T> — TypeScript 4.5+
type UnwrappedUser = Awaited<Promise<User>>; // User
type DeepUnwrapped = Awaited<Promise<Promise<User>>>; // User

// NoInfer<T> — TypeScript 5.4+ — previne inferența
function createState<T>(initial: T, transform: (value: NoInfer<T>) => T): T {
  return transform(initial);
}
```

---

## 8. Decorators (TypeScript 5 — Stage 3)

```typescript
// Decorator simplu pentru logging
function log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;

  descriptor.value = async function (...args: any[]) {
    console.log(`[${propertyKey}] called with:`, args);
    const result = await originalMethod.apply(this, args);
    console.log(`[${propertyKey}] returned:`, result);
    return result;
  };

  return descriptor;
}

class UserService {
  @log
  async findUser(id: string): Promise<User> {
    // ...
  }
}

// Decorator factory cu parametri
function validate(schema: ZodSchema) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = async function (data: unknown) {
      const parsed = schema.parse(data); // throws pe eroare
      return original.call(this, parsed);
    };
  };
}
```

---

## 9. Declaration Merging & Module Augmentation

```typescript
// Augmentarea Express Request (pattern foarte comun în Node.js)
declare global {
  namespace Express {
    interface Request {
      user?: User;
      correlationId: string;
    }
  }
}

// Acum poți folosi req.user în orice handler fără cast

// Module augmentation pentru environment variables
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: 'development' | 'production' | 'test';
      PORT: string;
      DATABASE_URL: string;
      JWT_SECRET: string;
    }
  }
}

// Acum process.env.NODE_ENV e type-safe
```

---

## 10. Satisfies operator (TypeScript 4.9+)

```typescript
// Problema: vrem type safety + păstrarea tipurilor precise
const config = {
  colors: {
    red: [255, 0, 0],     // vrem să fie number[], nu [number, number, number]
    green: '#00ff00',     // vrem să fie string, nu string | number[]
  }
} satisfies Record<string, Record<string, string | number[]>>;

// config.colors.red este number[] (nu string | number[])
// TypeScript verifică structura DAR inferează tipurile precise

// Fără satisfies:
// const config: Record<...> = {...} — pierde tipurile precise
// type Config = {...} — nu verifică la runtime shape-ul
```

---

## Întrebări de interviu — TypeScript

**Q: Care e diferența dintre `interface` și `type`?**
A: Ambele definesc contracte. Diferențele: `interface` suportă declaration merging (poți extinde ulterior), `type` suportă unions, intersections și mapped types mai expresiv. Pentru obiect shapes: `interface` e preferabil (mai performant în type checker, declaration merging). Pentru unions/transformări: `type`.

**Q: Cum funcționează type narrowing?**
A: TypeScript urmărește ramurile de cod (control flow analysis) și restrânge tipul în fiecare ramură. Trigger-e: `typeof`, `instanceof`, `in`, equality checks (`=== null`), type guards cu `is`, assertion functions cu `asserts`.

**Q: Ce e un discriminated union și de ce e util?**
A: Un union type unde fiecare member are o proprietate common cu o valoare literală unică (discriminator). TypeScript poate face narrowing automat pe switch/if pe discriminator. Util pentru: Result types, action payloads (Redux), event handling — evită `as` casting.

**Q: Cum ai face un tip care extrage toate cheile unui obiect care au valori de tip string?**
```typescript
type StringKeys<T> = {
  [K in keyof T]: T[K] extends string ? K : never;
}[keyof T];
```

**Q: Ce e `infer` și când îl folosești?**
A: `infer` declară o variabilă de tip în cadrul unui conditional type. O folosesc când vreau să extrag un tip "ascuns" — returnul unei funcții, tipul unui Promise, elementul unui array. Nu pot folosi `ReturnType<T>` built-in? Pot să-l scriu cu `infer`.

---

*[← Ghid principal](./00-Ghid-Pregatire.md) | [02 - JavaScript Core →](./02-JavaScript-Core-si-Event-Loop.md)*
