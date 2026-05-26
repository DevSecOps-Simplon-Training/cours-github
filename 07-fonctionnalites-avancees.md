# 07 - Fonctionnalités avancées

[← 06 - Sécurité](06-securite-bonnes-pratiques.md) | [🏠 Accueil](README.md) | [08 - Exercices →](08-exercices.md)

---

## Objectifs de cette partie

- Maîtriser GitHub CLI pour automatiser les tâches du quotidien
- Configurer GitHub Codespaces comme environnement IaC
- Utiliser GitHub Container Registry (GHCR) pour les images Docker
- Comprendre GitHub Packages

---

## 1. GitHub CLI (`gh`)

**gh** est l'outil officiel en ligne de commande de GitHub. Il permet de gérer repos, PRs, Issues et workflows directement depuis le terminal — indispensable pour automatiser les tâches DevOps.

### Installation

```bash
# macOS
brew install gh

# Linux (Ubuntu/Debian)
sudo apt install gh

# Windows
winget install GitHub.cli
```

### Authentification

```bash
gh auth login
# Choisissez GitHub.com → SSH → authentification navigateur
```

### Commandes DevOps du quotidien

```bash
# ── Dépôts ─────────────────────────────────────────
gh repo create mon-org/infra-azure --private --clone
gh repo view mon-org/infra-azure

# ── Pull Requests ───────────────────────────────────
gh pr create \
  --title "feat(aks): add spot node pool" \
  --body "Closes #87" \
  --reviewer alice,bob \
  --label "infrastructure"

gh pr list                          # Lister les PRs ouvertes
gh pr view 42                       # Détail d'une PR
gh pr checkout 42                   # Checkout local d'une PR (pour tester)
gh pr merge 42 --squash --delete-branch

# ── Issues ──────────────────────────────────────────
gh issue create \
  --title "Incident: AKS node pool saturé" \
  --label "incident,priority:critical" \
  --assignee @me

gh issue list --label "incident"
gh issue close 88 --comment "Résolu par PR #90"

# ── Workflows GitHub Actions ────────────────────────
gh workflow list
gh workflow run terraform.yml       # Déclencher manuellement
gh run list --workflow=terraform.yml
gh run view 12345 --log             # Voir les logs d'un run
gh run watch                        # Suivre un run en temps réel

# ── Secrets ─────────────────────────────────────────
gh secret set AZURE_CLIENT_ID
gh secret list
```

### Exemple : script d'automatisation

```bash
#!/bin/bash
# Script : créer une PR de hotfix et notifier l'équipe

BRANCH="hotfix/$(date +%Y%m%d)-nsg-fix"
git switch -c "$BRANCH"

# ... faire le fix ...

git add .
git commit -m "fix(security): restrict NSG inbound rule to corp CIDR"
git push -u origin "$BRANCH"

# Créer la PR via gh CLI
gh pr create \
  --title "🔒 Hotfix: NSG inbound restriction" \
  --body "Fixes #$(gh issue list --label security --json number -q '.[0].number')" \
  --reviewer @mon-org/security-team \
  --label "security,priority:critical"

echo "PR créée : $(gh pr view --json url -q .url)"
```

---

## 2. GitHub Codespaces — Environnement IaC dans le cloud

**Codespaces** est un environnement de développement complet dans le navigateur (VS Code). Configuré avec un `devcontainer.json`, il lance automatiquement un environnement avec Terraform, Azure CLI, kubectl et tous vos outils préinstallés.

### Configurer `.devcontainer/devcontainer.json`

```json
{
  "name": "DevOps Azure IaC",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-22.04",

  "features": {
    "ghcr.io/devcontainers/features/terraform:1": {
      "version": "1.7.0"
    },
    "ghcr.io/devcontainers/features/azure-cli:1": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/git:1": {}
  },

  "customizations": {
    "vscode": {
      "extensions": [
        "hashicorp.terraform",
        "ms-azuretools.vscode-azureterraform",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "ms-vscode.azure-account",
        "github.vscode-pull-request-github"
      ],
      "settings": {
        "editor.formatOnSave": true,
        "[terraform]": {
          "editor.defaultFormatter": "hashicorp.terraform"
        }
      }
    }
  },

  "postCreateCommand": "terraform --version && az --version && kubectl version --client"
}
```

### Avantages pour le DevOps

- Environnement identique pour toute l'équipe (plus de "ça marche sur ma machine")
- Accessible depuis n'importe quel navigateur, y compris tablette
- Gratuit jusqu'à 60h/mois sur les comptes personnels
- Idéal pour l'onboarding de nouveaux membres d'équipe

### Démarrer un Codespace

```
GitHub → votre repo → Code → Codespaces → Create codespace on main
```

---

## 3. GitHub Container Registry (GHCR)

**GHCR** (`ghcr.io`) est le registry Docker intégré à GitHub. Alternative à ACR pour héberger vos images de base ou outils internes.

### Publier une image sur GHCR

```yaml
# .github/workflows/publish-image.yml
name: Build and Publish to GHCR

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write  # Requis pour pusher sur GHCR

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}  # Token automatique, pas de secret à créer !

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/mon-app:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}/mon-app:latest
```

### Utiliser une image GHCR dans AKS

```yaml
# kubernetes/deployment.yaml
spec:
  containers:
    - name: mon-app
      image: ghcr.io/mon-org/infra-azure/mon-app:v1.2.0
      imagePullPolicy: IfNotPresent
```

Pour les images privées, créez un `imagePullSecret` avec un Personal Access Token GitHub.

---

## 4. Aide-mémoire : Workflow DevOps quotidien

```bash
# 1. Synchroniser main
git switch main && git pull origin main

# 2. Créer une branche feature
git switch -c feature/add-keyvault-module

# 3. Développer, valider, committer
terraform validate && terraform fmt
git add . && git commit -m "feat(keyvault): add Key Vault module with RBAC"

# 4. Pousser et créer la PR
git push -u origin feature/add-keyvault-module
gh pr create --title "feat(keyvault): add Key Vault module" --body "Closes #95"

# 5. Suivre la CI en temps réel
gh run watch

# 6. Intégrer les feedbacks de review
git commit -m "fix: address review comments"
git push

# 7. Merger après approbation
gh pr merge --squash --delete-branch

# 8. Vérifier le déploiement Azure
gh run list --workflow=terraform.yml
```

---

[← 06 - Sécurité](06-securite-bonnes-pratiques.md) | [🏠 Accueil](README.md) | [08 - Exercices →](08-exercices.md)
