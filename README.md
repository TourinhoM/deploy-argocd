# deploy-argocd

Repositório GitOps para gerenciar **Argo CD** e os **workloads** do cluster via pull request.

## Estrutura

- `bootstrap/`: instala/atualiza o Argo CD e cria o “root app”
- `apps/`: só CRDs do Argo (`AppProject`, `Application`, `ApplicationSet`)
- `envs/`: manifests por ambiente (Kustomize/Helm) referenciados por `apps/`
- `lib/`: peças reutilizáveis (patches/labels)

## Fluxo (GitOps)

1. Aplicar uma vez o Argo CD: `bootstrap/argocd/`
2. Aplicar uma vez o root app: `bootstrap/root-app/application.yaml`
3. A partir daí, o Argo CD reconcilia `apps/` e tudo que eles apontarem em `envs/`

## Como aplicar (primeiro bootstrap)

```bash
kubectl apply -k bootstrap/argocd
kubectl apply -f bootstrap/root-app/application.yaml
```

## External Secrets Operator + Bitwarden — bootstrap manual

A `Application` `external-secrets` instala o ESO + sidecar `bitwarden-sdk-server`
e a `external-secrets-config` cria o `ClusterSecretStore` `bitwarden-homelab`.
Dois segredos precisam ser criados **fora do Git** (chicken-and-egg):

1. **Access token** do machine account no Bitwarden Secrets Manager:

   ```bash
   sudo k3s kubectl -n external-secrets create secret generic bitwarden-access-token \
     --from-literal=token='<COLE_O_ACCESS_TOKEN>'
   ```

2. **Certificado TLS self-signed** para o `bitwarden-sdk-server`:

   ```bash
   bash scripts/bootstrap-bitwarden-sdk-tls.sh
   ```

   O script gera um cert válido por 10 anos com SAN cobrindo o DNS interno
   `bitwarden-sdk-server.external-secrets.svc.cluster.local` e cria/atualiza o
   Secret `bitwarden-tls-certs` (chaves: `tls.crt`, `tls.key`, `ca.crt`).

Validar:

```bash
sudo k3s kubectl get clustersecretstore bitwarden-homelab \
  -o jsonpath='{.status.conditions}'   # type=Ready, status=True
```

## Plano: o que fazer em cada arquivo

### `bootstrap/argocd/kustomization.yaml`
- **TODO**: decidir a forma de instalar o Argo CD (manifests vs Helm)
- **TODO**: manter a versão pinada (chart version ou release)
- **TODO**: ajustar `ingress.yaml` se você expõe a UI

### `bootstrap/argocd/values.yaml`
- **TODO**: configurar Ingress/hostname/TLS (se aplicável)
- **TODO**: configurar RBAC (admin desativado, SSO, etc) se necessário
- **TODO**: configurar repos/creds (idealmente via External Secrets)

### `bootstrap/root-app/application.yaml`
- **TODO**: preencher `spec.source.repoURL` com o URL real do repo
- **TODO**: ajustar o branch (`targetRevision`)
- **TODO**: decidir se `syncPolicy.automated` fica ligado no root (recomendado)

### `apps/projects/platform.yaml`
- **TODO**: restringir `sourceRepos` (somente o necessário)
- **TODO**: restringir `destinations` (namespaces por ambiente)

### `apps/applications/*.yaml`
- **TODO**: para cada app, apontar `spec.source.path` para o diretório em `envs/<ambiente>/...`
- **TODO**: definir `syncOptions` e política de prune/selfHeal

### `envs/dev/*` e `envs/prod/*`
- **TODO**: colocar aqui manifests reais (kustomize/helm values) por ambiente
- **TODO**: segredos: usar SOPS/SealedSecrets/ExternalSecrets (não commitar segredo em claro)