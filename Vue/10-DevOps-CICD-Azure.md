# DevOps, CI/CD și Azure (Interview Prep - Senior Frontend Architect)

> Azure DevOps Pipelines pentru Vue MFE, Azure Bicep (IaC), container deployment,
> environment management. Paralele pipeline Angular vs Vue.
> Pregătit pentru candidați cu experiență solidă în Angular DevOps.

---

## Cuprins

1. [Azure DevOps Pipeline YAML pentru Vue](#1-azure-devops-pipeline-yaml-pentru-vue)
2. [Pipeline pentru Multiple MFE-uri](#2-pipeline-pentru-multiple-mfe-uri)
3. [Build Strategies (Vite vs Webpack)](#3-build-strategies-vite-vs-webpack)
4. [Azure Bicep Basics (Infrastructure as Code)](#4-azure-bicep-basics-infrastructure-as-code)
5. [Container Deployment (Docker + Azure)](#5-container-deployment-docker--azure)
6. [Environment Management](#6-environment-management)
7. [Caching și Optimizări Pipeline](#7-caching-și-optimizări-pipeline)
8. [Monitoring și Alerting](#8-monitoring-și-alerting)
9. [Paralela: Pipeline Angular vs Vue](#9-paralela-pipeline-angular-vs-vue)
10. [Întrebări de interviu](#10-întrebări-de-interviu)

---

## 1. Azure DevOps Pipeline YAML pentru Vue

### 1.1 Concepte fundamentale

Un **Azure DevOps Pipeline** definit în YAML descrie întregul ciclu de viață al aplicației: de la commit la deployment. Pentru o aplicație Vue.js, pipeline-ul urmează aceeași structură ca pentru Angular, dar cu tooling diferit.

**Structura generală:**

```
trigger → pool → variables → stages → jobs → steps
```

- **trigger** - când rulează pipeline-ul (branch, path, tag)
- **pool** - pe ce mașină rulează (ubuntu-latest, windows-latest)
- **variables** - configurări reutilizabile
- **stages** - etape majore (Build, Test, Deploy Staging, Deploy Prod)
- **jobs** - unități de lucru în cadrul unui stage
- **steps** - comenzi individuale (script, task)

### 1.2 Pipeline simplu - Single Vue App

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/**
      - package.json
      - vite.config.ts
      - tsconfig*.json

pool:
  vmImage: 'ubuntu-latest'

variables:
  nodeVersion: '20.x'
  npmCache: $(Pipeline.Workspace)/.npm
  isMain: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]
  isDevelop: $[eq(variables['Build.SourceBranch'], 'refs/heads/develop')]

stages:
  # ─── Stage 1: Build & Test ───
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildJob
        displayName: 'Build Vue Application'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)
            displayName: 'Install Node.js $(nodeVersion)'

          - task: Cache@2
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              restoreKeys: |
                npm | "$(Agent.OS)"
              path: $(npmCache)
            displayName: 'Cache npm packages'

          - script: npm ci --cache $(npmCache)
            displayName: 'Install dependencies'

          - script: npx vue-tsc --noEmit
            displayName: 'TypeScript type check'

          - script: npm run lint
            displayName: 'ESLint check'

          - script: npm run test:unit -- --coverage --reporter=junit
            displayName: 'Unit tests (Vitest)'

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'junit-report.xml'
            displayName: 'Publish test results'

          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: 'coverage/cobertura-coverage.xml'
            displayName: 'Publish code coverage'

          - script: npm run build -- --mode $(Build.SourceBranchName)
            displayName: 'Build application'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'dist'
              ArtifactName: 'vue-app'
            displayName: 'Publish build artifacts'

  # ─── Stage 2: Deploy to Staging ───
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables.isDevelop, true))
    jobs:
      - deployment: DeployStagingJob
        displayName: 'Deploy to Staging Environment'
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureStaticWebApp@0
                  inputs:
                    app_location: '$(Pipeline.Workspace)/vue-app'
                    skip_app_build: true
                  displayName: 'Deploy to Azure Static Web App'

  # ─── Stage 3: Deploy to Production ───
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: Build
    condition: and(succeeded(), eq(variables.isMain, true))
    jobs:
      - deployment: DeployProdJob
        displayName: 'Deploy to Production Environment'
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'production-subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Upload static files to Azure Blob Storage
                      az storage blob upload-batch \
                        --source $(Pipeline.Workspace)/vue-app \
                        --destination '$web' \
                        --account-name prodstorageaccount \
                        --overwrite true

                      # Purge CDN cache
                      az cdn endpoint purge \
                        --resource-group rg-frontend \
                        --profile-name cdn-profile \
                        --name cdn-endpoint \
                        --content-paths '/*'
                  displayName: 'Upload to Storage & Purge CDN'
```

### 1.3 Explicații cheie

**`npm ci` vs `npm install`** - `npm ci` este mai rapid și mai deterministic. Șterge `node_modules` și instalează exact ce e în `package-lock.json`. Ideal pentru CI/CD.

**`vue-tsc --noEmit`** - Verificare TypeScript separată. Spre deosebire de Angular unde `ng build` include verificarea TS automat, în Vue trebuie rulat explicit.

**`--mode $(Build.SourceBranchName)`** - Vite folosește fișiere `.env.[mode]` pentru configurare per environment. Branch-ul `develop` va folosi `.env.develop`, iar `main` va folosi `.env.production`.

**`environment: production`** - Azure DevOps Environments permit approval gates, astfel încât deployment-ul în producție necesită aprobare manuală.

### Paralela cu Angular

| Aspect | Angular Pipeline | Vue Pipeline |
|--------|-----------------|--------------|
| Type check | Inclus în `ng build` | Separat: `vue-tsc --noEmit` |
| Build | `ng build --configuration=production` | `vite build --mode production` |
| Test | `ng test --watch=false --code-coverage` | `vitest run --coverage` |
| Lint | `ng lint` | `eslint .` sau `npm run lint` |
| Env config | `environment.ts` files (compile-time) | `.env` files / `import.meta.env` |

---

## 2. Pipeline pentru Multiple MFE-uri

### 2.1 Strategia generală

Într-o arhitectură **Micro-Frontend (MFE)**, fiecare MFE trebuie să poată fi:
- **Build-uit independent** - schimbarea într-un MFE nu rebuild-uiește tot
- **Deployat independent** - fără a afecta alte MFE-uri
- **Testat independent** - suite de teste per MFE
- **Versionat independent** - fiecare MFE are propriul ciclu de release

### 2.2 Template reutilizabil pentru MFE build

```yaml
# templates/mfe-build-template.yml
parameters:
  - name: mfeName
    type: string
  - name: mfePath
    type: string
  - name: deployContainer
    type: string
    default: '$web'
  - name: storageAccount
    type: string
    default: 'mfeprodstorageaccount'

stages:
  - stage: Build_${{ parameters.mfeName }}
    displayName: 'Build ${{ parameters.mfeName }}'
    jobs:
      - job: BuildAndTest
        displayName: 'Build & Test ${{ parameters.mfeName }}'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
            displayName: 'Install Node.js'

          - task: Cache@2
            inputs:
              key: 'npm | "$(Agent.OS)" | ${{ parameters.mfePath }}/package-lock.json'
              path: $(Pipeline.Workspace)/.npm
            displayName: 'Cache npm for ${{ parameters.mfeName }}'

          - script: |
              cd ${{ parameters.mfePath }}
              npm ci
            displayName: 'Install dependencies'

          - script: |
              cd ${{ parameters.mfePath }}
              npx vue-tsc --noEmit
            displayName: 'Type check'

          - script: |
              cd ${{ parameters.mfePath }}
              npm run lint
            displayName: 'Lint'

          - script: |
              cd ${{ parameters.mfePath }}
              npm run test:unit -- --coverage
            displayName: 'Unit tests'

          - script: |
              cd ${{ parameters.mfePath }}
              npm run build
            displayName: 'Build ${{ parameters.mfeName }}'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '${{ parameters.mfePath }}/dist'
              ArtifactName: '${{ parameters.mfeName }}'
            displayName: 'Publish artifacts'

  - stage: Deploy_${{ parameters.mfeName }}
    displayName: 'Deploy ${{ parameters.mfeName }}'
    dependsOn: Build_${{ parameters.mfeName }}
    condition: succeeded()
    jobs:
      - deployment: DeployMFE
        displayName: 'Deploy ${{ parameters.mfeName }} to Azure'
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'production-subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Upload MFE to its own container/path
                      az storage blob upload-batch \
                        --source $(Pipeline.Workspace)/${{ parameters.mfeName }} \
                        --destination '${{ parameters.deployContainer }}/${{ parameters.mfeName }}' \
                        --account-name ${{ parameters.storageAccount }} \
                        --overwrite true

                      # Purge CDN only for this MFE's path
                      az cdn endpoint purge \
                        --resource-group rg-frontend \
                        --profile-name cdn-profile \
                        --name cdn-endpoint \
                        --content-paths '/${{ parameters.mfeName }}/*'
                  displayName: 'Upload & Purge CDN'
```

### 2.3 Pipeline principal care folosește template-ul

```yaml
# azure-pipelines-mfe.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - apps/**

resources:
  repositories:
    - repository: templates
      type: git
      name: DevOps/pipeline-templates

stages:
  # Host / Shell application
  - template: templates/mfe-build-template.yml
    parameters:
      mfeName: 'shell'
      mfePath: 'apps/shell'
      deployContainer: '$web'

  # MFE Products
  - template: templates/mfe-build-template.yml
    parameters:
      mfeName: 'mfe-products'
      mfePath: 'apps/mfe-products'

  # MFE Checkout
  - template: templates/mfe-build-template.yml
    parameters:
      mfeName: 'mfe-checkout'
      mfePath: 'apps/mfe-checkout'

  # MFE User Profile
  - template: templates/mfe-build-template.yml
    parameters:
      mfeName: 'mfe-profile'
      mfePath: 'apps/mfe-profile'
```

### 2.4 Triggering doar MFE-urile modificate

```yaml
# Pipeline per MFE - apps/mfe-products/azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - apps/mfe-products/**
      - packages/shared-utils/**  # shared dependencies

pool:
  vmImage: 'ubuntu-latest'

# ... build & deploy steps doar pentru mfe-products
```

**Abordare alternativă - Change detection cu script:**

```yaml
# Detectare automată MFE-uri modificate
steps:
  - script: |
      # Determină ce MFE-uri s-au schimbat
      CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
      echo "Changed files: $CHANGED_FILES"

      # Setează variabile per MFE
      if echo "$CHANGED_FILES" | grep -q "apps/mfe-products/"; then
        echo "##vso[task.setvariable variable=buildProducts;isOutput=true]true"
      fi
      if echo "$CHANGED_FILES" | grep -q "apps/mfe-checkout/"; then
        echo "##vso[task.setvariable variable=buildCheckout;isOutput=true]true"
      fi
      if echo "$CHANGED_FILES" | grep -q "packages/shared/"; then
        echo "##vso[task.setvariable variable=buildAll;isOutput=true]true"
      fi
    name: detectChanges
    displayName: 'Detect changed MFEs'
```

### 2.5 Matrix strategy pentru build-uri paralele

```yaml
# Build paralel al tuturor MFE-urilor
jobs:
  - job: BuildMFEs
    strategy:
      matrix:
        shell:
          mfeName: 'shell'
          mfePath: 'apps/shell'
        products:
          mfeName: 'mfe-products'
          mfePath: 'apps/mfe-products'
        checkout:
          mfeName: 'mfe-checkout'
          mfePath: 'apps/mfe-checkout'
        profile:
          mfeName: 'mfe-profile'
          mfePath: 'apps/mfe-profile'
      maxParallel: 4
    steps:
      - task: NodeTool@0
        inputs:
          versionSpec: '20.x'

      - script: |
          cd $(mfePath)
          npm ci
          npm run test:unit
          npm run build
        displayName: 'Build $(mfeName)'

      - task: PublishBuildArtifacts@1
        inputs:
          PathtoPublish: '$(mfePath)/dist'
          ArtifactName: '$(mfeName)'
```

### 2.6 Dynamic remote URL management

```yaml
# Variables per stage/environment
variables:
  - name: mfeBaseUrl
    ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
      value: 'https://mfe-prod.azureedge.net'
    ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/develop') }}:
      value: 'https://mfe-staging.azureedge.net'

steps:
  - script: |
      # Injectează URL-urile remote în configurație
      cat > src/mfe-config.json << EOF
      {
        "remotes": {
          "mfeProducts": "$(mfeBaseUrl)/mfe-products/remoteEntry.js",
          "mfeCheckout": "$(mfeBaseUrl)/mfe-checkout/remoteEntry.js",
          "mfeProfile": "$(mfeBaseUrl)/mfe-profile/remoteEntry.js"
        }
      }
      EOF
    displayName: 'Generate MFE remote config'
```

**Runtime configuration injection** - alternativa la build-time config:

```typescript
// runtime-config.ts - încărcat la runtime, nu la build
export async function loadMfeConfig(): Promise<MfeConfig> {
  const response = await fetch('/config/mfe-config.json');
  return response.json();
}

// Avantaj: aceeași imagine Docker/build funcționează în orice environment
// Dezavantaj: cerere HTTP suplimentară la startup
```

### Paralela cu Angular

În Angular cu **Nx**, ai `affected:build` care detectează automat ce proiecte trebuie rebuild-uite pe baza grafului de dependențe. În Vue monorepo, această funcționalitate trebuie implementată manual (cu Turborepo, Lerna, sau scripturi custom) sau adoptat Nx și pentru proiecte Vue.

---

## 3. Build Strategies (Vite vs Webpack)

### 3.1 Vite build configuration

```typescript
// vite.config.ts - configurație completă pentru producție
import { defineConfig, type UserConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig(({ mode }): UserConfig => {
  const isProd = mode === 'production';

  return {
    plugins: [
      vue(),
      // Generează raport vizual al bundle-ului (util pentru pipeline)
      isProd && visualizer({
        filename: 'dist/bundle-report.html',
        gzipSize: true,
        brotliSize: true,
      }),
    ].filter(Boolean),

    build: {
      outDir: 'dist',
      sourcemap: isProd ? 'hidden' : true,  // hidden = fără referință în fișier
      minify: isProd ? 'esbuild' : false,
      target: 'es2020',

      rollupOptions: {
        output: {
          // Content-hash în numele fișierelor pentru cache busting
          entryFileNames: 'assets/[name].[hash].js',
          chunkFileNames: 'assets/[name].[hash].js',
          assetFileNames: 'assets/[name].[hash].[ext]',

          // Manual chunk splitting pentru optimizare
          manualChunks: {
            'vendor-vue': ['vue', 'vue-router', 'pinia'],
            'vendor-ui': ['element-plus'],
          },
        },
      },

      // Alertă dacă un chunk depășește limita
      chunkSizeWarningLimit: 500, // KB
    },

    // Environment variables
    envPrefix: 'VITE_',
    envDir: '.',
  };
});
```

### 3.2 Webpack build configuration (Module Federation)

```typescript
// webpack.config.js - pentru MFE cu Module Federation
const { defineConfig } = require('@vue/cli-service');
const { ModuleFederationPlugin } = require('webpack').container;

const isProd = process.env.NODE_ENV === 'production';

module.exports = defineConfig({
  publicPath: process.env.VUE_APP_PUBLIC_PATH || '/',

  configureWebpack: {
    plugins: [
      new ModuleFederationPlugin({
        name: 'mfeProducts',
        filename: 'remoteEntry.js',
        exposes: {
          './ProductList': './src/components/ProductList.vue',
          './ProductDetail': './src/views/ProductDetail.vue',
        },
        shared: {
          vue: { singleton: true, eager: true },
          pinia: { singleton: true },
          'vue-router': { singleton: true },
        },
      }),
    ],

    optimization: {
      splitChunks: {
        chunks: 'all',
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: 'vendors',
            chunks: 'all',
          },
        },
      },
    },

    devtool: isProd ? 'source-map' : 'eval-source-map',
  },
});
```

### 3.3 Environment-specific webpack config

```typescript
// webpack.config.prod.js
const baseConfig = require('./webpack.config.base');
const { merge } = require('webpack-merge');
const CompressionPlugin = require('compression-webpack-plugin');
const BundleAnalyzerPlugin =
  require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = merge(baseConfig, {
  mode: 'production',

  plugins: [
    // Pre-compress assets cu gzip și brotli
    new CompressionPlugin({
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
    }),
    new CompressionPlugin({
      algorithm: 'brotliCompress',
      test: /\.(js|css|html|svg)$/,
      filename: '[path][base].br',
    }),

    // Bundle analysis report (salvat ca HTML în artifacts)
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
    }),
  ],
});
```

### 3.4 Build output comparison

| Caracteristică | Vite (esbuild + Rollup) | Webpack 5 |
|---------------|------------------------|-----------|
| **Viteză build dev** | ~300ms (esbuild) | ~5-15s |
| **Viteză build prod** | ~3-8s | ~15-45s |
| **HMR** | <50ms | ~1-3s |
| **Module Federation** | Plugin experimental (vite-plugin-federation) | Nativ (container plugin) |
| **Tree shaking** | Rollup (excelent) | Webpack (bun) |
| **Code splitting** | Rollup manualChunks | SplitChunksPlugin |
| **Source maps** | Rapid (esbuild) | Mai lent |
| **Config complexity** | Minimal | Complex |
| **Ecosistem plugins** | Rollup + Vite plugins | Webpack loaders + plugins |
| **SSR support** | Nativ | Necesită config suplimentar |

### 3.5 Pipeline step diferențe

```yaml
# Vite build step
- script: npx vite build --mode production
  displayName: 'Vite Build'

# Webpack build step
- script: npx vue-cli-service build --mode production
  displayName: 'Webpack Build'

# SAU cu webpack direct
- script: npx webpack --config webpack.config.prod.js
  displayName: 'Webpack Build (direct)'
```

### 3.6 Bundle analysis în pipeline

```yaml
# Salvează raportul de bundle ca artifact
- script: |
    npm run build
    # Verifică dimensiunea bundle-ului
    MAIN_SIZE=$(stat -f%z dist/assets/index.*.js 2>/dev/null || stat -c%s dist/assets/index.*.js)
    MAX_SIZE=512000  # 500KB limit
    if [ "$MAIN_SIZE" -gt "$MAX_SIZE" ]; then
      echo "##vso[task.logissue type=warning]Main bundle size ($MAIN_SIZE bytes) exceeds limit ($MAX_SIZE bytes)"
    fi
  displayName: 'Build & Check bundle size'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: 'dist/bundle-report.html'
    ArtifactName: 'bundle-report'
  displayName: 'Publish bundle report'
```

### Paralela cu Angular

Angular folosește **Angular CLI** (care intern folosește Webpack sau esbuild). De la Angular 17+, esbuild este default. Vite pentru Vue oferă o experiență similară cu esbuild din Angular, dar cu mai mult control. Webpack rămâne necesar pentru Module Federation atât în Angular cât și în Vue (până când Vite Federation devine stabil).

---

## 4. Azure Bicep Basics (Infrastructure as Code)

### 4.1 Ce este Azure Bicep

**Azure Bicep** este limbajul declarativ al Azure pentru Infrastructure as Code (IaC). Este un **Domain-Specific Language (DSL)** care se compilează în **ARM templates** (JSON). Avantajele față de ARM templates:
- Sintaxă mult mai curată și lizibilă
- Type safety și IntelliSense în VS Code
- Modularitate prin **modules**
- Referințe simplificate între resurse
- Nicio dependență de state file (spre deosebire de Terraform)

### 4.2 Infrastructure pentru Vue MFE

```bicep
// main.bicep - Infrastructure completă pentru Vue MFE platform
@description('Deployment environment')
@allowed([
  'dev'
  'staging'
  'production'
])
param environment string

@description('Azure region')
param location string = resourceGroup().location

@description('Lista de MFE-uri care trebuie deployate')
param mfeNames array = [
  'shell'
  'mfe-products'
  'mfe-checkout'
  'mfe-profile'
]

// ─── Storage Account pentru Static Hosting ───
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mfe${environment}store${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: environment == 'production' ? 'Standard_GRS' : 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: true
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
  }
}

// Activare static website hosting
resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  parent: storageAccount
  name: 'default'
  properties: {
    cors: {
      corsRules: [
        {
          allowedOrigins: environment == 'production'
            ? ['https://app.mycompany.com']
            : ['*']
          allowedMethods: ['GET', 'HEAD', 'OPTIONS']
          allowedHeaders: ['*']
          exposedHeaders: ['*']
          maxAgeInSeconds: 86400
        }
      ]
    }
  }
}

// Container per MFE
resource mfeContainers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = [
  for mfeName in mfeNames: {
    parent: blobService
    name: mfeName
    properties: {
      publicAccess: 'Blob'
    }
  }
]

// ─── CDN Profile ───
resource cdnProfile 'Microsoft.Cdn/profiles@2023-05-01' = {
  name: 'cdn-mfe-${environment}'
  location: 'global'
  sku: {
    name: environment == 'production' ? 'Standard_Verizon' : 'Standard_Microsoft'
  }
}

// ─── CDN Endpoint ───
resource cdnEndpoint 'Microsoft.Cdn/profiles/endpoints@2023-05-01' = {
  parent: cdnProfile
  name: 'mfe-${environment}'
  location: 'global'
  properties: {
    originHostHeader: replace(
      replace(storageAccount.properties.primaryEndpoints.blob, 'https://', ''),
      '/',
      ''
    )
    origins: [
      {
        name: 'storage-origin'
        hostName: replace(
          replace(storageAccount.properties.primaryEndpoints.blob, 'https://', ''),
          '/',
          ''
        )
        httpsPort: 443
        httpPort: 80
      }
    ]
    isHttpAllowed: false
    isHttpsAllowed: true
    deliveryPolicy: {
      rules: [
        {
          name: 'NoCacheRemoteEntry'
          order: 1
          conditions: [
            {
              name: 'UrlFileName'
              parameters: {
                typeName: 'DeliveryRuleUrlFilenameConditionParameters'
                operator: 'Equal'
                matchValues: ['remoteEntry.js']
              }
            }
          ]
          actions: [
            {
              name: 'CacheExpiration'
              parameters: {
                typeName: 'DeliveryRuleCacheExpirationActionParameters'
                cacheBehavior: 'BypassCache'
                cacheType: 'All'
              }
            }
          ]
        }
        {
          name: 'CacheStaticAssets'
          order: 2
          conditions: [
            {
              name: 'UrlFileExtension'
              parameters: {
                typeName: 'DeliveryRuleUrlFileExtensionMatchConditionParameters'
                operator: 'Equal'
                matchValues: ['js', 'css', 'png', 'jpg', 'svg', 'woff2']
              }
            }
          ]
          actions: [
            {
              name: 'CacheExpiration'
              parameters: {
                typeName: 'DeliveryRuleCacheExpirationActionParameters'
                cacheBehavior: 'Override'
                cacheType: 'All'
                cacheDuration: '365.00:00:00'
              }
            }
          ]
        }
      ]
    }
  }
}

// ─── Application Insights pentru monitoring ───
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'ai-mfe-${environment}'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    RetentionInDays: environment == 'production' ? 90 : 30
  }
}

// ─── Outputs ───
output storageAccountName string = storageAccount.name
output cdnEndpointUrl string = 'https://${cdnEndpoint.properties.hostName}'
output appInsightsKey string = appInsights.properties.InstrumentationKey
output appInsightsConnectionString string = appInsights.properties.ConnectionString
```

### 4.3 Deployment Bicep din pipeline

```yaml
# Deploy infrastructure cu Bicep
- stage: Infrastructure
  displayName: 'Deploy Infrastructure'
  jobs:
    - job: DeployBicep
      steps:
        - task: AzureCLI@2
          inputs:
            azureSubscription: 'production-subscription'
            scriptType: 'bash'
            scriptLocation: 'inlineScript'
            inlineScript: |
              az deployment group create \
                --resource-group rg-mfe-$(environment) \
                --template-file infra/main.bicep \
                --parameters environment=$(environment) \
                --parameters location=westeurope
          displayName: 'Deploy Bicep template'
```

### 4.4 Bicep Modules - structură modulară

```bicep
// infra/modules/storage.bicep
@description('Environment name')
param environment string
param location string

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  // ... doar storage logic
}

output name string = storageAccount.name
output endpoint string = storageAccount.properties.primaryEndpoints.blob
```

```bicep
// infra/main.bicep - folosește modules
module storage 'modules/storage.bicep' = {
  name: 'storage-deployment'
  params: {
    environment: environment
    location: location
  }
}

module cdn 'modules/cdn.bicep' = {
  name: 'cdn-deployment'
  params: {
    environment: environment
    storageEndpoint: storage.outputs.endpoint
  }
}
```

### Paralela cu Angular

Bicep este **agnostic la framework** - aceeași infrastructură funcționează pentru Angular sau Vue. Diferența apare doar la:
- **CDN rules** - Angular are `ngsw-worker.js` (Service Worker) care necesită reguli specifice de caching, Vue nu
- **Output directory** - Angular produce `dist/project-name/`, Vue produce `dist/`
- **Fallback routing** - ambele necesită `index.html` ca fallback pentru SPA routing

---

## 5. Container Deployment (Docker + Azure)

### 5.1 Dockerfile pentru Vue app (Multi-stage)

```dockerfile
# ─── Stage 1: Build ───
FROM node:20-alpine AS build

# Argumente de build pentru environment
ARG VITE_API_URL
ARG VITE_APP_ENV=production

WORKDIR /app

# Copiază doar package files pentru cache layer
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

# Copiază sursa și build
COPY . .
RUN npm run build

# ─── Stage 2: Serve cu nginx ───
FROM nginx:1.25-alpine AS production

# Security: remove default config, run as non-root
RUN rm -rf /etc/nginx/conf.d/*

# Copiază artefactele de build
COPY --from=build /app/dist /usr/share/nginx/html

# Copiază configurația nginx
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:80/health || exit 1

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### 5.2 Dockerfile optimizat pentru MFE

```dockerfile
# Dockerfile.mfe - pentru Module Federation remote
FROM node:20-alpine AS build
ARG MFE_NAME
ARG PUBLIC_PATH=/

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Build cu public path corect pentru MFE
ENV PUBLIC_PATH=${PUBLIC_PATH}
RUN npm run build

# ─── Nginx serve ───
FROM nginx:1.25-alpine

ARG MFE_NAME
ENV MFE_NAME=${MFE_NAME}

COPY --from=build /app/dist /usr/share/nginx/html
COPY docker/nginx-mfe.conf /etc/nginx/templates/default.conf.template

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 5.3 nginx.conf pentru SPA

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript
               image/svg+xml;
    gzip_min_length 1000;
    gzip_vary on;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;

    # SPA routing - toate rutele duc la index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets agresiv (au hash în nume)
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # NU cache-ui index.html și remoteEntry.js
    location ~* (index\.html|remoteEntry\.js)$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
        add_header Expires "0";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}
```

### 5.4 De ce remoteEntry.js nu trebuie cache-uit

**remoteEntry.js** este fișierul manifest al unui Module Federation remote. Conține:
- Mapping-ul între module expuse și chunk-urile lor
- Versiunile librăriilor shared
- URL-urile chunk-urilor (care au hash)

Dacă cache-uiești remoteEntry.js, host-ul va încărca o versiune veche și va referi chunk-uri care **nu mai există** (au fost șterse la deploy). Rezultat: erori 404 și aplicație broken.

```
Timeline:
1. Deploy V1 → remoteEntry.js pointează la chunk-abc123.js
2. Deploy V2 → remoteEntry.js pointează la chunk-def456.js (abc123 șters)
3. User cu remoteEntry.js cached → cere chunk-abc123.js → 404!
```

### 5.5 Docker build în pipeline

```yaml
# Docker build & push to Azure Container Registry
stages:
  - stage: DockerBuild
    jobs:
      - job: BuildAndPush
        steps:
          - task: Docker@2
            inputs:
              containerRegistry: 'acr-connection'
              repository: 'mfe/$(mfeName)'
              command: 'buildAndPush'
              Dockerfile: 'Dockerfile'
              buildContext: '.'
              tags: |
                $(Build.BuildId)
                latest
            displayName: 'Build & Push Docker image'
```

### 5.6 Azure Container Apps deployment

```yaml
# Deploy la Azure Container Apps
- task: AzureCLI@2
  inputs:
    azureSubscription: 'production-subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az containerapp update \
        --name mfe-products \
        --resource-group rg-mfe \
        --image myacr.azurecr.io/mfe/products:$(Build.BuildId) \
        --set-env-vars \
          VITE_API_URL=https://api.mycompany.com \
          VITE_APP_ENV=production
  displayName: 'Deploy to Container Apps'
```

### 5.7 Azure Container Apps vs AKS vs App Service

| Caracteristică | Container Apps | AKS | App Service |
|---------------|---------------|-----|-------------|
| **Complexitate** | Scăzută | Ridicată | Scăzută |
| **Scaling** | Automat (KEDA) | Manual/HPA | Automat |
| **Cost** | Pay-per-use | Noduri fixe | Plan fixe |
| **Kubernetes** | Abstracție peste K8s | K8s complet | Fără K8s |
| **Use case MFE** | Ideal | Overkill | Simplu dar limitat |
| **Networking** | Ingress built-in | Ingress controller | Built-in |
| **Customizare** | Moderată | Totală | Limitată |

**Recomandare pentru MFE:** Azure Container Apps sau pur și simplu Azure Blob Storage + CDN (static hosting) dacă nu ai nevoie de SSR.

### Paralela cu Angular

Dockerfile-ul este aproape identic. Singura diferență:
- **Angular** - `ng build` produce output în `dist/project-name/`, deci COPY path diferit
- **Vue** - `vite build` produce output direct în `dist/`
- **Angular SSR** - necesită Node.js server (nu doar nginx)
- **Vue SSR (Nuxt)** - similar, necesită Node.js server

---

## 6. Environment Management

### 6.1 .env files per environment

```bash
# .env - defaults (committed to git)
VITE_APP_TITLE=My MFE Platform
VITE_API_TIMEOUT=30000

# .env.development - dev local (committed)
VITE_API_URL=http://localhost:3000/api
VITE_APP_ENV=development
VITE_ENABLE_DEVTOOLS=true

# .env.staging - staging (committed)
VITE_API_URL=https://api-staging.mycompany.com
VITE_APP_ENV=staging
VITE_ENABLE_DEVTOOLS=true

# .env.production - production (committed)
VITE_API_URL=https://api.mycompany.com
VITE_APP_ENV=production
VITE_ENABLE_DEVTOOLS=false

# .env.local - override-uri locale (in .gitignore!)
VITE_API_URL=http://localhost:5000/api
```

### 6.2 Accesarea env vars în Vue

```typescript
// Vite expune env vars cu prefixul VITE_ prin import.meta.env
console.log(import.meta.env.VITE_API_URL);
console.log(import.meta.env.VITE_APP_ENV);
console.log(import.meta.env.MODE);       // 'development' | 'production'
console.log(import.meta.env.DEV);         // true/false
console.log(import.meta.env.PROD);        // true/false

// Type-safe env vars
// env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_ENV: 'development' | 'staging' | 'production';
  readonly VITE_APP_TITLE: string;
  readonly VITE_ENABLE_DEVTOOLS: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### 6.3 Runtime configuration (recomandat pentru MFE)

```typescript
// config/runtime-config.ts
export interface RuntimeConfig {
  apiUrl: string;
  appEnv: string;
  mfeRemotes: Record<string, string>;
  featureFlags: Record<string, boolean>;
  appInsightsKey: string;
}

let config: RuntimeConfig | null = null;

export async function loadRuntimeConfig(): Promise<RuntimeConfig> {
  if (config) return config;

  const response = await fetch('/config/runtime-config.json');
  if (!response.ok) {
    throw new Error(`Failed to load runtime config: ${response.status}`);
  }

  config = await response.json();
  return config!;
}

export function getRuntimeConfig(): RuntimeConfig {
  if (!config) {
    throw new Error('Runtime config not loaded. Call loadRuntimeConfig() first.');
  }
  return config;
}
```

```json
// public/config/runtime-config.json (înlocuit la deployment)
{
  "apiUrl": "https://api.mycompany.com",
  "appEnv": "production",
  "mfeRemotes": {
    "products": "https://mfe-prod.azureedge.net/mfe-products/remoteEntry.js",
    "checkout": "https://mfe-prod.azureedge.net/mfe-checkout/remoteEntry.js"
  },
  "featureFlags": {
    "newCheckout": true,
    "darkMode": false
  },
  "appInsightsKey": "xxx-xxx-xxx"
}
```

```yaml
# Pipeline step: replace runtime config per environment
- script: |
    cat > dist/config/runtime-config.json << EOF
    {
      "apiUrl": "$(API_URL)",
      "appEnv": "$(ENVIRONMENT)",
      "mfeRemotes": {
        "products": "$(CDN_URL)/mfe-products/remoteEntry.js",
        "checkout": "$(CDN_URL)/mfe-checkout/remoteEntry.js"
      },
      "featureFlags": $(FEATURE_FLAGS_JSON),
      "appInsightsKey": "$(APP_INSIGHTS_KEY)"
    }
    EOF
  displayName: 'Inject runtime configuration'
```

### 6.4 Feature flags per environment

```typescript
// composables/useFeatureFlags.ts
import { ref, readonly } from 'vue';
import { getRuntimeConfig } from '@/config/runtime-config';

const flags = ref<Record<string, boolean>>({});

export function useFeatureFlags() {
  function loadFlags(): void {
    const config = getRuntimeConfig();
    flags.value = config.featureFlags;
  }

  function isEnabled(flagName: string): boolean {
    return flags.value[flagName] ?? false;
  }

  return {
    flags: readonly(flags),
    loadFlags,
    isEnabled,
  };
}
```

```vue
<!-- Utilizare în componente -->
<template>
  <NewCheckoutFlow v-if="isEnabled('newCheckout')" />
  <LegacyCheckout v-else />
</template>

<script setup lang="ts">
import { useFeatureFlags } from '@/composables/useFeatureFlags';
const { isEnabled } = useFeatureFlags();
</script>
```

### 6.5 Secrets management cu Azure Key Vault

```yaml
# Pipeline: citește secrets din Key Vault
variables:
  - group: 'mfe-secrets'  # Variable group linked la Key Vault

steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'production-subscription'
      KeyVaultName: 'kv-mfe-production'
      SecretsFilter: 'ApiKey,AppInsightsKey,CdnToken'
      RunAsPreJob: true
    displayName: 'Fetch secrets from Key Vault'

  - script: |
      # Secrets sunt disponibile ca variabile de environment
      echo "Using App Insights Key: $(AppInsightsKey)"
      # ATENȚIE: nu loga secretele! Azure le maskează automat în logs
    displayName: 'Use secrets in build'
```

**Regulă importantă:** Secretele nu trebuie NICIODATĂ puse în bundle-ul frontend. Ele se folosesc doar în pipeline (ex: pentru deployment) sau se injectează prin BFF (Backend for Frontend).

### Paralela cu Angular

| Aspect | Angular | Vue (Vite) |
|--------|---------|------------|
| Env files | `environment.ts` / `environment.prod.ts` | `.env` / `.env.production` |
| Accesare | `environment.apiUrl` (import direct) | `import.meta.env.VITE_API_URL` |
| Prefix obligatoriu | Nu | Da: `VITE_` |
| Build-time vs Runtime | Build-time (fileReplacements) | Build-time (.env) sau Runtime (fetch) |
| Typing | Fișier TypeScript nativ | `env.d.ts` (augmentation) |
| Runtime config | Posibil dar neconvențional | Pattern comun și recomandat |

---

## 7. Caching și Optimizări Pipeline

### 7.1 npm cache

```yaml
# Cache npm packages între rulări
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    restoreKeys: |
      npm | "$(Agent.OS)"
    path: $(Pipeline.Workspace)/.npm
  displayName: 'Cache npm'

# Folosește cache-ul la install
- script: npm ci --cache $(Pipeline.Workspace)/.npm
  displayName: 'Install with cache'
```

**Cum funcționează:**
1. Prima rulare: `npm ci` descarcă tot, salvează în cache
2. A doua rulare: dacă `package-lock.json` nu s-a schimbat, restaurează din cache
3. `npm ci` tot va recrea `node_modules`, dar pachetele vin din cache local (nu de pe registry)

### 7.2 Docker layer caching

```yaml
# Docker layer caching în Azure Pipelines
- task: Docker@2
  inputs:
    command: 'build'
    Dockerfile: 'Dockerfile'
    arguments: |
      --cache-from myacr.azurecr.io/mfe/products:cache
      --build-arg BUILDKIT_INLINE_CACHE=1
  displayName: 'Build with layer cache'

- task: Docker@2
  inputs:
    command: 'push'
    tags: |
      $(Build.BuildId)
      cache
  displayName: 'Push image + cache tag'
```

**Dockerfile optimizat pentru cache:**

```dockerfile
# Copiază package files SEPARAT de sursă
COPY package.json package-lock.json ./
RUN npm ci
# ^^^ Acest layer se cache-uiește dacă package files nu se schimbă

COPY . .
RUN npm run build
# ^^^ Acest layer se reconstruiește la orice schimbare de cod
```

### 7.3 Pipeline caching strategies avansate

```yaml
# Cache pentru multiple locații
variables:
  NPM_CACHE: $(Pipeline.Workspace)/.npm
  VITEST_CACHE: $(Pipeline.Workspace)/.vitest
  ESLINT_CACHE: $(Pipeline.Workspace)/.eslintcache

steps:
  # npm cache
  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      path: $(NPM_CACHE)
    displayName: 'Cache npm'

  # ESLint cache (persistă rezultatele lint-ului)
  - task: Cache@2
    inputs:
      key: 'eslint | "$(Agent.OS)" | $(Build.SourceVersion)'
      restoreKeys: |
        eslint | "$(Agent.OS)"
      path: $(ESLINT_CACHE)
    displayName: 'Cache ESLint'

  - script: npm run lint -- --cache --cache-location $(ESLINT_CACHE)
    displayName: 'Lint with cache'
```

### 7.4 Parallel builds

```yaml
# Rulează lint, type-check și tests în paralel
jobs:
  - job: Lint
    steps:
      - script: npm ci && npm run lint
        displayName: 'Lint'

  - job: TypeCheck
    steps:
      - script: npm ci && npx vue-tsc --noEmit
        displayName: 'Type check'

  - job: UnitTests
    steps:
      - script: npm ci && npm run test:unit
        displayName: 'Unit tests'

  - job: Build
    dependsOn:
      - Lint
      - TypeCheck
      - UnitTests
    steps:
      - script: npm ci && npm run build
        displayName: 'Build'
```

### 7.5 Optimizare: skip steps when unchanged

```yaml
# Condiționare pe baza fișierelor modificate
steps:
  - script: |
      CHANGED=$(git diff --name-only HEAD~1 HEAD)
      if echo "$CHANGED" | grep -qE '\.(ts|vue|tsx)$'; then
        echo "##vso[task.setvariable variable=hasCodeChanges]true"
      fi
      if echo "$CHANGED" | grep -q 'package-lock.json'; then
        echo "##vso[task.setvariable variable=hasDepsChanges]true"
      fi
    displayName: 'Detect change types'

  - script: npm run test:unit
    condition: eq(variables.hasCodeChanges, 'true')
    displayName: 'Tests (only if code changed)'
```

### 7.6 Pipeline metrics și trending

```yaml
# Track build times și artifact sizes
- script: |
    BUILD_START=$(date +%s)
    npm run build
    BUILD_END=$(date +%s)
    BUILD_DURATION=$((BUILD_END - BUILD_START))

    # Raportează ca metric
    echo "##vso[task.setvariable variable=buildDuration]$BUILD_DURATION"

    # Dimensiunea totală a dist/
    DIST_SIZE=$(du -sb dist | cut -f1)
    echo "##vso[task.setvariable variable=distSize]$DIST_SIZE"

    echo "Build duration: ${BUILD_DURATION}s, Dist size: ${DIST_SIZE} bytes"
  displayName: 'Build with metrics'
```

---

## 8. Monitoring și Alerting

### 8.1 Application Insights pentru Vue

```typescript
// plugins/app-insights.ts
import { type App } from 'vue';
import { ApplicationInsights } from '@microsoft/applicationinsights-web';
import { getRuntimeConfig } from '@/config/runtime-config';

let appInsights: ApplicationInsights | null = null;

export function initAppInsights(): ApplicationInsights {
  const config = getRuntimeConfig();

  appInsights = new ApplicationInsights({
    config: {
      connectionString: config.appInsightsConnectionString,
      enableAutoRouteTracking: true,   // Track SPA navigations
      enableCorsCorrelation: true,      // Correlation cu API calls
      enableRequestHeaderTracking: true,
      enableResponseHeaderTracking: true,
      disableFetchTracking: false,
    },
  });

  appInsights.loadAppInsights();

  // Custom properties pe fiecare telemetrie
  appInsights.addTelemetryInitializer((item) => {
    if (item.data) {
      item.data['mfeName'] = config.mfeName || 'shell';
      item.data['appVersion'] = config.appVersion || 'unknown';
      item.data['environment'] = config.appEnv;
    }
  });

  return appInsights;
}

export function getAppInsights(): ApplicationInsights {
  if (!appInsights) {
    throw new Error('App Insights not initialized');
  }
  return appInsights;
}

// Vue plugin
export const appInsightsPlugin = {
  install(app: App): void {
    const ai = initAppInsights();

    // Global error handler
    app.config.errorHandler = (err, instance, info) => {
      ai.trackException({
        exception: err as Error,
        properties: {
          componentName: instance?.$options?.name || 'Unknown',
          lifecycleHook: info,
        },
      });
      console.error('Vue Error:', err);
    };

    // Provide pentru composables
    app.provide('appInsights', ai);
  },
};
```

```typescript
// main.ts
import { createApp } from 'vue';
import { appInsightsPlugin } from '@/plugins/app-insights';

const app = createApp(App);
app.use(appInsightsPlugin);
app.mount('#app');
```

### 8.2 Web Vitals tracking

```typescript
// composables/useWebVitals.ts
import { onMounted } from 'vue';
import { onCLS, onFID, onLCP, onFCP, onTTFB, type Metric } from 'web-vitals';
import { getAppInsights } from '@/plugins/app-insights';

export function useWebVitals(): void {
  onMounted(() => {
    const ai = getAppInsights();

    function reportMetric(metric: Metric): void {
      ai.trackMetric({
        name: `WebVital_${metric.name}`,
        average: metric.value,
        properties: {
          id: metric.id,
          navigationType: metric.navigationType || 'unknown',
          rating: metric.rating,  // 'good' | 'needs-improvement' | 'poor'
        },
      });
    }

    // Core Web Vitals
    onCLS(reportMetric);   // Cumulative Layout Shift
    onFID(reportMetric);   // First Input Delay
    onLCP(reportMetric);   // Largest Contentful Paint

    // Supplementary metrics
    onFCP(reportMetric);   // First Contentful Paint
    onTTFB(reportMetric);  // Time to First Byte
  });
}
```

### 8.3 Error tracking cu Sentry (alternativă / complementar)

```typescript
// plugins/sentry.ts
import * as Sentry from '@sentry/vue';
import { type App } from 'vue';
import { type Router } from 'vue-router';

export function initSentry(app: App, router: Router): void {
  Sentry.init({
    app,
    dsn: import.meta.env.VITE_SENTRY_DSN,
    environment: import.meta.env.VITE_APP_ENV,
    release: import.meta.env.VITE_APP_VERSION,

    integrations: [
      Sentry.browserTracingIntegration({ router }),
      Sentry.replayIntegration({
        maskAllText: true,
        blockAllMedia: true,
      }),
    ],

    // Performance monitoring
    tracesSampleRate: import.meta.env.PROD ? 0.1 : 1.0,

    // Session replay (doar pentru erori)
    replaysSessionSampleRate: 0,
    replaysOnErrorSampleRate: 1.0,

    // Filtrează erori irelevante
    beforeSend(event) {
      if (event.exception?.values?.[0]?.type === 'ChunkLoadError') {
        // Chunk load error = user pe versiune veche, forțează reload
        window.location.reload();
        return null;
      }
      return event;
    },
  });
}
```

### 8.4 MFE Health Checks

```typescript
// composables/useMfeHealthCheck.ts
import { ref, type Ref } from 'vue';
import { getAppInsights } from '@/plugins/app-insights';

interface MfeHealth {
  name: string;
  status: 'healthy' | 'degraded' | 'unhealthy';
  responseTime: number;
  lastChecked: Date;
  error?: string;
}

export function useMfeHealthCheck(remotes: Record<string, string>) {
  const health: Ref<Map<string, MfeHealth>> = ref(new Map());

  async function checkMfe(name: string, url: string): Promise<MfeHealth> {
    const start = performance.now();
    try {
      const response = await fetch(url, {
        method: 'HEAD',
        cache: 'no-store',
      });
      const responseTime = performance.now() - start;

      return {
        name,
        status: response.ok ? 'healthy' : 'degraded',
        responseTime,
        lastChecked: new Date(),
      };
    } catch (error) {
      return {
        name,
        status: 'unhealthy',
        responseTime: performance.now() - start,
        lastChecked: new Date(),
        error: (error as Error).message,
      };
    }
  }

  async function checkAllMfes(): Promise<void> {
    const ai = getAppInsights();
    const results = await Promise.allSettled(
      Object.entries(remotes).map(([name, url]) => checkMfe(name, url))
    );

    results.forEach((result) => {
      if (result.status === 'fulfilled') {
        const mfeHealth = result.value;
        health.value.set(mfeHealth.name, mfeHealth);

        // Track în Application Insights
        ai.trackMetric({
          name: `MFE_Health_${mfeHealth.name}`,
          average: mfeHealth.responseTime,
          properties: {
            status: mfeHealth.status,
            error: mfeHealth.error || '',
          },
        });

        if (mfeHealth.status === 'unhealthy') {
          ai.trackException({
            exception: new Error(`MFE ${mfeHealth.name} is unhealthy`),
            properties: { url: remotes[mfeHealth.name] },
          });
        }
      }
    });
  }

  return { health, checkAllMfes, checkMfe };
}
```

### 8.5 Azure Monitor Alerts (Bicep)

```bicep
// alerts.bicep - Alerte pentru MFE platform
param appInsightsId string
param actionGroupId string

// Alert: Error rate peste 5%
resource errorRateAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'mfe-error-rate-high'
  location: 'global'
  properties: {
    description: 'MFE error rate is above 5%'
    severity: 2
    enabled: true
    scopes: [appInsightsId]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'ErrorRate'
          metricName: 'exceptions/count'
          operator: 'GreaterThan'
          threshold: 50
          timeAggregation: 'Count'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroupId
      }
    ]
  }
}

// Alert: Response time peste 3 secunde
resource responseTimeAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'mfe-response-time-slow'
  location: 'global'
  properties: {
    description: 'MFE response time is above 3 seconds'
    severity: 3
    enabled: true
    scopes: [appInsightsId]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'ResponseTime'
          metricName: 'requests/duration'
          operator: 'GreaterThan'
          threshold: 3000
          timeAggregation: 'Average'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroupId
      }
    ]
  }
}
```

### Paralela cu Angular

Monitoring este identic conceptual. Diferențele:
- **Angular** - `@sentry/angular-ivy` sau `@sentry/angular`, cu `ErrorHandler` personalizat
- **Vue** - `@sentry/vue`, cu `app.config.errorHandler`
- **Zone.js** - Angular folosește Zone.js care interceptează automat operațiile async. Vue nu are acest concept, deci tracking-ul trebuie explicit
- **Router tracking** - ambele suportă tracking automat de navigare prin integrările dedicate

---

## 9. Paralela: Pipeline Angular vs Vue

### 9.1 Comparație completă

| Aspect | Angular Pipeline | Vue Pipeline |
|--------|-----------------|--------------|
| **Build command** | `ng build --configuration=prod` | `vite build --mode production` |
| **Build tool** | Angular CLI (Webpack/esbuild) | Vite (esbuild + Rollup) / Webpack |
| **Type check** | Inclus în `ng build` | Separat: `vue-tsc --noEmit` |
| **Test runner** | Karma (legacy) / Jest / Vitest | Vitest (recomandat) / Jest |
| **E2E runner** | Protractor (legacy) / Cypress | Cypress / Playwright |
| **Lint** | `ng lint` (ESLint) | `eslint .` |
| **Output dir** | `dist/project-name/` | `dist/` |
| **Env config** | `environment.ts` (fileReplacements) | `.env` files / `import.meta.env` |
| **SSR build** | `ng build --ssr` | `nuxt build` |
| **Build time (med)** | ~15-45s | ~3-8s (Vite) |
| **HMR speed** | ~1-3s | <50ms (Vite) |
| **Bundle analyzer** | `ng build --stats-json` + webpack-bundle-analyzer | `rollup-plugin-visualizer` |
| **Monorepo tool** | Nx (popular) | Nx / Turborepo / pnpm workspaces |
| **Module Fed** | `@angular-architects/module-federation` | `@originjs/vite-plugin-federation` / webpack native |
| **Service Worker** | `@angular/service-worker` | `vite-plugin-pwa` |
| **Config file** | `angular.json` | `vite.config.ts` / `webpack.config.js` |

### 9.2 Pipeline structure comparison

```yaml
# ─── Angular Pipeline ───
stages:
  - stage: Build
    jobs:
      - job: BuildAngular
        steps:
          - script: npm ci
          - script: ng lint
          # Type check inclus în build
          - script: ng test --watch=false --code-coverage
          - script: ng build --configuration=production
          # Output: dist/my-app/

# ─── Vue Pipeline ───
stages:
  - stage: Build
    jobs:
      - job: BuildVue
        steps:
          - script: npm ci
          - script: eslint .
          - script: vue-tsc --noEmit          # Type check separat!
          - script: vitest run --coverage
          - script: vite build --mode production
          # Output: dist/
```

### 9.3 Diferențe cheie explicate

**Type checking:**
- Angular CLI compilează TypeScript ca parte din build. Dacă ai erori de tip, build-ul eșuează
- Vue necesită `vue-tsc` separat. `vite build` nu verifică tipuri (esbuild ignoră tipurile). Este o greșeală frecventă să nu rulezi `vue-tsc` în pipeline

**Environment variables:**
- Angular folosește `fileReplacements` din `angular.json` care înlocuiesc fișiere la build time
- Vue folosește `.env` files citite de Vite. Mai flexibil dar necesită prefix `VITE_`

**Module Federation:**
- Angular are soluția matură `@angular-architects/module-federation` cu schematics
- Vue are mai multe opțiuni dar niciuna la fel de matură. Plugin-ul Vite este experimental

**Build speed:**
- Vue cu Vite este semnificativ mai rapid datorită esbuild (Go-based bundler)
- Angular 17+ cu esbuild a redus mult gap-ul, dar Vue rămâne mai rapid

### 9.4 Ce se transferă direct din Angular DevOps

**Transferabil 1:1:**
- Structura pipeline YAML (stages, jobs, steps)
- Azure DevOps Environments cu approval gates
- Variable groups și secret management
- Caching strategies (npm, Docker layers)
- CDN purge și deployment strategies
- Bicep / IaC (agnostic la framework)
- Docker multi-stage builds (structură identică)
- Monitoring setup (Application Insights, Sentry)

**Necesită adaptare:**
- Build commands și configurare
- Type checking (separat în Vue)
- Test runner configuration
- Environment variable access pattern
- Module Federation setup

---

## 10. Întrebări de interviu

### Î1: Cum structurezi pipeline-ul pentru multiple MFE-uri într-un monorepo?

**Răspuns:**
Folosesc o abordare bazată pe **YAML templates** reutilizabile. Creez un template parametrizat care primește `mfeName` și `mfePath`, iar pipeline-ul principal îl instanțiază per MFE. Fiecare MFE are propriul stage de build și deploy. Folosesc **path triggers** pentru a rula pipeline-ul doar când fișierele MFE-ului respectiv se schimbă. Pentru shared packages, adaug `packages/shared/**` la trigger-ul fiecărui MFE. Ca alternativă, pot folosi **matrix strategy** pentru build-uri paralele. Cheia este independența: fiecare MFE trebuie să poată fi build-uit, testat și deployat fără a afecta celelalte.

### Î2: Cum faci deployment independent per MFE fără a afecta celelalte?

**Răspuns:**
Fiecare MFE are propriul **container sau folder** pe Azure Blob Storage, propriul pipeline și propriul trigger. La deployment, upload doar artefactele MFE-ului modificat și purg CDN-ul doar pentru path-ul acelui MFE (`/mfe-products/*` nu `/*`). **remoteEntry.js** este fișierul cheie - host-ul îl încarcă la runtime pentru a descoperi chunk-urile MFE-ului. Dacă un MFE nou eșuează, celelalte continuă să funcționeze deoarece au propriile remoteEntry.js. Rollback-ul înseamnă re-deployarea versiunii anterioare a unui singur MFE, nu a întregii platforme. Acest pattern este similar cu microserviciile backend.

### Î3: Cum gestionezi environment-urile (dev, staging, prod) pentru o platformă MFE?

**Răspuns:**
Folosesc o combinație de **build-time** și **runtime** configuration. La build time, `.env.[mode]` files definesc setările per environment în Vite. Dar pentru MFE-uri, prefer **runtime configuration** - un fișier `runtime-config.json` în `/public/config/` care se înlocuiește la deployment prin pipeline. Avantajul runtime config: aceeași imagine Docker funcționează în orice environment. Azure DevOps **Variable Groups** linkate la **Key Vault** gestionează secretele. Feature flags sunt parte din runtime config, controlate per environment. Fiecare environment are propriul **Azure DevOps Environment** cu approval gates pentru producție.

### Î4: Ce este Azure Bicep și cum îl integrezi în pipeline-ul MFE?

**Răspuns:**
**Azure Bicep** este limbajul declarativ IaC al Azure, compilat la ARM templates. Îl prefer față de ARM JSON pentru sintaxa curată și type safety. Pentru MFE, definesc în Bicep: **Storage Account** cu static website hosting, **CDN Profile** cu endpoint și cache rules (remoteEntry.js bypass cache), **Application Insights** pentru monitoring, și **Container Apps** dacă folosesc SSR. Bicep-ul rulează într-un **stage separat** al pipeline-ului, înainte de deployment-ul aplicației. Folosesc **modules** pentru reutilizabilitate și **parameters** pentru diferențiere per environment. Starea infrastructurii este gestionată de Azure (nu necesită state file ca Terraform).

### Î5: De ce remoteEntry.js nu trebuie cache-uit și cum configurezi asta?

**Răspuns:**
**remoteEntry.js** este manifestul Module Federation - conține mapping-ul între modulele expuse și chunk-urile fizice (care au hash). Dacă browser-ul sau CDN-ul cache-uiește remoteEntry.js, după un deployment nou, host-ul va cere chunk-uri vechi care au fost șterse, rezultând **erori 404**. Configurez no-cache la trei nivele: **nginx** (`location ~* remoteEntry\.js$ { expires -1; }`), **CDN rules** (Bicep delivery policy cu `BypassCache` pentru filename `remoteEntry.js`), și **response headers** (`Cache-Control: no-cache, no-store, must-revalidate`). Chunk-urile cu hash pot fi cache-uite agresiv deoarece hash-ul se schimbă la orice modificare.

### Î6: Cum faci rollback pentru un MFE care a introdus un bug în producție?

**Răspuns:**
Am mai multe strategii. Prima și cea mai rapidă: **re-run** pipeline-ul pe commit-ul anterior - se rebuild-uiește și se deployază versiunea precedentă. A doua: păstrez **artifact-urile** ultimelor N deployment-uri și am un pipeline de rollback care re-deployază un artifact existent fără rebuild. A treia, mai avansată: **blue-green deployment** unde am două sloturi (blue/green) și switch-ul între ele este instant (schimb doar CDN origin). La Container Apps, folosesc **revision management** - pot redirecționa traficul la o revizie anterioară instantaneu. Cheia este că rollback-ul afectează doar un MFE, celelalte rămân neafectate.

### Î7: Cum monitorizezi performanța MFE-urilor în producție?

**Răspuns:**
Implementez monitoring la trei nivele. **Application Insights** colectează telemetrie automată (page views, API calls, exceptions) cu custom properties per MFE (mfeName, version). **Web Vitals** (CLS, LCP, FID) se raportează prin composable-ul `useWebVitals` și se trackează ca custom metrics. **MFE Health Checks** - shell-ul verifică periodic disponibilitatea fiecărui remoteEntry.js și raportează status-ul. Creez **Azure Monitor Alerts** prin Bicep pentru: error rate peste 5%, response time peste 3s, și MFE unreachable. Dashboard-ul agregă metrici per MFE, permitând identificarea rapidă a MFE-ului problematic. Sentry complementează cu session replay pentru debugging.

### Î8: Cum implementezi blue-green deployment pentru MFE-uri?

**Răspuns:**
Blue-green pentru MFE-uri funcționează la nivel de **CDN origin swap**. Am două seturi de storage containers (blue și green). La deployment, upload noile artefacte în slotul inactiv, verific prin smoke tests, apoi schimb CDN origin-ul de la blue la green (sau invers). Switch-ul este instantaneu din perspectiva userilor. Pentru Container Apps, folosesc **traffic splitting** - pot direcționa 10% trafic la noua revizie, monitorizez, apoi cresc la 100%. Fiecare MFE poate avea propriul blue-green cycle independent. Complicația vine la shared dependencies - trebuie asigurat backward compatibility între versiuni.

### Î9: Cum gestionezi secrets în pipeline fără a le expune în bundle-ul frontend?

**Răspuns:**
Regula de aur: **niciun secret nu ajunge în bundle-ul frontend**. Secretele se stochează în **Azure Key Vault** și se accesează prin **Variable Groups** linkate. În pipeline, task-ul `AzureKeyVault@2` extrage secretele ca variabile de mediu masked (Azure le ascunde automat în logs). Le folosesc doar pentru: deployment credentials, CDN purge tokens, storage account keys, Application Insights connection strings (care sunt semi-publice dar le tratez ca secrets). API keys se accesează prin **BFF (Backend for Frontend)**, niciodată direct din frontend. `.env.local` cu secrets reale este în `.gitignore`. Pipeline variables marked as secret nu apar în logs.

### Î10: Care sunt diferențele principale între pipeline-ul Angular și Vue din perspectivă DevOps?

**Răspuns:**
Structura pipeline este **90% identică** - stages, deployment strategies, caching, Bicep, Docker, monitoring sunt framework-agnostic. Diferențele principale: **Type checking** - Angular include verificarea TS în `ng build`, Vue necesită `vue-tsc --noEmit` separat (greșeală frecventă: omiterea din pipeline). **Build speed** - Vue cu Vite este 3-5x mai rapid decât Angular cu Webpack. **Environment vars** - Angular folosește `fileReplacements` din `angular.json`, Vue folosește `.env` files cu prefix `VITE_`. **Output** - Angular: `dist/project-name/`, Vue: `dist/`. **Module Federation** - Angular are soluția `@angular-architects/module-federation` mai matură, Vue are opțiuni multiple dar mai puțin stabile. Ca arhitect, tranziția este lină deoarece conceptele DevOps sunt aceleași.

### Î11: Cum asiguri zero-downtime deployment pentru o platformă MFE?

**Răspuns:**
Zero-downtime se obține prin mai multe tehnici combinate. **Atomic uploads** - upload toate fișierele înainte de a actualiza remoteEntry.js (inversul cauzează 404). **CDN propagation** - după upload, aștept ca CDN-ul să propageze (sau purg selectiv). **Versioned paths** - fiecare deployment are un version prefix (`/v123/mfe-products/`) iar host-ul referă versiunea curentă. **Graceful degradation** - host-ul are fallback dacă un MFE nu răspunde (error boundary cu retry). **Rolling updates** în Container Apps cu `minReadySeconds` și health probes. **Backward compatibility** - MFE-ul nou trebuie să funcționeze cu shared state-ul creat de versiunea anterioară. Testing: smoke tests automate post-deployment care verifică că toate MFE-urile se încarcă corect.

### Î12: Cum gestionezi shared dependencies (Vue, Pinia) între MFE-uri la deployment?

**Răspuns:**
Module Federation gestionează shared dependencies prin configurarea `shared` din `webpack.config.js`. Declar `vue`, `pinia`, `vue-router` ca **singleton** și **eager** în host. La build, fiecare MFE declară aceleași dependențe shared. La runtime, Module Federation negociază: dacă host-ul a încărcat deja Vue 3.4, MFE-ul nu îl mai încarcă (folosește versiunea host-ului). **Riscul la deployment**: dacă MFE-ul A necesită Vue 3.5 dar host-ul are 3.4, runtime error. Soluția: **version ranges** stricte în shared config (`requiredVersion: '^3.4.0'`), smoke tests care verifică compatibilitatea, și un **compatibility matrix** în documentation. La upgrade major, coordonez deployment-ul tuturor MFE-urilor.

---

> **Notă finală:** DevOps pentru Vue MFE-uri este 90% identic cu Angular. Conceptele de pipeline,
> IaC, containerizare, monitoring și environment management sunt framework-agnostic.
> Diferențele sunt în tooling (build commands, config files, env vars) nu în arhitectură.
> Experiența ta din Angular DevOps se transferă direct.


---

**Următor :** [**11 - BFF cu C#/.NET** →](Vue/11-BFF-CSharp-dotNET.md)