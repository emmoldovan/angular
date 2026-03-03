# 02 - JavaScript Core & Event Loop

> Conceptele fundamentale JS pe care un Senior Engineer trebuie să le înțeleagă profund,
> nu să le memoreze — ci să poată **razona** despre ele în orice context.

---

## 1. Closure — mai mult decât "funcție care accesează outer scope"

```javascript
// Closure = funcție + referința la mediul lexical în care a fost creată
function makeCounter(initial = 0) {
  let count = initial;           // capturat în closure

  return {
    increment: () => ++count,
    decrement: () => --count,
    value: () => count,
    reset: () => { count = initial; },
  };
}

const counter = makeCounter(10);
counter.increment(); // 11
counter.increment(); // 12
counter.value();     // 12

// Bug clasic — closure capturează REFERINȚA, nu valoarea
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 3, 3, 3 (nu 0, 1, 2!)
}

// Fix 1: let (block scope)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0); // 0, 1, 2 ✓
}

// Fix 2: IIFE (Immediately Invoked Function Expression)
for (var i = 0; i < 3; i++) {
  ((j) => setTimeout(() => console.log(j), 0))(i); // 0, 1, 2 ✓
}
```

### Memory leak cu closures

```javascript
function createLeak() {
  const bigData = new Array(1000000).fill('leak');
  return function () {
    // bigData e capturat chiar dacă nu-l folosim
    return 'hello';
  };
}

const fn = createLeak();
// bigData rămâne în memorie cât timp `fn` există

// Fix: setezi bigData = null în funcție sau restrucurezi closure-ul
```

---

## 2. Prototype Chain & Inheritance

```javascript
// Fiecare obiect JS are un prototype intern [[Prototype]]
const animal = {
  breathe() { return 'breathing'; }
};

const dog = Object.create(animal);
dog.bark = function() { return 'woof'; };

dog.bark();     // 'woof' — proprietate proprie
dog.breathe();  // 'breathing' — moștenit din prototype

// Lanțul de prototype-uri:
// dog -> animal -> Object.prototype -> null

// class syntax e "sugar" pentru prototype-based inheritance
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  speak() {
    return `${this.name} barks`;
  }
}

// Sub capotă: Dog.prototype.__proto__ === Animal.prototype

// instanceof verifică lanțul de prototype-uri
const rex = new Dog('Rex');
rex instanceof Dog;    // true
rex instanceof Animal; // true
rex instanceof Object; // true
```

### `Object.create` vs `new` vs `class`

```javascript
// Object.create — prototype explicit
const userProto = {
  greet() { return `Hi, I'm ${this.name}`; }
};
const user = Object.create(userProto);
user.name = 'Emanuel';

// new — cu constructor function
function User(name) {
  this.name = name;
}
User.prototype.greet = function() { return `Hi, I'm ${this.name}`; };
const user2 = new User('Emanuel');

// class (preferat în cod modern)
class User3 {
  constructor(name) { this.name = name; }
  greet() { return `Hi, I'm ${this.name}`; }
}
```

---

## 3. `this` binding — 4 reguli

```javascript
// Regula 1: Default binding (strict mode: undefined; non-strict: global)
function show() {
  console.log(this); // global/window sau undefined (strict)
}

// Regula 2: Implicit binding (obiectul din stânga punctului)
const obj = {
  name: 'test',
  show() { console.log(this.name); } // 'test'
};
obj.show();

// Pierderea binding-ului:
const fn = obj.show;
fn(); // undefined — this nu mai e obj!

// Regula 3: Explicit binding (call, apply, bind)
fn.call(obj);       // 'test' ✓
fn.apply(obj, []);  // 'test' ✓
const bound = fn.bind(obj);
bound();            // 'test' ✓

// Regula 4: new binding
function Person(name) {
  this.name = name; // this = obiectul nou creat
}
const p = new Person('Emanuel'); // p.name = 'Emanuel'

// Arrow functions: NU au propriul this, capturează this lexical
class Timer {
  constructor() {
    this.count = 0;
  }

  start() {
    // fără arrow: this ar fi undefined în callback
    setInterval(() => {
      this.count++; // this = instanța Timer ✓
    }, 1000);
  }
}
```

---

## 4. Event Loop — fundamentul Node.js și browser-ului

### Arhitectura event loop

```
┌───────────────────────────────┐
│           Call Stack          │  ← Execuție sincronă
└───────────────┬───────────────┘
                │ empty
                ▼
┌───────────────────────────────┐
│       Microtask Queue         │  ← Promise.then, queueMicrotask, MutationObserver
│  (se golește COMPLET înainte  │
│   de următoarea macrotask)    │
└───────────────┬───────────────┘
                │ empty
                ▼
┌───────────────────────────────┐
│       Macrotask Queue         │  ← setTimeout, setInterval, I/O callbacks
│  (se ia câte O task)          │
└───────────────────────────────┘
```

### Demonstrație clasică

```javascript
console.log('1');                     // sync

setTimeout(() => console.log('2'), 0); // macrotask

Promise.resolve()
  .then(() => console.log('3'))        // microtask
  .then(() => console.log('4'));       // microtask

queueMicrotask(() => console.log('5')); // microtask

console.log('6');                     // sync

// Output: 1, 6, 3, 4, 5, 2
// Explicatie:
// Sync: 1, 6
// Microtasks: 3, 4 (din Promise chain), 5 (queueMicrotask)
// Macrotask: 2 (setTimeout)
```

### Event Loop în Node.js — fazele `libuv`

```
   ┌──────────────────────────┐
   │        timers            │ ← setTimeout, setInterval callbacks
   └──────────┬───────────────┘
              │
   ┌──────────▼───────────────┐
   │     pending callbacks    │ ← I/O errors din iterația anterioară
   └──────────┬───────────────┘
              │
   ┌──────────▼───────────────┐
   │       idle, prepare      │ ← intern
   └──────────┬───────────────┘
              │
   ┌──────────▼───────────────┐
   │          poll            │ ← așteaptă I/O events (cel mai important!)
   └──────────┬───────────────┘
              │
   ┌──────────▼───────────────┐
   │         check            │ ← setImmediate callbacks
   └──────────┬───────────────┘
              │
   ┌──────────▼───────────────┐
   │     close callbacks      │ ← socket.on('close', ...)
   └──────────────────────────┘

setImmediate vs setTimeout(fn, 0):
- În I/O callback: setImmediate rulează PRIMUL
- La nivel top-level: ordinea e nedeterministă (depinde de OS)
- process.nextTick rulează ÎNAINTE de orice fază (microtask special în Node)
```

```javascript
// process.nextTick vs Promise.then (Node.js specific)
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');

// Output: sync, nextTick, promise
// process.nextTick > Promise.then (ambele microtasks, dar nextTick are prioritate)
```

---

## 5. Promises — internals

```javascript
// Promise = object cu 3 stări: pending, fulfilled, rejected
// .then() returnează ÎNTOTDEAUNA un nou Promise — de aceea poți face chaining

const p = new Promise((resolve, reject) => {
  // executor rulează SINCRON!
  console.log('executor'); // apare primul
  resolve(42);
});

p.then(val => console.log(val)); // 42 — async (microtask)
console.log('after .then()');   // apare al doilea

// Output: executor, after .then(), 42

// Propagarea erorilor în chain
fetch('/api/user')
  .then(res => {
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  })
  .then(user => processUser(user))
  .catch(err => {
    // Prinde ORICE eroare din chain
    console.error('Failed:', err);
  })
  .finally(() => {
    // Rulează mereu, indiferent de succes/eroare
    setLoading(false);
  });

// Promise combinators
Promise.all([p1, p2, p3]);         // await toate; fail dacă oricare fail
Promise.allSettled([p1, p2, p3]);  // await toate; returnează status fiecăruia
Promise.race([p1, p2, p3]);        // primul care se rezolvă/respinge
Promise.any([p1, p2, p3]);         // primul succes; fail dacă TOATE fail (AggregateError)
```

---

## 6. Async/Await — ce se întâmplă sub capotă

```javascript
// async/await = sugar sintactic pentru Promises + generators
async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  const user = await res.json();
  return user;
}

// Echivalent cu:
function fetchUser(id) {
  return fetch(`/api/users/${id}`)
    .then(res => res.json())
    .then(user => user);
}

// Error handling async
async function safeFlow() {
  try {
    const user = await fetchUser('123');
    const orders = await fetchOrders(user.id);
    return { user, orders };
  } catch (err) {
    // Prinde și Promise rejections!
    console.error(err);
    throw err; // re-throw dacă vrei să propagui
  }
}

// Greșeală comună: await în for loop = SERIAL (lent)
async function slowWay(ids) {
  const results = [];
  for (const id of ids) {
    results.push(await fetchUser(id)); // unul câte unul!
  }
  return results;
}

// Corect: Promise.all = PARALEL
async function fastWay(ids) {
  return Promise.all(ids.map(id => fetchUser(id)));
}

// Cu rate limiting (câte 3 în paralel)
async function batchFetch(ids, batchSize = 3) {
  const results = [];
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    const batchResults = await Promise.all(batch.map(fetchUser));
    results.push(...batchResults);
  }
  return results;
}
```

---

## 7. Generators & Iterators

```javascript
// Generator = funcție care poate fi "pausată" și "reluată"
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) {
    yield i;
  }
}

const gen = range(0, 5);
gen.next(); // { value: 0, done: false }
gen.next(); // { value: 1, done: false }
// ...
gen.next(); // { value: 4, done: false }
gen.next(); // { value: undefined, done: true }

// Lazy evaluation — util pentru seturi mari de date
function* infiniteSequence() {
  let n = 0;
  while (true) {
    yield n++;
  }
}

const first5 = [...take(infiniteSequence(), 5)]; // [0, 1, 2, 3, 4]

// Async generators — pentru streaming data
async function* fetchPages(url) {
  let page = 1;
  while (true) {
    const res = await fetch(`${url}?page=${page}`);
    const data = await res.json();
    if (data.items.length === 0) break;
    yield data.items;
    page++;
  }
}

// Consum async generator
for await (const items of fetchPages('/api/users')) {
  processItems(items);
}
```

---

## 8. Proxy & Reflect — Metaprogramming

```javascript
// Proxy interceptează operații pe obiecte
const handler = {
  get(target, prop, receiver) {
    console.log(`Getting ${String(prop)}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`Setting ${String(prop)} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  }
};

const user = new Proxy({ name: 'Emanuel' }, handler);
user.name;       // "Getting name" → 'Emanuel'
user.age = 30;   // "Setting age = 30"

// Exemplu real: validare automată
function createValidated(obj, schema) {
  return new Proxy(obj, {
    set(target, prop, value) {
      if (prop in schema) {
        const validator = schema[prop];
        if (!validator(value)) {
          throw new TypeError(`Invalid value for ${String(prop)}: ${value}`);
        }
      }
      return Reflect.set(target, prop, value);
    }
  });
}

const user = createValidated({}, {
  age: (v) => typeof v === 'number' && v >= 0 && v <= 150,
  email: (v) => typeof v === 'string' && v.includes('@'),
});

user.age = 25;          // OK
user.age = -1;          // TypeError ✓
user.email = 'bad';     // TypeError ✓
```

---

## 9. WeakMap, WeakSet, WeakRef

```javascript
// WeakMap — chei slabe (garbage collected dacă nu există alte referințe)
const cache = new WeakMap();

function processUser(user) {
  if (cache.has(user)) {
    return cache.get(user); // cache hit
  }

  const result = expensiveComputation(user);
  cache.set(user, result); // cheie = obiectul user
  return result;
}

// Când user e garbage collected, entry-ul din cache dispare automat
// Nu poți itera WeakMap (nu are .keys(), .values(), .forEach())

// WeakRef — referință slabă care poate fi garbage collected
class ResourceManager {
  #resource;

  constructor() {
    this.#resource = new WeakRef(new ExpensiveResource());
  }

  use() {
    const resource = this.#resource.deref();
    if (!resource) {
      // A fost garbage collected
      this.#resource = new WeakRef(new ExpensiveResource());
      return this.use();
    }
    return resource.process();
  }
}

// FinalizationRegistry — callback când GC colectează un obiect
const registry = new FinalizationRegistry((label) => {
  console.log(`${label} was garbage collected`);
});

let obj = { data: 'important' };
registry.register(obj, 'my-object');
obj = null; // va declanșa callback-ul (nedeterminist)
```

---

## 10. Patterns importante JavaScript

### Module Pattern (IIFE)

```javascript
const UserModule = (() => {
  // Private
  let _users = [];
  const _validate = (user) => user.name && user.email;

  // Public API
  return {
    add(user) {
      if (!_validate(user)) throw new Error('Invalid user');
      _users.push(user);
    },
    getAll() {
      return [..._users]; // returnează copie, nu referința
    }
  };
})();
```

### Observer Pattern

```javascript
class EventEmitter {
  #listeners = new Map();

  on(event, listener) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(listener);
    return () => this.off(event, listener); // unsubscribe function
  }

  off(event, listener) {
    this.#listeners.get(event)?.delete(listener);
  }

  emit(event, ...args) {
    this.#listeners.get(event)?.forEach(listener => listener(...args));
  }

  once(event, listener) {
    const unsubscribe = this.on(event, (...args) => {
      listener(...args);
      unsubscribe();
    });
  }
}
```

---

## Întrebări de interviu — JavaScript

**Q: Explică event loop-ul. Ce se întâmplă cu `setTimeout(fn, 0)`?**
A: Event loop-ul monitorizează call stack-ul și queue-urile. `setTimeout(fn, 0)` adaugă callback-ul în macrotask queue. Chiar cu delay 0, va rula DUPĂ ce call stack-ul se golește și DUPĂ ce toate microtask-urile (Promises) se rezolvă. Deci nu e garantat că rulează "imediat".

**Q: Care e diferența dintre `null` și `undefined`?**
A: `undefined` = valoare nesetată (variabilă declarată dar neasignată, parametru lipsă, property inexistentă). `null` = absența intenționată a valorii (o setezi explicit). `typeof null === 'object'` e un bug din JS, `typeof undefined === 'undefined'`.

**Q: Cum funcționează closure-ul și ce probleme poate cauza?**
A: Closure e combinația dintre o funcție și mediul lexical (variabilele din scope-ul exterior) în care a fost creată. Probleme: memory leaks (closure ține referința la variabile mari care altfel ar fi GC-uite), bug-ul clasic cu `var` în for loops (closure capturează referința, nu valoarea).

**Q: `Promise.all` vs `Promise.allSettled` — când folosești care?**
A: `Promise.all` fail-fast — dacă oricare Promise rejectează, întregul all rejectează. Bun când vrei toate rezultatele sau nimic. `Promise.allSettled` — așteaptă toate, indiferent de succes/eroare, returnează `{ status, value/reason }` pentru fiecare. Bun pentru operații independente unde vrei să procesezi fiecare rezultat.

**Q: Ce e `this` și cum se determină valoarea lui?**
A: `this` e determinat la momentul apelului, nu la definire (excepție: arrow functions). 4 reguli în ordine: (1) `new` binding, (2) explicit binding (`call`/`apply`/`bind`), (3) implicit binding (obiectul din stânga `.`), (4) default binding (global/undefined în strict). Arrow functions capturează `this` din scope-ul lexical exterior.

---

*[← 01 - TypeScript](./01-TypeScript-Avansat.md) | [03 - Node.js →](./03-NodeJS-Express.md)*
