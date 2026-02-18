---
name: BeeCommerce GitOps Workflow
description: |
  GitOps Project Management Platform for BeeCommerce.
  Covers the full workflow: GitHub → Jenkins CI → Harbor Registry → ArgoCD → Kubernetes.
  Includes project creation, deployments, secrets management, and debugging.

  Key principles:
  - Git is the single source of truth
  - No manual changes on the cluster (ArgoCD enforces this)
  - Every change via Pull Request
  - Immutable deploys with commit SHA tags

  Covers: new-project, deploy, add-secret, debug-deploy, scale operations.

author: Bartosz Gaca (BeeCommerce)
version: 1.0.0
language: pl
triggers:
  - gitops
  - argocd
  - kubernetes
  - k8s
  - helm
  - jenkins
  - harbor
  - sealed secrets
  - nowy projekt k8s
  - deploy kubernetes
  - gitops-manifests
  - beecommerce gitops
  - karlik
  - felu
  - kube.beecommerce.pl
  - cookiecutter
  - app-of-apps
globs:
  - "**/Jenkinsfile"
  - "**/helm/values.yaml"
  - "**/argocd/*.yaml"
  - "**/gitops-manifests/**"
  - "**/sealed-secret.yaml"
---

# BeeCommerce GitOps Workflow

> **Autor:** Bartosz Gaca | BeeCommerce
> **Serwer K8s:** kube.beecommerce.pl
> **Harbor:** harbor.kube.beecommerce.pl
> **GitOps repo:** github.com/gacabartosz/gitops-manifests
> **Wersja:** 1.0.0

---

## ⚠️ ZASADA NADRZĘDNA

**Nikt nie dotyka klastra ręcznie. Co jest w `main` branch = co jest na K8s. Każda zmiana przez Pull Request.**

ArgoCD z `selfHeal: true` automatycznie cofa każdą ręczną zmianę do stanu z Git.

---

## 🏗️ Architektura

```
Developer → git push → GitHub → Jenkins CI → Harbor Registry
                                    ↓
                           gitops-manifests repo
                                    ↓
                           ArgoCD (auto-deploy)
                                    ↓
                           Kubernetes (OVH)
```

### Trzy typy repozytoriów

| Typ | Repo | Zawartość |
|-----|------|-----------|
| Template | `beecommerce/project-template` | Dockerfile, Jenkinsfile, Helm values szablon |
| App repos | `beecommerce/{app-name}` | Kod aplikacji + CI |
| GitOps manifests | `beecommerce/gitops-manifests` | Helm values per projekt, Sealed Secrets |

---

## 📋 OPERACJA: Nowy projekt

### Krok 1: Utwórz repo z template

```bash
gh repo create gacabartosz/nowy-projekt \
  --template gacabartosz/project-template \
  --private --clone
cd nowy-projekt
```

### Krok 2: Dodaj do gitops-manifests

```bash
git clone git@github.com:gacabartosz/gitops-manifests.git
cd gitops-manifests
cp -r apps/_template apps/nowy-projekt
```

Edytuj `apps/nowy-projekt/values.yaml`:

```yaml
name: nowy-projekt
replicaCount: 1

image:
  repository: harbor.kube.beecommerce.pl/library/nowy-projekt
  tag: latest
  imagePullPolicy: Always

service:
  enabled: true
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: cluster-issuer-letsencrypt
  hosts:
    - host: nowy-projekt.ai.beecommerce.pl
      paths: [/]
  tls:
    - secretName: nowy-projekt-tls
      hosts: [nowy-projekt.ai.beecommerce.pl]

autoscaling:
  enabled: false
```

### Krok 3: Commit i push

```bash
git add apps/nowy-projekt/
git commit -m "feat: add nowy-projekt to gitops"
git push
```

ArgoCD automatycznie wykryje nowy projekt (App-of-Apps pattern) i wdroży go.

---

## 🚀 OPERACJA: Deploy na produkcję

**Normalny flow (automatyczny):**

1. Developer pushuje do `main` w app repo
2. Jenkins CI automatycznie buduje i pushuje obraz do Harbor
3. Jenkins aktualizuje `tag` w `gitops-manifests/apps/{app}/values.yaml`
4. ArgoCD wykrywa zmianę → rolling update

**Manualny trigger Jenkins:**

```bash
# Przez GitHub Actions webhook lub bezpośrednio
curl -X POST https://jenkins.kube.beecommerce.pl/job/{app-name}/build \
  --user "user:token"
```

**Sprawdź status deploy:**

```bash
# Status ArgoCD sync
kubectl -n argocd get app {app-name}
kubectl -n argocd describe app {app-name}

# Krótka forma (ArgoCD CLI)
argocd app get {app-name}
argocd app sync {app-name}  # force sync jeśli auto nie zadziałało
```

---

## 🔐 OPERACJA: Dodaj Secret (Sealed Secrets)

### Zasada

Nigdy nie commituj plaintext secrets! Używaj `kubeseal` do szyfrowania.

### Workflow

```bash
# 1. Utwórz zwykły Secret (dry-run, nie aplikuj do klastra!)
kubectl create secret generic {app-name}-secrets \
  --dry-run=client -o yaml \
  --from-literal=API_KEY=twoja-wartosc \
  --from-literal=DB_PASSWORD=haslo \
  -n {namespace} \
  > /tmp/secret.yaml

# 2. Zaszyfruj kluczem publicznym klastra
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  < /tmp/secret.yaml \
  > gitops-manifests/apps/{app-name}/sealed-secret.yaml

# 3. Usuń plaintext plik!
rm /tmp/secret.yaml

# 4. Commituj zaszyfrowany plik (bezpieczny do Git)
cd gitops-manifests
git add apps/{app-name}/sealed-secret.yaml
git commit -m "feat: add secrets for {app-name}"
git push
```

### Weryfikacja

```bash
# Sprawdź czy Secret został odszyfrowany przez kontroler
kubectl get secret {app-name}-secrets -n {namespace}
kubectl get sealedsecret {app-name}-secrets -n {namespace}
```

---

## 🔍 OPERACJA: Debug deploy

### Kolejność sprawdzania

```bash
# 1. Stan aplikacji w ArgoCD
argocd app get {app-name}
# Szukaj: Health Status, Sync Status

# 2. Pod status
kubectl get pods -n {namespace} -l app={app-name}
kubectl describe pod {pod-name} -n {namespace}

# 3. Logi poda
kubectl logs -n {namespace} -l app={app-name} --tail=100
kubectl logs -n {namespace} {pod-name} --previous  # jeśli crashuje

# 4. Events w namespace
kubectl get events -n {namespace} --sort-by='.lastTimestamp' | tail -20

# 5. Ingress i certyfikat
kubectl get ingress -n {namespace}
kubectl describe ingress {app-name} -n {namespace}
kubectl get certificate -n {namespace}

# 6. Harbor - czy obraz istnieje?
# Sprawdź: https://harbor.kube.beecommerce.pl
```

### Najczęstsze problemy

| Problem | Przyczyna | Rozwiązanie |
|---------|-----------|-------------|
| `ImagePullBackOff` | Zły tag lub brak dostępu do Harbor | Sprawdź credentials imagePullSecret |
| `CrashLoopBackOff` | Błąd aplikacji przy starcie | `kubectl logs --previous` |
| ArgoCD OutOfSync | Konflikt między Git a klastrem | `argocd app sync --force` |
| Cert-manager pending | DNS nie propaguje | Sprawdź `kubectl describe certificate` |
| 502 Bad Gateway | Pod nie odpowiada | Sprawdź healthcheck i port |

---

## ⚖️ OPERACJA: Skalowanie

```bash
# Przez Git (rekomendowane!)
# Edytuj: gitops-manifests/apps/{app-name}/values.yaml
replicaCount: 3  # zmień liczbę

git commit -am "scale: {app-name} to 3 replicas"
git push
# ArgoCD automatycznie zastosuje zmianę

# Sprawdź
kubectl get pods -n {namespace} -l app={app-name}
```

---

## 📁 Struktura gitops-manifests

```
gitops-manifests/
├── apps/
│   ├── _template/           # szablon dla nowych projektów
│   │   ├── values.yaml
│   │   ├── sealed-secret.yaml
│   │   └── kustomization.yaml
│   ├── karlik-crm/
│   │   ├── values.yaml
│   │   └── sealed-secret.yaml
│   ├── felu-generator/
│   │   ├── values.yaml
│   │   └── sealed-secret.yaml
│   └── .../
├── base/
│   └── helm-chart/          # Universal Helm Chart
├── argocd/
│   ├── app-of-apps.yaml     # master Application
│   └── project.yaml
└── README.md
```

---

## 🔧 Jenkinsfile (szablon)

```groovy
pipeline {
    agent any
    environment {
        REGISTRY   = 'harbor.kube.beecommerce.pl'
        PROJECT    = 'library'
        APP_NAME   = env.JOB_BASE_NAME
        TAG        = "${env.GIT_COMMIT?.take(8) ?: 'latest'}"
    }
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t ${REGISTRY}/${PROJECT}/${APP_NAME}:${TAG} .'
            }
        }
        stage('Push to Harbor') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'harbor-robot',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh 'docker login ${REGISTRY} -u ${USER} -p ${PASS}'
                    sh 'docker push ${REGISTRY}/${PROJECT}/${APP_NAME}:${TAG}'
                }
            }
        }
        stage('Update GitOps Manifest') {
            steps {
                withCredentials([sshUserPrivateKey(
                    credentialsId: 'github-deploy-key',
                    keyFileVariable: 'SSH_KEY'
                )]) {
                    sh '''
                        git clone git@github.com:gacabartosz/gitops-manifests.git /tmp/gitops
                        cd /tmp/gitops
                        sed -i "s|tag:.*|tag: ${TAG}|" apps/${APP_NAME}/values.yaml
                        git commit -am "deploy: ${APP_NAME}:${TAG}"
                        git push
                    '''
                }
            }
        }
    }
}
```

---

## 🎯 ArgoCD App-of-Apps

```yaml
# argocd/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: beecommerce-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:gacabartosz/gitops-manifests.git
    targetRevision: main
    path: apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true      # usuwa zasoby nieistniejące w repo
      selfHeal: true   # cofa ręczne zmiany do stanu z Git
```

---

## 🛡️ Gwarancje niezmienności

| Mechanizm | Działanie |
|-----------|-----------|
| `ArgoCD selfHeal` | Cofa każdą ręczną zmianę w klastrze do stanu z Git |
| `ArgoCD prune` | Usuwa zasoby które nie istnieją w repo |
| `Branch protection` | `main` wymaga PR + review |
| `Image tags = commit SHA` | Brak `latest`, każdy deploy jest unikalny |
| `Harbor immutable tags` | Nie można nadpisać istniejącego tagu |
| `Git audit trail` | Pełna historia: kto, co, kiedy |

---

## 📊 Quick Reference

```bash
# === STATUS ===
argocd app list                              # wszystkie aplikacje
argocd app get {app}                         # szczegóły aplikacji
kubectl get pods -n {ns}                     # pody
kubectl get ingress -n {ns}                  # ingress

# === SYNC ===
argocd app sync {app}                        # force sync
argocd app sync {app} --force                # override conflicts

# === LOGI ===
kubectl logs -n {ns} -l app={app} -f         # live logs
kubectl logs -n {ns} {pod} --previous        # crash logi

# === SECRETS ===
kubeseal --fetch-cert > pub-cert.pem         # pobierz klucz publiczny
kubectl get sealedsecrets -n {ns}            # lista sealed secrets

# === HARBOR ===
docker login harbor.kube.beecommerce.pl      # logowanie
docker push harbor.kube.beecommerce.pl/library/{app}:{tag}

# === CERTYFIKATY ===
kubectl get certificate -n {ns}
kubectl describe certificaterequest -n {ns}
```

---

## 🗺️ Plan wdrożenia (fazy)

| Faza | Co | Czas |
|------|----|------|
| 0 | ArgoCD + Sealed Secrets na K8s, Harbor credentials w Jenkins | 1 dzień |
| 1 | Template repo, Jenkinsfile, Helm values.yaml | 1 dzień |
| 2 | **Pilot:** migracja `felu-generator` (najprostszy) | 1 dzień |
| 3 | gitops-manifests repo, ArgoCD App-of-Apps, sealed secrets | 1 dzień |
| 4 | Migracja: karlik-crm, porownanie-ai, ralph | 2-3 dni |
| 5 | Claude Code Skills: new-project, deploy, add-secret, debug | 2 dni |

**Zacznij od fazy 2 (pilot)** - waliduje cały flow na jednym projekcie.

---

*Opracowanie: Bartosz Gaca | BeeCommerce | kontakt@bartoszgaca.pl*
*Wersja 1.0.0 | Luty 2026*
