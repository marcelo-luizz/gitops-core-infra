# gitops-core-infra

Repositório de infraestrutura foundacional gerenciado via GitOps (ArgoCD + Crossplane).

## Escopo

Recursos de longa duração e alta criticidade que sustentam toda a plataforma:
- **Network**: VPCs, Subnets, NATs, Firewall Rules
- **Compute**: GKE Clusters, Compute Engine VMs
- **Database**: CloudSQL (PostgreSQL, MySQL), MemoryStore (Redis)
- **IAM**: Service Accounts de plataforma

## Estrutura

```
├── argocd/                    # Configuração do ArgoCD
│   ├── projects/              # AppProjects com RBAC
│   ├── applicationsets/       # ApplicationSets (automação de Applications)
│   └── cluster-addons/        # Addons Kubernetes (base + overlays)
├── crossplane/                # Definições do Crossplane
│   ├── providers/             # Providers GCP instalados
│   ├── provider-configs/      # Credenciais por ambiente
│   ├── xrds/                  # APIs de infraestrutura (XRDs)
│   └── compositions/          # Implementações padronizadas
├── environments/              # Claims por ambiente
│   ├── integration/
│   ├── pre-prod/
│   └── prod/
├── catalog-info.yaml          # Backstage Location
└── catalog/component.yaml     # Backstage Component
```

## Como funciona

1. Platform Team cria/edita um Claim em `environments/<env>/`
2. ArgoCD detecta a mudança e aplica o Claim no cluster
3. Crossplane resolve o Claim → XRD valida → Composition gera Managed Resources
4. Provider GCP provisiona o recurso na cloud

## Governança

- Todo PR requer aprovação do Platform Team
- PRs em `environments/prod/` requerem aprovação adicional do Security
- Sync windows restringem deploys em produção a horário comercial
