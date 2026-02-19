# 08 - System Design pentru Angular Principal Engineer

## Cuprins

1. [Cum sa abordezi un exercitiu de system design](#1-cum-sa-abordezi-un-exercitiu-de-system-design)
2. [Scalabilitate](#2-scalabilitate)
3. [Load Balancing si CDN](#3-load-balancing-si-cdn)
4. [Caching Strategies](#4-caching-strategies)
5. [Microservices vs Monolith](#5-microservices-vs-monolith)
6. [API Design](#6-api-design)
7. [WebSockets si Real-time](#7-websockets-si-real-time)
8. [Database Design Basics](#8-database-design-basics)
9. [CAP Theorem](#9-cap-theorem)
10. [Diagrame de arhitectura tipice](#10-diagrame-de-arhitectura-tipice)
11. [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## 1. Cum sa abordezi un exercitiu de system design

### De ce conteaza pentru un Principal Engineer

Chiar daca rolul principal este frontend (Angular), un Principal Engineer trebuie sa inteleaga sistemul end-to-end. Interviurile de system design evalueaza capacitatea de a gandi la scara larga, de a identifica trade-offs si de a comunica decizii tehnice clare.

### Cei 5 pasi fundamentali

#### Pasul 1: Clarifica cerintele (5-7 minute)

Nu sari direct la solutie. Pune intrebari pentru a intelege:

**Cerinte functionale** - ce trebuie sa faca sistemul:
- Cine sunt utilizatorii? Cati sunt?
- Ce actiuni principale fac? (CRUD, search, upload, streaming)
- Ce date manipulam? Ce relatii exista intre ele?

**Cerinte non-functionale** - cum trebuie sa se comporte:
- Latenta acceptabila (sub 200ms pentru UI, sub 1s pentru search)
- Availability target (99.9% = ~8.7h downtime/an, 99.99% = ~52 min/an)
- Consistency model (strong vs eventual)
- Security, compliance (GDPR, PCI-DSS)
- Scalabilitate (10K useri azi, 10M peste 2 ani?)

```
Exemplu de intrebari bune:
- "Care e volumul de date asteptat? Cate write-uri pe secunda?"
- "Datele trebuie sa fie consistent imediat sau eventual consistency e acceptabil?"
- "Care e ratio-ul read/write? Citim mai mult decat scriem?"
- "Exista cerinte de real-time (notificari, live updates)?"
- "Trebuie sa functioneze offline?"
```

#### Pasul 2: Estimari de capacitate (3-5 minute)

Demonstreaza ca poti dimensiona un sistem. Foloseste numere rotunde si back-of-the-envelope math.

```
Exemplu: Social media feed
- 100M useri activi lunar (MAU)
- 10M useri activi zilnic (DAU)
- Fiecare user vede 20 posturi/zi -> 200M read requests/zi
- 200M / 86400s = ~2300 reads/sec (peak = 3x -> ~7000 reads/sec)
- Fiecare user posteaza 1 post/saptamana -> ~1.4M writes/zi -> ~16 writes/sec
- Read/Write ratio = ~140:1 -> sistem read-heavy -> optimizam pentru reads

Stocare:
- 1 post = ~1KB text + metadata
- 1.4M posturi/zi * 1KB = 1.4GB/zi -> ~500GB/an doar text
- Imagini: 500KB medie * 500K posturi cu imagine/zi = 250GB/zi -> 91TB/an
- Total 5 ani: ~500TB (mostly media)

Bandwidth:
- Read: 7000 req/s * 1KB = 7MB/s text (+ imagini = mult mai mult)
- Concluzie: avem nevoie de CDN pentru media
```

#### Pasul 3: Design high-level (5-10 minute)

Deseneaza diagrama de componente principale:

```
Client (Browser/Mobile)
     |
     v
   CDN (assets statice + imagini)
     |
     v
Load Balancer
     |
     v
API Gateway (auth, rate limiting, routing)
     |
     +---> Service A (users)  ---> DB A (PostgreSQL)
     |
     +---> Service B (posts)  ---> DB B (PostgreSQL) + Cache (Redis)
     |
     +---> Service C (feed)   ---> Cache (Redis) + Message Queue
     |
     +---> Service D (media)  ---> Object Storage (S3)
```

Principii la acest pas:
- Identifica serviciile principale si responsabilitatile lor
- Arata flow-ul de date pentru operatiile principale (read path vs write path)
- Indica ce tip de storage foloseste fiecare serviciu
- Nu intra in detalii inca - asteapta intrebari

#### Pasul 4: Design detaliat (10-15 minute)

Deep dive pe componentele critice. Interviewer-ul va alege de obicei 1-2 componente.

Exemple de deep dives:
- **Feed generation**: Fan-out on write vs fan-out on read
- **Search**: Inverted index, Elasticsearch, ranking
- **Real-time notifications**: WebSocket connections, pub/sub
- **Media upload**: Pre-signed URLs, chunked upload, transcoding pipeline

La fiecare deep dive:
- Explica algoritmul sau strategia
- Discuta data flow pas cu pas
- Identifica potential bottlenecks
- Propune solutii concrete

#### Pasul 5: Trade-offs si bottlenecks (5 minute)

Arata maturitate tehnica discutand:
- Ce compromisuri ai facut si de ce
- Ce ar ceda primul sub load (single points of failure)
- Cum ai monitoriza si scala
- Ce ai face diferit cu mai mult timp/buget

### Framework-ul RESHADED

Un framework structurat pentru a nu uita nimic:

| Litera | Pas | Intrebari cheie |
|--------|-----|-----------------|
| **R** - Requirements | Cerinte functionale si non-functionale | Ce face? Pentru cine? Cat de rapid? |
| **E** - Estimation | Estimari de trafic, stocare, bandwidth | Cate requests/sec? Cat storage pe an? |
| **S** - Storage | Schema de date, alegere DB | SQL vs NoSQL? Ce indexuri? Sharding? |
| **H** - High-level | Diagrama de arhitectura | Ce servicii? Cum comunica? |
| **A** - API | Definirea API-urilor | REST/GraphQL? Ce endpoints? Ce payload? |
| **D** - Detailed | Deep dive pe componente critice | Cum functioneaza exact feature X? |
| **E** - Evaluation | Trade-offs, bottlenecks, failure modes | Ce cedeaza primul? Cum recuperam? |
| **D** - Deployment | CI/CD, monitoring, scaling | Cum deploy-uim? Cum monitorizam? |

```
Tip: Nu trebuie sa parcurgi TOATE pasii in 45 de minute.
Concentreaza-te pe R, H, A, D si E - restul le mentionezi pe scurt.
Interviewer-ul va ghida unde sa faci deep dive.
```

---

## 2. Scalabilitate

### Horizontal Scaling vs Vertical Scaling

**Vertical scaling (scale up)** - maresti resursele unei singure masini:
- Mai mult CPU, RAM, disk
- Simplu de implementat
- Limita fizica (cel mai mare server disponibil)
- Single point of failure
- Mai scump la scale mare

**Horizontal scaling (scale out)** - adaugi mai multe masini:
- Adaugi noduri identice
- Teoretic nelimitat
- Necesita load balancing
- Aplicatia trebuie sa fie stateless
- Mai complex dar mai resilient

```
Vertical Scaling:                Horizontal Scaling:

  +----------+                   +---------+  +---------+  +---------+
  |          |                   |  Node 1 |  |  Node 2 |  |  Node 3 |
  |  Server  |                   |  4 CPU  |  |  4 CPU  |  |  4 CPU  |
  |  64 CPU  |                   |  8GB    |  |  8GB    |  |  8GB    |
  | 256GB    |                   +---------+  +---------+  +---------+
  |          |                        |            |            |
  +----------+                   +----+------------+------------+----+
                                 |          Load Balancer            |
                                 +----------------------------------+
```

**Regula generala**: Incepe cu vertical scaling (simplu), treci la horizontal cand atingi limitele sau ai nevoie de high availability.

### Stateless vs Stateful Services

**Stateless services** - nu retin nimic intre requesturi:
- Fiecare request contine toata informatia necesara (JWT token, session ID)
- Orice instanta poate servi orice request
- Usor de scalat horizontal (adaugi instante identice)
- Ideal pentru API servers, microservices

**Stateful services** - retin date intre requesturi:
- WebSocket connections (conexiunea e legata de o instanta)
- In-memory cache local
- Session storage in memorie
- Greu de scalat - necesita sticky sessions sau state externalizat

```typescript
// STATELESS - orice instanta poate servi acest request
@Controller('users')
export class UsersController {
  @Get(':id')
  getUser(@Param('id') id: string, @Headers('authorization') token: string) {
    // Token-ul vine cu request-ul, nu depinde de state local
    const userId = this.jwtService.verify(token);
    return this.usersService.findById(id);
  }
}

// Solutie pentru a face servicii stateless:
// - Session -> externalizat in Redis
// - Cache -> externalizat in Redis
// - File uploads -> externalizat in S3
// - WebSocket state -> managed prin Redis pub/sub
```

### Database Sharding si Partitioning

**Partitioning** - impartirea datelor in bucati mai mici in acelasi server:
- **Horizontal partitioning (range-based)**: randuri impartite dupa un criteriu (date < 2024 in partitia A, date >= 2024 in partitia B)
- **Vertical partitioning**: coloane separate (tabel users cu date de baza vs tabel user_profiles cu detalii)

**Sharding** - distribuirea datelor pe mai multe servere fizice:

```
                    Shard Key: user_id % 3

User ID: 1,4,7,10...    User ID: 2,5,8,11...    User ID: 3,6,9,12...
  +----------+             +----------+             +----------+
  | Shard 0  |             | Shard 1  |             | Shard 2  |
  | Server A |             | Server B |             | Server C |
  +----------+             +----------+             +----------+
```

**Strategii de sharding**:
- **Hash-based**: `shard = hash(key) % num_shards` - distributie uniforma, greu de repartitionat
- **Range-based**: `shard = range(key)` - query-uri range eficiente, risc de hotspots
- **Directory-based**: lookup table care mapeaza key -> shard - flexibil dar adauga latenta
- **Geographic**: date stocate in regiunea utilizatorului - latenta scazuta, compliance GDPR

**Provocari ale sharding-ului**:
- JOIN-uri intre shard-uri - foarte costisitoare
- Rebalancing cand adaugi/elimini shard-uri
- Hotspots (un shard cu mult mai mult trafic - ex: celebrity users)
- Consistent hashing rezolva partial problema rebalancing-ului

### Connection Pooling

Deschiderea unei conexiuni la DB e costisitoare (TCP handshake, auth, TLS). Connection pooling mentine un set de conexiuni reutilizabile.

```
Fara pool:                      Cu pool:

Request 1 -> open conn -> query -> close     Request 1 --+
Request 2 -> open conn -> query -> close                  |    +--------+
Request 3 -> open conn -> query -> close     Request 2 --+--->| Pool   |---> DB
Request 4 -> open conn -> query -> close                  |   | (10    |
                                             Request 3 --+   | conns) |
~50ms overhead per request                                    +--------+
                                             ~0ms overhead (conexiune deja deschisa)
```

Parametri importanti:
- **min connections**: conexiuni mentinute activ (chiar fara trafic)
- **max connections**: limita superioara (protejeaza DB-ul)
- **idle timeout**: cat timp o conexiune idle ramane in pool
- **connection lifetime**: durata maxima a unei conexiuni (previne memory leaks)

---

## 3. Load Balancing si CDN

### Algoritmi de Load Balancing

**Round Robin** - distribuie requesturile secvential:
```
Request 1 -> Server A
Request 2 -> Server B
Request 3 -> Server C
Request 4 -> Server A  (se reia ciclul)
```
- Simplu, functioneaza bine cand serverele au capacitate egala
- Nu tine cont de load-ul actual al serverelor

**Weighted Round Robin** - servere cu capacitate diferita:
```
Server A (weight 3): primeste 3 din 6 requesturi
Server B (weight 2): primeste 2 din 6 requesturi
Server C (weight 1): primeste 1 din 6 requesturi
```

**Least Connections** - trimite la serverul cu cele mai putine conexiuni active:
- Bun cand requesturile au durate variabile
- Previne supraincarcarea unui server cu requesturi lente

**IP Hash** - hash pe IP-ul clientului determina serverul:
- Acelasi client ajunge mereu la acelasi server (sticky sessions)
- Util cand ai nevoie de session affinity
- Dezavantaj: daca un server cade, toti clientii lui trebuie redistribuiti

**Least Response Time** - combina conexiuni active + timp de raspuns:
- Cel mai inteligent dar si cel mai complex
- Necesita health monitoring activ

### L4 vs L7 Load Balancing

```
OSI Model (simplificat):

Layer 7 (Application) - HTTP, HTTPS, WebSocket   <-- L7 LB
Layer 6 (Presentation)
Layer 5 (Session)
Layer 4 (Transport)   - TCP, UDP                  <-- L4 LB
Layer 3 (Network)     - IP
Layer 2 (Data Link)
Layer 1 (Physical)
```

**L4 Load Balancing** (Transport Layer):
- Vede: IP sursa/destinatie, port, protocol (TCP/UDP)
- NU vede: continutul HTTP (URL, headers, cookies)
- Rapid (nu parseaza payload)
- Folosit pentru: TCP connections generice, database connections, gaming servers
- Exemple: AWS NLB, HAProxy in mod TCP

**L7 Load Balancing** (Application Layer):
- Vede: tot ce vede L4 + URL, HTTP headers, cookies, request body
- Poate ruta pe baza de: path (`/api` -> backend, `/` -> frontend), host, headers
- Poate face: SSL termination, compression, caching, rate limiting
- Mai lent (parseaza HTTP) dar mult mai flexibil
- Folosit pentru: web applications, API routing, A/B testing
- Exemple: AWS ALB, Nginx, Envoy, Traefik

```
L7 Load Balancer - Content-based routing:

Client ----> L7 LB ---> /api/*      ---> API Servers (pool A)
                   |
                   +--> /static/*   ---> CDN / Static Server
                   |
                   +--> /ws/*       ---> WebSocket Servers (pool B)
                   |
                   +--> /admin/*    ---> Admin Servers (pool C)
```

### CDN (Content Delivery Network)

CDN-ul distribuie continut static la edge servers aproape de utilizatori.

```
Fara CDN:                           Cu CDN:

User (Romania) ----5000km----->     User (Romania) ---50km---> Edge (Bucuresti)
Server (US East)                                                    |
RTT: ~120ms                                                    Cache HIT?
                                                               /         \
                                                             DA           NU
                                                              |            |
                                                         Serveste      Fetch de la origin
                                                         din cache     (o singura data)
                                                         RTT: ~5ms     apoi cache-uieste
```

**CDN pentru Angular SPA**:
- `index.html` - TTL scurt (5 min) sau no-cache (pentru deploy-uri rapide)
- `main.[hash].js`, `styles.[hash].css` - TTL lung (1 an) datorita content hashing
- Imagini, fonturi - TTL lung (30 zile - 1 an)
- API responses - de obicei NU se cache-uiesc pe CDN (date dinamice)

**Provideri populari**:
- **CloudFront** (AWS) - integrat cu S3, Lambda@Edge pentru logica la edge
- **Cloudflare** - CDN + DDoS protection + WAF + DNS, plan gratuit generos
- **Akamai** - cel mai mare CDN global, enterprise
- **Azure CDN** - integrat cu ecosistemul Azure

### Edge Caching pentru SPA

```nginx
# Configurare Nginx/CDN pentru Angular SPA

# index.html - mereu fresh (necesita deploy instant)
location = /index.html {
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header Pragma "no-cache";
}

# Assets cu hash in nume - cache agresiv
location ~* \.[a-f0-9]{16,}\.(js|css)$ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}

# Imagini si fonturi - cache moderat
location ~* \.(png|jpg|jpeg|gif|svg|woff2|ttf)$ {
    add_header Cache-Control "public, max-age=2592000";
}

# API - fara cache pe CDN
location /api/ {
    add_header Cache-Control "private, no-store";
    proxy_pass http://backend;
}
```

---

## 4. Caching Strategies

### Niveluri de Cache

```
+-----------+    +-----+    +-----+    +----------+    +----------+    +----+
|  Browser  |--->| CDN |--->| LB  |--->| App      |--->| Cache    |--->| DB |
|  Cache    |    |Cache|    |     |    | Server   |    | (Redis)  |    |    |
+-----------+    +-----+    +-----+    +----------+    +----------+    +----+
    L1             L2                      L3              L4            L5

Latenta tipica:
L1: 0ms (local)
L2: 5-50ms (edge)
L3: -
L4: 1-5ms (in-memory)
L5: 5-50ms (disk)
```

### Browser Cache (HTTP Caching)

**Cache-Control header** - controleaza comportamentul de caching:

```
Cache-Control: public, max-age=3600
  |              |          |
  |              |          +-- Fresh pentru 3600 secunde
  |              +-- Poate fi stocat de CDN si browser
  +-- Header-ul principal

Directive importante:
- public    : CDN si browser pot cache-ui
- private   : doar browser-ul (date specifice utilizatorului)
- no-cache  : cache-uieste dar revalideaza mereu cu serverul
- no-store  : nu stoca niciodata (date sensibile)
- max-age=N : fresh pentru N secunde
- s-maxage=N: max-age doar pentru CDN/proxy
- immutable : nu revalida niciodata (folosit cu content hashing)
- stale-while-revalidate=N : serveste stale si revalideaza in background
```

**ETag** - identificator unic al versiunii unui resurs:
```
Primul request:
GET /api/users/123
Response: 200 OK
ETag: "abc123"
Body: { "name": "Ion" }

Al doilea request (conditional):
GET /api/users/123
If-None-Match: "abc123"

Daca nu s-a schimbat:
Response: 304 Not Modified (fara body - economisim bandwidth)

Daca s-a schimbat:
Response: 200 OK
ETag: "def456"
Body: { "name": "Ion Updated" }
```

**Last-Modified** - timestamp-ul ultimei modificari:
```
Response: Last-Modified: Wed, 18 Feb 2026 10:00:00 GMT
Request:  If-Modified-Since: Wed, 18 Feb 2026 10:00:00 GMT
Response: 304 Not Modified (sau 200 cu date noi)
```

### Application Cache (Redis / Memcached)

**Redis** - in-memory data store, cel mai popular pentru caching:
- Structuri de date: strings, hashes, lists, sets, sorted sets
- Persistence optionala (RDB snapshots, AOF log)
- Pub/Sub pentru real-time
- Clustering si replication
- Expiry automat (TTL pe fiecare key)

**Memcached** - simplu, rapid, doar key-value:
- Fara persistence
- Multi-threaded (Redis e single-threaded cu I/O multiplexing)
- Ideal cand ai nevoie doar de simple key-value cache

```typescript
// Exemplu: Cache-aside pattern in Angular backend (NestJS)
@Injectable()
export class ProductsService {
  constructor(
    private redis: RedisService,
    private db: DatabaseService,
  ) {}

  async getProduct(id: string): Promise<Product> {
    // 1. Incearca din cache
    const cached = await this.redis.get(`product:${id}`);
    if (cached) {
      return JSON.parse(cached); // Cache HIT
    }

    // 2. Cache MISS - citeste din DB
    const product = await this.db.products.findById(id);

    // 3. Pune in cache cu TTL de 5 minute
    await this.redis.set(`product:${id}`, JSON.stringify(product), 'EX', 300);

    return product;
  }

  async updateProduct(id: string, data: UpdateProductDto): Promise<Product> {
    const product = await this.db.products.update(id, data);

    // Invalideaza cache-ul
    await this.redis.del(`product:${id}`);
    // SAU actualizeaza cache-ul (write-through)
    await this.redis.set(`product:${id}`, JSON.stringify(product), 'EX', 300);

    return product;
  }
}
```

### Cache Invalidation Strategies

**TTL (Time-To-Live)** - cea mai simpla:
- Seteaza un expiry pe fiecare entry
- Dupa expirare, urmatorul request va reincarca din DB
- Trade-off: TTL mic = date fresh dar mai multe cache miss; TTL mare = performanta dar date stale

**Event-based invalidation** - invalideaza cand datele se schimba:
- La write, sterge sau actualizeaza cache-ul
- Date mereu fresh
- Complexitate mai mare (trebuie sa stii ce cache entries sunt afectate)

**Write-through** - scrie simultan in cache si DB:
```
Client -> App Server -> scrie in Cache + scrie in DB (simultan)
Avantaj: cache mereu la zi
Dezavantaj: write-uri mai lente (dublu write), cache plin cu date neaccesate
```

**Write-behind (write-back)** - scrie in cache, DB se actualizeaza async:
```
Client -> App Server -> scrie in Cache -> (async) scrie in DB
Avantaj: write-uri foarte rapide
Dezavantaj: risc de pierdere date daca cache-ul cade inainte de sync cu DB
```

**Cache-aside (lazy loading)** - cel mai comun pattern:
```
READ: Citeste din cache -> daca miss -> citeste din DB -> pune in cache
WRITE: Scrie in DB -> invalideaza cache
Avantaj: doar datele accesate ajung in cache
Dezavantaj: primul request dupa invalidare e lent (cold cache)
```

### Angular Service Worker pentru Offline Caching

```typescript
// ngsw-config.json - configurare Angular Service Worker
{
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",     // descarca la instalare
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.webmanifest",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",          // descarca la prima accesare
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(png|jpg|svg)"
        ]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-performance",
      "urls": ["/api/products/**"],
      "cacheConfig": {
        "strategy": "performance",    // cache-first (stale-while-revalidate)
        "maxSize": 100,
        "maxAge": "1h",
        "timeout": "5s"              // daca network-ul dureaza > 5s, serveste din cache
      }
    },
    {
      "name": "api-freshness",
      "urls": ["/api/user/**"],
      "cacheConfig": {
        "strategy": "freshness",      // network-first (date fresh, fallback cache)
        "maxSize": 50,
        "maxAge": "30m",
        "timeout": "3s"
      }
    }
  ]
}
```

### Stale-While-Revalidate Pattern

Serveste datele din cache imediat (chiar daca sunt stale) si revalideaza in background:

```
Request 1 (cache miss):
  Client -> Cache MISS -> Server -> Response + cache
  Latenta: ~200ms

Request 2 (cache hit, fresh):
  Client -> Cache HIT (fresh) -> Response instant
  Latenta: ~1ms

Request 3 (cache hit, stale):
  Client -> Cache HIT (stale) -> Response instant (date vechi)
         -> Background: Server -> actualizeaza cache
  Latenta: ~1ms (utilizatorul vede date vechi dar primeste raspuns instant)

Request 4 (dupa revalidare):
  Client -> Cache HIT (fresh) -> Response instant (date noi)
```

```typescript
// Implementare in Angular cu RxJS
@Injectable({ providedIn: 'root' })
export class StaleWhileRevalidateService {
  private cache = new Map<string, { data: any; timestamp: number }>();
  private maxAge = 60_000; // 1 minut

  getData<T>(url: string): Observable<T> {
    const cached = this.cache.get(url);

    if (cached) {
      const isStale = Date.now() - cached.timestamp > this.maxAge;

      if (isStale) {
        // Serveste stale imediat, revalideaza in background
        return merge(
          of(cached.data as T),  // instant - date stale
          this.fetchAndCache<T>(url).pipe(skip(0))  // background update
        );
      }
      return of(cached.data as T);  // fresh din cache
    }

    return this.fetchAndCache<T>(url);  // prima data - fetch
  }

  private fetchAndCache<T>(url: string): Observable<T> {
    return this.http.get<T>(url).pipe(
      tap(data => this.cache.set(url, { data, timestamp: Date.now() }))
    );
  }
}
```

---

## 5. Microservices vs Monolith

### Monolith

Toata aplicatia intr-un singur deployable unit:

```
+---------------------------------------------------+
|                   MONOLITH                         |
|                                                    |
|  +----------+ +----------+ +----------+            |
|  |  Users   | |  Orders  | | Products |            |
|  |  Module  | |  Module  | |  Module  |            |
|  +----+-----+ +----+-----+ +----+-----+            |
|       |             |            |                  |
|  +----+-------------+------------+-----+            |
|  |          Shared Database            |            |
|  +-------------------------------------+            |
+---------------------------------------------------+
```

**Avantaje**:
- Simplu de dezvoltat si testat (un singur proiect)
- Simplu de deploy-uit (un singur artifact)
- Fara latenta inter-servicii (apeluri in-process)
- Tranzactii ACID simple (o singura baza de date)
- Usor de debug-uit (un singur stack trace)
- Ideal pentru echipe mici (< 10 dev) si stadiu incipient

**Dezavantaje**:
- Cu timpul devine greu de inteles si modificat ("big ball of mud")
- Un bug in orice modul poate afecta totul
- Deploy lent (toata aplicatia pentru o mica schimbare)
- Scaling e all-or-nothing (nu poti scala doar modulul care are nevoie)
- Coupling intre module creste cu timpul
- Tech stack fix (nu poti folosi limbaje diferite per modul)

### Microservices

Aplicatia descompusa in servicii independente, fiecare cu responsabilitate clara:

```
+----------+    +----------+    +----------+
|  Users   |    |  Orders  |    | Products |
| Service  |    | Service  |    | Service  |
|  (Node)  |    |  (Java)  |    |  (Go)    |
+----+-----+    +----+-----+    +----+-----+
     |               |              |
+----+-----+    +----+-----+    +----+-----+
| Users DB |    | Orders DB|    |Products  |
|(Postgres)|    | (Postgres|    | DB       |
+-----------+   +-----------+   |(MongoDB) |
                                +-----------+
```

**Avantaje**:
- Fiecare serviciu poate fi scalat independent
- Echipe autonome (fiecare echipa detine 1-2 servicii)
- Flexibilitate tehnologica (limbaj, framework, DB per serviciu)
- Deploy independent (schimbari izolate, rollback granular)
- Fault isolation (un serviciu down nu afecteaza celelalte)
- Mai usor de inteles individual (fiecare serviciu e mic)

**Dezavantaje**:
- Complexitate operationala enorma (deploy, monitoring, logging)
- Latenta retea intre servicii
- Tranzactii distribuite (saga pattern in loc de ACID)
- Debugging dificil (trace-uri distribuite)
- Data consistency challenges (eventual consistency)
- Necesita infrastructure matura (Kubernetes, service mesh, etc.)

### BFF (Backend For Frontend) Pattern

Un backend dedicat fiecarui tip de frontend:

```
+----------+    +-----------+    +----------+
|  Web App |    | Mobile App|    | Admin App|
| (Angular)|    |  (React   |    | (Angular)|
|          |    |   Native) |    |          |
+----+-----+    +-----+-----+    +----+-----+
     |                |               |
+----+-----+    +-----+-----+    +----+-----+
|  Web BFF |    | Mobile BFF|    | Admin BFF|
|          |    |           |    |          |
+----+-----+    +-----+-----+    +----+-----+
     |                |               |
     +--------+-------+-------+-------+
              |               |
        +-----+-----+  +-----+-----+
        |  Users    |  |  Orders   |
        |  Service  |  |  Service  |
        +-----------+  +-----------+
```

**De ce BFF**:
- Web-ul are nevoie de date diferite fata de mobile (ecran mare vs mic)
- Mobile-ul are bandwidth limitat - BFF agrega si comprima datele
- Admin-ul are nevoie de endpointuri speciale (bulk operations, reports)
- Fiecare BFF poate fi optimizat pentru clientul sau

### API Gateway Pattern

Un singur punct de intrare pentru toti clientii:

```
Clienti ----> API Gateway ----> Microservices
              |
              +-- Authentication / Authorization
              +-- Rate limiting
              +-- Request routing
              +-- Response aggregation
              +-- SSL termination
              +-- Logging / Monitoring
              +-- Circuit breaking
              +-- Request/Response transformation
```

Implementari populare: Kong, AWS API Gateway, Azure API Management, Envoy, Traefik.

### Cand migrezi de la monolith la microservices

**NU migra daca**:
- Echipa e mica (< 10 devs)
- Produsul e in stadiu incipient (nu stii ce va creste)
- Nu ai infrastructure matura (CI/CD, monitoring, Kubernetes)
- Performanta e OK si scaling nu e o problema

**Migra cand**:
- Echipele se blocheaza reciproc pe deploy-uri
- Un modul are nevoie de scaling diferit (ex: search are 100x trafic)
- Time-to-market sufera din cauza coupling-ului
- Ai nevoie de tech stack diferit pentru un modul

**Strategia Strangler Fig** - migrare incrementala:
```
Faza 1: Monolith serveste totul
        +---------------+
        |   Monolith    |
        +---------------+

Faza 2: Noul serviciu primeste o parte din trafic
        +------+   +--------+
        | Mono |   | Users  |
        | lith |   |Service |
        +------+   +--------+

Faza 3: Tot mai mult trafic migrat
        +----+  +------+ +------+ +--------+
        |Mono|  |Users | |Orders| |Products|
        |lith|  |Svc   | |Svc   | |Svc     |
        +----+  +------+ +------+ +--------+

Faza 4: Monolith-ul dispare
        +------+ +------+ +--------+ +------+
        |Users | |Orders| |Products| |Search|
        |Svc   | |Svc   | |Svc     | |Svc   |
        +------+ +------+ +--------+ +------+
```

### Modular Monolith - varianta de mijloc

Pastreaza avantajele monolith-ului (simplu de deploy) dar cu module bine izolate:

```
+----------------------------------------------------------+
|                    MODULAR MONOLITH                        |
|                                                           |
|  +----------+    +----------+    +----------+             |
|  |  Users   |    |  Orders  |    | Products |             |
|  |  Module  |    |  Module  |    |  Module  |             |
|  |          |    |          |    |          |             |
|  | - API    |    | - API    |    | - API    |             |
|  | - Domain |    | - Domain |    | - Domain |             |
|  | - Data   |    | - Data   |    | - Data   |             |
|  +----+-----+    +----+-----+    +----+-----+             |
|       |               |              |                    |
|  Module comunica DOAR prin interfete publice (API)        |
|  Fiecare modul are schema proprie in DB (isolation)       |
|                                                           |
|  +----+---------------+------------+-----+                |
|  |     Shared Database (schemas separate) |                |
|  | [users.*]    [orders.*]   [products.*] |                |
|  +----------------------------------------+                |
+----------------------------------------------------------+
```

**Avantaje**:
- Simplu de deploy-uit ca un monolith
- Module izolate ca microservices
- Poate fi descompus in microservices mai tarziu (daca e nevoie)
- Nu necesita infrastructure complexa

**Reguli**:
- Modulele comunica doar prin interfete publice
- Nu se acceseaza tabelele altui modul direct
- Fiecare modul are propriile entitati si repository-uri
- Dependente declarate explicit

---

## 6. API Design

### REST (Representational State Transfer)

**Principii fundamentale**:
- Resources identificate prin URI-uri
- Operatii prin HTTP methods (verbe)
- Stateless - fiecare request contine tot contextul
- Representation - resursa poate fi in JSON, XML, etc.

**HTTP Methods si semantica**:

| Method | Operatie | Idempotent | Body | Exemplu |
|--------|----------|------------|------|---------|
| GET | Citire | Da | Nu | `GET /api/users/123` |
| POST | Creare | Nu | Da | `POST /api/users` |
| PUT | Inlocuire completa | Da | Da | `PUT /api/users/123` |
| PATCH | Actualizare partiala | Nu* | Da | `PATCH /api/users/123` |
| DELETE | Stergere | Da | Nu | `DELETE /api/users/123` |

**Status Codes importante**:

```
2xx Success:
  200 OK              - Request reusit (GET, PUT, PATCH, DELETE)
  201 Created         - Resursa creata (POST)
  204 No Content      - Succes fara body (DELETE)

3xx Redirection:
  301 Moved Permanently
  304 Not Modified    - Cache valid (conditional GET)

4xx Client Error:
  400 Bad Request     - Payload invalid, validare esuata
  401 Unauthorized    - Neautentificat (lipseste token)
  403 Forbidden       - Autentificat dar nu are permisiuni
  404 Not Found       - Resursa nu exista
  409 Conflict        - Conflict (duplicate, version mismatch)
  422 Unprocessable   - Validare semantica esuata
  429 Too Many Reqs   - Rate limit atins

5xx Server Error:
  500 Internal Error  - Eroare neasteptata pe server
  502 Bad Gateway     - Upstream server error
  503 Unavailable     - Server indisponibil (maintenance)
  504 Gateway Timeout - Upstream timeout
```

**Pagination**:

```
# Offset-based (simplu dar problematic la scale)
GET /api/products?page=3&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "page": 3,
    "limit": 20,
    "total": 1500,
    "totalPages": 75
  }
}

# Cursor-based (performant, consistent)
GET /api/products?cursor=eyJpZCI6MTAwfQ&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

**Filtering si Sorting**:
```
GET /api/products?category=electronics&minPrice=100&maxPrice=500&sort=-createdAt,name
```

### GraphQL

**Schema-first** - definesti schema, clientul cere exact ce are nevoie:

```graphql
# Schema
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
  followers: [User!]!
  followersCount: Int!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  createdAt: DateTime!
}

type Query {
  user(id: ID!): User
  users(filter: UserFilter, pagination: PaginationInput): UserConnection!
  post(id: ID!): Post
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  updateUser(id: ID!, input: UpdateUserInput!): User!
}

type Subscription {
  postCreated(userId: ID!): Post!
  commentAdded(postId: ID!): Comment!
}
```

```graphql
# Query - clientul cere exact ce afiseaza
query {
  user(id: "123") {
    name
    posts {
      title
      comments {
        content
      }
    }
  }
}

# Response - exact structura ceruta, nimic in plus
{
  "data": {
    "user": {
      "name": "Ion",
      "posts": [
        {
          "title": "Primul post",
          "comments": [{ "content": "Super!" }]
        }
      ]
    }
  }
}
```

**Avantaje fata de REST**:
- Fara over-fetching (primesti doar campurile cerute)
- Fara under-fetching (un singur request in loc de multiple)
- Schema tipizata cu introspection
- Excelent pentru frontend-uri cu nevoi diverse (web, mobile)

**Dezavantaje**:
- Complexitate server-side (resolvers, dataloader pentru N+1)
- Caching mai dificil (POST pentru toate query-urile, nu se poate cache pe URL)
- File upload non-standard
- Potential pentru query-uri foarte costisitoare (rate limiting pe complexitate, nu pe request)

**Integrare Angular cu Apollo Client**:
```typescript
@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly GET_USER = gql`
    query GetUser($id: ID!) {
      user(id: $id) {
        name
        email
        posts {
          title
        }
      }
    }
  `;

  constructor(private apollo: Apollo) {}

  getUser(id: string): Observable<User> {
    return this.apollo.watchQuery<{ user: User }>({
      query: this.GET_USER,
      variables: { id },
    }).valueChanges.pipe(map(result => result.data.user));
  }
}
```

### gRPC

**Protocol Buffers** - serializare binara, mult mai eficienta decat JSON:

```protobuf
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);        // Server streaming
  rpc UploadAvatar (stream Chunk) returns (UploadResponse);      // Client streaming
  rpc Chat (stream Message) returns (stream Message);            // Bidirectional
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  repeated Post posts = 4;
}

message GetUserRequest {
  string id = 1;
}
```

**Cand folosesti gRPC**:
- Comunicare intre microservices (intern, nu public)
- Cand performanta e critica (10x mai rapid decat JSON/REST)
- Streaming bidirectional
- Polyglot environments (genereaza clienti in orice limbaj)
- NU pentru browser-to-server direct (necesita gRPC-Web proxy)

### API Versioning Strategies

```
# URL Path versioning (cel mai comun)
GET /api/v1/users
GET /api/v2/users

# Header versioning
GET /api/users
Accept: application/vnd.myapp.v2+json

# Query parameter
GET /api/users?version=2

# Content negotiation
GET /api/users
Accept: application/json; version=2
```

**Recomandare**: URL path versioning pentru simplitate. Mentine v1 activ cat timp are clienti. Deprecation policy: anunta cu 6-12 luni inainte de sunset.

### Rate Limiting

Protejeaza API-ul de abuz si asigura fair usage:

```
Algoritmi:
1. Fixed Window    - N requests per minut (simplu, burst la granita)
2. Sliding Window  - N requests in ultimele 60 secunde (mai precis)
3. Token Bucket    - tokens regenerate constant (permite burst-uri controlate)
4. Leaky Bucket    - proceseaza la rata fixa (output constant)

Headers standard:
X-RateLimit-Limit: 100          (limita)
X-RateLimit-Remaining: 42       (ramase)
X-RateLimit-Reset: 1708300800   (timestamp cand se reseteaza)
Retry-After: 30                 (cate secunde sa astepte - la 429)
```

### OpenAPI / Swagger Documentation

```yaml
# openapi.yaml - exemplu simplificat
openapi: 3.0.0
info:
  title: Products API
  version: 1.0.0

paths:
  /api/products:
    get:
      summary: List products
      parameters:
        - name: category
          in: query
          schema:
            type: string
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Product'
    post:
      summary: Create product
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateProductInput'
      responses:
        '201':
          description: Created

components:
  schemas:
    Product:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        price:
          type: number
```

---

## 7. WebSockets si Real-time

### WebSocket Protocol vs HTTP

```
HTTP (Request-Response):
Client ----Request----> Server
Client <---Response---- Server
(conexiune inchisa sau keep-alive idle)

WebSocket (Full-duplex):
Client ----HTTP Upgrade----> Server
Client <---101 Switching---- Server
Client <====Bidirectional====> Server  (conexiune persistenta)
Client <====Bidirectional====> Server
Client ----Close-----------> Server
```

**HTTP** - client initiaza mereu comunicarea, server raspunde:
- Simplu, bine inteles, caching nativ
- Overhead per request (headers, TCP handshake daca nu e keep-alive)
- Nu poate face server push (server-ul nu poate initia comunicarea)

**WebSocket** - conexiune full-duplex persistenta:
- Incepe cu HTTP Upgrade handshake, apoi trece pe protocolul ws://
- Overhead minimal per mesaj (2-14 bytes frame header vs sute de bytes HTTP headers)
- Server poate trimite date oricand (push)
- Mentine state (conexiunea e persistenta, stateful)
- Consum de resurse per conexiune activa

### Server-Sent Events (SSE)

Comunicare unidirectionala server -> client prin HTTP:

```
Client ----GET /events----> Server
Client <---data: msg1------- Server
Client <---data: msg2------- Server
Client <---data: msg3------- Server
(conexiune deschisa, server trimite cand vrea)
```

**Avantaje fata de WebSocket**:
- Functioneaza peste HTTP standard (nu necesita upgrade)
- Reconnection automata (browser-ul reconecteaza automat)
- Compatibil cu HTTP/2 multiplexing
- Simplu de implementat pe server (text/event-stream)

**Dezavantaje**:
- Unidirectional (doar server -> client)
- Limita de conexiuni per domeniu in HTTP/1.1 (6 per browser)
- Nu suporta binary data nativ

**Cand folosesti SSE**: notifications, live feed updates, stock prices, build status.

### Long Polling

```
Iteratia 1:
Client ----GET /updates----> Server
                              (server asteapta pana are date noi)
Client <---Response---------- Server (date noi disponibile)

Iteratia 2 (imediat dupa raspuns):
Client ----GET /updates----> Server
                              (server asteapta din nou...)
```

Emuleaza push prin requesturi HTTP repetate. Server-ul tine request-ul deschis pana are date sau timeout. Client-ul face imediat alt request dupa ce primeste raspuns.

**Avantaje**: functioneaza oriunde, fara protocol special
**Dezavantaje**: overhead mare, scalabilitate slaba (un thread per conexiune in servere traditionale)

### Comparatie

| Feature | WebSocket | SSE | Long Polling |
|---------|-----------|-----|--------------|
| Directie | Bidirectional | Server -> Client | Server -> Client |
| Protocol | ws:// | HTTP | HTTP |
| Overhead | Minim | Mic | Mare |
| Reconnection | Manual | Automat | Manual |
| Binary data | Da | Nu (base64) | Da |
| HTTP/2 compat | Separat | Nativ | Nativ |
| Scalabilitate | Buna | Buna | Slaba |
| Complexitate | Medie | Mica | Mica |
| Use case | Chat, gaming | Notifications | Fallback |

### Socket.IO

Librarie care abstractizeaza real-time communication cu fallback automat:
- Incearca WebSocket, daca nu merge -> long polling
- Reconnection automata cu exponential backoff
- Room-uri si namespace-uri (grupare logica a conexiunilor)
- Broadcasting (trimite la toti clientii dintr-un room)
- Acknowledgements (confirmare ca mesajul a ajuns)

### Angular Integration Example

```typescript
// websocket.service.ts
import { Injectable, OnDestroy } from '@angular/core';
import { Observable, Subject, timer, EMPTY } from 'rxjs';
import { webSocket, WebSocketSubject } from 'rxjs/webSocket';
import { retry, tap, switchMap, catchError } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class WebSocketService implements OnDestroy {
  private socket$: WebSocketSubject<any> | null = null;
  private messagesSubject = new Subject<any>();
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;

  messages$ = this.messagesSubject.asObservable();

  connect(url: string): void {
    if (this.socket$) {
      return; // deja conectat
    }

    this.socket$ = webSocket({
      url,
      openObserver: {
        next: () => {
          console.log('WebSocket connected');
          this.reconnectAttempts = 0;
        }
      },
      closeObserver: {
        next: (event) => {
          console.log('WebSocket closed, reconnecting...', event);
          this.socket$ = null;
          this.reconnect(url);
        }
      }
    });

    this.socket$.subscribe({
      next: (msg) => this.messagesSubject.next(msg),
      error: (err) => {
        console.error('WebSocket error:', err);
        this.socket$ = null;
        this.reconnect(url);
      }
    });
  }

  send(data: any): void {
    if (this.socket$) {
      this.socket$.next(data);
    }
  }

  private reconnect(url: string): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnect attempts reached');
      return;
    }

    this.reconnectAttempts++;
    const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
    // Exponential backoff: 2s, 4s, 8s, 16s, 30s, 30s...

    setTimeout(() => this.connect(url), delay);
  }

  ngOnDestroy(): void {
    this.socket$?.complete();
  }
}

// SSE cu Angular
@Injectable({ providedIn: 'root' })
export class SseService {
  private zone = inject(NgZone);

  getEvents(url: string): Observable<MessageEvent> {
    return new Observable(observer => {
      const eventSource = new EventSource(url);

      eventSource.onmessage = (event) => {
        // SSE ruleaza in afara NgZone - trebuie re-introdus
        this.zone.run(() => observer.next(event));
      };

      eventSource.onerror = (error) => {
        this.zone.run(() => observer.error(error));
      };

      // Cleanup la unsubscribe
      return () => eventSource.close();
    });
  }
}

// Componenta care foloseste WebSocket
@Component({
  selector: 'app-chat',
  template: `
    <div class="messages">
      @for (msg of messages(); track msg.id) {
        <div class="message">
          <strong>{{ msg.author }}:</strong> {{ msg.text }}
        </div>
      }
    </div>
    <input #input (keyup.enter)="send(input.value); input.value = ''" />
  `
})
export class ChatComponent implements OnInit, OnDestroy {
  private wsService = inject(WebSocketService);
  private destroyRef = inject(DestroyRef);

  messages = signal<ChatMessage[]>([]);

  ngOnInit(): void {
    this.wsService.connect('wss://api.example.com/chat');

    this.wsService.messages$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe(msg => {
        this.messages.update(msgs => [...msgs, msg]);
      });
  }

  send(text: string): void {
    this.wsService.send({ type: 'message', text, author: 'Me' });
  }

  ngOnDestroy(): void {
    // WebSocketService handles cleanup
  }
}
```

---

## 8. Database Design Basics

### SQL vs NoSQL

**SQL (Relational)** - date structurate cu relatii:

| Caracteristica | SQL | NoSQL |
|----------------|-----|-------|
| Schema | Fixa, predefinita | Flexibila, schema-less |
| Relatii | JOINs, foreign keys | Embedded documents, referinte |
| Scalare | Vertical (cu effort horizontal) | Horizontal nativ |
| Transactions | ACID | BASE (eventual consistency) |
| Query language | SQL standard | Variaza per DB |
| Best for | Date structurate, relatii complexe | Date semi-structurate, scale |

**Cand SQL** (PostgreSQL, MySQL):
- Relatii complexe intre entitati (e-commerce: users, orders, products, reviews)
- Nevoia de tranzactii ACID (banking, inventory)
- Date cu schema bine definita si stabila
- Query-uri complexe cu JOIN, GROUP BY, subqueries
- Reporting si analytics

**Cand NoSQL**:
- **Document store** (MongoDB, DynamoDB) - date semi-structurate, schema variabila, JSON nativ. Ex: CMS, cataloge de produse cu atribute diferite
- **Key-value** (Redis, DynamoDB) - caching, session storage, counters. Extrem de rapid
- **Wide-column** (Cassandra, HBase) - time-series data, IoT, log storage. Write-heavy la scale
- **Graph** (Neo4j, Neptune) - relatii complexe (social networks, recommendation engines, fraud detection)

### Normalization vs Denormalization

**Normalization** - elimina redundanta, date in tabele separate:

```sql
-- 3NF (Third Normal Form) - normalized
CREATE TABLE users (
  id UUID PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255) UNIQUE
);

CREATE TABLE addresses (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  street VARCHAR(255),
  city VARCHAR(100),
  country VARCHAR(100)
);

CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  total DECIMAL(10,2),
  created_at TIMESTAMP
);

-- Citire necesita JOIN
SELECT u.name, a.city, o.total
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN addresses a ON a.user_id = u.id
WHERE o.id = '...';
```

**Denormalization** - duplica date pentru performanta de citire:

```sql
-- Denormalized - date duplicate dar citire rapida
CREATE TABLE orders_denormalized (
  id UUID PRIMARY KEY,
  user_id UUID,
  user_name VARCHAR(100),        -- duplicat din users
  user_email VARCHAR(255),       -- duplicat din users
  shipping_city VARCHAR(100),    -- duplicat din addresses
  shipping_country VARCHAR(100), -- duplicat din addresses
  total DECIMAL(10,2),
  created_at TIMESTAMP
);

-- Citire fara JOIN
SELECT user_name, shipping_city, total
FROM orders_denormalized
WHERE id = '...';
```

**Trade-off**:
- Normalized = write simplu, read complex (JOIN-uri), fara redundanta
- Denormalized = write complex (trebuie actualizat in mai multe locuri), read simplu, redundanta

**In practica**: incepe normalized, denormalizeaza selectiv acolo unde ai bottleneck pe citire.

### Indexing Strategies

```sql
-- B-Tree index (default, cel mai comun)
-- Bun pentru: equality, range, sorting, prefix search
CREATE INDEX idx_users_email ON users(email);
-- Acum: SELECT * FROM users WHERE email = 'ion@x.com' e rapid

-- Composite index (ordine conteaza!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);
-- Ajuta: WHERE user_id = X AND created_at > Y
-- Ajuta: WHERE user_id = X ORDER BY created_at DESC
-- NU ajuta: WHERE created_at > Y (fara user_id - left prefix rule)

-- Partial index (doar un subset de date)
CREATE INDEX idx_orders_active ON orders(status) WHERE status = 'active';
-- Index mai mic, doar pentru query-uri pe ordere active

-- GIN index (pentru full-text search, JSONB, arrays)
CREATE INDEX idx_products_tags ON products USING GIN(tags);
-- Acum: WHERE tags @> '{"electronics"}' e rapid
```

**Reguli de indexing**:
- Indexeaza coloanele din WHERE, JOIN, ORDER BY
- Coloanele cu cardinalitate mare (multe valori distincte) beneficiaza cel mai mult
- Fiecare index incetineste write-urile (trebuie actualizat la INSERT/UPDATE)
- Foloseste EXPLAIN ANALYZE pentru a verifica planul de query

### ACID Properties

**A - Atomicity**: tranzactia se executa complet sau deloc (all or nothing)
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;  -- ambele sau niciuna
```

**C - Consistency**: DB-ul trece dintr-o stare valida in alta stare valida (constraints respectate)

**I - Isolation**: tranzactiile concurente nu se afecteaza reciproc
- READ UNCOMMITTED: vede date necommitted (dirty reads)
- READ COMMITTED: vede doar date committed (default PostgreSQL)
- REPEATABLE READ: vede un snapshot consistent (default MySQL InnoDB)
- SERIALIZABLE: executie ca si cum ar fi secventiale (cel mai strict)

**D - Durability**: o data committed, datele supravietuiesc crashuri (scrise pe disk, WAL)

### Eventual Consistency

In sistemele distribuite, nu poti avea mereu consistency imediat. Eventual consistency garanteaza ca, in absenta altor update-uri, toate replicile vor converge la aceeasi valoare.

```
Write -> Primary DB
              |
         Replication (async)
              |
    +---------+---------+
    |         |         |
 Replica 1  Replica 2  Replica 3
 (stale)    (stale)    (stale)
              |
         Dupa cateva ms...
              |
    +---------+---------+
    |         |         |
 Replica 1  Replica 2  Replica 3
 (fresh)    (fresh)    (fresh)     <- eventual consistent
```

**Implicatii practice**:
- Dupa un write, un read imediat pe o replica poate returna date vechi
- Read-after-write consistency: citeste de pe primary dupa write
- Acceptabil pentru: feed social, counters, recommendations
- Inacceptabil pentru: balanta cont bancar, inventar critic, booking sisteme

### Common Databases

| Database | Tip | Use Case | Notabil |
|----------|-----|----------|---------|
| **PostgreSQL** | Relational | General purpose, complex queries | JSONB, extensions, cel mai versatil |
| **MySQL** | Relational | Web apps, OLTP | Rapid, matur, MySQL vs MariaDB |
| **MongoDB** | Document | Flexible schema, prototyping | JSON-like, agregation framework |
| **Redis** | Key-Value | Cache, session, pub/sub | In-memory, sub 1ms latenta |
| **DynamoDB** | Key-Value/Document | Serverless, scale predictibil | AWS managed, single-digit ms |
| **Cassandra** | Wide-column | Write-heavy, time-series | Liniar scaling, no single point of failure |
| **Elasticsearch** | Search engine | Full-text search, logs | Inverted index, aggregations |
| **Neo4j** | Graph | Social networks, recommendations | Cypher query language |

---

## 9. CAP Theorem

### Cele trei proprietati

In orice sistem distribuit, poti garanta doar doua din trei:

**C - Consistency**: Toate nodurile vad aceleasi date in acelasi moment. Un read dupa un write returneaza mereu valoarea scrisa.

**A - Availability**: Fiecare request primeste un raspuns (success sau failure), chiar daca unele noduri sunt down.

**P - Partition Tolerance**: Sistemul continua sa functioneze chiar daca comunicarea intre noduri e intrerupta (network partition).

```
           C (Consistency)
          / \
         /   \
        /     \
       /  CP   \
      /  systems \
     /     |     \
    /      |      \
   --------+--------
  |   Partition    |
  |   Tolerance    |
   \      |      /
    \     |     /
     \ AP |    /
      \systems/
       \   /
        \ /
         A (Availability)

IMPORTANT: P nu e optional in sisteme distribuite.
Network partitions VOR aparea. Deci alegerea reala e intre CP si AP.
```

### CP Systems (Consistency + Partition Tolerance)

Sacrifica availability - unele requesturi pot fi refuzate in timpul unei partitii.

**Exemplu: MongoDB (cu majority read/write concern)**
- Daca primary-ul nu e disponibil, write-urile sunt refuzate pana se alege un nou primary
- Garanteaza ca nu vei citi date inconsistente
- **Cand**: banking, inventory management, booking systems

**Exemplu: HBase, Zookeeper**
- Prefera sa refuze serviciul decat sa returneze date potencial incorecte

### AP Systems (Availability + Partition Tolerance)

Sacrifica consistency - sistemul raspunde mereu dar datele pot fi temporar inconsistente.

**Exemplu: Cassandra**
- Scrie pe orice nod disponibil (chiar si in timpul partitiei)
- Conflicte rezolvate prin last-write-wins sau vector clocks
- **Cand**: social media feeds, counters, IoT data collection

**Exemplu: DynamoDB (default settings)**
- Eventually consistent reads (default) - rapid, potential stale
- Strongly consistent reads (optional) - mai lent, mereu fresh

### PACELC Theorem (extensie a CAP)

"If Partition, choose A or C; Else (normal operation), choose Latency or Consistency"

```
IF partition THEN:
  - PA: choose Availability (accept stale data)
  - PC: choose Consistency (reject some requests)

ELSE (no partition, normal operation) THEN:
  - EL: choose Latency (faster responses, eventual consistency)
  - EC: choose Consistency (slower responses, strong consistency)
```

| Sistem | Partition | Else (normal) | Clasificare |
|--------|-----------|---------------|-------------|
| DynamoDB | PA | EL | PA/EL |
| Cassandra | PA | EL | PA/EL |
| MongoDB | PC | EC | PC/EC |
| PostgreSQL (single) | - | EC | -/EC |
| MySQL + replicas | PC | EL | PC/EL |

### Implicatii practice pentru frontend (Angular)

Ca frontend engineer, CAP Theorem influenteaza cum construiesti UI-ul:

```typescript
// 1. Optimistic UI - presupune AP (availability first)
// Actualizeaza UI-ul imediat, rezolva conflicte mai tarziu
@Component({ ... })
export class TodoListComponent {
  todos = signal<Todo[]>([]);

  addTodo(text: string): void {
    const optimisticTodo: Todo = {
      id: crypto.randomUUID(),   // ID temporar
      text,
      completed: false,
      synced: false,             // flag: nu e inca pe server
    };

    // Actualizeaza UI imediat (optimistic)
    this.todos.update(todos => [...todos, optimisticTodo]);

    // Sync cu server-ul in background
    this.api.createTodo(text).pipe(
      tap(serverTodo => {
        // Inlocuieste todo-ul optimistic cu cel real
        this.todos.update(todos =>
          todos.map(t => t.id === optimisticTodo.id
            ? { ...serverTodo, synced: true }
            : t
          )
        );
      }),
      catchError(err => {
        // Rollback daca server-ul refuza
        this.todos.update(todos =>
          todos.filter(t => t.id !== optimisticTodo.id)
        );
        this.notification.error('Nu s-a putut adauga todo-ul');
        return EMPTY;
      })
    ).subscribe();
  }
}

// 2. Retry cu exponential backoff - handles network partitions
// Cand server-ul nu raspunde, reincearca cu delay crescator
this.http.get('/api/data').pipe(
  retry({
    count: 3,
    delay: (error, retryCount) => timer(1000 * Math.pow(2, retryCount))
  }),
  catchError(() => {
    // Fallback la date din cache (offline-first)
    return this.localCache.get('data');
  })
)

// 3. Conflict resolution in UI
// Cand doua persoane editeaza simultan, trebuie decis cine "castiga"
// Strategii:
// - Last-write-wins (simplu dar pierde date)
// - Show conflict to user (Google Docs style)
// - Merge automata (CRDTs - Conflict-free Replicated Data Types)
```

---

## 10. Diagrame de arhitectura tipice

### E-commerce Platform

```
                        +------------------+
                        |   User Browser   |
                        | (Angular SPA)    |
                        +--------+---------+
                                 |
                        +--------+---------+
                        |      CDN         |
                        | (CloudFront)     |
                        | JS/CSS/images    |
                        +--------+---------+
                                 |
                        +--------+---------+
                        |  WAF / DDoS      |
                        |  Protection      |
                        +--------+---------+
                                 |
                        +--------+---------+
                        | L7 Load Balancer |
                        | (ALB / Nginx)    |
                        +--------+---------+
                                 |
                        +--------+---------+
                        |   API Gateway    |
                        | - Auth (JWT)     |
                        | - Rate Limiting  |
                        | - Routing        |
                        +--------+---------+
                                 |
          +----------+-----------+----------+-----------+
          |          |           |          |           |
    +-----+----+ +--+------+ +-+-------+ ++---------+ +--+--------+
    | User     | | Product | | Order   | | Payment  | | Search    |
    | Service  | | Service | | Service | | Service  | | Service   |
    | (CRUD,   | | (catalog| | (cart,  | | (Stripe  | |(Elastic-  |
    |  auth)   | |  stock) | |  order) | |  integ.) | | search)   |
    +-----+----+ +--+------+ +---+-----+ ++--------++ +--+--------+
          |          |            |         |              |
    +-----+----+ +--+------+ +--+------+ +-+--------+ +--+--------+
    |PostgreSQL| |PostgreSQL| |PostgreSQL| |PostgreSQL| |Elastic-  |
    | (users)  | |(products)| | (orders)| | (payments| | search   |
    +----------+ +--+------+ +---+-----+ +----------+ +----------+
                    |             |
                +---+---+   +----+-----+
                | Redis |   | Message  |
                | Cache |   | Queue    |
                |(prods)|   |(RabbitMQ)|
                +-------+   +----+-----+
                                  |
                          +-------+--------+
                          |                |
                   +------+-----+   +------+------+
                   | Notification|  | Analytics   |
                   | Service     |  | Service     |
                   | (email,push)|  | (events,    |
                   +-------------+  |  metrics)   |
                                    +-------------+

Flow: Place Order
1. Client -> API Gateway -> Order Service: POST /orders
2. Order Service -> Product Service: verificare stoc (sync, gRPC)
3. Order Service -> Payment Service: initiere plata (sync)
4. Payment Service -> Stripe: procesare plata
5. Order Service -> Message Queue: order.created event
6. Notification Service <- MQ: trimite email confirmare (async)
7. Product Service <- MQ: decrementeaza stoc (async)
8. Analytics Service <- MQ: inregistreaza eveniment (async)
```

### Real-time Chat Application

```
                   +------------------+   +------------------+
                   | Web Client       |   | Mobile Client    |
                   | (Angular)        |   | (React Native)   |
                   +--------+---------+   +--------+---------+
                            |                      |
                   +--------+----------------------+---------+
                   |           L7 Load Balancer               |
                   |  (sticky sessions / IP hash              |
                   |   pentru WebSocket persistence)           |
                   +--------+---------+---------+-------------+
                            |         |         |
                   +--------+-+ +-----+----+ +--+----------+
                   | WS Server| |WS Server | | WS Server   |
                   | Node 1   | | Node 2   | | Node 3      |
                   +-----+----+ +----+-----+ +------+------+
                         |           |              |
                   +-----+-----------+--------------+------+
                   |            Redis Pub/Sub               |
                   |  (broadcast mesaje intre WS nodes)     |
                   +-----+-----------+---------------------+
                         |           |
                   +-----+----+ +---+--------+
                   |  Redis   | | MongoDB    |
                   |  (online | | (mesaje,   |
                   |  status, | |  conv.,    |
                   |  typing) | |  users)    |
                   +----------+ +---+--------+
                                    |
                              +-----+------+
                              | S3 / Blob  |
                              | (media,    |
                              |  fisiere)  |
                              +------------+

Flow: Send Message
1. Client -> WS Server Node 1: { type: "message", room: "abc", text: "Hello" }
2. WS Server Node 1 -> MongoDB: persist message
3. WS Server Node 1 -> Redis Pub/Sub: publish to channel "room:abc"
4. Redis -> ALL WS Servers: broadcast
5. WS Server Node 2 -> Clienti din room "abc" conectati la Node 2: push message
6. WS Server Node 3 -> Clienti din room "abc" conectati la Node 3: push message

Flow: Typing Indicator
1. Client -> WS Server: { type: "typing", room: "abc" }
2. WS Server -> Redis: SET "typing:abc:userId" EX 3 (expira in 3s)
3. WS Server -> Redis Pub/Sub: publish typing event
4. Redis -> Other WS Servers -> Clients: show "User is typing..."
```

### Content Management System (CMS)

```
                        Authors / Editors        Public Users
                              |                       |
                     +--------+---------+    +--------+---------+
                     | Admin Dashboard  |    | Public Website   |
                     | (Angular SPA)    |    | (Angular SSR /   |
                     +--------+---------+    |  Static / SSG)   |
                              |              +--------+---------+
                              |                       |
                     +--------+---------+    +--------+---------+
                     | Admin BFF        |    |      CDN         |
                     | (auth, file      |    | (cached pages,   |
                     |  upload)         |    |  assets, images) |
                     +--------+---------+    +--------+---------+
                              |                       |
                     +--------+-----------------------+---------+
                     |              API Gateway                  |
                     |  - JWT Auth (admin routes)                |
                     |  - API Key (public routes)                |
                     |  - Rate Limiting (public)                 |
                     +---------+----------+---------+-----------+
                               |          |         |
                    +----------+-+ +------+---+ +---+----------+
                    | Content    | | Media    | | Search       |
                    | Service    | | Service  | | Service      |
                    | - articles | | - upload | | - full-text  |
                    | - pages    | | - resize | | - faceted    |
                    | - taxonomy | | - CDN    | | - suggest    |
                    +-----+------+ +----+-----+ +---+----------+
                          |             |            |
                    +-----+------+ +---+------+ +---+----------+
                    | PostgreSQL | |   S3     | | Elasticsearch|
                    | (content,  | | (media)  | | (search      |
                    |  versions, | |          | |  index)      |
                    |  workflow) | +----------+ +--------------+
                    +-----+------+
                          |
                    +-----+------+      +---------------+
                    |   Redis    |      | Message Queue |
                    | - page     |      | - reindex     |
                    |   cache    |      | - image       |
                    | - session  |      |   processing  |
                    +------------+      | - webhook     |
                                        |   delivery    |
                                        +---------------+

Flow: Publish Article
1. Editor -> Admin Dashboard: editeaza articol (autosave la fiecare 30s)
2. Admin BFF -> Content Service: PUT /articles/:id/draft
3. Content Service -> PostgreSQL: salveaza draft + versiune noua
4. Editor -> "Publish" button
5. Content Service -> PostgreSQL: status = published, published_at = now()
6. Content Service -> Message Queue: article.published event
7. Search Service <- MQ: reindexeaza articolul in Elasticsearch
8. CDN <- MQ: invalideaza cache-ul pentru pagina articolului
9. Webhook Service <- MQ: notifica subscriberi (RSS, email, Slack)

Content Versioning:
+----+----------+---------+------------+
| v1 | Draft    | Author  | 2026-02-15 |
| v2 | Review   | Author  | 2026-02-16 |
| v3 | Approved | Editor  | 2026-02-17 |
| v4 | Published| Editor  | 2026-02-18 |  <- current
+----+----------+---------+------------+
```

---

## Intrebari frecvente de interviu

### 1. "Designeaza un URL Shortener (tip bit.ly)"

**Abordare**:
- Cerinte: shorten URL, redirect, analytics (click count, referrer)
- Estimari: 100M URLs create/luna, 10B redirects/luna (read-heavy 100:1)
- Key decision: cum generezi short code? Base62 encoding pe counter (auto-increment), hash + collision check, sau pre-generated IDs
- Storage: SQL (PostgreSQL) pentru URL mappings, Redis cache pentru top URLs
- Redirection: 301 (permanent, browser cache-uieste) vs 302 (temporary, fiecare click trece prin server - necesar pentru analytics)
- Scaling: cache agresiv (cele mai accesate URL-uri in Redis), sharding pe hash(shortCode)

### 2. "Designeaza un sistem de notifications (push, email, SMS)"

**Abordare**:
- Prioritize notifications (urgent vs non-urgent)
- Message Queue pentru decoupling si retry (SQS, RabbitMQ)
- Notification preferences per user (ce canal, ce frecventa)
- Rate limiting per user (nu spama)
- Template engine pentru mesaje
- Delivery tracking (sent, delivered, read)
- Batch processing pentru non-urgent (digest email zilnic)
- Angular: SSE sau WebSocket pentru push in browser, Service Worker pentru background notifications

### 3. "Designeaza un sistem de file upload si processing"

**Abordare**:
- Pre-signed URLs pentru upload direct la S3 (evita server-ul ca bottleneck)
- Chunked upload pentru fisiere mari (resumable uploads)
- Processing pipeline: upload -> message queue -> worker (resize, transcode, scan virus)
- CDN pentru serving
- Metadata in DB (PostgreSQL), fisiere in Object Storage (S3)
- Angular: progress tracking cu HttpClient reportProgress, drag-and-drop upload
- Limits: max file size, allowed types, rate limiting per user

### 4. "Designeaza un feed social media (tip Facebook/Twitter)"

**Abordare**:
- Fan-out on write (push model): cand un user posteaza, scrie in feed-ul fiecarui follower. Bun cand followeri < 1000. Problematic pentru celebrities (milioane de followeri)
- Fan-out on read (pull model): la fiecare load, agrega posturile din toti cei pe care ii urmaresti. Read-time computation, mai lent dar nu are problema celebrity
- Hybrid: fan-out on write pentru useri normali, fan-out on read pentru celebrities
- Ranking: chronological vs algorithmic (ML scoring)
- Infinite scroll cu cursor-based pagination
- Caching: pre-computed feed in Redis (lista de post IDs per user)
- Angular: virtual scrolling pentru performanta, intersection observer pentru lazy loading

### 5. "Designeaza un sistem de rate limiting"

**Abordare**:
- Algoritm: Token Bucket (permite burst-uri) sau Sliding Window Log (precis)
- Storage: Redis (INCR + EXPIRE pentru fixed window, sorted set pentru sliding window)
- Granularitate: per user, per IP, per API key, per endpoint
- Response: 429 Too Many Requests + Retry-After header
- Distributed: toate instante-le API citesc din acelasi Redis
- Tiers: free (100 req/min), pro (1000 req/min), enterprise (10000 req/min)
- Angular: interceptor care citeste X-RateLimit-Remaining si afiseaza warning

### 6. "Designeaza un dashboard de analytics real-time"

**Abordare**:
- Ingestion: events via Kafka/Kinesis (high throughput, partitioned)
- Processing: stream processing (Flink, Spark Streaming) pentru agregate real-time
- Storage: time-series DB (InfluxDB, TimescaleDB) sau pre-aggregated in Redis
- Serving: API care serveste agregate (nu raw data)
- Real-time updates: SSE sau WebSocket pentru push de date noi la dashboard
- Angular: change detection OnPush, signals pentru reactivity, chart library (D3, Chart.js), Web Workers pentru calcule grele, virtual scrolling pentru tabele mari

### 7. "Designeaza un sistem de search autocomplete"

**Abordare**:
- Trie (prefix tree) in memorie pentru sugestii rapide
- Elasticsearch cu completion suggester pentru productie
- Frecventa-based ranking (cele mai populare sugestii primele)
- Debounce pe client (300ms) - nu trimite request la fiecare keystroke
- Cache agresiv: top queries in Redis, browser cache pe sugestii
- Personalizare: search history per user (recent searches)
- Angular: debounceTime(300), distinctUntilChanged(), switchMap() (canceleaza request-ul precedent), async pipe

```typescript
// Exemplu classic Angular autocomplete
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  filter(term => term.length >= 2),
  switchMap(term => this.searchService.suggest(term)),
  takeUntilDestroyed()
).subscribe(suggestions => this.suggestions.set(suggestions));
```

### 8. "Designeaza un sistem de e-commerce checkout"

**Abordare**:
- Cart service: stocare in Redis (guest) sau DB (authenticated)
- Inventory: pessimistic locking (rezerva stoc la add-to-cart) vs optimistic (verifica la checkout)
- Payment: integrare Stripe/PayPal, idempotency key (previne double charge)
- Order saga: create order -> reserve inventory -> process payment -> confirm order. Daca payment esueaza -> compensating transaction (release inventory)
- Concurrency: ce se intampla daca 2 persoane cumpara ultimul produs simultan? Optimistic locking cu version number
- Angular: multi-step form cu state management, payment form in iframe (PCI compliance), retry logic pentru network errors

### 9. "Cum ai migra o aplicatie Angular de la monolith la micro-frontends?"

**Abordare**:
- Evaluare: e chiar necesar? (complexitate organizationala vs beneficii)
- Module Federation (Webpack 5 / Native Federation): share dependencies, lazy load remote modules
- Single-SPA: framework agnostic, poate mixa Angular + React
- Web Components: encapsulare nativa, framework agnostic
- Shared state: custom events, shared services via dependency injection
- Routing: shell app gestioneaza routing top-level, fiecare micro-frontend gestioneaza sub-routes
- Deployment: fiecare MFE se deployaza independent, versioning prin manifest
- Incremental migration: incepe cu un modul nou ca MFE, apoi migreaza treptat

### 10. "Designeaza o platforma de video streaming"

**Abordare**:
- Upload: chunked upload cu resumability, pre-signed URLs
- Processing: transcoding pipeline (multiple rezolutii: 360p, 720p, 1080p, 4K), adaptive bitrate (HLS/DASH)
- Storage: Object storage (S3) pentru video files, CDN pentru delivery
- Streaming: HLS (HTTP Live Streaming) - video impartit in segmente de 2-10s, manifest file cu toate rezolutiile
- CDN: edge caching pentru segmente populare, origin shield
- Metadata: PostgreSQL (video info, users, comments), Elasticsearch (search)
- Recommendations: separate service cu ML model, pre-computed recommendations in Redis
- Angular: video.js sau HLS.js player, lazy loading comments, infinite scroll pentru feed, picture-in-picture API

---

## Sfaturi finale pentru interviul de System Design

1. **Comunica constant** - gandeste cu voce tare, nu sta in liniste 5 minute
2. **Conduce conversatia** - nu astepta sa fii ghidat la fiecare pas
3. **Cere feedback** - "Are sens abordarea? Vrei sa fac deep dive undeva?"
4. **Prioritizeaza** - nu poti acoperi tot in 45 min; concentreaza-te pe componentele critice
5. **Arata trade-offs** - nu exista solutie perfecta; explica DE CE ai ales o varianta
6. **Gandeste la failure modes** - "Ce se intampla daca X cade?"
7. **Foloseste numere** - estimarile de trafic/stocare arata gandire inginereasca
8. **Adapteaza la scale** - solutia pentru 1000 useri difera de cea pentru 100M useri
9. **Ca Principal Engineer** - arata ca intelegi nu doar Angular, ci intreg ecosistemul
10. **Practica** - deseneaza pe hartie/whiteboard, cronometreaza-te, repeta cu un coleg
