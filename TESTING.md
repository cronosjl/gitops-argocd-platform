# Guide de Test — GitOps ArgoCD Platform (Todo API)

Tests couvrant les 5 étapes du projet : validation Kustomize, déploiement ArgoCD, déploiements progressifs blue/green, pipeline CI et sécurité.

---

## Prérequis

```bash
k3d version && kubectl version --client && kustomize version
argocd version --client && kubeseal --version
kubectl cluster-info && kubectl get nodes
```

---

## 1. Validation locale des manifests (sans cluster)

### 1.1 Build de tous les overlays

```bash
kustomize build apps/todo-api/overlays/dev
kustomize build apps/todo-api/overlays/staging
kustomize build apps/todo-api/rollouts/staging
kustomize build infrastructure/kubernetes
```

Chaque commande doit se terminer sans erreur.

### 1.2 Vérifications attendues

**Overlay dev :**
- `namespace: todo-api-dev`
- `replicas: 1`
- `resources.requests` et `resources.limits` présents
- `livenessProbe` et `readinessProbe` présents

**Overlay staging :**
- `namespace: todo-api-staging`
- `replicas: 3`

**Rollout staging (blue/green) :**
- `kind: Rollout`
- `strategy.blueGreen` présent
- `activeService: todo-api-active`
- `previewService: todo-api-preview`
- `autoPromotionEnabled: false`

### 1.3 Audit secrets en clair

```bash
grep -r "password:\|token:\|secret:" apps/ argocd/ infrastructure/ \
  --include="*.yaml" \
  | grep -v "secretKeyRef\|sealed\|encryptedData"
```

**Attendu : aucune ligne.**

### 1.4 Vérifier les Applications ArgoCD

```bash
grep -E "repoURL:|targetRevision:|path:" argocd/applications/*.yaml
```

**Attendu :**
- `repoURL: https://github.com/cronosjl/gitops-argocd-platform.git`
- `targetRevision: main`

---

## 2. Bootstrap ArgoCD

### 2.1 Cluster k3d

```bash
k3d cluster create argocd-platform \
  --port "8080:80@loadbalancer" \
  --port "8443:443@loadbalancer"

kubectl get nodes
# Attendu : 1 node Ready
```

### 2.2 Installation ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=available --timeout=120s deployment/argocd-server -n argocd
kubectl get pods -n argocd
# Attendu : tous les pods Running
```

### 2.3 Login ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

argocd login localhost:8080 --insecure --username admin --password "$ARGOCD_PWD"
echo "UI → https://localhost:8080 (admin / $ARGOCD_PWD)"
```

### 2.4 Installation Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl wait --for=condition=available --timeout=120s \
  deployment/argo-rollouts -n argo-rollouts
kubectl argo rollouts version
```

### 2.5 Déploiement des Applications ArgoCD

```bash
kubectl apply -f argocd/applications/ -n argocd

argocd app list
# Attendu : 4 apps (todo-api-dev, todo-api-staging, todo-api-rollout, infrastructure)
```

---

## 3. Vérification du déploiement

### 3.1 Attendre la synchronisation

```bash
argocd app wait todo-api-dev --sync --health --timeout 180
argocd app wait todo-api-staging --sync --health --timeout 180
argocd app wait infrastructure --sync --health --timeout 180
```

### 3.2 Ressources déployées — dev

```bash
kubectl -n todo-api-dev get deploy,po,svc -l app=todo-api
```

**Attendu :**
```
NAME                   READY   STATUS    RESTARTS
pod/todo-api-<hash>    1/1     Running   0

NAME           TYPE        PORT(S)
service/todo-api  ClusterIP  80/TCP
```

### 3.3 Ressources déployées — staging

```bash
kubectl -n todo-api-staging get deploy,po,svc -l app=todo-api
kubectl argo rollouts list rollouts -n todo-api-staging
```

**Attendu :**
- Services `todo-api-active` et `todo-api-preview` présents
- Rollout `todo-api` en état `Healthy`

### 3.4 Probes et resource limits

```bash
kubectl -n todo-api-dev describe pod -l app=todo-api \
  | grep -A5 "Liveness\|Readiness\|Limits\|Requests"
```

**Attendu :**
```
Limits:    cpu: 500m  memory: 256Mi
Requests:  cpu: 100m  memory: 128Mi
Liveness:  http-get ...
Readiness: http-get ...
```

### 3.5 Infrastructure

```bash
kubectl get ns todo-api-dev todo-api-staging
kubectl get networkpolicies -A
kubectl get roles,rolebindings -n todo-api-dev
```

---

## 4. Tests fonctionnels de l'API

```bash
kubectl port-forward svc/todo-api -n todo-api-dev 3000:80 &

# GET /todos (liste vide au départ)
curl -s http://localhost:3000/todos
# Attendu : []

# POST /todos
curl -s -X POST http://localhost:3000/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Test GitOps", "completed": false}'
# Attendu : {"id": 1, "title": "Test GitOps", "completed": false}

# GET /todos après création
curl -s http://localhost:3000/todos

# DELETE /todos/1
curl -s -X DELETE http://localhost:3000/todos/1

kill %1
```

---

## 5. Test du self-heal et du prune ArgoCD

### 5.1 Self-heal (correction des drifts)

```bash
# Modifier manuellement le nombre de replicas
kubectl -n todo-api-dev scale deployment todo-api --replicas=5

# Attendre ~1 minute (selfHeal ArgoCD)
watch kubectl -n todo-api-dev get deploy todo-api
# Attendu : revient à 1 replica automatiquement
```

### 5.2 Auto-prune (suppression des ressources orphelines)

```bash
# Créer une ressource hors GitOps
kubectl -n todo-api-dev create configmap orphan-test --from-literal=key=value

# Attendre ~3 minutes (ArgoCD prune)
sleep 180
kubectl -n todo-api-dev get configmap orphan-test
# Attendu : "Error from server (NotFound)"
```

---

## 6. Déploiement Blue/Green (Argo Rollouts)

### 6.1 État initial

```bash
kubectl argo rollouts get rollout todo-api -n todo-api-staging
# Attendu : status = Healthy, strategy = BlueGreen
```

### 6.2 Déclencher un blue/green

```bash
kubectl argo rollouts set image todo-api \
  todo-api=docker.io/shatri/todo-api-node:v2 \
  -n todo-api-staging

kubectl argo rollouts get rollout todo-api -n todo-api-staging --watch
```

**Attendu :**
- Pods preview créés avec la nouvelle image
- `todo-api-active` toujours sur l'ancienne version
- `todo-api-preview` sur la nouvelle version

### 6.3 Valider la version preview

```bash
kubectl port-forward svc/todo-api-preview -n todo-api-staging 3002:80 &
curl -s http://localhost:3002/todos
kill %1
```

### 6.4 Promouvoir

```bash
kubectl argo rollouts promote todo-api -n todo-api-staging
kubectl argo rollouts get rollout todo-api -n todo-api-staging --watch
# Attendu : bascule active → nouvelle version, Healthy
```

### 6.5 Rollback

```bash
kubectl argo rollouts set image todo-api \
  todo-api=docker.io/shatri/todo-api-node:v-bad \
  -n todo-api-staging

kubectl argo rollouts abort todo-api -n todo-api-staging
kubectl argo rollouts undo todo-api -n todo-api-staging

kubectl argo rollouts get rollout todo-api -n todo-api-staging
# Attendu : Healthy sur l'ancienne version
```

### 6.6 Dashboard

```bash
kubectl argo rollouts dashboard -n todo-api-staging
# → http://localhost:3100
```

---

## 7. Tests sécurité

### 7.1 Absence de secrets en clair

```bash
grep -r "password:\|token:\|apiKey:\|secret:" \
  apps/ argocd/ infrastructure/ .github/ \
  --include="*.yaml" \
  | grep -v "secretKeyRef\|sealed\|encryptedData"
# Attendu : aucune ligne
```

### 7.2 Sealed Secrets

```bash
kubectl get pods -n kube-system | grep sealed-secrets
kubectl get sealedsecrets -A
```

### 7.3 Network Policies

```bash
kubectl get networkpolicies -n todo-api-dev
kubectl get networkpolicies -n todo-api-staging
kubectl describe networkpolicy -n todo-api-staging
```

### 7.4 RBAC

```bash
kubectl get roles,rolebindings -n todo-api-dev
```

### 7.5 Historique Git propre

```bash
git log --oneline -15
# Vérifier : aucun "wip", "fix", "test" isolés sans contexte
```

---

## 8. Logs et diagnostic

```bash
# Logs de l'application
kubectl -n todo-api-dev logs -l app=todo-api --tail=50
kubectl -n todo-api-staging logs -l app=todo-api --tail=50

# Events
kubectl -n todo-api-staging get events --sort-by='.lastTimestamp' | head -20

# Logs ArgoCD
kubectl -n argocd logs deploy/argocd-application-controller --tail=30

# Logs Argo Rollouts
kubectl -n argo-rollouts logs deploy/argo-rollouts --tail=30
```

---

## 9. Nettoyage

```bash
argocd app delete todo-api-dev todo-api-staging todo-api-rollout infrastructure --yes
kubectl delete ns todo-api-dev todo-api-staging
k3d cluster delete argocd-platform
```

---

## Checklist de validation

| # | Test | Commande | Attendu |
|---|------|----------|---------|
| 1 | Kustomize build dev | `kustomize build overlays/dev` | Pas d'erreur |
| 2 | Kustomize build staging | `kustomize build overlays/staging` | Pas d'erreur |
| 3 | Kustomize build rollout | `kustomize build rollouts/staging` | kind: Rollout, blueGreen |
| 4 | Pas de secrets en clair | `grep password: ...` | 0 ligne |
| 5 | ArgoCD opérationnel | `argocd app list` | 4 apps Synced + Healthy |
| 6 | Pod dev running | `kubectl -n todo-api-dev get po` | 1/1 Running |
| 7 | Pod staging running | `kubectl -n todo-api-staging get po` | Running |
| 8 | API répond | `curl localhost:3000/todos` | `[]` |
| 9 | CRUD Todo | POST + GET + DELETE | Opérations OK |
| 10 | Self-heal | Scale manuel → watch | Revient à la valeur Git |
| 11 | Auto-prune | ConfigMap orphelin | Supprimé par ArgoCD |
| 12 | Blue/Green promote | `rollouts promote` | Bascule instantanée |
| 13 | Rollback | `rollouts abort + undo` | Retour version stable |
| 14 | Sealed Secrets | `kubectl get sealedsecrets` | Présents |
| 15 | Network Policies | `kubectl get netpol` | Présentes dans chaque ns |
| 16 | Historique Git | `git log --oneline` | Commits propres |
