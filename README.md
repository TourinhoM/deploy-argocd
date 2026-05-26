# deploy-argocd

Repositório de bootstrap do cluster: instala o Argo CD, aplica o root-app e configura
os pré-requisitos de infraestrutura (Vault, ESO) que o restante da stack depende.

## Estrutura

```
bootstrap/       instala/atualiza o Argo CD e cria o root-app
apps/            AppProjects e Applications (Argo CD CRDs)
cluster-config/  manifests aplicados por Applications deste repo (ESO, monitoring, argocd)
scripts/         onboarding.sh + bootstrap-vault.sh
```

## Quickstart — cluster do zero

```bash
bash scripts/onboarding.sh
```

O script executa as seguintes etapas em ordem:

| Etapa | O que faz |
|-------|-----------|
| 1/6 | `kubectl apply -k bootstrap/argocd` — instala o Argo CD |
| 2/6 | Aguarda `argocd-server` ficar Ready |
| 3/6 | `kubectl apply -f bootstrap/root-app/application.yaml` — inicia reconciliação GitOps |
| 4/6 | Aguarda namespace `external-secrets` e controller do ESO |
| 5/6 | Chama `scripts/bootstrap-vault.sh` — init, unseal, policies e secrets |
| 6/6 | Aguarda `ClusterSecretStore vault-homelab` ficar Ready |

### O que o bootstrap-vault.sh faz

O script `scripts/bootstrap-vault.sh` é chamado automaticamente pelo `onboarding.sh`
e pode também ser executado isoladamente em caso de reinicialização do Vault.

```bash
KUBECTL="sudo k3s kubectl" bash scripts/bootstrap-vault.sh
```

Etapas internas:

1. Aguarda o pod do Vault ficar Ready
2. `vault operator init -key-shares=1 -key-threshold=1` — gera unseal key e root token
3. Cria o Secret `vault-unseal-key` (usado pelo postStart hook para auto-unseal em restarts)
4. Unseal do Vault
5. Cria `vault-bootstrap-token` e reinicia o Job `vault-bootstrap` — configura KV v2,
   Kubernetes auth, policies (`eso-read`, `crossplane-write`) e roles (`eso`, `provider-vault`)
6. Coleta interativamente os secrets de aplicação e os escreve no Vault

> **Root token**: exibido na etapa 2 e não pode ser recuperado depois.
> Guarde em local seguro antes de continuar.

### Secrets criados interativamente

O script solicita os seguintes valores. A entrada não é ecoada no terminal.

| Path no Vault | Propriedades | Origem |
|---------------|-------------|--------|
| `secret/cluster/argocd` | `github_app_id`, `github_app_installation_id`, `github_app_private_key` | GitHub App |
| `secret/cluster/grafana` | `admin_password`, `db_password` | Definido pelo operador |
| `secret/cluster/postgresql` | `superuser_password` | Definido pelo operador |
| `secret/cluster/keycloak` | `admin_password` | Definido pelo operador |
| `secret/cluster/dtrack` | `db_password` | Definido pelo operador |

> **`secret/cluster/dtrack.api_key`** não pode ser criado no bootstrap porque é gerado
> pelo Dependency Track no primeiro boot. Após a aplicação subir, execute:
> ```bash
> vault kv patch secret/cluster/dtrack api_key=<valor>
> ```

### Após o onboarding

Force-sync dos ExternalSecrets para materializar todos os Secrets k8s imediatamente:

```bash
kubectl annotate externalsecret --all -A force-sync=$(date +%s) --overwrite
```

Apps individuais que precisam de TLS no Ingress trazem seu próprio script
(ex.: `deploy-keycloak/scripts/bootstrap-tls.sh`).

## Bootstrap manual (passo a passo)

Se preferir executar sem o script:

```bash
# 1. Instala Argo CD
kubectl apply -k bootstrap/argocd

# 2. Inicia reconciliação GitOps
kubectl apply -f bootstrap/root-app/application.yaml

# 3. Bootstrap do Vault (após pod Ready)
bash scripts/bootstrap-vault.sh

# 4. Valida ClusterSecretStore
kubectl get clustersecretstore vault-homelab \
  -o jsonpath='{.status.conditions}'   # type=Ready, status=True
```

## Lint / validação CI

Validado pelo workflow `lint-k8s.yml` do
[`org-ci-platform`](https://github.com/TourinhoM/org-ci-platform) —
3 scanners em paralelo sobre o output do `kustomize build`.

| Scanner | O que valida |
|---------|-------------|
| **kubeconform** | Schema das APIs k8s + CRDs via datreeio catalog |
| **kube-linter** | resources, probes, securityContext, capabilities |
| **polaris** | Security/reliability/efficiency; gate em `danger` |

### Patches em `bootstrap/argocd/` por causa do lint

- **`resources-patch.yaml`** — `resources.requests/limits`, `imagePullPolicy: Always`,
  `containerPort: 9001` no notifications-controller, `securityContext` pod-level
- **`polaris-rbac-exempt-patch.yaml`** — exemptions para ClusterRoles/Bindings do Argo CD
  que precisam de permissões amplas por design (application-controller, server)

Ver [ARCHITECTURE.md](ARCHITECTURE.md) para decisões de design detalhadas.
