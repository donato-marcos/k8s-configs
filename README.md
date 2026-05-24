# 📦 k8s-configs — GitOps Lab

Repositório GitOps para gerenciamento de aplicações e infraestrutura Kubernetes via **ArgoCD + Kustomize + Helm**.


## 🗂️ Estrutura

```
k8s-configs/
├── k8s-apps/
│   ├── apps/                           # Aplicações (Kustomize)
│   │   └── atv4-compassuol/
│   │       ├── base/                   # Manifests canônicos
│   │       └── overlays/               # dev/, prod/
│   │
│   ├── argocd/                         # Definições do ArgoCD
│   │   ├── root-app.yaml               # App-of-Apps (entrypoint)
│   │   ├── apps/                       # Apps de negócio
│   │   └── infra/                      # Apps de infra
│   │
│   └── infra/                          # Infraestrutura
│       ├── metrics-server/
│       │   ├── kustomization.yaml
│       │   └── values.yaml
│       └── kube-prometheus-stack/
│           ├── kustomization.yaml
│           ├── values.yaml
│           └── httproute-grafana.yaml
│
└── README.md
```

## Quick Start

### Primeiro deploy (bootstrap)
```bash
# Aplicar o root-app (único comando manual necessário)
ansible@k8s-master01:~$ kubectl apply -f https://raw.githubusercontent.com/donato-marcos/k8s-configs/refs/heads/main/k8s-apps/argocd/root-app.yaml

# Verificar apps criadas
kubectl get applications -n argocd
```

### Validar localmente antes do commit
```bash
# App com Kustomize
kustomize build k8s-apps/apps/atv4-compassuol/overlays/dev | \
  kubectl apply --dry-run=server -f -

# Infra com Helm via Kustomize
kustomize build --enable-helm k8s-apps/infra/kube-prometheus-stack | \
  kubectl apply --dry-run=server -f -
```

## Como Adicionar...

### Nova aplicação (Kustomize)
1. Crie a estrutura em `apps/minha-app/{base,overlays}`
2. Adicione `kustomization.yaml` na base e nos overlays
3. Crie `argocd/apps/minha-app.yaml` apontando para o overlay
4. Commit → push → ArgoCD sincroniza automaticamente

### Nova infraestrutura (Helm + Kustomize)
1. Crie `infra/meu-chart/{kustomization.yaml, values.yaml}`
2. No `kustomization.yaml`:
   ```yaml
   helmCharts:
     - name: meu-chart
       repo: https://meu-repo.github.io/charts
       version: 1.2.3        # Sempre pinar!
       releaseName: meu-chart
       namespace: meu-ns
       valuesFile: values.yaml
   ```
3. Crie `argocd/infra/meu-chart.yaml` com `syncOptions: [EnableHelm=true]`
4. Commit → push → pronto!

---

## Comandos Úteis

```bash
# ArgoCD
argocd app list -n argocd
argocd app get <app> -n argocd
argocd app sync <app> -n argocd --force
argocd app logs <app> -n argocd --follow

# Kustomize + Helm (local)
kustomize build <path>                      # Apps
kustomize build --enable-helm <path>        # Infra com Helm

# Kubernetes
kubectl get pods -n monitoring
kubectl get httproute -A
kubectl top nodes                           # Se metrics-server ativo
```

## 📌 Regras de Ouro

1. **Nunca** usar `kubectl apply` direto no cluster para recursos gerenciados
2. **Sempre** pinar versão de charts Helm (`version: x.y.z`, nunca `*`)
3. **Sempre** validar com `--dry-run` antes de commitar
4. **Infra crítica**: usar `prune: false` para evitar deleção acidental
5. **Mudança = Git**: editou, commitou, pushou → ArgoCD aplica

> 🎯 **Lab Status**: ✅ Operacional  
> 🔄 **Tudo versionado**: rollback = `git revert`  
