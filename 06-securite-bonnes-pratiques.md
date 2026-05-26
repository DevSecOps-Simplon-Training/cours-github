# 06 - Sécurité et Branch Protection

[← 05 - GitHub Actions](05-github-actions-cicd.md) | [🏠 Accueil](README.md) | [07 - Fonctionnalités avancées →](07-fonctionnalites-avancees.md)

---

## Objectifs de cette partie

- Configurer les GitHub Rulesets pour protéger les branches
- Activer et comprendre GitHub Advanced Security (GHAS)
- Utiliser Dependabot pour les providers Terraform
- Configurer Secret Scanning avec push protection
- Créer un README professionnel pour un projet IaC

---

## 1. GitHub Rulesets (Branch Protection moderne)

Les **Rulesets** sont le successeur des Branch Protection Rules. Plus flexibles et exportables, ils permettent de définir des règles sur plusieurs branches à la fois.

### Créer un Ruleset pour `main`

**Settings → Rules → Rulesets → New ruleset**

```
Nom          : protect-main
Enforcement  : Active
Target       : Branch name matches "main"

Règles à activer :
✅ Restrict creations          → personne ne peut créer/supprimer main
✅ Restrict deletions
✅ Require pull request        → tout passe par une PR
   ✅ Required approvals : 1 minimum
   ✅ Dismiss stale reviews    → re-approval si nouveau commit poussé
   ✅ Require review from Code Owners
✅ Require status checks       → CI doit passer avant le merge
   Ajouter : "terraform-validate / 🔍 Validate"
✅ Block force pushes          → interdit même pour les admins
✅ Require signed commits      → traçabilité renforcée (optionnel)
```

> 💡 **Différence avec Branch Protection** : Les Rulesets s'appliquent par pattern (ex : `release/*`) et peuvent être exportés/importés entre dépôts. Ils supportent aussi les **bypass lists** pour les workflows automatisés.

---

## 2. GitHub Advanced Security (GHAS)

GHAS regroupe trois fonctionnalités de sécurité automatisées. Pour les dépôts publics, GHAS est gratuit.

### Secret Scanning — Push Protection

Bloque tout push contenant un secret reconnu (credential Azure, token GitHub, clé AWS...) **avant** qu'il n'atteigne GitHub.

**Activer :**
Settings → Code security → Secret scanning → **Enable** + **Push protection → Enable**

Si un secret est détecté lors d'un `git push` :
```
remote: — Push Protection ————————————————————
remote: Secrets detected in push:
remote:   AZURE_CLIENT_SECRET in terraform.tfvars (line 3)
remote: Push blocked. Revoke the secret and remove it from your commits.
```

**Si vous commitez un secret par erreur :**
1. Révoquez-le immédiatement dans le portail Azure
2. Supprimez-le de l'historique avec `git filter-repo --path-glob '*.tfvars' --invert-paths`
3. Force-push (après avoir désactivé temporairement la protection)
4. Ne faites PAS qu'un simple commit de suppression — le secret reste dans l'historique !

### Dependabot — Providers Terraform à jour

Dependabot crée automatiquement des PR pour mettre à jour les dépendances, y compris les providers Terraform.

**Configurer `.github/dependabot.yml` :**

```yaml
version: 2
updates:
  # Providers Terraform
  - package-ecosystem: "terraform"
    directory: "/environments/prod"
    schedule:
      interval: "weekly"
    labels:
      - "terraform"
      - "dependencies"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "ci-cd"
      - "dependencies"
```

> ⚠️ **Attention** : les mises à jour de providers Terraform peuvent introduire des breaking changes. Configurez toujours une branch protection pour que Dependabot passe par la CI avant de merger.

### Code Scanning — CodeQL pour IaC

CodeQL analyse votre code pour détecter des vulnérabilités. Pour l'IaC, des outils spécialisés comme **Checkov** ou **tfsec** sont plus adaptés.

**Ajouter tfsec dans votre CI :**

```yaml
# .github/workflows/security-scan.yml
name: IaC Security Scan

on:
  pull_request:
    paths: ['**.tf', '**.bicep']

jobs:
  tfsec:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          soft_fail: false   # Bloque la PR si des failles sont trouvées

  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: environments/
          framework: terraform
          soft_fail: false
```

---

## 3. README professionnel pour un projet IaC

```markdown
# Infrastructure Azure — CloudOps Corp

![Terraform CI](https://github.com/mon-org/infra-azure/workflows/Terraform%20CI%2FCD%20Azure/badge.svg)
![Security Scan](https://github.com/mon-org/infra-azure/workflows/IaC%20Security%20Scan/badge.svg)
![Terraform](https://img.shields.io/badge/Terraform-1.7-purple)
![Azure](https://img.shields.io/badge/Azure-AKS%20%7C%20ACR%20%7C%20KeyVault-blue)

Infrastructure Azure gérée en IaC (Terraform) pour CloudOps Corp.

## 🏗️ Architecture

- **AKS** : Cluster Kubernetes pour les workloads applicatifs
- **ACR** : Registry Docker privé
- **Key Vault** : Gestion des secrets
- **Virtual Network** : Isolation réseau

## 🚀 Déploiement

### Prérequis

- Azure CLI >= 2.50 (`az --version`)
- Terraform >= 1.7 (`terraform --version`)
- Accès Contributor sur la subscription Azure

### Déployer un environnement

```bash
az login
cd environments/prod
terraform init
terraform plan
terraform apply
```

## 📁 Structure

```
├── modules/          # Modules Terraform réutilisables
│   ├── aks/
│   ├── network/
│   └── keyvault/
├── environments/     # Configuration par environnement
│   ├── dev/
│   └── prod/
└── .github/          # CI/CD et configuration GitHub
    ├── workflows/
    ├── CODEOWNERS
    └── pull_request_template.md
```

## 🔒 Sécurité

- Authentification Azure via **OIDC** (pas de credentials long-lived)
- **CODEOWNERS** : toute modification prod nécessite 2 approbations
- **Secret Scanning** : push protection activée
- **tfsec + Checkov** : scan de sécurité à chaque PR

## 📖 Documentation

- Runbook AKS : `docs/runbook-aks.md`
- Procédure de rollback : `docs/rollback.md`
```

---

[← 05 - GitHub Actions](05-github-actions-cicd.md) | [🏠 Accueil](README.md) | [07 - Fonctionnalités avancées →](07-fonctionnalites-avancees.md)
