# Análise Completa e Sugestões — gitops-core-infra

## 📸 Mapa Atual do Repositório

```
gitops-core-infra/
├── argocd/
│   ├── applicationsets/
│   │   ├── core-infra-environments.yaml   ✅ ApplicationSet (git generator por environments/*)
│   │   └── core-infra-platform.yaml       ✅ ApplicationSet (list generator para providers/XRDs/compositions)
│   ├── cluster-bootstrap/
│   │   ├── integration/                   ✅ Kustomization com gateway, external-secrets, titlis, datadog
│   │   ├── pre-prod/                      ⚠️  kustomize.yaml vazio
│   │   └── prod/                          ⚠️  kustomize.yaml vazio
│   └── projects/
│       ├── platform-engineering-project.yaml  ✅ Único preenchido (platform AppProject)
│       ├── integration-project.yaml           ❌ Vazio
│       ├── pre-prod-project.yaml              ❌ Vazio
│       ├── prod-project.yaml                  ❌ Vazio
│       ├── prod-1-0-project.yaml              ❌ Vazio
│       ├── homolog-1-0-project.yaml           ❌ Vazio
│       └── stage-1-0-project.yaml             ❌ Vazio
│
├── backstage-templates/
│   ├── gcs-bucket/          ✅ Template + skeleton (Managed Resource direto)
│   ├── pubsub/              ✅ Template + skeleton (MR com DLQ + Subscription)
│   ├── iam/service-account/ ✅ Template + skeleton
│   └── compute/
│       ├── compute-engine/  ✅ Template sofisticado (scope app/core dinâmico)
│       ├── cloudsql/        ✅ Template + skeleton
│       ├── gke/             ❌ Vazio
│       └── memory-store/    ❌ Vazio
│
├── crossplane/
│   ├── providers/               ✅ 5 providers (gcp, compute, storage, pubsub, sql)
│   ├── provider-config/         ✅ ProviderConfig GCP (ImpersonateServiceAccount)
│   ├── composite-resource-definition/
│   │   ├── gcs-bucket/          ❌ Vazio
│   │   ├── pubsub/              ❌ Vazio
│   │   ├── compute/             ❌ Subpastas vazias (cloudsql, compute-engine, gke, memory-store)
│   │   └── iam/service-accounts/❌ Vazio
│   └── composition/
│       ├── gcs-bucket/          ❌ Vazio
│       ├── pubsub/              ❌ Vazio
│       ├── compute/             ❌ Subpastas vazias (cloudsql, compute-engine, gke, memory-store)
│       └── iam/service-accounts/❌ Vazio
│
├── environments/
│   ├── integration/         ✅ Estrutura completa, poucos arquivos preenchidos
│   ├── homolog/             ⚠️  Cópia idêntica do integration
│   ├── stage/               ⚠️  Cópia idêntica do integration
│   ├── pre-prod/            ⚠️  Cópia idêntica do integration
│   ├── prod/                ⚠️  Cópia idêntica do integration
│   └── prod-1-0/            ⚠️  Cópia com bootstrap apontando errado
│
├── catalog/component.yaml   ✅ Backstage component
├── catalog-info.yaml        ✅ Backstage location
├── docs/index.md            ⚠️  Placeholder genérico
└── mkdocs.yaml              ✅ TechDocs config
```

---

## 🔴 Problemas Críticos Identificados

### 1. Sem XRDs e Compositions — O repositório não faz "core infra"

Este é o problema mais grave. As pastas `crossplane/composite-resource-definition/` e `crossplane/composition/` estão **todas vazias**. Isso significa que:

- Vocês estão usando **Managed Resources diretos** (ex: `compute.gcp.m.upbound.io/v1beta1 Instance`) em vez de abstrações.
- O developer ou o SRE precisa saber os detalhes do provider GCP para criar qualquer recurso.
- Não existe nenhuma padronização de segurança forçada na origem.
- Todos os skeletons do Backstage geram MRs puros — qualquer parâmetro perigoso (network `default`, zona `us-central1`, sem encryption) fica exposto.

**Impacto**: Esse é o oposto de "Security by Design". Qualquer um pode gerar um recurso fora dos padrões.

### 2. Identidade confusa entre Core e App

O repo se chama `gitops-core-infra`, mas:
- Os backstage templates de **gcs-bucket, pubsub** estão aqui (são recursos de App Infra, não Core).
- Os templates fazem `publish:github:pull-request` para `gitops-app-infra` (repo separado), mas vivem dentro do repo core.
- O template de compute-engine é o mais maduro e já resolve o scope `app-infra` vs `core-infra` dinamicamente — mas esse padrão não foi replicado para os outros.
- O `gitops-app-infra` está praticamente vazio (só catalog + docs).

### 3. 6 ambientes duplicados sem diferenciação real

Existem 6 pastas de ambiente (`integration`, `homolog`, `stage`, `pre-prod`, `prod`, `prod-1-0`) com:
- Exatamente a **mesma estrutura de pastas** espelhada
- Os **mesmos arquivos YAML idênticos** (todas as VMs são `e2-medium`, `debian-11`, `us-central1-a`, `network: default`)
- O `bootstrap-prod.yaml` dentro de `prod-1-0/` aponta para `environments/integration` (cópia não ajustada)
- Não existe diferenciação de sizing, HA, backup, encryption entre dev e prod

### 4. Recursos hardcoded sem parametrização

Todos os Managed Resources nos environments e skeletons estão com:
- `zone: us-central1-a` (não `southamerica-east1` como na estrutura de pastas sugere)
- `network: default` (perigoso em produção)
- Sem labels de ownership, cost-center, environment
- Sem encryption configurada
- Sem lifecycle policies

### 5. ApplicationSet de environments se auto-referencia

O `core-infra-environments.yaml` usa path `gitops-core-infra/environments/*`, mas os bootstrap YAMLs **dentro** de cada environment também são Applications do ArgoCD. Isso cria um loop: o ApplicationSet gera uma Application por environment, que dentro tem OUTRO Application (`bootstrap-*.yaml`) que faz a mesma coisa.

### 6. cluster-bootstrap mistura conceitos

O `cluster-bootstrap/` gerencia ferramentas do cluster (gateways, external-secrets, datadog) mas vive dentro de um repo de **cloud infrastructure**. São coisas de camadas diferentes — Kubernetes workloads/addons vs Cloud resources.

---

## 🏗️ Sugestão de Arquitetura Ideal

### Princípio Central: Separação por "Quem Pede" e "Quem Define"

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        BACKSTAGE (Developer Portal)                     │
│  Templates vivem AQUI ou em repo dedicado, NÃO nos repos de gitops     │
└────────────┬───────────────────────────────────┬────────────────────────┘
             │ PR para Core                      │ PR para App
             ▼                                   ▼
┌──────────────────────────┐     ┌──────────────────────────────────────┐
│   gitops-core-infra      │     │   gitops-app-infra                   │
│                          │     │                                      │
│ O Platform Team é dono.  │     │ Os Devs geram PRs via Backstage.     │
│ Pouca automação.         │     │ Self-service com guardrails.         │
│ Aprovação rigorosa.      │     │                                      │
│                          │     │ Usa CLAIMS (XRC) que referenciam     │
│ Contém:                  │     │ as abstrações definidas no core.     │
│ - VPCs, Subnets, NATs    │     │                                      │
│ - GKE Clusters           │     │ Contém:                              │
│ - CloudSQL instances     │     │ - Buckets, PubSub, Cloud Functions   │
│ - Memory Store           │     │ - Service Accounts de apps           │
│ - Service Accounts base  │     │ - Cloud Scheduler                    │
│ - Firewall rules         │     │ - Permissões app-specific            │
│                          │     │                                      │
│ + XRDs e Compositions    │     │                                      │
│   (as "APIs" de infra)   │     │                                      │
└──────────┬───────────────┘     └────────────────┬─────────────────────┘
           │                                      │
           ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ARGOCD (Management Cluster)                          │
│                                                                         │
│  AppProject: core-infra  ─────►  Crossplane ─────►  GCP                │
│  AppProject: app-infra   ─────►  Crossplane ─────►  GCP                │
│  AppProject: platform    ─────►  Helm/Kustomize ──► Clusters           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 📂 Estrutura de Pastas Sugerida

### gitops-core-infra (Reestruturado)

```
gitops-core-infra/
│
├── README.md
├── catalog-info.yaml
├── catalog/component.yaml
├── mkdocs.yaml
├── docs/
│   └── index.md
│
├── argocd/
│   │
│   ├── projects/
│   │   ├── core-infra.yaml              # AppProject para recursos de cloud foundation
│   │   └── platform.yaml                # AppProject para addons de cluster (já existe)
│   │
│   ├── applicationsets/
│   │   ├── crossplane-platform.yaml     # Synca providers, XRDs, compositions (cluster-scoped)
│   │   └── core-environments.yaml       # Um ApplicationSet que gera 1 App por environment
│   │
│   └── cluster-addons/                  # RENOMEADO de cluster-bootstrap
│       ├── base/                        # ← Kustomize base compartilhada
│       │   ├── kustomization.yaml
│       │   ├── external-secrets/
│       │   │   ├── application.yaml
│       │   │   └── cluster-secret-store.yaml
│       │   ├── gateway/
│       │   │   ├── external-gateway.yaml
│       │   │   └── internal-gateway.yaml
│       │   └── monitoring/
│       │       └── datadog.yaml
│       └── overlays/                    # ← Override por ambiente via Kustomize
│           ├── integration/
│           │   └── kustomization.yaml   # patches: values, namespaces, etc.
│           ├── pre-prod/
│           │   └── kustomization.yaml
│           └── prod/
│               └── kustomization.yaml
│
├── crossplane/
│   │
│   ├── providers/                       # ✅ Mantém (já está bom)
│   │   ├── provider-family-gcp.yaml     # Considerar migrar para provider-family
│   │   ├── provider-gcp-compute.yaml
│   │   ├── provider-gcp-storage.yaml
│   │   ├── provider-gcp-pubsub.yaml
│   │   └── provider-gcp-sql.yaml
│   │
│   ├── provider-configs/                # ✅ Mantém (plural para clareza)
│   │   ├── gcp-default.yaml            # ProviderConfig padrão
│   │   └── gcp-prod.yaml               # ProviderConfig separado para prod (outro project)
│   │
│   ├── xrds/                            # 💡 RENOMEADO de composite-resource-definition
│   │   ├── xnetwork.yaml               # Define API: XNetwork (VPC + Subnet + NAT + Firewall)
│   │   ├── xcluster.yaml               # Define API: XCluster (GKE + NodePools)
│   │   ├── xdatabase.yaml              # Define API: XDatabase (CloudSQL)
│   │   ├── xcache.yaml                 # Define API: XCache (MemoryStore)
│   │   ├── xbucket.yaml                # Define API: XBucket
│   │   ├── xpubsub.yaml               # Define API: XPubSub (Topic + Sub + DLQ)
│   │   └── xserviceaccount.yaml        # Define API: XServiceAccount
│   │
│   └── compositions/                    # 💡 1 arquivo por composição, nomeados claramente
│       ├── network-standard.yaml        # Composition: VPC + Subnets + NAT + Firewall
│       ├── cluster-gke-standard.yaml    # Composition: GKE private cluster + node pools
│       ├── cluster-gke-autopilot.yaml   # Composition: GKE Autopilot
│       ├── database-postgresql.yaml     # Composition: CloudSQL PostgreSQL HA
│       ├── database-mysql.yaml          # Composition: CloudSQL MySQL
│       ├── cache-redis.yaml             # Composition: MemoryStore Redis
│       ├── bucket-standard.yaml         # Composition: GCS com encryption + lifecycle
│       ├── pubsub-standard.yaml         # Composition: Topic + Sub + DLQ + IAM
│       └── serviceaccount-standard.yaml # Composition: SA + IAM bindings
│
└── environments/                        # 💡 APENAS Claims/XRCs, nada mais
    ├── integration/
    │   ├── network/
    │   │   └── main.yaml                # XNetworkClaim: VPC + subnets de integration
    │   ├── compute/
    │   │   ├── gke-integration.yaml     # XClusterClaim: cluster de integration
    │   │   └── vms/                     # (se necessário)
    │   │       └── bastion.yaml
    │   ├── database/
    │   │   ├── pg-main.yaml             # XDatabaseClaim: postgres principal
    │   │   └── redis-session.yaml       # XCacheClaim: redis de sessão
    │   └── iam/
    │       └── core-service-accounts.yaml
    │
    ├── pre-prod/
    │   ├── network/
    │   │   └── main.yaml                # Mesmo padrão, valores diferentes
    │   ├── compute/
    │   │   └── gke-preprod.yaml
    │   ├── database/
    │   │   ├── pg-main.yaml
    │   │   └── redis-session.yaml
    │   └── iam/
    │       └── core-service-accounts.yaml
    │
    └── prod/
        ├── network/
        │   └── main.yaml                # VPC maior, mais subnets, multi-AZ
        ├── compute/
        │   ├── gke-prod.yaml            # Cluster maior, HA
        │   └── gke-prod-batch.yaml      # Cluster secundário se necessário
        ├── database/
        │   ├── pg-main.yaml             # HA, backups, read replicas
        │   └── redis-session.yaml       # Cluster redis
        └── iam/
            └── core-service-accounts.yaml
```

### gitops-app-infra (Proposta)

```
gitops-app-infra/
│
├── README.md
├── catalog-info.yaml
├── catalog/component.yaml
├── mkdocs.yaml
├── docs/index.md
│
├── argocd/
│   ├── projects/
│   │   └── app-infra.yaml               # AppProject com permissões restritas
│   └── applicationsets/
│       └── app-environments.yaml         # Git generator: environments/*/teams/*
│
└── environments/
    ├── integration/
    │   └── teams/
    │       ├── payments/
    │       │   ├── storage/
    │       │   │   └── receipts-bucket.yaml       # XBucketClaim
    │       │   ├── messaging/
    │       │   │   └── payment-events.yaml        # XPubSubClaim
    │       │   └── iam/
    │       │       └── payment-processor-sa.yaml   # XServiceAccountClaim
    │       └── notifications/
    │           ├── messaging/
    │           │   └── notification-events.yaml
    │           └── iam/
    │               └── notification-sa.yaml
    │
    ├── pre-prod/
    │   └── teams/
    │       └── [mesma estrutura]
    │
    └── prod/
        └── teams/
            └── [mesma estrutura]
```

### backstage-templates (Repo separado ou dentro do Backstage)

> **Decisão de design**: NÃO separar templates por `app-infra/` vs `core-infra/`.
> Um mesmo recurso (bucket, database, service-account) pode ser usado em ambos os contextos.
> A separação é feita **pelo parâmetro `scope`** dentro do template, não pela pasta.

```
backstage-templates/
├── catalog-info.yaml                    # Registra todos os templates no Backstage
│
├── bucket/
│   ├── template.yaml                    # Pergunta: scope = app-infra | core-infra
│   └── skeleton/
│       └── bucket-claim.yaml            # ← Gera XBucketClaim
│
├── pubsub/
│   ├── template.yaml
│   └── skeleton/
│       └── pubsub-claim.yaml            # ← Gera XPubSubClaim
│
├── service-account/
│   ├── template.yaml
│   └── skeleton/
│       └── sa-claim.yaml
│
├── database/
│   ├── template.yaml                    # Pergunta: engine (postgres/mysql), scope, sizing
│   └── skeleton/
│       └── database-claim.yaml          # ← Gera XDatabaseClaim
│
├── cache/
│   ├── template.yaml
│   └── skeleton/
│       └── cache-claim.yaml             # ← Gera XCacheClaim
│
├── network/
│   ├── template.yaml                    # Restrito: só platform-team via allowedGroups
│   └── skeleton/
│       └── network-claim.yaml
│
└── cluster/
    ├── template.yaml                    # Restrito: só platform-team via allowedGroups
    └── skeleton/
        └── cluster-claim.yaml
```

**Como funciona o `scope` no template:**

O template pergunta ao usuário o scope e direciona o PR para o repo correto:

```yaml
# Exemplo: backstage-templates/bucket/template.yaml
parameters:
  - title: Escopo do recurso
    properties:
      scope:
        type: string
        title: Onde este recurso pertence?
        enum:
          - app-infra       # Recurso de um time/serviço
          - core-infra      # Recurso de plataforma/fundação
        description: |
          app-infra: O time é dono. PR vai para gitops-app-infra.
          core-infra: Platform Team é dono. PR vai para gitops-core-infra.

steps:
  - id: publish
    action: publish:github:pull-request
    input:
      repoUrl: github.com?owner=jeitto&repo=gitops-${{ parameters.scope }}
      # scope=app-infra  → PR para gitops-app-infra
      # scope=core-infra → PR para gitops-core-infra
```

**Controle de acesso por complexidade do recurso, não por pasta:**

| Recurso | Quem pode usar | Como controlar |
|---|---|---|
| bucket, pubsub, service-account | Qualquer time | `allowedGroups: ['*']` no template |
| database, cache | Times com aprovação | `allowedGroups: ['*']` + PR review obrigatório |
| network, cluster | Só Platform Team | `allowedGroups: ['platform-team']` no template |

Dessa forma o **mesmo template de bucket** serve para:
- Um dev criando bucket de uploads → scope `app-infra` → PR no `gitops-app-infra`
- O platform team criando bucket de backups → scope `core-infra` → PR no `gitops-core-infra`

---

## 🔑 Recomendações Detalhadas

### 1. Crie XRDs e Compositions — É a peça que falta

Hoje vocês fazem:
```
Backstage → Managed Resource (apiVersion: compute.gcp.m.upbound.io/v1beta1) → GCP
```

Deveriam fazer:
```
Backstage → Claim (apiVersion: jeitto.io/v1alpha1) → XRD → Composition → Managed Resources → GCP
```

**Exemplo prático — XBucket:**

```yaml
# crossplane/xrds/xbucket.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xbuckets.jeitto.io
spec:
  group: jeitto.io
  names:
    kind: XBucket
    plural: xbuckets
  claimNames:
    kind: BucketClaim
    plural: bucketclaims
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  required: [environment, team]
                  properties:
                    environment:
                      type: string
                      enum: [integration, pre-prod, prod]
                    team:
                      type: string
                    storageClass:
                      type: string
                      enum: [STANDARD, NEARLINE, COLDLINE]
                      default: STANDARD
                    location:
                      type: string
                      default: southamerica-east1
                    retentionDays:
                      type: integer
                      default: 90
```

```yaml
# crossplane/compositions/bucket-standard.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: bucket-standard
  labels:
    provider: gcp
    service: storage
spec:
  compositeTypeRef:
    apiVersion: jeitto.io/v1alpha1
    kind: XBucket
  resources:
    - name: bucket
      base:
        apiVersion: storage.gcp.m.upbound.io/v1beta2
        kind: Bucket
        spec:
          forProvider:
            location: southamerica-east1
            publicAccessPrevention: enforced     # ← Segurança forçada!
            uniformBucketLevelAccess: true        # ← Padrão forçado!
            versioning:
              - enabled: true                    # ← Versionamento obrigatório!
            encryption:
              - defaultKmsKeyName: "projects/jeitto-prod/locations/southamerica-east1/keyRings/storage/cryptoKeys/default"
            labels:
              managed-by: crossplane
          providerConfigRef:
            name: default
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.location
          toFieldPath: spec.forProvider.location
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.storageClass
          toFieldPath: spec.forProvider.storageClass
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.team
          toFieldPath: spec.forProvider.labels.team
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.environment
          toFieldPath: spec.forProvider.labels.environment
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.environment
          toFieldPath: spec.providerConfigRef.name
          transforms:
            - type: map
              map:
                integration: default
                pre-prod: default
                prod: gcp-prod          # ← Prod usa outro ProviderConfig/Project!
```

**Então o claim gerado pelo Backstage fica simples e seguro:**

```yaml
# environments/integration/teams/payments/storage/receipts-bucket.yaml
apiVersion: jeitto.io/v1alpha1
kind: BucketClaim
metadata:
  name: payments-receipts
  namespace: payments
  labels:
    team: payments
    environment: integration
spec:
  parameters:
    environment: integration
    team: payments
    storageClass: STANDARD
    location: southamerica-east1
  compositionRef:
    name: bucket-standard
```

O developer **nunca toca** em `publicAccessPrevention`, `encryption`, ou `uniformBucketLevelAccess`. O Platform Team define isso na Composition.

### 2. Reduza os ambientes para 3

Vocês têm **6 ambientes** com conteúdo idêntico. Isso é manutenção exponencial sem benefício.

| Atual | Proposta | Justificativa |
|---|---|---|
| integration | **integration** | Ambiente de desenvolvimento/testes |
| homolog | ❌ Eliminar | Duplicação sem diferenciação |
| stage | ❌ Eliminar | Duplicação sem diferenciação |
| pre-prod | **pre-prod** | Validação pré-produção |
| prod | **prod** | Produção |
| prod-1-0 | ❌ Eliminar | Cópia idêntica com bootstrap errado |

Se precisar de mais ambientes no futuro, o ApplicationSet faz isso automaticamente — basta criar a pasta.

### 3. Elimine os bootstrap-*.yaml dentro de environments

Hoje cada environment tem um `bootstrap-*.yaml` que é uma Application do ArgoCD. Mas o `core-infra-environments.yaml` (ApplicationSet) **já faz isso automaticamente** com o git generator. Isso gera duplicação e conflito.

**Remover**: `environments/*/bootstrap-*.yaml`

**O ApplicationSet já resolve:**
```yaml
# O git generator já cria uma Application por pasta em environments/*
generators:
  - git:
      directories:
        - path: environments/*
```

### 4. Corrija o ApplicationSet de platform

O `core-infra-platform.yaml` usa o path `gitops-core-infra/crossplane/...` com um prefixo `gitops-core-infra/`. Isso só funciona se o repo estiver dentro de uma monorepo. Se é um repo isolado, o path deveria ser `crossplane/...`.

Além disso, ele aponta **tudo** para `crossplane-system`, inclusive XRDs e Compositions que são **cluster-scoped** (não pertencem a nenhum namespace). O namespace não importa para cluster-scoped resources, mas é mais limpo omitir ou usar um pattern diferente.

### 5. Separe os Backstage templates do gitops-core-infra

Templates do Backstage não são infraestrutura — são **formulários** que geram PRs. Eles devem viver:
- Dentro do próprio Backstage (se monorepo)
- Em um repo dedicado `backstage-templates`
- Nunca dentro do repo que o ArgoCD monitora (senão ele tenta aplicar `template.yaml` como K8s resource)

Atualmente o ArgoCD com `recurse: true` e `include: "*.yaml"` vai tentar aplicar os templates do Backstage como Kubernetes objects — isso vai gerar erros de sync.

### 6. Refatore cluster-addons com Kustomize base/overlays

O `cluster-bootstrap` atual tem a mesma estrutura copiada 3 vezes (integration, pre-prod, prod). Use Kustomize base + overlays para evitar duplicação:

```yaml
# argocd/cluster-addons/base/kustomization.yaml
resources:
  - external-secrets/application.yaml
  - gateway/external-gateway.yaml
  - gateway/internal-gateway.yaml

# argocd/cluster-addons/overlays/integration/kustomization.yaml
resources:
  - ../../base
patchesStrategicMerge:
  - external-secrets-patch.yaml     # override version, namespace, etc.
```

### 7. Use ArgoCD Projects com RBAC real

Só o `platform-engineering-project.yaml` está preenchido. Cada ambiente deveria ter um `AppProject` que restringe:
- Quais repos podem fazer deploy
- Quais namespaces são alvos válidos
- Quais resource types são permitidos

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: core-infra-prod
  namespace: argocd
spec:
  description: "Core infrastructure for production"
  sourceRepos:
    - "https://github.com/jeitto/gitops-core-infra.git"
  destinations:
    - namespace: crossplane-system
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: "jeitto.io"
      kind: "*"                  # Permite XRDs/Claims
    - group: "*.gcp.m.upbound.io"
      kind: "*"                  # Permite Managed Resources
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  syncWindows:
    - kind: allow
      schedule: "0 6-22 * * 1-5"  # Sync apenas horário comercial para prod
      duration: 16h
      applications: ["*"]
```

---

## 📊 Resumo Visual: Antes vs Depois

```
ANTES (Atual)                              DEPOIS (Sugerido)
─────────────────────                      ─────────────────────

Backstage                                  Backstage
    │                                          │
    ▼                                          ▼
Gera Managed Resource puro                 Gera Claim (XRC) padronizado
(apiVersion: compute.gcp.m.upbound.io)     (apiVersion: jeitto.io/v1alpha1)
    │                                          │
    ▼                                          ▼
PR no gitops-core-infra                    PR no repo CORRETO
(templates dentro do repo core!)           (core ou app, conforme scope)
    │                                          │
    ▼                                          ▼
ArgoCD aplica MR diretamente               ArgoCD aplica o Claim
(sem guardrails)                           (XRD valida, Composition padroniza)
    │                                          │
    ▼                                          ▼
Crossplane → GCP                           Crossplane → MRs gerados → GCP
(network: default, sem encryption,         (encryption forçada, labels,
 us-central1, sem labels)                   southamerica-east1, team tag)
```

---

## ✅ Plano de Ação (Priorizado)

| # | Ação | Impacto | Esforço |
|---|------|---------|---------|
| 1 | Criar os XRDs e Compositions para bucket e pubsub | 🔴 Crítico | Médio |
| 2 | Migrar skeletons do Backstage para gerar Claims em vez de MRs | 🔴 Crítico | Baixo |
| 3 | Mover backstage-templates para repo separado (sem dividir app/core — usar param `scope`) | 🟡 Alto | Baixo |
| 4 | Remover bootstrap-*.yaml duplicados dos environments | 🟡 Alto | Baixo |
| 5 | Eliminar environments duplicados (homolog, stage, prod-1-0) | 🟡 Alto | Baixo |
| 6 | Corrigir paths no ApplicationSet (remover prefixo `gitops-core-infra/`) | 🟡 Alto | Baixo |
| 7 | Preencher os ArgoCD Projects por ambiente | 🟠 Médio | Médio |
| 8 | Refatorar cluster-bootstrap com Kustomize base/overlays | 🟠 Médio | Médio |
| 9 | Estruturar gitops-app-infra com environments/teams/* | 🟠 Médio | Médio |
| 10 | Corrigir zona/região dos resources (us-central1 → southamerica-east1) | 🟡 Alto | Baixo |
| 11 | Criar Compositions para GKE, CloudSQL, MemoryStore | 🟠 Médio | Alto |
| 12 | Configurar ProviderConfig separado por projeto GCP (dev vs prod) | 🟠 Médio | Baixo |

---

Quer que eu comece a implementar algum desses itens? Posso começar pelas XRDs e Compositions (item 1) ou pela reestruturação de pastas (itens 3-6).
