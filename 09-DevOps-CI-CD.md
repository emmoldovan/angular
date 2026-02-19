# DevOps & CI/CD pentru Angular

## Cuprins

1. [Pipeline CI/CD pentru Angular](#1-pipeline-cicd-pentru-angular)
2. [Docker pentru aplicatii Angular](#2-docker-pentru-aplicatii-angular)
3. [Deployment strategies](#3-deployment-strategies)
4. [Nginx configurare pentru SPA](#4-nginx-configurare-pentru-spa)
5. [Environment management](#5-environment-management)
6. [Monitoring si Observability](#6-monitoring-si-observability)
7. [Error tracking cu Sentry](#7-error-tracking-cu-sentry)
8. [Feature flags](#8-feature-flags)
9. [Semantic versioning si release management](#9-semantic-versioning-si-release-management)
10. [Intrebari frecvente de interviu](#intrebari-frecvente-de-interviu)

---

## 1. Pipeline CI/CD pentru Angular

### Concepte fundamentale

Un pipeline CI/CD (Continuous Integration / Continuous Delivery) automatizeaza intreg ciclul de viata al unei aplicatii Angular: de la validarea codului (lint, teste) pana la build si deployment in productie. La nivel de Principal Engineer, trebuie sa stii sa proiectezi pipeline-uri eficiente, rapide si fiabile.

**Etapele clasice ale unui pipeline Angular:**

```
lint --> test --> build --> deploy
```

- **Lint** - Verifica stilul si calitatea codului (ESLint, Prettier)
- **Test** - Ruleaza teste unitare (Karma/Jest) si eventual E2E (Cypress/Playwright)
- **Build** - Compilare AOT, tree-shaking, bundling cu optimizari de productie
- **Deploy** - Publicarea artefactelor pe serverul tinta (CDN, Nginx, cloud)

---

### GitHub Actions - Workflow complet

```yaml
# .github/workflows/ci-cd.yml
name: Angular CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# Un singur workflow per branch la un moment dat
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'
  ANGULAR_CLI_VERSION: '17'

jobs:
  # ============================================
  # ETAPA 1: Lint - Verificare calitate cod
  # ============================================
  lint:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'  # Cache automat pe baza package-lock.json

      - name: Install dependencies
        run: npm ci  # ci e mai rapid si mai determinist decat npm install

      - name: Run ESLint
        run: npx ng lint --format=json --output-file=lint-results.json
        continue-on-error: false

      - name: Check Prettier formatting
        run: npx prettier --check "src/**/*.{ts,html,scss}"

      - name: Upload lint results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: lint-results
          path: lint-results.json

  # ============================================
  # ETAPA 2: Test - Teste unitare si coverage
  # ============================================
  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint  # Ruleaza doar daca lint-ul trece
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Headless Chrome pentru teste in CI
      - name: Run unit tests with coverage
        run: |
          npx ng test \
            --watch=false \
            --browsers=ChromeHeadless \
            --code-coverage \
            --source-map=true

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

      # Optional: Coverage gate - esueaza daca coverage scade sub prag
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | \
            jq '.total.statements.pct')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage sub 80%! Pipeline esuat."
            exit 1
          fi

  # ============================================
  # ETAPA 3: Build - Compilare productie
  # ============================================
  build:
    name: Production Build
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build for production
        run: npx ng build --configuration=production
        env:
          # Variabile de mediu disponibile la build time
          NG_APP_API_URL: ${{ vars.API_URL }}
          NG_APP_VERSION: ${{ github.sha }}

      # Analiza dimensiune bundle
      - name: Analyze bundle size
        run: |
          echo "## Bundle Size Report" >> $GITHUB_STEP_SUMMARY
          du -sh dist/my-app/browser/* | while read size file; do
            echo "- $file: $size" >> $GITHUB_STEP_SUMMARY
          done

      # Salveaza artefactele pentru deploy
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: dist/
          retention-days: 7

  # ============================================
  # ETAPA 4: Deploy - Publicare in productie
  # ============================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.app.example.com
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: dist/

      - name: Deploy to staging server
        run: |
          # Exemplu cu AWS S3 + CloudFront
          aws s3 sync dist/my-app/browser/ s3://${{ secrets.S3_STAGING_BUCKET }} \
            --delete
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_STAGING_DIST_ID }} \
            --paths "/*"

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.example.com
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-${{ github.sha }}
          path: dist/

      - name: Deploy to production
        run: |
          aws s3 sync dist/my-app/browser/ s3://${{ secrets.S3_PROD_BUCKET }} \
            --delete
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CF_PROD_DIST_ID }} \
            --paths "/*"

      - name: Notify deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployed ${{ github.sha }} to production"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

### GitLab CI - Pipeline complet

```yaml
# .gitlab-ci.yml
image: node:20-alpine

stages:
  - install
  - lint
  - test
  - build
  - deploy

# Cache global - partajat intre toate job-urile
cache:
  key:
    files:
      - package-lock.json  # Cache invalidat cand se schimba dependintele
  paths:
    - node_modules/
    - .npm/
  policy: pull  # Default: doar citeste din cache

# ============================================
# Instalare dependinte (o singura data)
# ============================================
install:
  stage: install
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
      - .npm/
    policy: pull-push  # Acest job actualizeaza cache-ul
  script:
    - npm ci --cache .npm --prefer-offline
  rules:
    - when: always

# ============================================
# Lint
# ============================================
lint:
  stage: lint
  script:
    - npx ng lint
    - npx prettier --check "src/**/*.{ts,html,scss}"
  rules:
    - when: on_success

# ============================================
# Teste unitare
# ============================================
test:unit:
  stage: test
  image: node:20  # Imagine completa, nu alpine (Chrome are nevoie de librarii)
  before_script:
    # Instaleaza Chrome headless pentru Karma
    - apt-get update && apt-get install -y chromium
    - export CHROME_BIN=/usr/bin/chromium
  script:
    - npx ng test --watch=false --browsers=ChromeHeadless --code-coverage
  coverage: '/Statements\s*:\s*(\d+\.?\d*)%/'  # Regex pentru extractia coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 7 days

# Teste E2E (paralele cu cele unitare)
test:e2e:
  stage: test
  image: cypress/browsers:latest
  script:
    - npx ng e2e --configuration=ci
  artifacts:
    paths:
      - cypress/screenshots/
      - cypress/videos/
    when: on_failure
    expire_in: 3 days

# ============================================
# Build
# ============================================
build:staging:
  stage: build
  script:
    - npx ng build --configuration=staging
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

build:production:
  stage: build
  script:
    - npx ng build --configuration=production
  artifacts:
    paths:
      - dist/
    expire_in: 30 days
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ============================================
# Deploy
# ============================================
deploy:staging:
  stage: deploy
  needs: [build:staging]
  environment:
    name: staging
    url: https://staging.app.example.com
  script:
    - echo "Deploying to staging..."
    # Docker build + push catre registry
    - docker build -t $CI_REGISTRY_IMAGE:staging .
    - docker push $CI_REGISTRY_IMAGE:staging
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy:production:
  stage: deploy
  needs: [build:production]
  environment:
    name: production
    url: https://app.example.com
  script:
    - echo "Deploying to production..."
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG .
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
    - docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: $CI_COMMIT_TAG  # Doar pe tag-uri (release-uri)
  when: manual  # Necesita aprobare manuala
```

---

### Optimizari CI/CD critice

**Caching eficient:**

```yaml
# GitHub Actions - Cache strategy avansata
- name: Cache node_modules
  uses: actions/cache@v4
  id: npm-cache
  with:
    path: |
      node_modules
      ~/.npm
      ~/.cache/Cypress  # Cache pentru Cypress binaries
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Install dependencies
  if: steps.npm-cache.outputs.cache-hit != 'true'
  run: npm ci
```

**Teste headless Chrome - Configurare Karma:**

```typescript
// karma.conf.js
module.exports = function (config) {
  config.set({
    browsers: ['ChromeHeadless'],  // Fara interfata grafica
    customLaunchers: {
      ChromeHeadlessCI: {
        base: 'ChromeHeadless',
        flags: [
          '--no-sandbox',           // Necesar in containere Docker
          '--disable-gpu',          // Stabilitate in CI
          '--disable-translate',
          '--disable-extensions',
          '--remote-debugging-port=9222'
        ]
      }
    },
    singleRun: true,  // Ruleaza o data si iese
    restartOnFileChange: false,
  });
};
```

**Versioning automata a artefactelor:**

```yaml
# Tagare automata cu SHA-ul commit-ului + timestamp
- name: Tag build artifact
  run: |
    VERSION=$(node -p "require('./package.json').version")
    BUILD_ID="${VERSION}-${GITHUB_SHA::8}-$(date +%Y%m%d%H%M%S)"
    echo "BUILD_ID=$BUILD_ID" >> $GITHUB_ENV
    echo "Build: $BUILD_ID"
```

> **Perspectiva Principal Engineer:** Un pipeline bun are sub 10 minute pentru feedback pe PR-uri. Daca depaseste, trebuie optimizat: paralelizare, caching agresiv, teste relevante pe affected paths. Pipeline-urile lente ucid productivitatea echipei.

---

## 2. Docker pentru aplicatii Angular

### Multi-stage Dockerfile

Cea mai importanta tehnica Docker pentru Angular este **multi-stage build**: o etapa de build (Node.js) si o etapa de servire (Nginx). Rezultatul este o imagine mica, fara Node.js, fara node_modules, doar fisierele statice compilate.

```dockerfile
# ============================================
# STAGE 1: Build - Compilare Angular
# ============================================
FROM node:20-alpine AS build

# Setam directorul de lucru
WORKDIR /app

# Copiem DOAR fisierele de dependinte mai intai
# Aceasta permite Docker layer caching: daca package*.json nu se schimba,
# npm ci nu se ruleaza din nou (cache hit)
COPY package.json package-lock.json ./

# npm ci este determinist si mai rapid in CI decat npm install
RUN npm ci --no-audit --no-fund

# Acum copiem restul codului sursa
COPY . .

# Build de productie Angular
# --output-path seteaza locatia artefactelor
RUN npx ng build --configuration=production

# ============================================
# STAGE 2: Serve - Nginx pentru productie
# ============================================
FROM nginx:1.25-alpine AS production

# Stergem configuratia default Nginx
RUN rm /etc/nginx/conf.d/default.conf

# Copiem configuratia noastra custom
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Copiem artefactele din etapa de build
# ATENTIE: path-ul depinde de versiunea Angular
# Angular 17+: dist/<project-name>/browser/
# Angular <17: dist/<project-name>/
COPY --from=build /app/dist/my-app/browser/ /usr/share/nginx/html/

# Script pentru injectarea variabilelor de mediu la runtime
COPY docker/env-config.sh /docker-entrypoint.d/40-env-config.sh
RUN chmod +x /docker-entrypoint.d/40-env-config.sh

# Expunem portul 80
EXPOSE 80

# Nginx ruleaza in foreground (necesar pentru Docker)
CMD ["nginx", "-g", "daemon off;"]
```

---

### .dockerignore - Best practices

```dockerfile
# .dockerignore
# Echivalentul .gitignore pentru Docker context

# Dependinte - se instaleaza in container
node_modules
.npm

# Output build local
dist
.angular

# Git
.git
.gitignore

# IDE
.vscode
.idea
*.swp

# Teste
coverage
e2e
cypress
*.spec.ts

# Docker files (evita recursivitate)
Dockerfile
docker-compose*.yml
.dockerignore

# Documentatie
README.md
CHANGELOG.md
docs/

# CI/CD configs
.github
.gitlab-ci.yml

# Environment files cu secrete
.env
.env.local
*.env
```

> **De ce conteaza .dockerignore:** Fara el, `COPY . .` trimite TOT la Docker daemon - inclusiv node_modules (sute de MB), .git, si fisiere sensibile. Cu un .dockerignore bun, build context-ul scade dramatic, build-ul e mai rapid si imaginea e mai sigura.

---

### Docker Compose pentru dezvoltare locala

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============================================
  # Angular App (development cu hot reload)
  # ============================================
  angular-app:
    build:
      context: .
      dockerfile: Dockerfile.dev  # Dockerfile separat pentru dev
    ports:
      - "4200:4200"
    volumes:
      # Bind mount pentru hot reload - codul local e sincronizat in container
      - .:/app
      # Named volume pentru node_modules (evita conflicte OS)
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=development
      - API_URL=http://api:3000
    depends_on:
      api:
        condition: service_healthy
    networks:
      - app-network

  # ============================================
  # Backend API
  # ============================================
  api:
    build:
      context: ../api
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - JWT_SECRET=dev-secret-key
      - CORS_ORIGIN=http://localhost:4200
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - app-network

  # ============================================
  # Baza de date PostgreSQL
  # ============================================
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - app-network

volumes:
  node_modules:  # Volume persistent pentru dependinte
  pgdata:        # Volume persistent pentru date PostgreSQL

networks:
  app-network:
    driver: bridge
```

**Dockerfile.dev (pentru dezvoltare cu hot reload):**

```dockerfile
# Dockerfile.dev
FROM node:20-alpine

WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# NU copiem codul sursa - vine prin bind mount din docker-compose
EXPOSE 4200

# ng serve cu host 0.0.0.0 (altfel nu e accesibil din afara containerului)
CMD ["npx", "ng", "serve", "--host", "0.0.0.0", "--poll", "2000"]
```

---

### Optimizare imagine Docker

```dockerfile
# Imagine NEOPTIMIZATA - 1.2 GB
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npx ng build --configuration=production
# Problema: Node.js + node_modules + cod sursa raman in imagine finala

# Imagine OPTIMIZATA - 25 MB
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --no-audit
COPY . .
RUN npx ng build --configuration=production

FROM nginx:1.25-alpine
COPY --from=build /app/dist/my-app/browser/ /usr/share/nginx/html/
# Doar Nginx + fisiere statice
```

**Comparatie dimensiuni:**

| Abordare | Dimensiune imagine |
|---|---|
| node:20 + tot codul | ~1.2 GB |
| Multi-stage cu node:20 | ~180 MB |
| Multi-stage cu node:20-alpine + nginx:alpine | **~25 MB** |

---

### Variabile de mediu: Build time vs Runtime

Aceasta este o distinctie critica pentru Angular in Docker:

**Build time** - Variabilele sunt "coapte" (baked) in bundle la compilare. Nu pot fi schimbate dupa build.

```typescript
// environment.prod.ts - valori fixate la build
export const environment = {
  production: true,
  apiUrl: 'https://api.prod.example.com',  // FIXAT in bundle
};
```

**Runtime** - Variabilele sunt injectate cand containerul porneste. Aceeasi imagine Docker functioneaza in orice mediu.

```bash
#!/bin/sh
# docker/env-config.sh - Script executat la pornirea containerului

# Genereaza un fisier JS cu configuratia din environment variables
cat <<EOF > /usr/share/nginx/html/assets/env-config.js
(function(window) {
  window.__env = window.__env || {};
  window.__env.apiUrl = '${API_URL:-http://localhost:3000}';
  window.__env.sentryDsn = '${SENTRY_DSN:-}';
  window.__env.featureFlags = '${FEATURE_FLAGS:-}';
  window.__env.appVersion = '${APP_VERSION:-unknown}';
})(this);
EOF
```

```html
<!-- index.html - Incarcat INAINTE de bundle-ul Angular -->
<script src="assets/env-config.js"></script>
```

```typescript
// env-config.service.ts
@Injectable({ providedIn: 'root' })
export class EnvConfigService {
  private config: Record<string, string>;

  constructor() {
    this.config = (window as any).__env || {};
  }

  get apiUrl(): string {
    return this.config['apiUrl'] || 'http://localhost:3000';
  }

  get sentryDsn(): string {
    return this.config['sentryDsn'] || '';
  }
}
```

> **Perspectiva Principal Engineer:** Prefer intotdeauna configuratie runtime pentru URL-uri si feature flags. O singura imagine Docker care functioneaza pe staging, QA si productie elimina o intreaga categorie de buguri de tip "merge in staging dar nu in productie". Build time configuration e acceptabila doar pentru optimizari care necesita tree-shaking (ex: `isDevMode()`).

---

## 3. Deployment strategies

### Blue-Green Deployment

Doua medii identice (Blue si Green). Unul e live, celalalt primeste noua versiune. Traficul se comuta instantaneu.

```
            Load Balancer
                 |
        +--------+--------+
        |                 |
   [Blue - v1.0]    [Green - v1.1]
    (LIVE)           (STANDBY)

    Dupa validare:

            Load Balancer
                 |
        +--------+--------+
        |                 |
   [Blue - v1.0]    [Green - v1.1]
    (STANDBY)        (LIVE)       <-- Switch instant
```

```yaml
# Kubernetes - Blue-Green cu servicii
# service.yml - Selector-ul se schimba intre blue si green
apiVersion: v1
kind: Service
metadata:
  name: angular-app
spec:
  selector:
    app: angular-app
    version: green  # Schimba la 'blue' pentru rollback instant
  ports:
    - port: 80
      targetPort: 80
```

**Avantaje:**
- Rollback instant (comuta inapoi la versiunea veche)
- Zero downtime
- Testare completa pe mediul nou inainte de switch

**Dezavantaje:**
- Necesita dublu resurse (doua medii complete)
- Migratiile de baza de date pot fi complexe
- Cost ridicat

---

### Canary Deployment

Noua versiune primeste un procent mic de trafic (ex: 5%). Daca metricile sunt bune, procentul creste treptat pana la 100%.

```
            Load Balancer
           /             \
         95%              5%
          |                |
   [v1.0 - 10 pods]  [v1.1 - 1 pod]  <-- Canary

   Dupa validare (etape):
   5% --> 25% --> 50% --> 75% --> 100%
```

```yaml
# Kubernetes - Canary cu Istio VirtualService
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: angular-app
spec:
  http:
    - route:
        - destination:
            host: angular-app
            subset: stable
          weight: 95
        - destination:
            host: angular-app
            subset: canary
          weight: 5
---
# DestinationRule - defineste subseturile
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: angular-app
spec:
  host: angular-app
  subsets:
    - name: stable
      labels:
        version: v1.0
    - name: canary
      labels:
        version: v1.1
```

**Metrici de urmarit in canary:**
- Error rate (rata de erori nu trebuie sa creasca)
- Latenta (P50, P95, P99)
- Apdex score (satisfactia utilizatorilor)
- Business metrics (conversion rate, bounce rate)

**Rollback automat:**

```yaml
# Argo Rollouts - Canary cu analiza automata
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: angular-app
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }     # Asteapta 5 minute
        - analysis:                     # Analiza automata
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: angular-app
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      # Daca analiza esueaza, rollback automat
      rollbackWindow:
        revisions: 2
```

---

### Rolling Deployment

Instante vechi sunt inlocuite progresiv cu cele noi. La orice moment, ambele versiuni ruleaza simultan.

```
Starea initiala:  [v1] [v1] [v1] [v1]
Pasul 1:          [v2] [v1] [v1] [v1]  - Un pod la un moment dat
Pasul 2:          [v2] [v2] [v1] [v1]
Pasul 3:          [v2] [v2] [v2] [v1]
Pasul 4:          [v2] [v2] [v2] [v2]  - Complet
```

```yaml
# Kubernetes - Rolling Update (default strategy)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: angular-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Maxim 1 pod in plus peste replicas
      maxUnavailable: 0   # Niciodata sub numarul de replicas
  template:
    spec:
      containers:
        - name: angular-app
          image: angular-app:v1.1
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
```

---

### A/B Testing Deployment

Diferit de canary - nu e despre versiune noua vs veche, ci despre doua variante de feature testate pe segmente diferite de utilizatori.

```typescript
// Routare bazata pe header-e sau cookies
// Nginx upstream cu split pe baza de cookie
// nginx.conf

// upstream A (control group)
// upstream B (experiment group)

// split_clients bazat pe cookie/header
```

```nginx
# nginx.conf - A/B split
split_clients "${remote_addr}" $variant {
    50%    server_a;
    50%    server_b;
}

upstream server_a {
    server app-variant-a:80;
}

upstream server_b {
    server app-variant-b:80;
}

server {
    location / {
        proxy_pass http://$variant;
    }
}
```

---

### Cand sa folosesti fiecare strategie

| Strategie | Cazuri de utilizare | Complexitate | Cost |
|---|---|---|---|
| **Blue-Green** | Aplicatii critice, rollback instant necesar, schimbari majore | Medie | Ridicat (x2 resurse) |
| **Canary** | Productie cu volum mare, risc ridicat, metrici bune | Ridicata | Mediu |
| **Rolling** | Aplicatii standard, update-uri frecvente, zero downtime | Scazuta | Scazut |
| **A/B Testing** | Experimentare pe produs, optimizare conversii | Ridicata | Mediu |

> **Perspectiva Principal Engineer:** In practica, majoritatea aplicatiilor Angular (fiind SPA-uri servite ca fisiere statice) folosesc Blue-Green cu CDN. Publici noua versiune pe CDN, invalidezi cache-ul, done. Canary e mai relevant cand Angular face SSR (server-side rendering) cu un server Node.js. Alegerea strategiei depinde de arhitectura, nu de framework.

---

## 4. Nginx configurare pentru SPA

### Configuratie completa pentru Angular

O aplicatie Angular SPA necesita o configuratie Nginx speciala: toate rutele trebuie redirectionate la `index.html` (pentru ca rutarea se face pe client, nu pe server).

```nginx
# /etc/nginx/nginx.conf - Configuratie principala
user nginx;
worker_processes auto;  # Autodetecteaza numarul de CPU cores
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # ============================================
    # Logging format
    # ============================================
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time';

    access_log /var/log/nginx/access.log main;

    # ============================================
    # Performanta
    # ============================================
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 10m;

    # ============================================
    # Compresie Gzip
    # ============================================
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;  # Balans intre compresie si CPU
    gzip_min_length 1024;  # Nu comprima fisiere sub 1KB
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml
        font/woff2;

    # ============================================
    # Compresie Brotli (mai eficienta decat gzip)
    # Necesita modulul ngx_brotli
    # ============================================
    # brotli on;
    # brotli_comp_level 6;
    # brotli_types text/plain text/css application/javascript
    #              application/json image/svg+xml;

    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# /etc/nginx/conf.d/default.conf - Configuratie server Angular
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # ============================================
    # CRITICAL: HTML5 Routing Support
    # Fara asta, refresh pe /dashboard va da 404
    # ============================================
    location / {
        try_files $uri $uri/ /index.html;

        # index.html NU trebuie cached (contine referinte la bundle-uri cu hash)
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }

    # ============================================
    # Cache agresiv pentru fisiere cu hash
    # Angular genereaza: main.abc123.js, styles.def456.css
    # Hash-ul se schimba la fiecare build => cache pe termen lung e sigur
    # ============================================
    location ~* \.(js|css)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Cache pentru fonturi si imagini
    location ~* \.(ico|gif|jpe?g|png|webp|avif|svg|woff2?|ttf|eot)$ {
        expires 6M;
        add_header Cache-Control "public";
        access_log off;
    }

    # ============================================
    # Security Headers
    # ============================================
    # Previne clickjacking - pagina nu poate fi inclusa in iframe
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Previne MIME type sniffing
    add_header X-Content-Type-Options "nosniff" always;

    # XSS Protection (browser legacy)
    add_header X-XSS-Protection "1; mode=block" always;

    # Referrer Policy
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Permissions Policy (inlocuieste Feature-Policy)
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;

    # Content Security Policy - ADAPTEAZA LA NEVOILE APLICATIEI
    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' 'unsafe-inline' 'unsafe-eval';
        style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
        font-src 'self' https://fonts.gstatic.com;
        img-src 'self' data: https:;
        connect-src 'self' https://api.example.com https://*.sentry.io;
        frame-ancestors 'self';
        base-uri 'self';
        form-action 'self';
    " always;

    # HSTS - Forteaza HTTPS (doar daca ai HTTPS configurat)
    # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # ============================================
    # Reverse Proxy catre API Backend
    # ============================================
    location /api/ {
        proxy_pass http://backend-api:3000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (daca e necesar)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeout-uri
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # ============================================
    # Health check endpoint
    # ============================================
    location /health {
        access_log off;
        return 200 "OK";
        add_header Content-Type text/plain;
    }

    # Ascunde fisierele sensibile
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Custom error pages
    error_page 404 /index.html;  # SPA fallback
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

---

### Strategia de caching explicata

```
Cerere browser:                          Nginx:

  GET /index.html          --->    Cache-Control: no-cache
  (la fiecare navigare)            (intotdeauna fisierul proaspat)

  GET /main.a1b2c3.js      --->   Cache-Control: immutable, max-age=1y
  (hash in numele fisierului)      (servit din cache browser 1 an)

  GET /styles.d4e5f6.css   --->   Cache-Control: immutable, max-age=1y
  (hash in numele fisierului)      (servit din cache browser 1 an)
```

**De ce aceasta strategie functioneaza:**
- `index.html` contine referinte la `main.HASH.js` si `styles.HASH.css`
- Cand faci un build nou, hash-urile se schimba (ex: `main.x7y8z9.js`)
- Browser-ul cere `index.html` proaspat (no-cache), gaseste noul hash, descarca noile fisiere
- Fisierele vechi raman in cache dar nu mai sunt referentiate - vor fi eventual evicted

> **Perspectiva Principal Engineer:** Aceasta strategie de cache (index.html fara cache, assets cu immutable cache) este standardul de facto pentru orice SPA. E simpla, eficienta si elimina problema "utilizatorul vede versiunea veche". Orice alta abordare e o complicatie inutila.

---

## 5. Environment management

### Angular environments (build time)

Angular ofera un mecanism nativ de environment configuration prin `fileReplacements` in `angular.json`.

```typescript
// src/environments/environment.ts (development - default)
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api',
  sentryDsn: '',
  enableDebugTools: true,
  logLevel: 'debug',
};
```

```typescript
// src/environments/environment.prod.ts (productie)
export const environment = {
  production: true,
  apiUrl: 'https://api.example.com/api',
  sentryDsn: 'https://abc@sentry.io/123',
  enableDebugTools: false,
  logLevel: 'error',
};
```

```typescript
// src/environments/environment.staging.ts (staging)
export const environment = {
  production: true,  // Staging se comporta ca productie
  apiUrl: 'https://api-staging.example.com/api',
  sentryDsn: 'https://abc@sentry.io/456',
  enableDebugTools: false,
  logLevel: 'warn',
};
```

**Configurare in angular.json:**

```json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.prod.ts"
                }
              ],
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kb",
                  "maximumError": "1mb"
                }
              ],
              "outputHashing": "all"
            },
            "staging": {
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.staging.ts"
                }
              ],
              "outputHashing": "all"
            }
          }
        }
      }
    }
  }
}
```

```bash
# Utilizare
ng build --configuration=production   # Foloseste environment.prod.ts
ng build --configuration=staging      # Foloseste environment.staging.ts
ng build                              # Foloseste environment.ts (dev)
```

---

### Runtime configuration cu APP_INITIALIZER

Pentru configuratie care trebuie sa fie flexibila (nu baked la build time), folosim `APP_INITIALIZER` pentru a incarca un fisier JSON la pornirea aplicatiei.

```typescript
// app-config.model.ts
export interface AppConfig {
  apiUrl: string;
  authDomain: string;
  sentryDsn: string;
  featureFlags: Record<string, boolean>;
  appVersion: string;
}
```

```typescript
// app-config.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { firstValueFrom } from 'rxjs';
import { AppConfig } from './app-config.model';

@Injectable({ providedIn: 'root' })
export class AppConfigService {
  private config!: AppConfig;

  constructor(private http: HttpClient) {}

  /** Incarca configuratia la startup - apelat de APP_INITIALIZER */
  async loadConfig(): Promise<void> {
    try {
      // Fisierul config.json este servit din /assets/
      // Poate fi generat dinamic de Docker/Kubernetes la startup
      const config = await firstValueFrom(
        this.http.get<AppConfig>('/assets/config.json')
      );
      this.config = config;
      console.log('App config loaded successfully');
    } catch (error) {
      console.error('Failed to load app config:', error);
      // Fallback la valori default
      this.config = {
        apiUrl: '/api',
        authDomain: '',
        sentryDsn: '',
        featureFlags: {},
        appVersion: 'unknown',
      };
    }
  }

  get apiUrl(): string {
    return this.config.apiUrl;
  }

  get authDomain(): string {
    return this.config.authDomain;
  }

  getFeatureFlag(flag: string): boolean {
    return this.config.featureFlags?.[flag] ?? false;
  }

  getConfig(): Readonly<AppConfig> {
    return Object.freeze({ ...this.config });
  }
}
```

```typescript
// app.config.ts (Angular 17+ standalone)
import {
  ApplicationConfig,
  APP_INITIALIZER,
} from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { AppConfigService } from './app-config.service';

/** Factory function pentru APP_INITIALIZER */
function initializeApp(configService: AppConfigService): () => Promise<void> {
  return () => configService.loadConfig();
}

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    {
      provide: APP_INITIALIZER,
      useFactory: initializeApp,
      deps: [AppConfigService],
      multi: true,  // Permite mai multi APP_INITIALIZER
    },
  ],
};
```

```json
// src/assets/config.json (pentru development local)
{
  "apiUrl": "http://localhost:3000/api",
  "authDomain": "dev.auth.example.com",
  "sentryDsn": "",
  "featureFlags": {
    "newDashboard": true,
    "darkMode": false
  },
  "appVersion": "local-dev"
}
```

---

### Environment variables via Docker / Kubernetes

```bash
#!/bin/sh
# docker/generate-config.sh
# Genereaza config.json din variabile de mediu la pornirea containerului

CONFIG_PATH="/usr/share/nginx/html/assets/config.json"

cat > $CONFIG_PATH <<EOF
{
  "apiUrl": "${API_URL:-http://localhost:3000/api}",
  "authDomain": "${AUTH_DOMAIN:-}",
  "sentryDsn": "${SENTRY_DSN:-}",
  "featureFlags": ${FEATURE_FLAGS:-{}},
  "appVersion": "${APP_VERSION:-unknown}"
}
EOF

echo "Generated config.json:"
cat $CONFIG_PATH
```

```yaml
# Kubernetes ConfigMap + Deployment
apiVersion: v1
kind: ConfigMap
metadata:
  name: angular-config
data:
  API_URL: "https://api.prod.example.com"
  AUTH_DOMAIN: "auth.example.com"
  FEATURE_FLAGS: '{"newDashboard": true, "darkMode": false}'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: angular-app
spec:
  template:
    spec:
      containers:
        - name: angular-app
          image: angular-app:1.2.3
          envFrom:
            - configMapRef:
                name: angular-config
          env:
            # Secretele vin din Kubernetes Secrets, nu ConfigMap
            - name: SENTRY_DSN
              valueFrom:
                secretKeyRef:
                  name: angular-secrets
                  key: sentry-dsn
```

---

### Secrets management

```yaml
# Kubernetes Secrets (codificate base64, NU criptate!)
apiVersion: v1
kind: Secret
metadata:
  name: angular-secrets
type: Opaque
data:
  sentry-dsn: aHR0cHM6Ly9hYmNAc2VudHJ5LmlvLzEyMw==

# ATENTIE: Kubernetes Secrets sunt doar base64, nu criptate.
# Pentru secretele reale, foloseste:
# - HashiCorp Vault
# - AWS Secrets Manager
# - Azure Key Vault
# - Sealed Secrets (Bitnami)
# - External Secrets Operator
```

**Reguli de aur pentru secrets in frontend:**

```
1. NICIODATA nu pune secrete reale in codul frontend
   - API keys publice (Stripe publishable key) = OK
   - API keys private (Stripe secret key) = NICIODATA

2. Orice in bundle-ul JavaScript este PUBLIC
   - Oricine poate face View Source
   - Environment variables "baked" in build sunt vizibile

3. Secretele sensibile apartin backend-ului
   - Frontend-ul cere backend-ului sa faca operatia secreta
   - Backend-ul are access la secretele reale

4. .env files nu se commiteaza NICIODATA in git
   - .env.example cu valori placeholder = OK
   - .env cu valori reale = .gitignore
```

> **Perspectiva Principal Engineer:** Am vazut echipe care puneau API keys secrete in environment.prod.ts. Aceasta e o greseala grava de securitate. Tot ce ajunge in bundle-ul JavaScript e public. Foloseste APP_INITIALIZER pentru configuratie non-secreta si lasa secretele pe backend. Singurul "secret" acceptabil in frontend e un DSN public (cum ar fi Sentry), care e proiectat sa fie public.

---

## 6. Monitoring si Observability

### Cele trei piloni ai observability

```
                    OBSERVABILITY
                    /     |      \
               LOGS    METRICS   TRACES

  Ce s-a         Cat de        Cum s-a
  intamplat?     bine merge?   propagat?

  ELK Stack      Prometheus    Jaeger
  Loki           Grafana       Zipkin
  CloudWatch     Datadog       OpenTelemetry
```

**Logs** - Inregistrari textuale ale evenimentelor. Raspund la "Ce s-a intamplat?"
**Metrics** - Valori numerice agregate in timp. Raspund la "Cat de bine functioneaza?"
**Traces** - Parcursul unei cereri prin sisteme distribuite. Raspund la "Unde e bottleneck-ul?"

---

### Prometheus + Grafana pentru metrici

```typescript
// Metricile frontend se colecteaza diferit de cele backend.
// In Angular, colectam Web Vitals si le trimitem la un endpoint de metrici.

// performance-metrics.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface PerformanceMetric {
  name: string;
  value: number;
  timestamp: number;
  labels: Record<string, string>;
}

@Injectable({ providedIn: 'root' })
export class PerformanceMetricsService {
  private metricsBuffer: PerformanceMetric[] = [];
  private flushInterval = 30_000; // 30 secunde

  constructor(private http: HttpClient) {
    this.startCollection();
    this.startFlushing();
  }

  private startCollection(): void {
    // Core Web Vitals via PerformanceObserver
    if (typeof PerformanceObserver !== 'undefined') {
      // Largest Contentful Paint (LCP)
      const lcpObserver = new PerformanceObserver((list) => {
        const entries = list.getEntries();
        const lastEntry = entries[entries.length - 1];
        this.record('web_vital_lcp', lastEntry.startTime, {
          rating: lastEntry.startTime < 2500 ? 'good' : 'poor',
        });
      });
      lcpObserver.observe({ type: 'largest-contentful-paint', buffered: true });

      // First Input Delay (FID) / Interaction to Next Paint (INP)
      const fidObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          this.record('web_vital_fid', (entry as any).processingStart - entry.startTime, {
            rating: (entry as any).processingStart - entry.startTime < 100 ? 'good' : 'poor',
          });
        }
      });
      fidObserver.observe({ type: 'first-input', buffered: true });

      // Cumulative Layout Shift (CLS)
      let clsValue = 0;
      const clsObserver = new PerformanceObserver((list) => {
        for (const entry of list.getEntries()) {
          if (!(entry as any).hadRecentInput) {
            clsValue += (entry as any).value;
          }
        }
        this.record('web_vital_cls', clsValue, {
          rating: clsValue < 0.1 ? 'good' : 'poor',
        });
      });
      clsObserver.observe({ type: 'layout-shift', buffered: true });
    }

    // Navigation Timing
    window.addEventListener('load', () => {
      setTimeout(() => {
        const nav = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming;
        if (nav) {
          this.record('page_load_time', nav.loadEventEnd - nav.startTime);
          this.record('dom_interactive', nav.domInteractive - nav.startTime);
          this.record('ttfb', nav.responseStart - nav.requestStart);
        }
      }, 0);
    });
  }

  record(name: string, value: number, labels: Record<string, string> = {}): void {
    this.metricsBuffer.push({
      name,
      value,
      timestamp: Date.now(),
      labels: {
        ...labels,
        url: window.location.pathname,
        userAgent: navigator.userAgent.substring(0, 100),
      },
    });
  }

  /** Trimite metricile catre backend (care le expune la Prometheus) */
  private startFlushing(): void {
    setInterval(() => {
      if (this.metricsBuffer.length === 0) return;

      const metrics = [...this.metricsBuffer];
      this.metricsBuffer = [];

      // Foloseste sendBeacon pentru fiabilitate (functioneaza si la page unload)
      const blob = new Blob([JSON.stringify(metrics)], { type: 'application/json' });
      const sent = navigator.sendBeacon('/api/metrics', blob);

      if (!sent) {
        // Fallback la fetch daca sendBeacon esueaza
        this.http.post('/api/metrics', metrics).subscribe({
          error: () => {
            // Re-adauga metricile in buffer daca trimiterea esueaza
            this.metricsBuffer.push(...metrics);
          },
        });
      }
    }, this.flushInterval);
  }
}
```

**Dashboard Grafana - Metrici esentiale de monitorizat:**

```
Frontend Dashboard:
+----------------------------------------------+
| LCP (Largest Contentful Paint)               |
| P50: 1.2s | P75: 2.1s | P95: 3.8s          |
| Target: < 2.5s                               |
+----------------------------------------------+
| CLS (Cumulative Layout Shift)                |
| P50: 0.02 | P75: 0.08 | P95: 0.25          |
| Target: < 0.1                                |
+----------------------------------------------+
| JS Error Rate         | API Error Rate       |
| 0.3% of sessions      | 1.2% of requests     |
+----------------------------------------------+
| Bundle Size Trend     | Page Load Time P95   |
| main.js: 245KB (gzip) | 3.2s                 |
+----------------------------------------------+
```

---

### ELK Stack pentru logs

```typescript
// structured-logger.service.ts
// Loguri structurate (JSON) sunt esentiale pentru ELK Stack

import { Injectable, ErrorHandler } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../environments/environment';

export type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface LogEntry {
  timestamp: string;
  level: LogLevel;
  message: string;
  context?: string;
  userId?: string;
  sessionId: string;
  url: string;
  userAgent: string;
  data?: Record<string, unknown>;
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
}

@Injectable({ providedIn: 'root' })
export class StructuredLoggerService {
  private readonly sessionId = crypto.randomUUID();
  private logBuffer: LogEntry[] = [];
  private readonly LOG_LEVELS: Record<LogLevel, number> = {
    debug: 0,
    info: 1,
    warn: 2,
    error: 3,
  };

  constructor(private http: HttpClient) {
    // Flush periodic
    setInterval(() => this.flush(), 10_000);
    // Flush la page unload
    window.addEventListener('beforeunload', () => this.flush());
  }

  debug(message: string, data?: Record<string, unknown>): void {
    this.log('debug', message, data);
  }

  info(message: string, data?: Record<string, unknown>): void {
    this.log('info', message, data);
  }

  warn(message: string, data?: Record<string, unknown>): void {
    this.log('warn', message, data);
  }

  error(message: string, error?: Error, data?: Record<string, unknown>): void {
    this.log('error', message, {
      ...data,
      ...(error && {
        error: {
          name: error.name,
          message: error.message,
          stack: error.stack,
        },
      }),
    });
  }

  private log(level: LogLevel, message: string, data?: Record<string, unknown>): void {
    // Respecta nivelul minim de logging din configuratie
    const configLevel = environment.logLevel as LogLevel;
    if (this.LOG_LEVELS[level] < this.LOG_LEVELS[configLevel]) return;

    const entry: LogEntry = {
      timestamp: new Date().toISOString(),
      level,
      message,
      sessionId: this.sessionId,
      url: window.location.href,
      userAgent: navigator.userAgent,
      data,
    };

    // In development, afiseaza si in consola
    if (!environment.production) {
      const consoleFn = level === 'error' ? console.error :
                        level === 'warn' ? console.warn : console.log;
      consoleFn(`[${level.toUpperCase()}] ${message}`, data);
    }

    this.logBuffer.push(entry);

    // Flush imediat pe erori
    if (level === 'error') {
      this.flush();
    }
  }

  private flush(): void {
    if (this.logBuffer.length === 0) return;
    const logs = [...this.logBuffer];
    this.logBuffer = [];

    // Trimite la backend-ul care scrie in Elasticsearch / Logstash
    navigator.sendBeacon('/api/logs', JSON.stringify(logs));
  }
}
```

**Architectura ELK pentru frontend:**

```
Browser (Angular)
    |
    | POST /api/logs (JSON structurat)
    |
Backend API
    |
    | Forward la Logstash (sau direct Elasticsearch)
    |
Logstash (parsare, enrichment)
    |
    | Indexare
    |
Elasticsearch (storage + cautare)
    |
    | Query
    |
Kibana (vizualizare, dashboards, alerte)
```

---

### Real User Monitoring (RUM)

RUM colecteaza date de performanta de la utilizatorii reali (nu din teste sintetice). Este esential pentru a intelege experienta reala.

```typescript
// rum.service.ts
import { Injectable } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class RumService {
  private navigationStart = 0;

  constructor(private router: Router) {
    this.trackRouteChanges();
    this.trackResourceLoading();
    this.trackLongTasks();
  }

  /** Masoara timpul fiecarei navigari Angular (SPA route change) */
  private trackRouteChanges(): void {
    this.router.events.pipe(
      filter((event): event is NavigationEnd => event instanceof NavigationEnd)
    ).subscribe((event) => {
      const duration = performance.now() - this.navigationStart;
      this.reportMetric('spa_navigation', {
        url: event.urlAfterRedirects,
        duration,
        timestamp: Date.now(),
      });
    });

    // Captureaza start-ul navigarii
    this.router.events.subscribe((event) => {
      if (event.constructor.name === 'NavigationStart') {
        this.navigationStart = performance.now();
      }
    });
  }

  /** Monitorizeaza resursele incarcate lent */
  private trackResourceLoading(): void {
    if (typeof PerformanceObserver === 'undefined') return;

    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        const resource = entry as PerformanceResourceTiming;
        if (resource.duration > 1000) { // Resurse peste 1 secunda
          this.reportMetric('slow_resource', {
            name: resource.name,
            duration: resource.duration,
            type: resource.initiatorType,
            size: resource.transferSize,
          });
        }
      }
    });
    observer.observe({ type: 'resource', buffered: false });
  }

  /** Detecteaza task-urile care blocheaza main thread-ul */
  private trackLongTasks(): void {
    if (typeof PerformanceObserver === 'undefined') return;

    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.duration > 50) { // Long task = peste 50ms
          this.reportMetric('long_task', {
            duration: entry.duration,
            url: window.location.pathname,
          });
        }
      }
    });

    try {
      observer.observe({ type: 'longtask', buffered: true });
    } catch {
      // longtask nu e suportat in toate browserele
    }
  }

  private reportMetric(name: string, data: Record<string, unknown>): void {
    // Trimite la backend de metrici
    navigator.sendBeacon('/api/rum', JSON.stringify({ name, ...data }));
  }
}
```

> **Perspectiva Principal Engineer:** Observability nu e un "nice-to-have" la nivel de Principal - e o cerinta fundamentala. Daca nu poti masura, nu poti imbunatati. Cele trei intrebari pe care trebuie sa le poti raspunde oricand: (1) Ce erori au utilizatorii in productie? (2) Cat de repede se incarca aplicatia? (3) Ce se degradeaza in timp? Fara monitoring, esti orb.

---

## 7. Error tracking cu Sentry

### Integrare Sentry cu Angular

Sentry este cel mai popular serviciu de error tracking pentru aplicatii web. Captureaza erori, exceptii, si performanta cu context complet (stack traces, breadcrumbs, user info).

```bash
# Instalare
npm install @sentry/angular @sentry/tracing
```

```typescript
// sentry.config.ts - Configurare centralizata
import * as Sentry from '@sentry/angular';
import { environment } from '../environments/environment';

export function initSentry(): void {
  if (!environment.sentryDsn) {
    console.warn('Sentry DSN not configured, error tracking disabled');
    return;
  }

  Sentry.init({
    dsn: environment.sentryDsn,

    // Identificarea versiunii (legata de source maps)
    release: `my-app@${environment.appVersion}`,

    // Mediul curent
    environment: environment.production ? 'production' : 'development',

    // Integrations
    integrations: [
      // Tracking pentru performanta (tranzactii)
      Sentry.browserTracingIntegration(),
      // Replay sesiuni cu erori (inregistrare video-like)
      Sentry.replayIntegration({
        maskAllText: true,     // GDPR: ascunde textul
        blockAllMedia: true,   // GDPR: ascunde media
      }),
    ],

    // Sample rates
    tracesSampleRate: environment.production ? 0.1 : 1.0,  // 10% in prod
    replaysSessionSampleRate: 0.01,  // 1% din sesiuni normale
    replaysOnErrorSampleRate: 1.0,   // 100% din sesiuni cu erori

    // Filtrare erori irelevante
    beforeSend(event) {
      // Ignora erori de la extensii de browser
      if (event.exception?.values?.[0]?.stacktrace?.frames?.some(
        (frame) => frame.filename?.includes('extensions://')
      )) {
        return null;
      }

      // Ignora erori de retea pentru resurse externe
      if (event.exception?.values?.[0]?.value?.includes('Loading chunk')) {
        // Chunk loading error - utilizatorul are o versiune veche cached
        // Forteaza reload
        window.location.reload();
        return null;
      }

      return event;
    },

    // Ignora erori cunoscute si irelevante
    ignoreErrors: [
      'ResizeObserver loop limit exceeded',
      'ResizeObserver loop completed with undelivered notifications',
      'Non-Error exception captured',
      /Loading chunk [\d]+ failed/,
      'AbortError',
    ],

    // Nu trimite PII (Personally Identifiable Information) fara consimtamant
    sendDefaultPii: false,
  });
}
```

---

### ErrorHandler custom pentru Angular

```typescript
// sentry-error-handler.ts
import { ErrorHandler, Injectable, inject } from '@angular/core';
import * as Sentry from '@sentry/angular';
import { HttpErrorResponse } from '@angular/common/http';

@Injectable()
export class SentryErrorHandler implements ErrorHandler {
  handleError(error: unknown): void {
    // Extrage eroarea reala din wrapper-ele Angular
    const extractedError = this.extractError(error);

    // Clasifica si raporteaza
    if (extractedError instanceof HttpErrorResponse) {
      this.handleHttpError(extractedError);
    } else if (extractedError instanceof Error) {
      this.handleJsError(extractedError);
    } else {
      // Eroare necunoscuta
      Sentry.captureMessage(`Unknown error: ${String(error)}`, 'error');
    }

    // Logheaza si in consola pentru debugging local
    console.error('Global error caught:', extractedError);
  }

  private handleHttpError(error: HttpErrorResponse): void {
    // Nu raporta 401/403 (sunt flow-uri normale de auth)
    if (error.status === 401 || error.status === 403) {
      return;
    }

    // Nu raporta 0 (cereri anulate de utilizator)
    if (error.status === 0) {
      return;
    }

    Sentry.withScope((scope) => {
      scope.setTag('error.type', 'http');
      scope.setTag('http.status', String(error.status));
      scope.setTag('http.url', error.url || 'unknown');
      scope.setContext('http_error', {
        status: error.status,
        statusText: error.statusText,
        url: error.url,
        message: error.message,
      });

      // Severitate bazata pe status code
      if (error.status >= 500) {
        scope.setLevel('error');
      } else {
        scope.setLevel('warning');
      }

      Sentry.captureException(
        new Error(`HTTP ${error.status}: ${error.url}`)
      );
    });
  }

  private handleJsError(error: Error): void {
    Sentry.withScope((scope) => {
      scope.setTag('error.type', 'javascript');

      // Adauga breadcrumbs cu navigarea recenta
      const currentRoute = window.location.pathname;
      scope.addBreadcrumb({
        category: 'navigation',
        message: `User was on: ${currentRoute}`,
        level: 'info',
      });

      Sentry.captureException(error);
    });
  }

  private extractError(error: unknown): Error | HttpErrorResponse {
    // Angular wrapper-eaza erorile - trebuie sa extragem cauza reala
    if (error instanceof Error) {
      // Verifica daca are o cauza nested
      const cause = (error as any).rejection || (error as any).originalError;
      return cause instanceof Error ? cause : error;
    }

    if (error instanceof HttpErrorResponse) {
      return error;
    }

    // Creeaza o eroare din orice altceva
    return new Error(String(error));
  }
}
```

```typescript
// app.config.ts - Inregistrare
import { ApplicationConfig, ErrorHandler } from '@angular/core';
import { SentryErrorHandler } from './sentry-error-handler';
import { initSentry } from './sentry.config';

// Initializeaza Sentry inainte de Angular bootstrap
initSentry();

export const appConfig: ApplicationConfig = {
  providers: [
    // Inlocuieste ErrorHandler-ul default cu Sentry
    { provide: ErrorHandler, useClass: SentryErrorHandler },

    // Sentry trace propagation pentru HTTP requests
    {
      provide: Sentry.TraceService,
      deps: [Router],
    },
    {
      provide: APP_INITIALIZER,
      useFactory: () => () => {},
      deps: [Sentry.TraceService],
      multi: true,
    },
  ],
};
```

---

### Source maps upload

Source maps permit Sentry sa traduca stack trace-urile minificate in cod lizibil. Fara ele, erorile arata ca `main.a1b2c3.js:1:45678` in loc de `user.service.ts:42`.

```bash
# Instalare Sentry CLI
npm install -D @sentry/cli
```

```yaml
# GitHub Actions - Upload source maps la deploy
- name: Upload source maps to Sentry
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
    SENTRY_ORG: my-org
    SENTRY_PROJECT: angular-app
  run: |
    VERSION="my-app@$(node -p 'require("./package.json").version')"

    # Creeaza un release in Sentry
    npx sentry-cli releases new "$VERSION"

    # Upload source maps
    npx sentry-cli releases files "$VERSION" upload-sourcemaps \
      ./dist/my-app/browser/ \
      --url-prefix "~/" \
      --rewrite

    # Marcheaza release-ul ca deployed
    npx sentry-cli releases finalize "$VERSION"
    npx sentry-cli releases deploys "$VERSION" new \
      --env production

    # IMPORTANT: sterge source maps din bundle-ul deploiat
    # Nu vrei ca utilizatorii sa vada codul sursa original
    find ./dist -name "*.map" -delete
```

```json
// .sentryclirc (sau sentry.properties)
{
  "defaults": {
    "org": "my-org",
    "project": "angular-app"
  }
}
```

> **Perspectiva Principal Engineer:** Sentry fara source maps e aproape inutil - nu poti debugga `e.a is not a function at main.js:1:238947`. Source maps sunt OBLIGATORII, dar trebuie sterse din productie dupa upload la Sentry. Altfel expui codul sursa original utilizatorilor. Pipeline-ul de CI/CD trebuie sa faca: build --> upload source maps la Sentry --> sterge .map din dist --> deploy.

---

## 8. Feature flags

### Ce sunt si de ce le folosim

Feature flags (sau feature toggles) sunt mecanisme care permit activarea/dezactivarea functionalitatilor fara a face un nou deployment. Sunt esentiale pentru:

- **Trunk-based development** - toti developerii lucreaza pe `main`, codul incomplet e ascuns sub flag-uri
- **Progressive rollout** - activeaza o functionalitate pentru 5% din utilizatori, apoi 25%, apoi 100%
- **Kill switch** - dezactiveaza instant o functionalitate problematica fara rollback
- **A/B testing** - varianta A vs varianta B pe segmente diferite de utilizatori
- **Permissioning** - anumite features doar pentru plan-ul Premium

```
Fara feature flags:
  develop -> feature-branch -> merge -> deploy -> DISPONIBIL TUTUROR
  Rollback = nou deployment

Cu feature flags:
  main -> deploy (flag OFF) -> activare graduala -> DISPONIBIL CONTROLAT
  Rollback = dezactiveaza flag-ul (instant, fara deploy)
```

---

### Implementare custom in Angular

```typescript
// feature-flag.service.ts
import { Injectable, signal, computed } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { interval, switchMap, retry } from 'rxjs';

export interface FeatureFlag {
  name: string;
  enabled: boolean;
  variant?: string;     // Pentru A/B testing
  percentage?: number;   // Pentru progressive rollout
}

@Injectable({ providedIn: 'root' })
export class FeatureFlagService {
  /** Sursa de adevar reactiva pentru toate flag-urile */
  private flags = signal<Map<string, FeatureFlag>>(new Map());

  constructor(private http: HttpClient) {}

  /** Incarca flag-urile de pe server (apelat la startup via APP_INITIALIZER) */
  async initialize(): Promise<void> {
    try {
      const flags = await fetch('/api/feature-flags').then(r => r.json());
      const flagMap = new Map<string, FeatureFlag>();
      for (const flag of flags) {
        flagMap.set(flag.name, flag);
      }
      this.flags.set(flagMap);
    } catch (error) {
      console.error('Failed to load feature flags, using defaults:', error);
    }

    // Polling pentru actualizari (fiecare 60 secunde)
    this.startPolling();
  }

  /** Verifica daca un flag e activ - reactive cu signals */
  isEnabled(flagName: string): boolean {
    const flag = this.flags().get(flagName);
    if (!flag) return false;

    // Verificare bazata pe procent (consistent per utilizator)
    if (flag.percentage !== undefined && flag.percentage < 100) {
      return this.isInPercentage(flagName, flag.percentage);
    }

    return flag.enabled;
  }

  /** Signal reactiv pentru un flag specific */
  isEnabled$(flagName: string) {
    return computed(() => {
      const flag = this.flags().get(flagName);
      return flag?.enabled ?? false;
    });
  }

  /** Obtine varianta A/B pentru un flag */
  getVariant(flagName: string): string | undefined {
    return this.flags().get(flagName)?.variant;
  }

  /** Hash deterministic pentru consistent bucketing */
  private isInPercentage(flagName: string, percentage: number): boolean {
    const userId = this.getUserId();
    const hash = this.simpleHash(`${flagName}:${userId}`);
    return (hash % 100) < percentage;
  }

  private getUserId(): string {
    // Persistent per browser (sessionStorage sau localStorage)
    let id = localStorage.getItem('_ff_user_id');
    if (!id) {
      id = crypto.randomUUID();
      localStorage.setItem('_ff_user_id', id);
    }
    return id;
  }

  private simpleHash(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash |= 0;
    }
    return Math.abs(hash);
  }

  private startPolling(): void {
    interval(60_000).pipe(
      switchMap(() => this.http.get<FeatureFlag[]>('/api/feature-flags')),
      retry({ count: 3, delay: 5000 })
    ).subscribe((flags) => {
      const flagMap = new Map<string, FeatureFlag>();
      for (const flag of flags) {
        flagMap.set(flag.name, flag);
      }
      this.flags.set(flagMap);
    });
  }
}
```

---

### Directiva Angular pentru feature flags

```typescript
// feature-flag.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef,
  OnInit,
  OnDestroy,
  effect,
  inject,
} from '@angular/core';
import { FeatureFlagService } from './feature-flag.service';

@Directive({
  selector: '[featureFlag]',
  standalone: true,
})
export class FeatureFlagDirective implements OnInit {
  @Input({ required: true }) featureFlag!: string;
  @Input() featureFlagElse?: TemplateRef<unknown>;

  private templateRef = inject(TemplateRef<unknown>);
  private viewContainer = inject(ViewContainerRef);
  private featureFlagService = inject(FeatureFlagService);
  private hasView = false;

  ngOnInit(): void {
    // Reactiv: se actualizeaza automat cand flag-urile se schimba
    effect(() => {
      const isEnabled = this.featureFlagService.isEnabled$(this.featureFlag)();

      if (isEnabled && !this.hasView) {
        this.viewContainer.clear();
        this.viewContainer.createEmbeddedView(this.templateRef);
        this.hasView = true;
      } else if (!isEnabled) {
        this.viewContainer.clear();
        if (this.featureFlagElse) {
          this.viewContainer.createEmbeddedView(this.featureFlagElse);
        }
        this.hasView = false;
      }
    });
  }
}
```

**Utilizare in template:**

```html
<!-- Arata componenta doar daca flag-ul e activ -->
<div *featureFlag="'newDashboard'">
  <app-new-dashboard />
</div>

<!-- Cu template alternativ (else) -->
<div *featureFlag="'newCheckout'; else oldCheckout">
  <app-new-checkout />
</div>
<ng-template #oldCheckout>
  <app-legacy-checkout />
</ng-template>

<!-- Guard pe rute -->
```

```typescript
// feature-flag.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { FeatureFlagService } from './feature-flag.service';

export function featureFlagGuard(flagName: string): CanActivateFn {
  return () => {
    const featureFlagService = inject(FeatureFlagService);
    const router = inject(Router);

    if (featureFlagService.isEnabled(flagName)) {
      return true;
    }

    // Redirect la pagina principala daca flag-ul nu e activ
    return router.createUrlTree(['/']);
  };
}

// Utilizare in routes
export const routes: Routes = [
  {
    path: 'new-feature',
    loadComponent: () => import('./new-feature/new-feature.component'),
    canActivate: [featureFlagGuard('newFeaturePage')],
  },
];
```

---

### Servicii externe: LaunchDarkly, Unleash

```typescript
// Exemplu integrare LaunchDarkly
import * as LDClient from 'launchdarkly-js-client-sdk';

@Injectable({ providedIn: 'root' })
export class LaunchDarklyService {
  private client!: LDClient.LDClient;
  private flags = signal<Record<string, boolean>>({});

  async initialize(userId: string): Promise<void> {
    const context: LDClient.LDContext = {
      kind: 'user',
      key: userId,
      // Atribute pentru targeting
      email: 'user@example.com',
      plan: 'premium',
      country: 'RO',
    };

    this.client = LDClient.initialize('your-client-side-id', context);
    await this.client.waitForInitialization();

    // Reactualizare la schimbari in timp real (streaming)
    this.client.on('change', () => {
      this.flags.set(this.client.allFlags());
    });

    this.flags.set(this.client.allFlags());
  }

  isEnabled(flag: string, defaultValue = false): boolean {
    return this.client?.variation(flag, defaultValue) ?? defaultValue;
  }
}
```

---

### Strategia de cleanup pentru feature flags

Feature flags vechi, abandonate, sunt o forma de technical debt. Trebuie curatate sistematic.

```
Ciclul de viata al unui feature flag:

1. CREAT     - Developer adauga flag-ul in cod si pe platforma
2. DEZVOLTAT - Codul din spatele flag-ului e implementat
3. TESTAT    - Flag activat pe staging, testat
4. ROLLED OUT - Flag activat in productie (gradual sau complet)
5. PERMANENT - Dupa validare, flag-ul e 100% activ de X zile
6. CLEANUP   - Sterge flag-ul din cod si platforma

Regula: Daca un flag e 100% activ de peste 30 de zile, trebuie sters.
```

```typescript
// Conventie: adauga data de expirare in comentariu
// TODO(feature-flag): Remove 'newDashboard' flag after 2026-04-01
// Ticket: JIRA-1234

// Cleanup checklist:
// [ ] Sterge directiva/guard-ul cu flag-ul
// [ ] Sterge ramura 'else' din template (pastreaza doar ramura activa)
// [ ] Sterge flag-ul din backend/platforma
// [ ] Actualizeaza testele
```

> **Perspectiva Principal Engineer:** Feature flags sunt una dintre cele mai puternice unelte dintr-un toolkit de Principal Engineer. Permit decuplarea deploy-ului de release: deployezi cod in productie oricand, dar il activezi cand e gata. Problema: prea multe flag-uri abandonate transforma codebase-ul intr-un labirint de `if/else`. Impune o politica stricta de cleanup (ex: nu mai mult de 20 de flag-uri active simultan, reviewuri lunare, alertare pe flag-uri expirate).

---

## 9. Semantic versioning si release management

### SemVer: MAJOR.MINOR.PATCH

```
Versiune: 2.4.1
           | | |
           | | +-- PATCH: Bug fixes, fara breaking changes
           | +---- MINOR: Functionalitati noi, backward compatible
           +------ MAJOR: Breaking changes, API incompatibile

Reguli:
  - Bug fix fara impact API        -> PATCH: 2.4.1 --> 2.4.2
  - Feature nou, backward compat   -> MINOR: 2.4.2 --> 2.5.0
  - Breaking change                 -> MAJOR: 2.5.0 --> 3.0.0

Pre-release:
  - 3.0.0-alpha.1  (instabil, experimental)
  - 3.0.0-beta.1   (feature complete, testare)
  - 3.0.0-rc.1     (release candidate, aproape final)
```

---

### Conventional Commits

Conventional Commits este o conventie de mesaje de commit care permite generarea automata a changelog-urilor si determinarea tipului de versiune.

```bash
# Format: <type>(<scope>): <description>

# Patch bump (bug fix)
git commit -m "fix(auth): resolve token refresh race condition"

# Minor bump (feature)
git commit -m "feat(dashboard): add real-time notifications widget"

# Major bump (breaking change) - notat cu ! sau BREAKING CHANGE in footer
git commit -m "feat(api)!: change authentication to OAuth 2.0"
# sau
git commit -m "feat(api): change authentication to OAuth 2.0

BREAKING CHANGE: API endpoints now require Bearer token instead of API key"

# Alte tipuri (nu afecteaza versiunea)
git commit -m "docs(readme): update installation instructions"
git commit -m "chore(deps): update Angular to v17.1"
git commit -m "refactor(core): extract validation logic to separate service"
git commit -m "test(auth): add unit tests for login flow"
git commit -m "ci(github): add caching to CI pipeline"
git commit -m "perf(list): implement virtual scrolling for large datasets"
```

**Commit lint automation:**

```json
// package.json
{
  "devDependencies": {
    "@commitlint/cli": "^18.0.0",
    "@commitlint/config-conventional": "^18.0.0",
    "husky": "^9.0.0"
  }
}
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'scope-enum': [2, 'always', [
      'core', 'auth', 'dashboard', 'shared',
      'api', 'ui', 'deps', 'ci', 'docs'
    ]],
    'subject-max-length': [2, 'always', 72],
  },
};
```

```bash
# Husky - ruleaza commitlint la fiecare commit
npx husky add .husky/commit-msg 'npx commitlint --edit $1'
```

---

### Changelog automat si release

```bash
# Instalare standard-version (sau semantic-release)
npm install -D standard-version

# Generare release automata:
# - Analizeaza commit-urile de la ultimul tag
# - Determina tipul de bump (major/minor/patch)
# - Actualizeaza package.json
# - Genereaza CHANGELOG.md
# - Creeaza commit + tag
npx standard-version

# Release minor explicit
npx standard-version --release-as minor

# Pre-release
npx standard-version --prerelease alpha
```

**CHANGELOG.md generat automat:**

```markdown
# Changelog

## [2.5.0](https://github.com/org/app/compare/v2.4.2...v2.5.0) (2026-02-18)

### Features

* **dashboard:** add real-time notifications widget ([a1b2c3d](https://github.com/...))
* **auth:** implement biometric login support ([e4f5g6h](https://github.com/...))

### Bug Fixes

* **api:** resolve token refresh race condition ([i7j8k9l](https://github.com/...))
* **ui:** fix layout shift on mobile navigation ([m1n2o3p](https://github.com/...))

### Performance Improvements

* **list:** implement virtual scrolling for datasets > 10k rows ([q4r5s6t](https://github.com/...))
```

---

### Semantic Release - Automatizare completa

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Necesar pentru analiza istoricului complet
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci

      # semantic-release face totul automat:
      # 1. Analizeaza commit-urile de la ultimul release
      # 2. Determina versiunea noua
      # 3. Genereaza changelog
      # 4. Creeaza GitHub Release cu tag
      # 5. Publica pe npm (daca e o librarie)
      - name: Semantic Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: npx semantic-release
```

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    [
      "@semantic-release/npm",
      {
        "npmPublish": false
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["package.json", "CHANGELOG.md"],
        "message": "chore(release): ${nextRelease.version}\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

---

### Monorepo versioning (Nx)

```bash
# In monorepo cu Nx, fiecare librarie/app poate avea versiunea ei
# sau pot fi versionate impreuna

# Independent versioning (fiecare pachet separat)
npx nx release version --projects=ui-components
npx nx release changelog --projects=ui-components
npx nx release publish --projects=ui-components

# Fixed versioning (toate pachetele au aceeasi versiune)
npx nx release version
npx nx release changelog
npx nx release publish
```

```json
// nx.json - Release configuration
{
  "release": {
    "projects": ["packages/*"],
    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "groupByScope": true
      }
    },
    "changelog": {
      "projectChangelogs": true,
      "workspaceChangelog": {
        "file": "CHANGELOG.md",
        "createRelease": "github"
      }
    }
  }
}
```

---

### Release branches vs Trunk-based development

```
RELEASE BRANCHES:                    TRUNK-BASED:

main ----*----*----*----*--->        main ----*----*----*----*--->
          \                                   |    |    |    |
  release/1.0 ---*---*--->                    v    v    v    v
               \                           deploy deploy deploy deploy
    release/1.1 ---*--->                   (feature flags controleaza
                                            vizibilitatea)

Pro:                                 Pro:
+ Versiuni stabile izolate           + Feedback rapid
+ Hotfix-uri clare                   + Integrare continua reala
+ Bun pentru produse enterprise      + Mai putin merge hell

Con:                                 Con:
- Merge hell intre branches          - Necesita feature flags
- Feedback lent                      - Necesita disciplina de echipa
- Cherry-pick-uri complexe           - CI/CD matur obligatoriu
```

> **Perspectiva Principal Engineer:** Pentru aplicatii web Angular (SPA), recomand trunk-based development cu feature flags. Release branches au sens pentru SDK-uri, librarii publice sau produse care se livreaza on-premise. Conventional commits + semantic-release automatizeaza complet procesul de versioning, eliminand dezbaterile "ce versiune punem?". E o investitie de 2 ore de setup care economiseste saptamani pe an.

---

## Intrebari frecvente de interviu

### 1. Cum ai structura un pipeline CI/CD pentru o aplicatie Angular enterprise?

**Raspuns:**

Pipeline-ul trebuie optimizat pentru viteza (sub 10 minute pe PR) si fiabilitate. L-as structura pe etape:

**Lint** (1-2 min) - ESLint si Prettier ruleaza primele, sunt cele mai rapide. Daca codul nu e formatat, nu are sens sa rulezi teste.

**Test** (3-5 min) - Teste unitare cu Jest (nu Karma - Jest e mai rapid si nu necesita browser). Coverage gate la 80%. Teste E2E doar pe main, nu pe fiecare PR (sunt lente).

**Build** (2-3 min) - Build de productie cu AOT si budget checks. Daca bundle-ul depaseste budget-ul, pipeline-ul esueaza.

**Deploy** - Pe develop: deploy automat pe staging. Pe main: deploy automat pe productie (daca toate etapele trec) sau manual trigger.

**Optimizari critice:**
- Cache npm cu `actions/cache` sau `cache: 'npm'` in setup-node
- `npm ci` in loc de `npm install` (determinist si rapid)
- Paralelizare: lint si test pot rula simultan (nu depind unul de altul)
- Affected testing: intr-un monorepo Nx, ruleaza teste doar pentru proiectele afectate de schimbari

---

### 2. Explica diferenta intre Blue-Green si Canary deployment. Cand ai folosi fiecare?

**Raspuns:**

**Blue-Green** mentine doua medii identice. Traficul se comuta instant intre ele. Avantajul principal: rollback in secunde (comuta inapoi). Dezavantajul: dublu resurse.

**Canary** redirecteaza un procent mic de trafic (5-10%) catre noua versiune. Daca metricile sunt bune (error rate, latenta, business metrics), procentul creste gradual. Avantajul: risc minim. Dezavantajul: necesita observability matura si mai multa complexitate.

**Cand folosesc fiecare:**
- **Blue-Green**: pentru aplicatii Angular clasice (SPA pe CDN). E simplu: publici noile fisiere pe CDN, invalidezi cache-ul. "Mediul blue" e pur si simplu versiunea veche cached la CDN edge.
- **Canary**: cand Angular face SSR cu un server Node.js, sau cand am un API backend critic. Canary imi permite sa detectez probleme (memory leaks, erori de integrare) cu impact minimal.

In practica, pentru un SPA pur, Blue-Green cu CDN e suficient si mult mai simplu de operat.

---

### 3. Cum gestionezi environment variables intr-o aplicatie Angular deployata cu Docker?

**Raspuns:**

Exista doua abordari fundamentale si trebuie inteles trade-off-ul:

**Build time** (`environment.ts` + `fileReplacements`): Variabilele sunt "baked" in bundle. Pro: tree-shaking, simplitate. Con: o imagine Docker per mediu.

**Runtime** (preferata mea): Un script shell genereaza un `config.json` sau `env-config.js` la pornirea containerului Docker, din variabilele de mediu. Angular il incarca via `APP_INITIALIZER` sau `<script>` tag.

**De ce prefer runtime:** O singura imagine Docker (`angular-app:1.2.3`) care functioneaza pe dev, staging, QA si productie. Doar variabilele de mediu se schimba. Aceasta elimina o intreaga categorie de buguri ("merge pe staging dar nu pe prod") si simplifica pipeline-ul CI/CD (un singur build, nu patru).

**Pattern concret:** Script in `/docker-entrypoint.d/` genereaza `config.json` --> Angular il incarca cu `APP_INITIALIZER` --> serviciile injecteaza `AppConfigService`.

---

### 4. Ce security headers configurezi pe Nginx pentru o aplicatie Angular si de ce?

**Raspuns:**

- **X-Frame-Options: SAMEORIGIN** - Previne clickjacking (pagina nu poate fi incadrata in iframe pe alt domeniu)
- **X-Content-Type-Options: nosniff** - Previne MIME type sniffing (browserul respecta Content-Type-ul declarat)
- **Content-Security-Policy** - Cel mai important si complex. Defineste de unde se pot incarca resurse. Pentru Angular trebuie `'unsafe-inline'` pentru stiluri (Angular genereaza stiluri inline) si atentie la `'unsafe-eval'` (necesar doar in dev mode).
- **Referrer-Policy: strict-origin-when-cross-origin** - Controleaza ce informatii se trimit in header-ul Referer
- **Permissions-Policy** - Dezactiveaza API-uri ale browserului pe care nu le folosesti (camera, microfon, geolocalizare)
- **Strict-Transport-Security** (HSTS) - Forteaza HTTPS. Trebuie activat DOAR dupa ce HTTPS e configurat corect.

CSP este cel mai dificil de configurat corect. Recomand sa incepi cu `Content-Security-Policy-Report-Only` pentru a vedea ce ar fi blocat fara sa strici aplicatia, apoi treci la enforcement.

---

### 5. Cum implementezi feature flags in Angular si care e strategia de cleanup?

**Raspuns:**

**Implementare:** Un `FeatureFlagService` care incarca flag-urile de la un endpoint API la startup (via `APP_INITIALIZER`) si le actualizeza periodic (polling sau SSE/WebSocket). O directiva structurala `*featureFlag="'flagName'"` care arata/ascunde elemente din template. Un guard `featureFlagGuard('flagName')` pentru rute. Serviciul expune signals reactive pentru ca UI-ul sa se actualizeze automat cand flag-urile se schimba.

**Platforma:** Pentru echipe mici, un serviciu custom cu flag-uri in baza de date e suficient. Pentru echipe mari, LaunchDarkly sau Unleash ofera targeting avansat (pe utilizator, pe tara, pe plan), A/B testing, si audit trail.

**Cleanup - partea critica pe care multi o ignora:**
1. Fiecare flag are o data de expirare (ex: 30 de zile dupa 100% rollout)
2. Review lunar: stergem flag-urile expirate
3. Limita: maxim 15-20 flag-uri active simultan
4. Fiecare flag are un ticket JIRA de cleanup asociat
5. Linter custom care avertizeaza pe flag-uri expirate

Fara cleanup, feature flags devin technical debt masiv - am vazut codebaze cu 200+ flag-uri abandonate, unde nimeni nu stia care mai sunt relevante.

---

### 6. Cum configurezi Sentry pentru Angular si ce erori filtrezi?

**Raspuns:**

**Setup:**
1. `Sentry.init()` in `main.ts` inainte de Angular bootstrap, cu DSN, release, environment
2. `ErrorHandler` custom care clasifica erorile (HTTP vs JS) si adauga context
3. Source maps upload in pipeline-ul de CI/CD (si stergere din productie!)
4. Sample rates: 10% tracing in productie (altfel costa mult), 100% pe erori

**Filtrare - esentiala pentru a nu fi inundat de zgomot:**
- **Erori de extensii de browser** - `extensions://` in stack trace --> ignora
- **ResizeObserver loop** - Bug de browser, nu al aplicatiei --> ignora
- **Chunk loading failed** - Utilizatorul are versiune veche cached --> reload automat, nu raporta
- **401/403 HTTP** - Flow normal de autentificare --> nu raporta
- **Status 0** - Cerere anulata de utilizator (navigare rapida) --> nu raporta
- **AbortError** - La fel, navigare/anulare normala

**Ce raportam:** erori JavaScript reale, erori HTTP 500+, erori de API neasteptate, performance degradation. Scopul e sa ai sub 50 de erori relevante pe zi, nu mii de false positives.

---

### 7. Explica strategia de caching Nginx pentru o aplicatie Angular SPA.

**Raspuns:**

Strategia se bazeaza pe un principiu simplu: **index.html fara cache, tot restul cu cache agresiv**.

**index.html** (`Cache-Control: no-cache`) - Browserul cere mereu versiunea proaspata. Fisierul e mic (~4KB), deci costul e neglijabil. index.html contine referintele la bundle-uri cu hash in nume.

**JS/CSS cu hash** (`Cache-Control: public, immutable, max-age=31536000`) - Fisierele Angular au hash in nume (`main.a1b2c3.js`). Cand faci un build nou, hash-ul se schimba. Browserul descarca noul fisier (alt URL). Fisierul vechi ramane in cache dar nu mai e referentiat.

**Fonturi/imagini** (`max-age=6M`) - Cache mediu. Se schimba rar, dar nu au hash in nume.

**try_files $uri $uri/ /index.html** - Esential pentru SPA! Cand utilizatorul acceseaza `/dashboard/users`, Nginx nu gaseste un fisier `/dashboard/users`, asa ca serveste `index.html`, iar Angular Router preia rutarea pe client.

Aceasta strategie garanteaza ca utilizatorii vad mereu versiunea curenta (prin index.html proaspat) fara sa descarce de fiecare data bundle-urile mari (care sunt cached agresiv).

---

### 8. Ce inseamna Conventional Commits si cum le integrezi in workflow-ul de release?

**Raspuns:**

Conventional Commits e o conventie de mesaje de commit cu format structurat: `type(scope): description`. Tipurile standard: `feat` (feature), `fix` (bug fix), `chore`, `docs`, `refactor`, `test`, `perf`, `ci`.

**Beneficiul principal:** Automatizarea completa a release-ului. Unelte ca `semantic-release` analizeaza commit-urile de la ultimul release si determina automat versiunea:
- `fix:` --> PATCH bump (1.0.0 --> 1.0.1)
- `feat:` --> MINOR bump (1.0.0 --> 1.1.0)
- `feat!:` sau `BREAKING CHANGE:` --> MAJOR bump (1.0.0 --> 2.0.0)

**Workflow complet:**
1. `commitlint` + Husky valideaza mesajele la fiecare commit
2. La merge pe main, `semantic-release` determina versiunea, genereaza CHANGELOG, creeaza GitHub Release si tag
3. Pipeline-ul de CI/CD detecteaza noul tag si face deploy

**In monorepo:** Nx Release suporta versioning independent per librarie sau fix (toate la aceeasi versiune), tot bazat pe conventional commits.

Eliminam complet dezbaterile manuale despre versiuni - commit-urile spun povestea, tooling-ul face restul.

---

### 9. Cum ai implementa monitoring pentru o aplicatie Angular la scara mare?

**Raspuns:**

As structura monitoring-ul pe trei niveluri:

**Nivel 1 - Error tracking (Sentry):** Captureaza fiecare eroare JavaScript si HTTP din productie, cu stack trace lizibil (source maps), breadcrumbs (ce a facut utilizatorul inainte de eroare), si session replay. Alerteaza echipa pe Slack cand error rate depaseste un prag.

**Nivel 2 - Performance monitoring (Web Vitals + RUM):** Colectez LCP, CLS, INP/FID de la utilizatorii reali (nu din teste sintetice). Datele ajung in Prometheus, vizualizate in Grafana. Dashboards pe: P50/P75/P95 per metrica, breakdown pe pagina/ruta, trend in timp. Alertez cand P75 LCP depaseste 2.5s.

**Nivel 3 - Business metrics:** Conversion rate, bounce rate, feature adoption. Acestea conecteaza performanta tehnica de impactul business. Daca LCP se degradeaza si conversion rate scade simultan, avem dovada clara pentru prioritizarea optimizarii.

**Principiul cheie:** Monitoring-ul trebuie sa fie proactiv, nu reactiv. Nu astept ca un utilizator sa raporteze o problema - alertele ma anunta automat. La scara mare (milioane de utilizatori), sample rate de 1-10% e suficient si economic.

---

### 10. Descrie un Dockerfile optimizat pentru Angular. Ce greseli comune ai vazut?

**Raspuns:**

**Dockerfile optimizat:** Multi-stage build cu doua etape. Prima etapa: `node:20-alpine`, copiez `package*.json`, `npm ci`, copiez codul, `ng build --configuration=production`. A doua etapa: `nginx:alpine`, copiez doar artefactele din prima etapa. Imagine finala: ~25MB cu doar Nginx si fisiere statice.

**Greseli comune pe care le-am corectat in echipe:**

1. **Single-stage build** - Lasa Node.js si node_modules (1GB+) in imagine de productie. Fix: multi-stage.

2. **`COPY . .` inainte de `npm ci`** - Orice schimbare in cod invalideaza cache-ul Docker pentru `npm ci`. Fix: copiaza `package*.json` separat, instaleaza dependintele, apoi copiaza restul.

3. **Lipsa .dockerignore** - Trimite node_modules, .git, fisierele .env la Docker daemon. Build context e enorm, build-ul e lent, si risc de securitate.

4. **`npm install` in loc de `npm ci`** - `npm install` poate schimba `package-lock.json`, e non-determinist. `npm ci` respecta exact lock file-ul.

5. **Nu sterg source maps** - `.map` files in productie expun codul sursa original.

6. **Ruleaza Nginx ca root** - Fix: `user nginx;` in nginx.conf sau `--user` flag la `docker run`.

7. **Variabile de mediu baked la build** - Necesita rebuild per mediu. Fix: runtime configuration cu script la container startup.