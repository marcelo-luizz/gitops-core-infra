# gitops-core-infra

Repositório de infraestrutura foundacional gerenciado via GitOps.

## Componentes

| Categoria | Recursos | Composition |
|-----------|----------|-------------|
| Network | VPC, Subnet, NAT, Firewall | `network-standard` |
| Compute | GKE Clusters | `cluster-gke-standard` |
| Database | CloudSQL PostgreSQL/MySQL | `database-postgresql`, `database-mysql` |
| Cache | MemoryStore Redis | `cache-redis` |
| IAM | Service Accounts de plataforma | `serviceaccount-standard` |

## Como criar um recurso

1. Crie um arquivo Claim em `environments/<env>/<categoria>/`
2. Abra um PR para `main`
3. Aguarde aprovação do Platform Team
4. Após merge, o ArgoCD aplica automaticamente

## Contato

Platform Engineering — #platform-engineering no Slack
