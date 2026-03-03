# 06 - Gândire & Problem Solving

> Cum să gândești cu voce tare în interviu, live coding mindset și
> tehnici de rezolvare a problemelor.

---

## Mentalitatea corectă în interviu

**Ce caută intervievatorii:**
- Nu neapărat soluția perfectă — ci **procesul** tău de gândire
- Cum reacționezi la **ambiguitate** (pui întrebări bune?)
- Cum **comunici** în timp ce rezolvi
- Cum **gestionezi** blocajele (te panichezi sau pivotezi?)
- Dacă **verifici** și **rafinezi** soluția

**Formula câștigătoare:**
```
Clarificare → Exemple → Brute force → Optimizare → Cod → Test → Edge cases
```

---

## Pasul 1: Clarificare (NU sări direct la cod)

```
Problema: "Scrie o funcție care găsește duplicate-le dintr-un array"

Întrebări bune:
- "Ce tip de elemente poate conține array-ul? (numere, string-uri, obiecte?)"
- "Ce returnăm? Primele duplicate? Toate duplicatele? Indexul lor?"
- "Array-ul e sortat sau nesortat?"
- "Avem constrângeri de performanță? (O(n) vs O(n log n))"
- "Ce facem cu array-ul gol sau cu un singur element?"
- "Sunt duplicate multiple ale aceluiași element posibile?"
```

---

## Pasul 2: Gândește cu voce tare

```typescript
// Intervievator: "Găsește primul element non-repetitiv dintr-un string"

// TU: "Ok, să gândim. Am string-ul 'leetcode'.
// Vreau să găsesc primul caracter care apare o singură dată.
// Prima abordare care îmi vine: parcurg string-ul, număr aparițiile fiecărei litere,
// apoi parcurg din nou și returnez primul cu count 1.
// Compexitate: O(n) timp, O(k) spațiu unde k e numărul de caractere unice.
// Există ceva mai bun? Nu cred, trebuie să văd tot string-ul cel puțin o dată.
// Să implementăm asta."

function firstNonRepeating(s: string): string | null {
  // Pasul 1: numărăm aparițiile
  const counts = new Map<string, number>();
  for (const char of s) {
    counts.set(char, (counts.get(char) ?? 0) + 1);
  }

  // Pasul 2: găsim primul cu count 1
  for (const char of s) {
    if (counts.get(char) === 1) return char;
  }

  return null;
}

// "Să testăm mental:
// 'leetcode': l=1, e=2, t=1, c=1, o=1, d=1
// Parcurgem: l → count 1 → return 'l' ✓
// 'aabb': a=2, b=2 → return null ✓
// '' → return null ✓"
```

---

## Tehnici de rezolvare pentru probleme comune

### Array & String problems

```typescript
// Pattern: Two Pointers
function reverseString(s: string[]): void {
  let left = 0, right = s.length - 1;
  while (left < right) {
    [s[left], s[right]] = [s[right], s[left]]; // destructuring swap
    left++;
    right--;
  }
}

// Pattern: Sliding Window
function maxSumSubarray(nums: number[], k: number): number {
  let windowSum = nums.slice(0, k).reduce((a, b) => a + b, 0);
  let maxSum = windowSum;

  for (let i = k; i < nums.length; i++) {
    windowSum += nums[i] - nums[i - k];
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}

// Pattern: HashMap pentru O(1) lookup
function twoSum(nums: number[], target: number): [number, number] | null {
  const seen = new Map<number, number>(); // value → index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i];
    if (seen.has(complement)) {
      return [seen.get(complement)!, i];
    }
    seen.set(nums[i], i);
  }

  return null;
}
```

### Tree / Recursive problems

```typescript
interface TreeNode {
  value: number;
  left?: TreeNode;
  right?: TreeNode;
}

// DFS — recursiv
function maxDepth(node: TreeNode | undefined): number {
  if (!node) return 0;
  return 1 + Math.max(maxDepth(node.left), maxDepth(node.right));
}

// BFS — queue (level by level)
function levelOrder(root: TreeNode | undefined): number[][] {
  if (!root) return [];

  const result: number[][] = [];
  const queue: TreeNode[] = [root];

  while (queue.length > 0) {
    const levelSize = queue.length;
    const level: number[] = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift()!;
      level.push(node.value);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(level);
  }

  return result;
}
```

### Async / Real-world problems

```typescript
// "Implementează un debounce function"
function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout>;

  return function (...args: Parameters<T>) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

// "Implementează throttle"
function throttle<T extends (...args: any[]) => any>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;

  return function (...args: Parameters<T>) {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => { inThrottle = false; }, limit);
    }
  };
}

// "Implementează Promise.all de la zero"
function myPromiseAll<T>(promises: Promise<T>[]): Promise<T[]> {
  return new Promise((resolve, reject) => {
    if (promises.length === 0) {
      resolve([]);
      return;
    }

    const results: T[] = new Array(promises.length);
    let completed = 0;

    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(value => {
        results[index] = value;
        completed++;
        if (completed === promises.length) {
          resolve(results);
        }
      }).catch(reject);
    });
  });
}

// "Implementează retry cu exponential backoff"
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;

      const delay = baseDelay * Math.pow(2, attempt - 1); // 1s, 2s, 4s
      const jitter = Math.random() * 200; // ±200ms pentru a evita thundering herd
      await new Promise(resolve => setTimeout(resolve, delay + jitter));
    }
  }
  throw new Error('This should never happen');
}
```

---

## Design problems (system design light)

### "Proiectează un rate limiter"

```typescript
// Interviu: gândești cu voce tare

// "Ok, rate limiter = limităm numărul de request-uri per client per fereastră de timp.
// Câteva abordări:
// 1. Fixed window — simplu, dar burst la granița ferestrei
// 2. Sliding window — mai precis
// 3. Token bucket — permite burst controlat
// 4. Leaky bucket — smoothing

// Pentru un API Express, o implementare simplă cu Redis:"

class RateLimiter {
  constructor(
    private readonly redis: Redis,
    private readonly limit: number,     // max requests
    private readonly windowMs: number   // fereastră în ms
  ) {}

  async isAllowed(clientId: string): Promise<{ allowed: boolean; remaining: number }> {
    const key = `rate:${clientId}:${Math.floor(Date.now() / this.windowMs)}`;

    const count = await this.redis.incr(key);

    // Setăm TTL doar la primul request din fereastră
    if (count === 1) {
      await this.redis.pexpire(key, this.windowMs);
    }

    return {
      allowed: count <= this.limit,
      remaining: Math.max(0, this.limit - count),
    };
  }
}

// "Trade-offs:
// + Simplu, O(1) per request
// - Fixed window: burst la granița (100 requests în ultimele ms + 100 în primele ms = 200 în 2ms)
// Pentru sliding window: Redis sorted sets cu timestamps
// Pentru producție: express-rate-limit cu Redis store"
```

### "Proiectează un cache LRU"

```typescript
// "LRU = Least Recently Used. Evictăm cel mai vechi accesat.
// Structura de date optimă: HashMap + Doubly Linked List
// HashMap → O(1) lookup, LinkedList → O(1) move to front/evict tail"

class LRUCache<K, V> {
  private readonly map = new Map<K, V>();

  constructor(private readonly capacity: number) {}

  get(key: K): V | undefined {
    if (!this.map.has(key)) return undefined;

    // Map în JS menține insertion order — poți folosi ca LRU!
    const value = this.map.get(key)!;
    this.map.delete(key);
    this.map.set(key, value); // re-insert = move to end
    return value;
  }

  set(key: K, value: V): void {
    if (this.map.has(key)) {
      this.map.delete(key);
    } else if (this.map.size >= this.capacity) {
      // Evict LRU (primul element din Map)
      const firstKey = this.map.keys().next().value;
      this.map.delete(firstKey);
    }
    this.map.set(key, value);
  }

  get size(): number {
    return this.map.size;
  }
}

// "Interesant: Map în JS menține ordinea de inserare,
// deci nu am nevoie de LinkedList explicit — refac entry-ul la fiecare get."
```

---

## Edge cases pe care să le menționezi mereu

```typescript
// Înainte să trimiți codul, verifică:

// 1. Input-uri goale / null / undefined
if (!array || array.length === 0) return [];

// 2. Un singur element
if (array.length === 1) return array[0];

// 3. Overflow pentru numere
// Sum-ul poate depăși Number.MAX_SAFE_INTEGER?
// Folosești BigInt pentru valori mari?

// 4. Caractere speciale / Unicode
// "naïve" are 5 caractere sau 6? (ï e un code point, dar în UTF-16 poate fi 2 code units)
'naïve'.length;           // 5 în JS (corect)
[...'naïve'].length;      // 5 (spread iterează code points)

// Emoji:
'😀'.length;              // 2! (surrogate pair în UTF-16)
[...'😀'].length;         // 1 (corect)

// 5. Referințe vs valori (mutabilitate)
function addToArray(arr: number[], item: number): number[] {
  return [...arr, item]; // NU arr.push(item) — nu muți input-ul!
}

// 6. Floating point
0.1 + 0.2 === 0.3; // false!
Math.abs((0.1 + 0.2) - 0.3) < Number.EPSILON; // true ✓
```

---

## Cum gestionezi blocajul

```
Nu știi cum să continui? Nu taci — verbalizează:

1. "Mă gândesc la asta... Am câteva abordări în minte"
2. "Știu că soluția naivă ar fi O(n²) — mă gândesc cum s-o optimizez"
3. "Pot să implementez varianta simplă mai întâi și apoi să rafinăm?"
4. "Pot să vă întreb o întrebare pentru a clarifica?"
5. "Nu am mai lucrat exact cu asta, dar aș aborda-o prin..."

NU face:
- Tăcere lungă fără să explici ce faci
- "Nu știu" și te oprești
- Scrii cod aleatoriu în speranța că merge
```

---

## Întrebări "tricky" frecvente

```typescript
// Q: Cum funcționează asta?
console.log(0.1 + 0.2);         // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3); // false — floating point!

// Q: Ce printează?
const obj = { a: 1 };
const obj2 = obj;
obj2.a = 2;
console.log(obj.a); // 2 — referință, nu copie!

// Q: Ce printează?
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 3, 3, 3 — var e function-scoped, closure capturează referința

// Q: Ce e hoisting?
console.log(x); // undefined (nu ReferenceError!)
var x = 5;
console.log(x); // 5
// var e hoisted (declarația), dar nu și asignarea

console.log(y); // ReferenceError — let nu e hoisted (temporal dead zone)
let y = 5;

// Q: Cum copiezi un obiect profund?
const deep = JSON.parse(JSON.stringify(obj)); // ok pentru JSON-safe
const deep2 = structuredClone(obj);           // modern, handles Date, Map, Set, etc.

// Q: == vs ===
null == undefined;   // true (coercion)
null === undefined;  // false
0 == false;          // true
0 === false;         // false
// Folosește ÎNTOTDEAUNA ===
```

---

## Probleme tipice pentru interviuri TypeScript/Node.js

```typescript
// 1. "Implementează un EventEmitter tip-safe"
// (deja în 02-JavaScript-Core)

// 2. "Implementează memoize"
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map<string, ReturnType<T>>();

  return ((...args: Parameters<T>): ReturnType<T> => {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key)!;

    const result = fn(...args);
    cache.set(key, result);
    return result;
  }) as T;
}

// 3. "Flatten nested array"
function flatten<T>(arr: (T | T[])[]): T[] {
  return arr.reduce<T[]>((flat, item) => {
    return flat.concat(Array.isArray(item) ? flatten(item) : [item]);
  }, []);
}
// Sau simplu: arr.flat(Infinity)

// 4. "Deep equal"
function deepEqual(a: unknown, b: unknown): boolean {
  if (a === b) return true;
  if (typeof a !== typeof b) return false;
  if (typeof a !== 'object' || a === null || b === null) return false;

  const aKeys = Object.keys(a as object);
  const bKeys = Object.keys(b as object);
  if (aKeys.length !== bKeys.length) return false;

  return aKeys.every(key =>
    deepEqual((a as any)[key], (b as any)[key])
  );
}

// 5. "Pipe / Compose functions"
const pipe = <T>(...fns: Array<(x: T) => T>) =>
  (value: T): T => fns.reduce((acc, fn) => fn(acc), value);

const transform = pipe(
  (x: number) => x * 2,
  (x: number) => x + 1,
  (x: number) => x ** 2
);
transform(3); // ((3*2)+1)^2 = 49
```

---

*[← 05 - Patterns](./05-Patterns-si-Arhitectura.md) | [07 - React →](./07-React-Overview.md)*
