# 07 - React Overview (Bonus)

> Nu e focusul principal al interviului, dar cunoașterea bazelor React
> (mai ales hooks și state management) poate fi un avantaj solid.
> Dacă ai background Angular, tranziția e mai ușoară decât crezi.

---

## Angular → React: mapping rapid

| Angular | React |
|---------|-------|
| Component class | Function component |
| `@Input()` / `input()` | Props |
| `@Output()` / EventEmitter | Callback props |
| `ngOnInit()` | `useEffect(() => {}, [])` |
| `ngOnDestroy()` | Return function din `useEffect` |
| `Signal` / `BehaviorSubject` | `useState` / `useReducer` |
| `computed()` | `useMemo` |
| `effect()` | `useEffect` |
| Service singleton | Context + custom hook |
| `ng-content` | `children` prop |
| `*ngFor` | `.map()` în JSX |
| `*ngIf` | Ternary sau `&&` în JSX |
| RxJS Observable | Promise / custom hook |
| `ChangeDetectionStrategy.OnPush` | `React.memo` |

---

## Hooks fundamentale

```tsx
import { useState, useEffect, useCallback, useMemo, useRef } from 'react';

// useState — state local
function Counter() {
  const [count, setCount] = useState(0);

  // Update pe baza valorii anterioare (important pentru async!)
  const increment = () => setCount(prev => prev + 1);

  return <button onClick={increment}>Count: {count}</button>;
}

// useEffect — side effects
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false; // previne state update pe component demontat

    async function fetchUser() {
      try {
        setLoading(true);
        const data = await api.getUser(userId);
        if (!cancelled) setUser(data);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    fetchUser();

    return () => { cancelled = true; }; // cleanup
  }, [userId]); // re-rulează când userId se schimbă

  if (loading) return <div>Loading...</div>;
  if (!user) return <div>User not found</div>;
  return <div>{user.name}</div>;
}

// useCallback — memoizează funcții (evită re-render copii)
function ParentComponent() {
  const [count, setCount] = useState(0);

  // Fără useCallback: handleClick e o funcție nouă la fiecare render
  // Cu useCallback: aceeași referință → Child nu se re-renderează inutil
  const handleClick = useCallback(() => {
    setCount(prev => prev + 1);
  }, []); // dependențe goale = niciodată re-creată

  return <ChildComponent onClick={handleClick} />;
}

// useMemo — memoizează valori computate costisitoare
function ProductList({ products, searchTerm }: Props) {
  const filteredProducts = useMemo(() => {
    return products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [products, searchTerm]); // re-calculează doar când products sau searchTerm se schimbă

  return <ul>{filteredProducts.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// useRef — referință stabilă (nu cauzează re-render)
function StopWatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef<ReturnType<typeof setInterval>>();

  const start = () => {
    intervalRef.current = setInterval(() => {
      setTime(prev => prev + 1);
    }, 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
  };

  return (
    <div>
      <p>{time}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

---

## Custom Hooks — echivalentul serviciilor/composables

```tsx
// Custom hook = logică reutilizabilă (ca un service/composable)
// Convenție: începe cu "use"

function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setStoredValue = useCallback((newValue: T | ((prev: T) => T)) => {
    setValue(prev => {
      const resolved = typeof newValue === 'function'
        ? (newValue as (prev: T) => T)(prev)
        : newValue;
      localStorage.setItem(key, JSON.stringify(resolved));
      return resolved;
    });
  }, [key]);

  return [value, setStoredValue] as const;
}

// useApi — fetch cu loading/error state
function useApi<T>(url: string) {
  const [state, setState] = useState<{
    data: T | null;
    loading: boolean;
    error: string | null;
  }>({ data: null, loading: true, error: null });

  useEffect(() => {
    const controller = new AbortController();

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => setState({ data, loading: false, error: null }))
      .catch(err => {
        if (err.name !== 'AbortError') {
          setState({ data: null, loading: false, error: err.message });
        }
      });

    return () => controller.abort();
  }, [url]);

  return state;
}

// Folosire
function UserPage({ id }: { id: string }) {
  const { data: user, loading, error } = useApi<User>(`/api/users/${id}`);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <UserCard user={user!} />;
}
```

---

## State Management

```tsx
// useReducer — pentru state complex cu mai multe acțiuni
type State = {
  items: CartItem[];
  total: number;
};

type Action =
  | { type: 'ADD_ITEM'; payload: Product }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'CLEAR' };

function cartReducer(state: State, action: Action): State {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(i => i.id === action.payload.id);
      const items = existing
        ? state.items.map(i => i.id === action.payload.id
            ? { ...i, quantity: i.quantity + 1 }
            : i)
        : [...state.items, { ...action.payload, quantity: 1 }];
      return { items, total: items.reduce((sum, i) => sum + i.price * i.quantity, 0) };
    }
    case 'REMOVE_ITEM': {
      const items = state.items.filter(i => i.id !== action.payload);
      return { items, total: items.reduce((sum, i) => sum + i.price * i.quantity, 0) };
    }
    case 'CLEAR':
      return { items: [], total: 0 };
    default:
      return state;
  }
}

// Context + useReducer = global state fără library
const CartContext = React.createContext<{
  state: State;
  dispatch: React.Dispatch<Action>;
} | null>(null);

function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });
  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  );
}

function useCart() {
  const context = useContext(CartContext);
  if (!context) throw new Error('useCart must be used within CartProvider');
  return context;
}
```

---

## Performance — React.memo, lazy, Suspense

```tsx
// React.memo — previne re-render dacă props nu s-au schimbat
const ExpensiveList = React.memo(function ExpensiveList({ items }: { items: Item[] }) {
  return (
    <ul>
      {items.map(item => <ListItem key={item.id} item={item} />)}
    </ul>
  );
});

// React.lazy + Suspense — code splitting
const Dashboard = React.lazy(() => import('./Dashboard'));
const Analytics = React.lazy(() => import('./Analytics'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}

// Key concept: Lists și reconciliation
// Key-urile ajută React să identifice ce s-a schimbat
// ❌ Nu folosi index ca key dacă lista poate fi reordonată/filtrată
{items.map((item, index) => <Item key={index} item={item} />)} // GREȘIT

// ✅ Folosește ID stabil
{items.map(item => <Item key={item.id} item={item} />)} // CORECT
```

---

## TypeScript în React

```tsx
// Props cu TypeScript
interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
  children?: React.ReactNode;
}

const Button: React.FC<ButtonProps> = ({
  label,
  onClick,
  variant = 'primary',
  disabled = false,
  children,
}) => (
  <button
    className={`btn btn-${variant}`}
    onClick={onClick}
    disabled={disabled}
  >
    {children ?? label}
  </button>
);

// Generic component
interface SelectProps<T> {
  options: T[];
  value: T;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string;
}

function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select
      value={getValue(value)}
      onChange={e => {
        const selected = options.find(o => getValue(o) === e.target.value);
        if (selected) onChange(selected);
      }}
    >
      {options.map(option => (
        <option key={getValue(option)} value={getValue(option)}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  );
}

// Event types
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setValue(e.target.value);
};

const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
  // ...
};
```

---

## Întrebări de interviu — React

**Q: Explică diferența dintre `useState` și `useRef`.**
A: `useState` declanșează re-render la fiecare schimbare. `useRef` returnează o referință stabilă care nu cauzează re-render. Folosești `useRef` pentru: referințe DOM, valori care se schimbă dar nu afectează UI (timer IDs, previous values), valori care trebuie să persistă între render-uri fără a cauza re-render.

**Q: Când folosești `useCallback` și `useMemo`?**
A: Ambele pentru optimizare, nu by default. `useCallback` când pasezi o funcție ca prop unui component memo-izat sau ca dependență la alt hook. `useMemo` pentru computații costisitoare. Regula: măsoară mai întâi (React DevTools Profiler), optimizează după. Premature optimization = complexity fără beneficiu.

**Q: Ce e lifting state up?**
A: Când două componente sibling trebuie să partajeze state, muți state-ul în cel mai apropiat ancestor comun. Componenta parinte deține state-ul și pasează valorile + funcțiile de update ca props. Alternativa la scară: Context API sau state management library (Zustand, Redux Toolkit).

**Q: `useEffect` cleanup — când e important?**
A: Când efectul creează resurse care trebuie eliberate: subscripții (WebSocket, EventEmitter), timers (setInterval), fetch requests (AbortController), DOM mutations. Fără cleanup: memory leaks, setState pe componente demontate (warning), comportament nedefinit.

---

*[← 06 - Gândire](./06-Gandire-si-Problem-Solving.md) | [08 - Q&A →](./08-Intrebari-si-Raspunsuri.md)*
