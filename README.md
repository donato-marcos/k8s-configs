# k8s-configs — GitOps Lab

Repositório GitOps para gerenciamento de aplicações e infraestrutura Kubernetes via **ArgoCD + Kustomize + Helm**.

> Essse repositório faz parte do LAB: [cluster-kubernetes-cilium-ipv4](https://github.com/donato-marcos/Laboratorio-Kubernetes-com-Cilium-e-GatewayAPI/blob/main/cluster-kubernetes-cilium-ipv4.md)


## 🗂️ Estrutura

```bash
k8s-configs/
├── apps
│   ├── atv4-compassuol
│   │   ├── base
│   │   │   ├── backend
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── hpa.yaml
│   │   │   │   ├── httproute.yaml
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── service.yaml
│   │   │   ├── frontend
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── hpa.yaml
│   │   │   │   ├── httproute.yaml
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays
│   │       ├── dev
│   │       │   ├── kustomization.yaml
│   │       │   ├── namespace.yaml
│   │       │   ├── patch-backend.yaml
│   │       │   └── patch-frontend.yaml
│   │       └── prod
│   └── other-apps
├── clusters
│   ├── homelab
│   │   ├── apps
│   │   │   └── atv4-compassuol.yaml
│   │   ├── bootstrap
│   │   │   ├── root-apps.yaml
│   │   │   └── root-infra.yaml
│   │   └── infra
│   │       ├── argocd.yaml
│   │       ├── cert-manager.yaml
│   │       ├── cilium.yaml
│   │       ├── external-dns.yaml
│   │       ├── kube-prometheus-stack.yaml
│   │       ├── longhorn.yaml
│   │       └── metrics-server.yaml
│   └── other-labs
├── infra
│   ├── gitops
│   │   └── argocd
│   │       ├── httproute.yaml
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── networking
│   │   ├── cilium
│   │   │   ├── bgp
│   │   │   │   ├── advertisement-ipv4.yaml
│   │   │   │   ├── advertisement-ipv6.yaml
│   │   │   │   ├── cluster-config.yaml
│   │   │   │   ├── peer-config-ipv4.yaml
│   │   │   │   └── peer-config-ipv6.yaml
│   │   │   ├── gateway-api
│   │   │   │   └── gateway.yaml
│   │   │   ├── hubble-httproute.yaml
│   │   │   ├── ipam
│   │   │   │   └── loadbalancer-pool.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── external-dns
│   │       ├── cloudflare-api-token-secret.yaml
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── observability
│   │   ├── kube-prometheus-stack
│   │   │   ├── httproute-grafana.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── metrics-server
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── security
│   │   └── cert-manager
│   │       ├── certificates
│   │       │   ├── wildcard-prod.yaml
│   │       │   └── wildcard-staging.yaml
│   │       ├── cloudflare-api-token-secret.yaml
│   │       ├── issuers
│   │       │   ├── prod-clusterissuer.yaml
│   │       │   └── staging-clusterissuer.yaml
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   └── storage
│       └── longhorn
│           ├── httproute.yaml
│           ├── kustomization.yaml
│           └── values.yaml
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

---
# Pensando em uma nova árvore:

```bash
k8s-configs/
├── clusters
│   ├── homelab
│   │   ├── applications
│   │   │   ├── apps
│   │   │   │   └── atv4-compassuol.yaml
│   │   │   ├── bootstrap
│   │   │   │   ├── root-apps.yaml
│   │   │   │   └── root-infra.yaml
│   │   │   └── infra
│   │   │       ├── argocd.yaml
│   │   │       ├── cert-manager.yaml
│   │   │       ├── cilium.yaml
│   │   │       ├── external-dns.yaml
│   │   │       ├── kube-prometheus-stack.yaml
│   │   │       └── metrics-server.yaml
│   │   └── config
│   │       ├── bgp
│   │       │   ├── advertisement-ipv4.yaml
│   │       │   ├── advertisement-ipv6.yaml
│   │       │   ├── cluster-config.yaml
│   │       │   ├── peer-config-ipv4.yaml
│   │       │   └── peer-config-ipv6.yaml
│   │       ├── ipam
│   │       │   └── loadbalancer-pool.yaml
│   │       ├── certificates
│   │       │   ├── wildcard-prod.yaml
│   │       │   └── wildcard-staging.yaml
│   │       ├── issuers
│   │       │   ├── prod-clusterissuer.yaml
│   │       │   └── staging-clusterissuer.yaml
│   │       ├── secrets
│   │       │   ├── external-dns-cloudflare-token.yaml
│   │       │   └── cert-manager-cloudflare-token.yaml
│   │       └── gateway-api
│   │           ├── httproute-hubble.yaml
│   │           ├── httproute-grafana.yaml
│   │           ├── httproute-argocd.yaml
│   │           └── gateway.yaml
│   └── other-labs
├── apps
│   ├── atv4-compassuol
│   │   ├── base
│   │   │   ├── backend
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── hpa.yaml
│   │   │   │   ├── httproute.yaml
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── service.yaml
│   │   │   ├── frontend
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── hpa.yaml
│   │   │   │   ├── httproute.yaml
│   │   │   │   ├── kustomization.yaml
│   │   │   │   └── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays
│   │       ├── dev
│   │       │   ├── kustomization.yaml
│   │       │   ├── namespace.yaml
│   │       │   ├── patch-backend.yaml
│   │       │   └── patch-frontend.yaml
│   │       └── prod
│   └── other-apps
├── infra
│   ├── gitops
│   │   └── argocd
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── networking
│   │   ├── cilium
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── external-dns
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   ├── observability
│   │   ├── kube-prometheus-stack
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── metrics-server
│   │       ├── kustomization.yaml
│   │       └── values.yaml
│   └── security
│       └── cert-manager
│           ├── kustomization.yaml
│           └── values.yaml
└── README.md
```