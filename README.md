# k8s-configs — GitOps Lab

Repositório GitOps para gerenciamento de aplicações e infraestrutura Kubernetes via **ArgoCD + Kustomize + Helm**.

> **Esse** repositório faz parte do LAB: [cluster-kubernetes-cilium-ipv4](https://github.com/donato-marcos/Laboratorio-Kubernetes-com-Cilium-e-GatewayAPI/blob/main/cluster-kubernetes-cilium-ipv4.md)

---

## ⚠️ Configuração Inicial Obrigatória

> 🚨 **Antes do bootstrap**, realize os passos abaixo manualmente:

### 1. Cloudflare API Token
O token é compartilhado entre **cert-manager** e **external-dns**.
> 🔐 Gerar em: https://dash.cloudflare.com/profile/api-tokens

```bash
# Criar secret manualmente (ainda não usamos external-secrets):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "seu-token-aqui"
EOF

# Repetir para o namespace do external-dns (ou usar um secret global com RBAC):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: external-dns
type: Opaque
stringData:
  api-token: "seu-token-aqui"
EOF
```

> 💡 **Permissões mínimas do token Cloudflare**:
> - `Zone:DNS:Edit` (para cert-manager + external-dns)
> - `Zone:Read` (opcional, para validações)
> - Escopo: **Apenas a zona do seu domínio**

### 2. Configurar Issuers do cert-manager
Edite os arquivos `infra/security/cert-manager/issuers/*.yaml`:

```yaml
# infra/security/cert-manager/issuers/prod-clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: seu-email@dominio.com  # ⚠️ ALTERAR: usado para notificações da Let's Encrypt
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token-secret  # ✅ Mesmo secret criado acima
              key: api-token
```

> 🔁 Repita o processo para `staging-clusterissuer.yaml` (útil para testes sem rate-limit).

### 3. Aplicar secrets antes do bootstrap
Como ainda **não utilizamos external-secrets-operator**, os secrets devem existir no cluster antes da sincronização:

```bash
# Aplicar manualmente antes do root-infra.yaml:
kubectl apply -f infra/security/cert-manager/cloudflare-api-token-secret.yaml
kubectl apply -f infra/networking/external-dns/cloudflare-api-token-secret.yaml

# Depois prosseguir com o bootstrap normal:
kubectl apply -f clusters/homelab/bootstrap/root-infra.yaml
```

## 🗂️ Estrutura

```bash
k8s-configs/
├── apps/                          # Aplicações gerenciadas via Kustomize
│   ├── atv4-compassuol/
│   │   ├── base/
│   │   │   ├── backend/
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── hpa.yaml
│   │   │   │   ├── httproute.yaml
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── service.yaml
│   │   │   ├── frontend/
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── hpa.yaml
│   │   │   │   ├── httproute.yaml
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   ├── namespace.yaml
│   │       │   ├── patch-backend.yaml
│   │       │   └── patch-frontend.yaml
│   │       └── prod/
│   └── other-apps/
│
├── clusters/                      # Configurações por cluster (ArgoCD App-of-Apps)
│   └── homelab/
│       ├── apps/                  # Applications de aplicações
│       │   └── atv4-compassuol.yaml
│       ├── bootstrap/             # Bootstrap do GitOps (root apps)
│       │   ├── root-apps.yaml
│       │   └── root-infra.yaml
│       └── infra/                 # Applications de infraestrutura
│           ├── argocd.yaml
│           ├── cert-manager.yaml
│           ├── cilium.yaml
│           ├── external-dns.yaml
│           ├── kube-prometheus-stack.yaml
│           └── metrics-server.yaml
│
├── infra/                         # Módulos de infraestrutura reutilizáveis
│   ├── gitops/
│   │   └── argocd/
│   │       ├── httproute.yaml
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── networking/
│   │   ├── cilium/
│   │   │   ├── bgp/
│   │   │   │   ├── advertisement-ipv4.yaml
│   │   │   │   ├── advertisement-ipv6.yaml
│   │   │   │   ├── cluster-config.yaml
│   │   │   │   ├── peer-config-ipv4.yaml
│   │   │   │   └── peer-config-ipv6.yaml
│   │   │   ├── gateway-api/
│   │   │   │   └── gateway.yaml
│   │   │   ├── hubble-httproute.yaml
│   │   │   ├── ipam/
│   │   │   │   └── loadbalancer-pool.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── external-dns/
│   │       ├── cloudflare-api-token-secret.yaml  # ⚠️ Aplicar manualmente!
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── observability/
│   │   ├── kube-prometheus-stack/
│   │   │   ├── httproute-grafana.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── metrics-server/
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   └── security/
│       └── cert-manager/
│           ├── certificates/
│           │   ├── wildcard-prod.yaml
│           │   └── wildcard-staging.yaml
│           ├── cloudflare-api-token-secret.yaml  # ⚠️ Aplicar manualmente!
│           ├── issuers/
│           │   ├── prod-clusterissuer.yaml       # ⚠️ Editar email + solver dns01
│           │   └── staging-clusterissuer.yaml
│           ├── kustomization.yaml
│           └── values.yaml
│
└── README.md
```

## 🚀 Quick Start

### Pré-requisitos
- `kubectl` configurado para o cluster alvo
- `kustomize` instalado (para validação local)
- Acesso de leitura ao repositório pelo ArgoCD
- ✅ Secrets da Cloudflare aplicados manualmente (ver seção acima)

### Primeiro deploy (bootstrap)
```bash
# 1. Aplicar o root-infra.yaml (instala ArgoCD e componentes de infra)
kubectl apply -f https://raw.githubusercontent.com/donato-marcos/k8s-configs/refs/heads/main/clusters/homelab/bootstrap/root-infra.yaml

# 2. Aguardar estabilização da infra (ArgoCD, Cilium, cert-manager, etc.)
kubectl wait --for=condition=Available deployment -n argocd --all --timeout=300s

# 3. Aplicar o root-apps.yaml (sincroniza as aplicações)
kubectl apply -f https://raw.githubusercontent.com/donato-marcos/k8s-configs/refs/heads/main/clusters/homelab/bootstrap/root-apps.yaml

# 4. Verificar aplicações no ArgoCD
kubectl get applications -n argocd
```

### Validar localmente antes do commit
```bash
# App com Kustomize (overlay dev)
kustomize build apps/atv4-compassuol/overlays/dev | \
  kubectl apply --dry-run=server -f -

# Infra com Helm via Kustomize
kustomize build --enable-helm infra/observability/kube-prometheus-stack | \
  kubectl apply --dry-run=server -f -
```

## ➕ Como Adicionar...

### Nova aplicação (Kustomize)
1. Crie a estrutura em `apps/minha-app/{base,overlays}`
2. Adicione `kustomization.yaml` na base e em cada overlay
3. Crie o Application do ArgoCD em `clusters/homelab/apps/minha-app.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: minha-app
     namespace: argocd
   spec:
     source:
       repoURL: https://github.com/donato-marcos/k8s-configs.git
       path: apps/minha-app/overlays/dev
       targetRevision: HEAD
     destination:
       server: https://kubernetes.default.svc
       namespace: minha-app-ns
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```
4. Commit → push → ArgoCD sincroniza automaticamente

### Nova infraestrutura (Helm + Kustomize)
1. Crie `infra/meu-chart/{kustomization.yaml, values.yaml}`
2. No `kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   helmCharts:
     - name: meu-chart
       repo: https://meu-repo.github.io/charts
       version: 1.2.3        # ✅ Sempre pinar versão!
       releaseName: meu-chart
       namespace: meu-ns
       valuesFile: values.yaml
   ```
3. Crie o Application em `clusters/homelab/infra/meu-chart.yaml` com:
   ```yaml
   spec:
     source:
       repoURL: https://github.com/donato-marcos/k8s-configs.git
       path: infra/meu-chart
       targetRevision: HEAD
     
   ```
4. Commit → push → pronto! 🚀

---

## 🛠️ Comandos Úteis

```bash
# ===== ArgoCD =====
argocd app list -n argocd
argocd app get <app-name> -n argocd
argocd app sync <app-name> -n argocd --force
argocd app logs <app-name> -n argocd --follow

# ===== Kustomize + Helm (validação local) =====
kustomize build <path>                      # Apps puras com Kustomize
kustomize build --enable-helm <path>        # Infra com Helm charts

# ===== Kubernetes =====
kubectl get pods -n monitoring
kubectl get httproute -A
kubectl top nodes                           # Requer metrics-server ativo
kubectl describe application <app> -n argocd

# ===== Cert-Manager & DNS =====
kubectl get certificaterequest -A
kubectl get challenges -A                   # Debug de validação dns01
kubectl logs -n cert-manager -l app.kubernetes.io/name=cert-manager

# ===== Debug de sincronização =====
kubectl get events -n argocd --field-selector involvedObject.name=<app-name>
```

---

## 📌 Regras de Ouro

| # | Regra | Por quê? |
|---|-------|----------|
| 1 | **Nunca** use `kubectl apply` direto para recursos gerenciados pelo ArgoCD | Evita drift e conflitos de gerenciamento |
| 2 | **Sempre** pinar versão de Helm charts (`version: x.y.z`) | Garante reprodutibilidade e evita breaking changes |
| 3 | **Sempre** valide com `--dry-run=server` antes de commitar | Previne erros de sintaxe/schema antes do deploy |
| 4 | Use `prune: false` em recursos críticos de infra | Evita deleção acidental de componentes essenciais |
| 5 | **Mudança = Git**: editou → commitou → pushou → ArgoCD aplica | Mantém o cluster como extensão declarativa do repositório |
| 6 | Documente patches e overrides nos overlays | Facilita manutenção e onboarding da equipe |
| 7 | **Secrets manuais primeiro**: aplique antes do GitOps | Evita falhas de deploy por dependência de secrets |

---

## 🔄 Rollback & Recuperação

```bash
# Reverter última mudança no Git
git revert HEAD --no-edit && git push

# Forçar re-sync de uma aplicação no ArgoCD
argocd app sync <app-name> -n argocd --force

# Restaurar versão anterior de um Helm chart
# 1. Reverta o pin de versão no kustomization.yaml
# 2. Commit + push
# 3. ArgoCD aplica a versão anterior automaticamente

# Debug de certificados pendentes
kubectl describe certificate <nome> -n <namespace>
kubectl get challenges -A | grep -v Valid
```

---

> 🎯 **Lab Status**: ✅ Operacional  
> 🔄 **Tudo versionado**: rollback = `git revert`  
> 🌐 **IPv4/IPv6 ready**: BGP + Gateway API + Dual Stack  
> 🔐 **Certificados**: dns01 + Cloudflare (secret manual)  
