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
version: 1.1.0
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
  - "**/application.yaml"
---

# BeeCommerce GitOps Workflow

> **Autor:** Bartosz Gaca | BeeCommerce
> **Serwer K8s:** kube.beecommerce.pl
> **Harbor:** harbor.kube.beecommerce.pl
> **GitOps repo:** github.com/gacabartosz/gitops-manifests
> **Wersja:** 1.1.0

---

## ⚠️ ZASADA NADRZĘDNA

**Nikt nie dotyka klastra ręcznie. Co jest w `main` branch = co jest na K8s. Każda zmiana przez Pull Request.**

ArgoCD z `selfHeal: true` automatycznie cofa każdą ręczną zmianę do stanu z Git.

---

## 🔧 PREREQ - co musi być zainstalowane lokalnie

```bash
# macOS
brew install kubectl helm argocd kubeseal gh docker

# Weryfikacja
kubectl version --client
helm version
argocd version --client
kubeseal --version
gh --version
docker --version
```

**Na klastrze K8s (jednorazowe):**

```bash
# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Sealed Secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace \
  --set crds.enabled=true
```

---

## 🏗️ Architektura

```
Developer (lokalnie)
    docker compose up → testy lokalne
    git push → GitHub
                 ↓ (webhook)
            Jenkins CI
         1. docker build
         2. docker push → Harbor Registry
         3. sed tag → gitops-manifests push
                              ↓ (ArgoCD watches)
                         ArgoCD sync
                              ↓
                    Kubernetes (OVH)
                    + cert-manager (SSL)
                    + Sealed Secrets
```

### Trzy typy repozytoriów

| Typ | Repo | Zawartość |
|-----|------|-----------|
| Template | `gacabartosz/project-template` | Dockerfile, Jenkinsfile, Helm values szablon |
| App repos | `gacabartosz/{app-name}` | Kod aplikacji + CI |
| GitOps manifests | `gacabartosz/gitops-manifests` | Application YAML + Helm values + Sealed Secrets per projekt |

---

## 📋 OPERACJA: Nowy projekt (krok po kroku)

### Krok 1: Utwórz repo z template

```bash
gh repo create gacabartosz/nowy-projekt \
  --template gacabartosz/project-template \
  --private --clone
cd nowy-projekt
```

### Krok 2: Praca lokalna

```bash
# Napisz kod, uruchom lokalnie
docker compose up --build

# Testy
docker compose run app npm test   # lub pytest, go test, etc.

# Gdy działa lokalnie - push
git add .
git commit -m "feat: initial implementation"
git push origin main
```

### Krok 3: Dodaj aplikację do gitops-manifests

```bash
git clone git@github.com:gacabartosz/gitops-manifests.git
cd gitops-manifests
cp -r apps/_template apps/nowy-projekt
```

**Edytuj `apps/nowy-projekt/values.yaml`:**

```yaml
name: nowy-projekt
replicaCount: 1

image:
  repository: harbor.kube.beecommerce.pl/library/nowy-projekt
  tag: placeholder          # Jenkins nadpisze przy pierwszym buildzie
  imagePullPolicy: Always

imagePullSecrets:
  - name: harbor-pull-secret  # WYMAGANE - Harbor jest prywatny!

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

**Utwórz `apps/nowy-projekt/application.yaml`:**
> ⚠️ Ten plik jest KLUCZOWY - bez niego ArgoCD nie wykryje aplikacji!

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nowy-projekt
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@github.com:gacabartosz/gitops-manifests.git
    targetRevision: main
    path: base/helm-chart
    helm:
      valueFiles:
        - ../../apps/nowy-projekt/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: nowy-projekt
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true    # tworzy namespace jeśli nie istnieje
```

### Krok 4: Skonfiguruj Harbor Pull Secret w K8s

```bash
# Jednorazowo per namespace (lub globalnie przez operator)
kubectl create namespace nowy-projekt
kubectl create secret docker-registry harbor-pull-secret \
  --docker-server=harbor.kube.beecommerce.pl \
  --docker-username=robot-account \
  --docker-password=robot-token \
  -n nowy-projekt
```

### Krok 5: Commit i push do gitops-manifests

```bash
git add apps/nowy-projekt/
git commit -m "feat: add nowy-projekt to gitops"
git push
```

ArgoCD (App-of-Apps) wykryje `application.yaml` i automatycznie wdroży projekt.

### Krok 6: Skonfiguruj GitHub webhook dla Jenkinsa

W GitHub repo `gacabartosz/nowy-projekt` → Settings → Webhooks → Add webhook:

```
Payload URL: https://jenkins.kube.beecommerce.pl/github-webhook/
Content type: application/json
Events: Just the push event
```

### Krok 7: Stwórz job w Jenkinsie

W Jenkins → New Item → Pipeline → wybierz "Pipeline script from SCM":
- SCM: Git
- Repository URL: `git@github.com:gacabartosz/nowy-projekt.git`
- Branch: `*/main`
- Script Path: `Jenkinsfile`

### Krok 8: Weryfikacja

```bash
# Sprawdź czy ArgoCD wykrył aplikację
argocd app list
argocd app get nowy-projekt

# Sprawdź pody
kubectl get pods -n nowy-projekt

# Sprawdź ingress i certyfikat
kubectl get ingress -n nowy-projekt
kubectl get certificate -n nowy-projekt
```

---

## 🚀 OPERACJA: Deploy na produkcję

**Normalny flow (automatyczny po skonfigurowaniu):**

```
git push → GitHub webhook → Jenkins → Harbor push → gitops-manifests update → ArgoCD sync
```

**Manualny trigger Jenkins (jeśli webhook nie zadziałał):**

```bash
curl -X POST https://jenkins.kube.beecommerce.pl/job/nowy-projekt/build \
  --user "admin:jenkins-api-token"
```

**Force sync ArgoCD:**

```bash
argocd app sync nowy-projekt
argocd app sync nowy-projekt --force   # override conflicts
```

**Sprawdź status:**

```bash
argocd app get nowy-projekt
# Szukaj: Health=Healthy, Sync=Synced
```

---

## 🔐 OPERACJA: Dodaj Secret (Sealed Secrets)

Nigdy nie commituj plaintext secrets! Używaj `kubeseal`.

```bash
# 1. Pobierz klucz publiczny klastra (jednorazowo)
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > /tmp/pub-cert.pem

# 2. Utwórz Secret (dry-run - NIE aplikuje do klastra)
kubectl create secret generic nowy-projekt-secrets \
  --dry-run=client -o yaml \
  --from-literal=API_KEY=twoja-wartosc \
  --from-literal=DB_PASSWORD=haslo \
  -n nowy-projekt \
  > /tmp/secret.yaml

# 3. Zaszyfruj
kubeseal --format yaml \
  --cert /tmp/pub-cert.pem \
  < /tmp/secret.yaml \
  > gitops-manifests/apps/nowy-projekt/sealed-secret.yaml

# 4. Usuń plaintext
rm /tmp/secret.yaml

# 5. Commituj (zaszyfrowany plik jest bezpieczny)
cd gitops-manifests
git add apps/nowy-projekt/sealed-secret.yaml
git commit -m "feat: add secrets for nowy-projekt"
git push
```

**Weryfikacja:**

```bash
kubectl get sealedsecret nowy-projekt-secrets -n nowy-projekt
kubectl get secret nowy-projekt-secrets -n nowy-projekt   # odszyfrowany przez kontroler
```

---

## 🔍 OPERACJA: Debug deploy

```bash
# 1. ArgoCD status
argocd app get nowy-projekt
# Szukaj: Health Status, Sync Status, ostatnie eventy

# 2. Pody
kubectl get pods -n nowy-projekt
kubectl describe pod {pod-name} -n nowy-projekt

# 3. Logi
kubectl logs -n nowy-projekt -l app=nowy-projekt --tail=100
kubectl logs -n nowy-projekt {pod-name} --previous   # jeśli crashuje

# 4. Events
kubectl get events -n nowy-projekt --sort-by='.lastTimestamp' | tail -20

# 5. Ingress i certyfikat
kubectl get ingress -n nowy-projekt
kubectl describe ingress nowy-projekt -n nowy-projekt
kubectl describe certificate nowy-projekt-tls -n nowy-projekt

# 6. Czy obraz istnieje w Harbor?
# https://harbor.kube.beecommerce.pl → library → nowy-projekt → Tags
```

### Najczęstsze problemy

| Problem | Przyczyna | Rozwiązanie |
|---------|-----------|-------------|
| `ImagePullBackOff` | Brak `harbor-pull-secret` w namespace | `kubectl create secret docker-registry harbor-pull-secret ...` |
| `ImagePullBackOff` | Zły tag - Jenkins jeszcze nie buildował | Ręcznie triggeruj Jenkins build |
| `CrashLoopBackOff` | Błąd aplikacji przy starcie | `kubectl logs --previous` |
| ArgoCD `OutOfSync` | Konflikt Git ↔ klaster | `argocd app sync --force` |
| Cert-manager `pending` | DNS nie propaguje jeszcze | Poczekaj lub sprawdź `kubectl describe certificate` |
| `502 Bad Gateway` | Pod nie odpowiada na port | Sprawdź `targetPort` w values.yaml vs port aplikacji |
| ArgoCD nie widzi aplikacji | Brak `application.yaml` w `apps/{app}/` | Utwórz plik Application CRD |

---

## ⚖️ OPERACJA: Skalowanie

```bash
# Edytuj gitops-manifests/apps/nowy-projekt/values.yaml
replicaCount: 3

# Commituj przez Git (nie ręcznie kubectl scale!)
cd gitops-manifests
git commit -am "scale: nowy-projekt to 3 replicas"
git push

# ArgoCD zastosuje zmianę automatycznie, sprawdź:
kubectl get pods -n nowy-projekt
```

---

## 📁 Struktura gitops-manifests

```
gitops-manifests/
├── apps/
│   ├── _template/
│   │   ├── application.yaml    # KLUCZOWY - Application CRD
│   │   ├── values.yaml         # Helm override values
│   │   └── sealed-secret.yaml  # opcjonalnie
│   ├── karlik-crm/
│   │   ├── application.yaml
│   │   ├── values.yaml
│   │   └── sealed-secret.yaml
│   ├── felu-generator/
│   │   ├── application.yaml
│   │   ├── values.yaml
│   │   └── sealed-secret.yaml
│   └── .../
├── base/
│   └── helm-chart/             # Universal Helm Chart (submodule lub lokalny)
├── argocd/
│   ├── app-of-apps.yaml        # master Application
│   └── project.yaml
└── README.md
```

---

## 🔧 Jenkinsfile (kompletny szablon)

```groovy
pipeline {
    agent any
    environment {
        REGISTRY = 'harbor.kube.beecommerce.pl'
        PROJECT  = 'library'
        APP_NAME = env.JOB_BASE_NAME
        TAG      = "${env.GIT_COMMIT?.take(8) ?: 'latest'}"
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
                        export GIT_SSH_COMMAND="ssh -i ${SSH_KEY} -o StrictHostKeyChecking=no"
                        rm -rf /tmp/gitops-${BUILD_NUMBER}
                        git clone git@github.com:gacabartosz/gitops-manifests.git \
                            /tmp/gitops-${BUILD_NUMBER}
                        cd /tmp/gitops-${BUILD_NUMBER}
                        git config user.email "jenkins@beecommerce.pl"
                        git config user.name "Jenkins CI"
                        sed -i "s|tag:.*|tag: ${TAG}|" apps/${APP_NAME}/values.yaml
                        git commit -am "deploy: ${APP_NAME}:${TAG}"
                        git push
                        rm -rf /tmp/gitops-${BUILD_NUMBER}
                    '''
                }
            }
        }
    }
    post {
        failure {
            echo "Build failed for ${APP_NAME}:${TAG}"
        }
    }
}
```

---

## 🎯 ArgoCD App-of-Apps

```yaml
# argocd/app-of-apps.yaml
# Aplikuje się JEDNORAZOWO: kubectl apply -f argocd/app-of-apps.yaml -n argocd
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
    path: apps                  # szuka Application YAML w podkatalogach
    directory:
      recurse: true             # rekurencyjnie we wszystkich apps/*/
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Jednorazowa konfiguracja ArgoCD:**

```bash
# Dodaj gitops-manifests repo do ArgoCD
argocd repo add git@github.com:gacabartosz/gitops-manifests.git \
  --ssh-private-key-path ~/.ssh/id_ed25519

# Zaaplikuj App-of-Apps (jednorazowo)
kubectl apply -f argocd/app-of-apps.yaml -n argocd

# Sprawdź
argocd app list
```

---

## 🛡️ Gwarancje niezmienności

| Mechanizm | Działanie |
|-----------|-----------|
| `ArgoCD selfHeal` | Cofa każdą ręczną zmianę w klastrze do stanu z Git |
| `ArgoCD prune` | Usuwa zasoby które nie istnieją w repo |
| `Branch protection` | `main` wymaga PR + review, brak direct push |
| `Image tags = commit SHA` | Brak `:latest` w produkcji, każdy deploy jest unikalny |
| `Harbor immutable tags` | Nie można nadpisać istniejącego tagu obrazu |
| `Git audit trail` | Pełna historia: kto, co, kiedy zmienił |

---

## 📊 Quick Reference

```bash
# === PREREQ CHECK ===
kubectl cluster-info                         # połączenie z K8s
argocd login kube.beecommerce.pl             # login do ArgoCD
kubeseal --fetch-cert > /tmp/pub-cert.pem    # klucz do szyfrowania

# === STATUS ===
argocd app list                              # wszystkie aplikacje
argocd app get {app}                         # szczegóły
kubectl get pods -n {ns}                     # pody
kubectl get ingress -n {ns}                  # ingress

# === SYNC ===
argocd app sync {app}                        # force sync
argocd app sync {app} --force                # override conflicts

# === LOGI ===
kubectl logs -n {ns} -l app={app} -f         # live logs
kubectl logs -n {ns} {pod} --previous        # crash logi

# === SECRETS ===
kubectl get sealedsecrets -n {ns}            # lista sealed secrets
kubectl get secrets -n {ns}                  # lista odszyfrowanych secrets

# === HARBOR ===
docker login harbor.kube.beecommerce.pl
docker push harbor.kube.beecommerce.pl/library/{app}:{tag}

# === CERTYFIKATY ===
kubectl get certificate -n {ns}
kubectl describe certificaterequest -n {ns}

# === NAMESPACE ===
kubectl create namespace {app}
kubectl create secret docker-registry harbor-pull-secret \
  --docker-server=harbor.kube.beecommerce.pl \
  --docker-username=robot \
  --docker-password=token \
  -n {app}
```

---

## 🗺️ Plan wdrożenia (fazy)

| Faza | Co | Rezultat | Czas |
|------|----|----------|------|
| 0 | ArgoCD + Sealed Secrets + cert-manager na K8s | Infrastruktura gotowa | 1 dzień |
| 1 | Harbor credentials w Jenkins, deploy key GitHub | CI/CD gotowe | 1 dzień |
| 2 | Template repo + gitops-manifests repo z `_template` | Szablon gotowy | 1 dzień |
| 3 | App-of-Apps w ArgoCD, dodaj repo | ArgoCD zarządza deployami | 1 dzień |
| 4 | **Pilot:** `felu-generator` (najprostszy projekt) | Pierwszy projekt na GitOps | 1 dzień |
| 5 | Migracja: karlik-crm, porownanie-ai, ralph | Wszystkie projekty na GitOps | 2-3 dni |

**Zacznij od fazy 4 (pilot)** - waliduje cały flow na jednym projekcie zanim migrujesz resztę.

---

*Opracowanie: Bartosz Gaca | BeeCommerce | kontakt@bartoszgaca.pl*
*Wersja 1.1.0 | Luty 2026*
